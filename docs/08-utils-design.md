# 工具模块设计文档 (utils)

## 1. 模块概述

### 1.1 职责
- CPU 亲和性绑定
- 高性能消息队列 (SPSC)
- 时间工具函数
- 通用工具函数

### 1.2 依赖关系
- **依赖**: `core` (Error)
- **被依赖**: 所有模块

### 1.3 文件结构
```
src/utils/
├── mod.rs           # 模块导出
├── affinity.rs      # CPU 亲和性
├── spsc.rs         # SPSC 队列
├── time.rs         # 时间工具
└── misc.rs         # 杂项工具
```

---

## 2. CPU 亲和性

### 2.1 Core Pinner

```rust
use core_affinity::{CoreId, get_core_ids};
use std::thread;

/// CPU 核心 Pin 管理器
///
/// 用于将线程绑定到特定 CPU 核心，减少上下文切换和 Cache Miss
pub struct CorePinner {
    /// 已分配的核心
    allocated: Vec<usize>,

    /// 总核心数
    total_cores: usize,
}

impl CorePinner {
    /// 创建新的 Pin 管理器
    pub fn new() -> Result<Self> {
        let core_ids = get_core_ids().ok_or_else(|| {
            Error::System("Failed to get CPU core IDs".to_string())
        })?;

        Ok(Self {
            allocated: Vec::new(),
            total_cores: core_ids.len(),
        })
    }

    /// 获取 CPU 核心数
    pub fn core_count(&self) -> usize {
        self.total_cores
    }

    /// 保留核心（不分配）
    pub fn reserve(&mut self, core_id: usize) -> Result<()> {
        if core_id >= self.total_cores {
            return Err(Error::System(format!(
                "Core {} exceeds total cores {}",
                core_id, self.total_cores
            )));
        }

        if !self.allocated.contains(&core_id) {
            self.allocated.push(core_id);
        }

        Ok(())
    }

    /// 分配一个核心
    pub fn allocate(&mut self) -> Option<usize> {
        for core_id in 0..self.total_cores {
            if !self.allocated.contains(&core_id) {
                self.allocated.push(core_id);
                return Some(core_id);
            }
        }
        None
    }

    /// 分配多个核心
    pub fn allocate_many(&mut self, count: usize) -> Vec<usize> {
        let mut allocated = Vec::new();
        for _ in 0..count {
            if let Some(core_id) = self.allocate() {
                allocated.push(core_id);
            } else {
                break;
            }
        }
        allocated
    }

    /// 绑定当前线程到核心
    pub fn pin_current(core_id: usize) -> Result<()> {
        let core_ids = get_core_ids().ok_or_else(|| {
            Error::System("Failed to get CPU core IDs".to_string())
        })?;

        if core_id >= core_ids.len() {
            return Err(Error::System(format!(
                "Core {} exceeds total cores {}",
                core_id, core_ids.len()
            )));
        }

        if core_affinity::set_for_current(core_ids[core_id].clone()) {
            Ok(())
        } else {
            Err(Error::System("Failed to pin thread to core".to_string()))
        }
    }

    /// 绑定线程句柄到核心
    pub fn pin_thread(handle: &thread::Thread, core_id: usize) -> Result<()> {
        Self::pin_current(core_id)
    }

    /// 获取未分配的核心
    pub fn available(&self) -> Vec<usize> {
        (0..self.total_cores)
            .filter(|c| !self.allocated.contains(c))
            .collect()
    }
}

impl Default for CorePinner {
    fn default() -> Self {
        Self::new().unwrap()
    }
}

/// 优化的核心分配方案
///
/// 参考设计文档中的分配策略
#[derive(Debug, Clone)]
pub struct CoreAllocation {
    /// I/O 核心 (WebSocket, REST API)
    pub io_cores: Vec<usize>,

    /// 策略核心
    pub strategy_cores: Vec<usize>,

    /// 存储和监控核心
    pub storage_core: usize,
}

impl CoreAllocation {
    /// 创建分配方案
    pub fn allocate(pinner: &mut CorePinner, num_strategies: usize) -> Result<Self> {
        // 保留 Core 0-1 给 I/O
        pinner.reserve(0)?;
        pinner.reserve(1)?;

        // 分配策略核心
        let strategy_cores = pinner.allocate_many(num_strategies);

        // 最后一个核心给存储和监控
        let storage_core = pinner.allocate()
            .ok_or_else(|| Error::System("No core available for storage".to_string()))?;

        Ok(Self {
            io_cores: vec![0, 1],
            strategy_cores,
            storage_core,
        })
    }

    /// 绑定 I/O 线程
    pub fn bind_io(&self, index: usize) -> Result<()> {
        let core = self.io_cores.get(index)
            .ok_or_else(|| Error::System("IO core index out of range".to_string()))?;
        CorePinner::pin_current(*core)
    }

    /// 绑定策略线程
    pub fn bind_strategy(&self, index: usize) -> Result<()> {
        let core = self.strategy_cores.get(index)
            .ok_or_else(|| Error::System("Strategy core index out of range".to_string()))?;
        CorePinner::pin_current(*core)
    }

    /// 绑定存储线程
    pub fn bind_storage(&self) -> Result<()> {
        CorePinner::pin_current(self.storage_core)
    }
}
```

