# 存储模块设计文档 (storage)

## 1. 模块概述

### 1.1 职责
- 高性能 Parquet 文件写入
- 数据分区管理和生命周期
- 内存映射文件访问
- 数据压缩和编码优化

### 1.2 依赖关系
- **依赖**: `core` (Tick, Error)
- **被依赖**: `data`, `backtest`

### 1.3 文件结构
```
src/storage/
├── mod.rs           # 模块导出
├── writer.rs        # Parquet 写入器
├── reader.rs        # Parquet 读取器
├── partition.rs     # 分区管理
└── mmap.rs          # 内存映射访问
```

---

## 2. Parquet 写入器

### 2.1 批量写入设计

```rust
use parquet::{
    arrow::arrow_writer::ArrowWriter,
    file::properties::{WriterProperties, WriterVersion},
};

/// Tick 写入器
///
/// 高性能批量写入 Ticks 到 Parquet 文件
pub struct TickWriter {
    /// Arrow 写入器
    writer: Option<Box<ArrowWriter<BufWriter<File>>>>,

    /// 缓冲区
    buffer: Vec<Tick>,

    /// 批量大小
    batch_size: usize,

    /// 文件路径
    path: PathBuf,

    /// 写入配置
    config: WriterConfig,

    /// 写入统计
    stats: WriterStats,
}

/// 写入器配置
#[derive(Debug, Clone)]
pub struct WriterConfig {
    /// 压缩算法
    pub compression: Compression,

    /// 行组大小（行数）
    pub row_group_size: usize,

    /// 数据页大小（字节）
    pub data_page_size: usize,

    /// 字典页面大小（字节）
    pub dictionary_page_size: usize,

    /// 启用统计信息
    pub enable_statistics: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Compression {
    /// Zstandard (最佳压缩比)
    Zstd(i32),
    /// Snappy (最快)
    Snappy,
    /// Gzip (兼容性好)
    Gzip(i32),
    /// 无压缩
    None,
}

impl Default for WriterConfig {
    fn default() -> Self {
        Self {
            compression: Compression::Zstd(3),
            row_group_size: 100_000,
            data_page_size: 1024 * 1024, // 1MB
            dictionary_page_size: 256 * 1024, // 256KB
            enable_statistics: true,
        }
    }
}

/// 写入器统计
#[derive(Debug, Clone, Default)]
pub struct WriterStats {
    /// 写入的总 Tick 数
    pub total_ticks: u64,
    /// 写入的总字节数
    pub total_bytes: u64,
    /// 刷盘次数
    pub flush_count: u64,
}

impl TickWriter {
    /// 创建新的写入器
    pub fn new(path: impl AsRef<Path>, config: WriterConfig) -> Result<Self> {
        let file = File::create(&path)
            .map_err(|e| Error::Storage(format!("Create file: {}", e)))?;

        let props = Self::build_properties(&config);

        let writer = ArrowWriter::try_new(
            BufWriter::new(file),
            Arc::new(Tick::arrow_schema()),
            Some(props),
        ).map_err(|e| Error::Storage(format!("Create ArrowWriter: {}", e)))?;

        Ok(Self {
            writer: Some(Box::new(writer)),
            buffer: Vec::with_capacity(config.row_group_size),
            batch_size: config.row_group_size,
            path: path.as_ref().to_path_buf(),
            config,
            stats: WriterStats::default(),
        })
    }

    /// 构建 WriterProperties
    fn build_properties(config: &WriterConfig) -> WriterProperties {
        let mut builder = WriterProperties::builder()
            .set_writer_version(WriterVersion::PARQUET_2_0)
            .set_row_group_size(config.row_group_size)
            .set_data_page_size(config.data_page_size)
            .set_dictionary_page_size(config.dictionary_page_size);

        match config.compression {
            Compression::Zstd(level) => {
                builder = builder.set_compression(parquet::basic::Compression::ZSTD(
                    parquet::basic::ZstdLevel::try_new(level).unwrap(),
                ));
            }
            Compression::Snappy => {
                builder = builder.set_compression(parquet::basic::Compression::SNAPPY);
            }
            Compression::Gzip(level) => {
                builder = builder.set_compression(parquet::basic::Compression::GZIP(
                    parquet::basic::GzipLevel::try_new(level).unwrap(),
                ));
            }
            Compression::None => {
                builder = builder.set_compression(parquet::basic::Compression::UNCOMPRESSED);
            }
        }

        builder.build()
    }

    /// 写入单个 Tick
    pub fn write_tick(&mut self, tick: Tick) -> Result<()> {
        self.buffer.push(tick);
        self.stats.total_ticks += 1;

        if self.buffer.len() >= self.batch_size {
            self.flush()?;
        }

        Ok(())
    }

    /// 批量写入 Ticks
    pub fn write_ticks(&mut self, ticks: &[Tick]) -> Result<usize> {
        self.buffer.extend_from_slice(ticks);
        self.stats.total_ticks += ticks.len() as u64;

        if self.buffer.len() >= self.batch_size {
            self.flush()?;
        }

        Ok(ticks.len())
    }

    /// 刷盘
    pub fn flush(&mut self) -> Result<()> {
        if self.buffer.is_empty() {
            return Ok(());
        }

        let batch = Tick::to_record_batch(&self.buffer);

        if let Some(writer) = self.writer.as_mut() {
            writer.write(&batch)
                .map_err(|e| Error::Storage(format!("Write batch: {}", e)))?;
        }

        self.stats.flush_count += 1;
        self.buffer.clear();

        Ok(())
    }

    /// 关闭写入器
    pub fn close(mut self) -> Result<()> {
        self.flush()?;
        if let Some(writer) = self.writer.take() {
            writer.close()
                .map_err(|e| Error::Storage(format!("Close writer: {}", e)))?;
        }
        Ok(())
    }

    /// 获取统计信息
    pub fn stats(&self) -> &WriterStats {
        &self.stats
    }
}

impl Drop for TickWriter {
    fn drop(&mut self) {
        let _ = self.flush();
    }
}
```

---

## 3. 内存映射读取器

### 3.1 零拷贝访问

```rust
use memmap2::Mmap;

/// 内存映射 Tick 读取器
///
/// 直接映射 Parquet 文件到内存，零拷贝访问
pub struct MmapTickReader {
    /// 内存映射
    mmap: Mmap,

    /// 文件大小
    size: usize,

    /// Parquet 文件元数据
    metadata: ParquetMetaData,
}

impl MmapTickReader {
    /// 打开文件并映射到内存
    pub fn open(path: impl AsRef<Path>) -> Result<Self> {
        let file = File::open(&path)
            .map_err(|e| Error::Storage(format!("Open file: {}", e)))?;

        let mmap = unsafe {
            Mmap::map(&file)
                .map_err(|e| Error::Storage(format!("Mmap file: {}", e)))?
        };

        let size = mmap.len();

        // 读取 Parquet 元数据（简化版）
        let metadata = Self::read_metadata(&mmap)?;

        Ok(Self { mmap, size, metadata })
    }

    /// 读取 Parquet 元数据
    fn read_metadata(mmap: &[u8]) -> Result<ParquetMetaData> {
        // 实际实现需要解析 Parquet 文件格式
        // 这里简化处理
        todo!("Implement Parquet metadata parsing")
    }

    /// 获取指定时间范围内的 Ticks（零拷贝）
    pub fn range(&self, start_ts: i64, end_ts: i64) -> Result<Vec<Tick>> {
        // 根据元数据定位行组
        // 使用 Arrow Reader 读取指定范围
        todo!("Implement range query with zero-copy")
    }

    /// 迭代所有 Ticks
    pub fn iter(&self) -> impl Iterator<Item = Tick> + '_ {
        // 实现迭代器
        todo!("Implement iterator")
    }
}

/// 迭代器实现
pub struct TickIterator<'a> {
    reader: &'a MmapTickReader,
    row_group_idx: usize,
    row_idx: usize,
}

impl<'a> Iterator for TickIterator<'a> {
    type Item = Tick;

    fn next(&mut self) -> Option<Self::Item> {
        // 实现迭代逻辑
        todo!("Implement next")
    }
}
```

---

## 4. 分区管理

### 4.1 分区策略