---

## 3. SPSC 队列

### 3.1 无锁队列实现

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::mem::ManuallyDrop;

/// 单生产者单消费者无锁队列
///
/// 基于环形缓冲区的实现，无需锁，适合高频场景
pub struct SpscQueue<T> {
    /// 缓冲区
    buffer: Vec<ManuallyDrop<T>>,

    /// 容量（必须是 2 的幂）
    capacity: usize,

    /// 掩码（用于快速取模）
    mask: usize,

    /// 写入位置（生产者）
    write: AtomicUsize,

    /// 读取位置（消费者）
    read: AtomicUsize,
}

impl<T> SpscQueue<T> {
    /// 创建新的队列
    ///
    /// 容量会向上取整到最近的 2 的幂
    pub fn new(min_capacity: usize) -> Self {
        let capacity = min_capacity.next_power_of_two();
        let mask = capacity - 1;

        let buffer: Vec<_> = (0..capacity)
            .map(|_| ManuallyDrop::new(unsafe { std::mem::zeroed() }))
            .collect();

        Self {
            buffer,
            capacity,
            mask,
            write: AtomicUsize::new(0),
            read: AtomicUsize::new(0),
        }
    }

    /// 尝试推送元素
    pub fn try_push(&self, value: T) -> Result<(), T> {
        let write_pos = self.write.load(Ordering::Relaxed);
        let read_pos = self.read.load(Ordering::Acquire);

        // 检查队列是否已满
        if write_pos.wrapping_sub(read_pos) >= self.capacity {
            return Err(value);
        }

        unsafe {
            let slot = &mut self.buffer[write_pos & self.mask] as *mut _ as *mut T;
            std::ptr::write(slot, value);
        }

        self.write.store(write_pos.wrapping_add(1), Ordering::Release);
        Ok(())
    }

    /// 推送元素（阻塞直到成功）
    pub fn push(&self, value: T) {
        while self.try_push(value).is_err() {
            std::hint::spin_loop();
        }
    }

    /// 尝试弹出元素
    pub fn try_pop(&self) -> Option<T> {
        let read_pos = self.read.load(Ordering::Relaxed);
        let write_pos = self.write.load(Ordering::Acquire);

        // 检查队列是否为空
        if read_pos == write_pos {
            return None;
        }

        unsafe {
            let slot = &self.buffer[read_pos & self.mask] as *const _ as *const T;
            let value = std::ptr::read(slot);
            self.read.store(read_pos.wrapping_add(1), Ordering::Release);
            Some(value)
        }
    }

    /// 弹出元素（阻塞直到有值）
    pub fn pop(&self) -> T {
        loop {
            if let Some(value) = self.try_pop() {
                return value;
            }
            std::hint::spin_loop();
        }
    }

    /// 获取队列大小
    pub fn len(&self) -> usize {
        let write = self.write.load(Ordering::Acquire);
        let read = self.read.load(Ordering::Acquire);
        write.wrapping_sub(read)
    }

    /// 检查队列是否为空
    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    /// 检查队列是否已满
    pub fn is_full(&self) -> bool {
        self.len() >= self.capacity
    }

    /// 获取容量
    pub const fn capacity(&self) -> usize {
        self.capacity
    }
}

impl<T> Drop for SpscQueue<T> {
    fn drop(&mut self) {
        let write = self.write.load(Ordering::Relaxed);
        let read = self.read.load(Ordering::Relaxed);

        // 丢弃所有剩余元素
        for i in read..write {
            unsafe {
                let slot = &self.buffer[i & self.mask] as *mut _ as *mut T;
                std::ptr::drop_in_place(slot);
            }
        }
    }
}

/// 批量读取的 SPSC 队列
///
/// 优化批量读取场景
pub struct SpscBatchQueue<T> {
    inner: SpscQueue<T>,
}

impl<T> SpscBatchQueue<T> {
    pub fn new(capacity: usize) -> Self {
        Self {
            inner: SpscQueue::new(capacity),
        }
    }

    pub fn push(&self, value: T) {
        self.inner.push(value);
    }

    /// 批量弹出（最多 n 个）
    pub fn drain(&self, n: usize) -> Vec<T> {
        let mut result = Vec::with_capacity(n);
        for _ in 0..n {
            match self.inner.try_pop() {
                Some(value) => result.push(value),
                None => break,
            }
        }
        result
    }

    /// 批量弹出（直到队列为空）
    pub fn drain_all(&self) -> Vec<T> {
        let mut result = Vec::new();
        while let Some(value) = self.inner.try_pop() {
            result.push(value);
        }
        result
    }
}
```

### 3.2 Disruptor 风格的序列器

```rust
/// LMAX Disruptor 风格的序列号
///
/// 用于高并发场景的事件序列化
pub struct Sequence {
    value: AtomicU64,
}

impl Sequence {
    pub fn new(initial: u64) -> Self {
        Self {
            value: AtomicU64::new(initial),
        }
    }

    /// 获取当前值
    pub const fn get(&self) -> u64 {
        self.value.load(Ordering::Acquire)
    }

    /// 设置值
    pub fn set(&self, value: u64) {
        self.value.store(value, Ordering::Release);
    }

    /// 比较并设置
    pub fn compare_and_set(&self, expected: u64, new: u64) -> bool {
        self.value.compare_exchange(
            expected,
            new,
            Ordering::AcqRel,
            Ordering::Acquire
        ).is_ok()
    }

    /// 增加并返回旧值
    pub fn increment_and_get(&self) -> u64 {
        self.value.fetch_add(1, Ordering::AcqRel)
    }

    /// 获取并增加
    pub fn get_and_increment(&self) -> u64 {
        self.value.fetch_add(1, Ordering::AcqRel)
    }

    /// 等待序列号达到某个值
    pub fn wait_for(&self, target: u64) -> u64 {
        loop {
            let current = self.get();
            if current >= target {
                return current;
            }
            std::hint::spin_loop();
        }
    }
}

/// 序列号组
///
/// 用于跟踪多个序列号的最小值
pub struct SequenceGroup {
    sequences: Vec<Sequence>,
}

impl SequenceGroup {
    pub fn new(size: usize) -> Self {
        Self {
            sequences: (0..size)
                .map(|_| Sequence::new(0))
                .collect(),
        }
    }

    /// 获取最小序列号
    pub fn minimum(&self) -> u64 {
        self.sequences.iter()
            .map(|s| s.get())
            .min()
            .unwrap_or(0)
    }

    /// 设置序列号
    pub fn set(&self, index: usize, value: u64) {
        if let Some(seq) = self.sequences.get(index) {
            seq.set(value);
        }
    }
}
```

---

## 4. 时间工具

### 4.1 时间函数

```rust
use std::time::{SystemTime, UNIX_EPOCH};
use chrono::{DateTime, Utc};

/// 获取当前纳秒时间戳
#[inline]
pub const fn now_nanos() -> i64 {
    use std::time::Instant;
    // 这是一个简化的实现
    // 实际应该使用更精确的时间源
    Instant::now().elapsed().as_nanos() as i64
}

/// 获取当前毫秒时间戳
#[inline]
pub fn now_millis() -> i64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_millis() as i64
}

/// 获取当前秒时间戳
#[inline]
pub fn now_secs() -> i64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs() as i64
}

/// 获取今天开始时间戳
pub fn day_start_ts() -> i64 {
    let now = Utc::now();
    let midnight = now.date_naive().and_hms_opt(0, 0, 0).unwrap();
    DateTime::<Utc>::from_naive_utc_and_offset(midnight, Utc)
        .timestamp_nanos_opt()
        .unwrap_or(0)
}

/// 获取本周开始时间戳
pub fn week_start_ts() -> i64 {
    let now = Utc::now();
    let weekday = now.weekday().num_days_from_monday();
    let start = now - chrono::Duration::days(weekday as i64);
    start.date_naive().and_hms_opt(0, 0, 0).unwrap()
        .and_utc()
        .timestamp_nanos_opt()
        .unwrap_or(0)
}

/// 纳秒时间戳格式化
pub fn format_nanos(ts: i64) -> String {
    let secs = ts / 1_000_000_000;
    let nanos = (ts % 1_000_000_000) as u32;
    let dt = DateTime::<Utc>::from_timestamp(secs, nanos).unwrap();
    dt.format("%Y-%m-%d %H:%M:%S%.9f UTC").to_string()
}

/// 时间戳转 DateTime
pub fn to_datetime(ts: i64) -> Option<DateTime<Utc>> {
    let secs = ts / 1_000_000_000;
    let nanos = (ts % 1_000_000_000) as u32;
    DateTime::<Utc>::from_timestamp(secs, nanos)
}