```rust
/// 数据分区管理器
pub struct PartitionManager {
    /// 根目录
    root: PathBuf,

    /// 分区策略
    strategy: PartitionStrategy,

    /// 当前写入分区
    current_partition: Option<PartitionInfo>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PartitionStrategy {
    /// 按小时分区
    ByHour,
    /// 按天分区
    ByDay,
    /// 按月分区
    ByMonth,
}

#[derive(Debug, Clone)]
struct PartitionInfo {
    path: PathBuf,
    start_ts: i64,
    end_ts: i64,
}

impl PartitionManager {
    pub fn new(root: PathBuf, strategy: PartitionStrategy) -> Self {
        Self {
            root,
            strategy,
            current_partition: None,
        }
    }

    /// 获取 Tick 应该写入的分区
    pub fn get_partition(&mut self, tick: &Tick) -> Result<PathBuf> {
        let partition_path = self.build_partition_path(tick.ts);

        // 确保目录存在
        std::fs::create_dir_all(&partition_path)
            .map_err(|e| Error::Storage(format!("Create dir: {}", e)))?;

        Ok(partition_path)
    }

    /// 构建分区路径
    fn build_partition_path(&self, ts: i64) -> PathBuf {
        let datetime = chrono::DateTime::<chrono::Utc>::from_timestamp_nanos(ts)
            .unwrap();

        let (year, month, day, hour) = (
            datetime.year(),
            datetime.month(),
            datetime.day(),
            datetime.hour(),
        );

        let path = self.root
            .join("tick")
            .join(format!("{:04}", year))
            .join(format!("{:02}", month));

        match self.strategy {
            PartitionStrategy::ByHour => {
                path.join(format!("{:02}", day))
                    .join(format!("{:02}", hour))
            }
            PartitionStrategy::ByDay => {
                path.join(format!("{:02}", day))
            }
            PartitionStrategy::ByMonth => {
                path
            }
        }
    }

    /// 获取指定时间范围的所有分区
    pub fn list_partitions(&self, start_ts: i64, end_ts: i64) -> Result<Vec<PathBuf>> {
        let start = chrono::DateTime::<chrono::Utc>::from_timestamp_nanos(start_ts).unwrap();
        let end = chrono::DateTime::<chrono::Utc>::from_timestamp_nanos(end_ts).unwrap();

        let mut partitions = Vec::new();
        let mut current = start;

        while current <= end {
            let path = self.build_partition_path(current.ts_nanos());
            if path.exists() {
                partitions.push(path);
            }
            // 移动到下一个分区
            current = match self.strategy {
                PartitionStrategy::ByHour => current + chrono::Duration::hours(1),
                PartitionStrategy::ByDay => current + chrono::Duration::days(1),
                PartitionStrategy::ByMonth => {
                    let year = current.year();
                    let month = current.month() as i32;
                    if month == 12 {
                        chrono::NaiveDate::from_ymd_opt(year + 1, 1, 1)
                    } else {
                        chrono::NaiveDate::from_ymd_opt(year, month + 1, 1)
                    }
                    .and_then(|d| d.and_hms_opt(0, 0, 0))
                    .map(|dt| chrono::DateTime::<chrono::Utc>::from_naive_utc_and_offset(dt, chrono::Utc))
                    .unwrap()
                }
            };
        }

        Ok(partitions)
    }
}
```

---

## 5. 数据生命周期管理

### 5.1 数据清理策略

```rust
/// 数据清理器
pub struct DataCleaner {
    /// 分区管理器
    partition_manager: PartitionManager,

    /// 清理策略
    retention_policy: RetentionPolicy,
}

#[derive(Debug, Clone)]
pub enum RetentionPolicy {
    /// 保留 N 天
    Days(u32),
    /// 保留 N 个分区
    Partitions(u32),
    /// 自定义
    Custom(Box<dyn Fn(&Path, i64) -> bool + Send + Sync>),
}

impl DataCleaner {
    pub fn new(partition_manager: PartitionManager, policy: RetentionPolicy) -> Self {
        Self {
            partition_manager,
            retention_policy: policy,
        }
    }

    /// 执行清理
    pub fn cleanup(&self, dry_run: bool) -> Result<Vec<PathBuf>> {
        let mut to_remove = Vec::new();
        let now = utils::now_nanos();

        // 遍历所有分区
        for entry in walkdir::WalkDir::new(&self.partition_manager.root)
            .min_depth(4)
            .max_depth(5)
            .into_iter()
            .filter_map(|e| e.ok())
        {
            if !entry.path().is_dir() {
                continue;
            }

            // 从路径解析时间戳
            if let Some(ts) = self.extract_timestamp(entry.path()) {
                if self.should_remove(now, ts) {
                    to_remove.push(entry.path().to_path_buf());
                }
            }
        }

        if !dry_run {
            for path in &to_remove {
                std::fs::remove_dir_all(path)
                    .map_err(|e| Error::Storage(format!("Remove {}: {}", path.display(), e)))?;
            }
        }

        Ok(to_remove)
    }

    /// 从路径提取时间戳
    fn extract_timestamp(&self, path: &Path) -> Option<i64> {
        // 根据分区策略从路径提取时间戳
        todo!("Extract timestamp from path")
    }

    /// 判断是否应该删除
    fn should_remove(&self, now: i64, partition_ts: i64) -> bool {
        match &self.retention_policy {
            RetentionPolicy::Days(days) => {
                let cutoff = now - (*days as i64) * 24 * 3600 * 1_000_000_000;
                partition_ts < cutoff
            }
            RetentionPolicy::Partitions(_) => false,
            RetentionPolicy::Custom(f) => {
                // 这里需要分区路径和时间戳
                // 简化处理
                false
            }
        }
    }
}
```

---

## 6. 模块导出

```rust
// src/storage/mod.rs

pub mod writer;
pub mod reader;
pub mod partition;
pub mmap;

pub use writer::{TickWriter, WriterConfig, Compression, WriterStats};
pub use reader::{TickReader, MmapTickReader};
pub use partition::{PartitionManager, PartitionStrategy};
```

---

## 7. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| 写入吞吐量 | > 1M ticks/s | 批量写入测试 |
| 查询延迟 | < 10ms | 单次查询测试 |
| 内存占用 | < 100MB | 内存分析 |
| 压缩比 | > 10:1 | 文件大小对比 |

---

## 8. 未来扩展

- **列式存储优化**: 按查询模式优化列顺序
- **增量快照**: 定期创建增量快照
- **数据归档**: 将旧数据迁移到冷存储