/// DateTime 转纳秒时间戳
pub fn from_datetime(dt: DateTime<Utc>) -> i64 {
    dt.timestamp_nanos_opt().unwrap_or(0)
}

/// 高精度计时器
pub struct PrecisionTimer {
    start: std::time::Instant,
}

impl PrecisionTimer {
    pub fn start() -> Self {
        Self {
            start: std::time::Instant::now(),
        }
    }

    pub fn elapsed_nanos(&self) -> u64 {
        self.start.elapsed().as_nanos() as u64
    }

    pub fn elapsed_micros(&self) -> u64 {
        self.start.elapsed().as_micros() as u64
    }

    pub fn elapsed_millis(&self) -> u64 {
        self.start.elapsed().as_millis()
    }
}

impl Default for PrecisionTimer {
    fn default() -> Self {
        Self::start()
    }
}
```

---

## 5. 杂项工具

### 5.1 常用工具函数

```rust
/// 字节数组转换
pub struct Bytes;

impl Bytes {
    /// 将 u64 转换为字节数组
    pub fn from_u64(value: u64) -> [u8; 8] {
        value.to_be_bytes()
    }

    /// 将字节数组转换为 u64
    pub fn to_u64(bytes: [u8; 8]) -> u64 {
        u64::from_be_bytes(bytes)
    }

    /// 将 u32 转换为字节数组
    pub fn from_u32(value: u32) -> [u8; 4] {
        value.to_be_bytes()
    }

    /// 将字节数组转换为 u32
    pub fn to_u32(bytes: [u8; 4]) -> u32 {
        u32::from_be_bytes(bytes)
    }
}

/// 延迟初始化
pub struct Lazy<T, F>(Option<T>, F);

impl<T, F> Lazy<T, F>
where
    F: FnOnce() -> T,
{
    pub const fn new(init: F) -> Self {
        Self(None, init)
    }

    pub fn get(&mut self) -> &T {
        if self.0.is_none() {
            self.0 = Some((self.1)());
        }
        self.0.as_ref().unwrap()
    }
}

/// 快速哈希
pub use std::collections::hash_map::DefaultHasher;
pub use std::hash::{Hash, Hasher};

pub fn fast_hash<T: Hash>(value: &T) -> u64 {
    let mut hasher = DefaultHasher::new();
    value.hash(&mut hasher);
    hasher.finish()
}

/// 字符串 intern
///
/// 用于减少字符串内存占用
pub struct StringInterner {
    strings: parking_lot::RwLock<HashMap<String, u32>>,
    strings_reverse: parking_lot::RwLock<HashMap<u32, String>>,
    next_id: AtomicU32,
}

impl StringInterner {
    pub fn new() -> Self {
        Self {
            strings: parking_lot::RwLock::new(HashMap::new()),
            strings_reverse: parking_lot::RwLock::new(HashMap::new()),
            next_id: AtomicU32::new(1),
        }
    }

    /// 获取字符串的 ID
    pub fn intern(&self, s: &str) -> u32 {
        let read = self.strings.read();
        if let Some(&id) = read.get(s) {
            return id;
        }
        drop(read);

        let mut write = self.strings.write();
        // 再次检查（double-checked locking）
        if let Some(&id) = write.get(s) {
            return id;
        }

        let id = self.next_id.fetch_add(1, Ordering::Relaxed);
        write.insert(s.to_string(), id);
        self.strings_reverse.write().insert(id, s.to_string());
        id
    }

    /// 根据 ID 获取字符串
    pub fn lookup(&self, id: u32) -> Option<String> {
        self.strings_reverse.read().get(&id).cloned()
    }
}

impl Default for StringInterner {
    fn default() -> Self {
        Self::new()
    }
}
```

---

## 6. 模块导出

```rust
// src/utils/mod.rs

pub mod affinity;
pub mod spsc;
pub mod time;
pub mod misc;

pub use affinity::{CorePinner, CoreAllocation};
pub use spsc::{SpscQueue, SpscBatchQueue, Sequence, SequenceGroup};
pub use time::{now_nanos, now_millis, now_secs, day_start_ts, PrecisionTimer};
pub use misc::{Bytes, Lazy, fast_hash, StringInterner};
```

---

## 7. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| SPSC 队列延迟 | < 100ns | 循环延迟测试 |
| SPSC 队列吞吐 | > 50M ops/s | 批量测试 |
| 时间戳精度 | < 1μs | 系统时钟测试 |

---

## 8. 未来扩展

- **NUMA 感知**: 跨 NUMA 节点的内存分配优化
- **RDMA 支持**: 远程直接内存访问
- **SIMD 工具**: 通用的 SIMD 包装
