# 数据模块设计文档 (data)

## 1. 模块概述

### 1.1 职责
- 从外部数据源获取实时行情数据
- 提供历史行情数据的查询接口
- 管理数据源的生命周期（连接、重连、断开）
- 数据格式转换和验证

### 1.2 依赖关系
- **依赖**: `core` (Tick, Event, Error)
- **被依赖**: `strategy`, `backtest`, `execution`, `storage`

### 1.3 文件结构
```
src/data/
├── mod.rs           # 模块导出
├── feed.rs          # 行情源抽象定义
├── websocket.rs     # WebSocket 行情源实现
├── rest.rs          # REST API 数据源实现
├── query.rs         # DataFusion 查询引擎
└── cache.rs         # 行情数据缓存
```

---

## 2. 行情源抽象设计

### 2.1 Feed Trait 定义

```rust
/// 行情数据源 Trait
///
/// # Thread Safety
/// 实现必须是 `Send` 的，支持跨线程传递
///
/// # Lifecycle
/// 1. connect() - 建立连接
/// 2. subscribe() - 订阅交易对
/// 3. next() - 接收行情（阻塞或异步）
/// 4. disconnect() - 断开连接
pub trait Feed: Send + Sync {
    /// 建立连接
    ///
    /// # Errors
    /// 返回连接失败的具体原因
    async fn connect(&mut self) -> Result<()>;

    /// 断开连接
    async fn disconnect(&mut self) -> Result<()>;

    /// 订阅交易对
    ///
    /// # Parameters
    /// - `symbols`: 要订阅的交易对列表
    ///
    /// # Errors
    /// 返回订阅失败的具体原因
    async fn subscribe(&mut self, symbols: &[Symbol]) -> Result<()>;

    /// 取消订阅
    async fn unsubscribe(&mut self, symbols: &[Symbol]) -> Result<()>;

    /// 接收下一个 Tick
    ///
    /// # Returns
    /// - `Some(tick)`: 收到新的 Tick
    /// - `None`: 连接已关闭
    async fn next(&mut self) -> Result<Option<MarketEvent>>;

    /// 获取连接状态
    fn is_connected(&self) -> bool;

    /// 获取支持的交易对列表
    async fn supported_symbols(&self) -> Result<Vec<Symbol>>;
}

/// 行情源配置
#[derive(Debug, Clone)]
pub struct FeedConfig {
    /// 数据源类型
    pub feed_type: FeedType,

    /// 连接地址
    pub url: String,

    /// API Key（可选）
    pub api_key: Option<String>,

    /// API Secret（可选）
    pub api_secret: Option<String>,

    /// 重连间隔（毫秒）
    pub retry_interval_ms: u64,

    /// 最大重连次数
    pub max_retries: Option<u32>,

    /// 心跳间隔（秒）
    pub heartbeat_interval_secs: Option<u64>,

    /// 是否启用压缩
    pub compression: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FeedType {
    /// WebSocket 实时行情
    WebSocket,
    /// REST API 轮询
    Rest,
    /// 文件回放
    File,
}
```

### 2.2 Feed Manager

```rust
/// 行情源管理器
///
/// 负责管理多个行情源的连接状态、重连逻辑、订阅管理
pub struct FeedManager {
    /// 注册的行情源
    feeds: Vec<Box<dyn Feed>>,

    /// 重连任务句柄
    reconnect_handles: Vec<tokio::task::JoinHandle<()>>,

    /// 订阅状态
    subscriptions: HashMap<FeedId, HashSet<Symbol>>,

    /// 事件发送器（SPSC）
    event_tx: crossbeam::channel::Sender<MarketEvent>,

    /// 配置
    config: FeedConfig,
}

impl FeedManager {
    /// 创建新的行情管理器
    pub fn new(config: FeedConfig) -> Self {
        let (tx, rx) = crossbeam::channel::unbounded();
        Self {
            feeds: Vec::new(),
            reconnect_handles: Vec::new(),
            subscriptions: HashMap::new(),
            event_tx: tx,
            config,
        }
    }

    /// 注册行情源
    pub fn register_feed(&mut self, feed: Box<dyn Feed>) -> FeedId {
        let id = FeedId(self.feeds.len() as u32);
        self.feeds.push(feed);
        id
    }

    /// 启动所有行情源
    pub async fn start(&mut self) -> Result<()> {
        for (id, feed) in self.feeds.iter_mut().enumerate() {
            if let Err(e) = feed.connect().await {
                tracing::error!("Feed {} connect failed: {}", id, e);
                continue;
            }
        }
        Ok(())
    }

    /// 获取事件接收器
    pub fn event_receiver(&self) -> crossbeam::channel::Receiver<MarketEvent> {
        // 需要在构造时克隆并存储
        // 实际实现中需要调整
        todo!()
    }
}
```

---

## 3. WebSocket 行情源实现

### 3.1 Binance WebSocket 示例

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::StreamExt;

/// Binance WebSocket 行情源
pub struct BinanceWebSocketFeed {
    /// WebSocket 写入端
    write: Option<tokio_tungstenite::WebSocketStream<...>>,

    /// WebSocket 读取端
    read: Option<tokio_tungstenite::WebSocketStream<...>>,

    /// 配置
    config: FeedConfig,

    /// 是否已连接
    connected: bool,

    /// 订阅的交易对
    subscribed_symbols: HashSet<Symbol>,
}

impl BinanceWebSocketFeed {
    /// 创建新的 Binance WebSocket Feed
    pub fn new(config: FeedConfig) -> Self {
        Self {
            write: None,
            read: None,
            config,
            connected: false,
            subscribed_symbols: HashSet::new(),
        }
    }

    /// 构建订阅消息
    fn build_subscribe_msg(&self, symbols: &[Symbol]) -> String {
        let streams: Vec<String> = symbols
            .iter()
            .map(|s| format!("{}@trade", s.to_lowercase()))
            .collect();
        json!({
            "method": "SUBSCRIBE",
            "params": streams,
            "id": 1
        }).to_string()
    }

    /// 解析行情消息
    fn parse_trade_msg(msg: &str) -> Result<Tick> {
        let value: serde_json::Value = serde_json::from_str(msg)?;
        let data = &value["data"];

        let ts = data["E"].as_i64().ok_or_else(|| {
            Error::Validation("Missing timestamp".to_string())
        })?;

        let price = data["p"].as_str().ok_or_else(|| {
            Error::Validation("Missing price".to_string())
        })?.parse::<f64>()?;

        let qty = data["q"].as_str().ok_or_else(|| {
            Error::Validation("Missing quantity".to_string())
        })?.parse::<f64>()?;

        Ok(Tick::with_f64_price(
            ts * 1_000_000, // ms to ns
            price,
            qty as u64,
        ))
    }
}

#[async_trait::async_trait]
impl Feed for BinanceWebSocketFeed {
    async fn connect(&mut self) -> Result<()> {
        let url = format!("{}?streams=btcusdt@trade", self.config.url);
        let (ws_stream, _) = connect_async(&url).await
            .map_err(|e| Error::Network(format!("WS connect: {}", e)))?;

        let (write, read) = ws_stream.split();
        self.write = Some(write);
        self.read = Some(read);
        self.connected = true;

        tracing::info!("Binance WebSocket connected");
        Ok(())
    }

    async fn disconnect(&mut self) -> Result<()> {
        self.write = None;
        self.read = None;
        self.connected = false;
        Ok(())
    }

    async fn subscribe(&mut self, symbols: &[Symbol]) -> Result<()> {
        if !self.connected {
            return Err(Error::Network("Not connected".to_string()));
        }

        let msg = self.build_subscribe_msg(symbols);
        if let Some(write) = &mut self.write {
            write.send(Message::Text(msg)).await
                .map_err(|e| Error::Network(format!("Send subscribe: {}", e)))?;
        }

        for symbol in symbols {
            self.subscribed_symbols.insert(*symbol);
        }

        Ok(())
    }

    async fn unsubscribe(&mut self, symbols: &[Symbol]) -> Result<()> {
        // 构建取消订阅消息并发送
        for symbol in symbols {
            self.subscribed_symbols.remove(symbol);
        }
        Ok(())
    }

    async fn next(&mut self) -> Result<Option<MarketEvent>> {
        if let Some(read) = &mut self.read {
            match read.next().await {
                Some(Ok(Message::Text(text))) => {
                    let tick = Self::parse_trade_msg(&text)?;
                    return Ok(Some(MarketEvent::Tick {
                        symbol: Symbol::from("BTCUSDT"), // 从消息中解析
                        tick,
                    }));
                }
                Some(Ok(Message::Ping(data))) => {
                    if let Some(write) = &mut self.write {
                        write.send(Message::Pong(data)).await?;
                    }
                    return self.next().await;
                }
                Some(Ok(Message::Close(_))) => {
                    self.connected = false;
                    return Ok(None);
                }
                Some(Err(e)) => {
                    return Err(Error::Network(format!("WS error: {}", e)));
                }
                None => return Ok(None),
                _ => {}
            }
        }
        Ok(None)
    }

    fn is_connected(&self) -> bool {
        self.connected
    }

    async fn supported_symbols(&self) -> Result<Vec<Symbol>> {
        // 通过 REST API 获取
        Ok(vec![])
    }
}
```

---

## 4. REST API 数据源实现

### 4.1 REST Feed 设计

```rust
/// REST API 行情源（轮询模式）
pub struct RestFeed {
    /// HTTP 客户端
    client: reqwest::Client,

    /// 基础 URL
    base_url: String,

    /// 轮询间隔
    interval: Duration,

    /// 当前订阅的交易对
    symbols: Vec<Symbol>,

    /// 上次获取的时间戳（用于增量获取）
    last_ts: HashMap<Symbol, i64>,
}

impl RestFeed {
    pub fn new(base_url: String, interval: Duration) -> Self {
        Self {
            client: reqwest::Client::new(),
            base_url,
            interval,
            symbols: Vec::new(),
            last_ts: HashMap::new(),
        }
    }

    async fn fetch_trades(&self, symbol: Symbol, from_ts: i64) -> Result<Vec<Tick>> {
        let url = format!(
            "{}/api/v1/trades?symbol={}&limit=1000",
            self.base_url,
            symbol
        );

        let response = self.client.get(&url).send().await?;
        let trades: Vec<serde_json::Value> = response.json().await?;

        let mut ticks = Vec::new();
        for trade in trades {
            let price = trade["price"].as_str().unwrap().parse::<f64>()?;
            let qty = trade["qty"].as_str().unwrap().parse::<f64>()?;
            let ts = trade["time"].as_i64().unwrap() * 1_000_000;

            ticks.push(Tick::with_f64_price(ts, price, qty as u64));
        }

        Ok(ticks)
    }
}

#[async_trait::async_trait]
impl Feed for RestFeed {
    async fn connect(&mut self) -> Result<()> {
        Ok(())
    }

    async fn disconnect(&mut self) -> Result<()> {
        Ok(())
    }

    async fn subscribe(&mut self, symbols: &[Symbol]) -> Result<()> {
        self.symbols = symbols.to_vec();
        Ok(())
    }

    async fn unsubscribe(&mut self, symbols: &[Symbol]) -> Result<()> {
        self.symbols.retain(|s| !symbols.contains(s));
        Ok(())
    }

    async fn next(&mut self) -> Result<Option<MarketEvent>> {
        tokio::time::sleep(self.interval).await;

        for symbol in &self.symbols {
            let last_ts = *self.last_ts.get(symbol).unwrap_or(&0);
            let ticks = self.fetch_trades(*symbol, last_ts).await?;

            for tick in ticks {
                self.last_ts.insert(*symbol, tick.ts);
                return Ok(Some(MarketEvent::Tick {
                    symbol: *symbol,
                    tick,
                }));
            }
        }

        Ok(None)
    }

    fn is_connected(&self) -> bool {
        true
    }

    async fn supported_symbols(&self) -> Result<Vec<Symbol>> {
        let url = format!("{}/api/v1/exchangeInfo", self.base_url);
        let response = self.client.get(&url).send().await?;
        let info: serde_json::Value = response.json().await?;

        let symbols = info["symbols"]
            .as_array()
            .map(|arr| {
                arr.iter()
                    .filter_map(|s| s["symbol"].as_str())
                    .filter_map(|s| Symbol::try_from(s).ok())
                    .collect()
            })
            .unwrap_or_default();

        Ok(symbols)
    }
}
```

---

## 5. DataFusion 查询引擎

### 5.1 查询引擎设计

```rust
use datafusion::prelude::*;
use datafusion::error::DataFusionError;

/// 历史数据查询引擎
///
/// 基于 DataFusion SQL 引擎，支持对 Parquet 文件进行高效查询
pub struct QueryEngine {
    /// DataFusion Session Context
    ctx: SessionContext,

    /// 数据根目录
    data_root: PathBuf,
}

impl QueryEngine {
    /// 创建新的查询引擎
    pub fn new(data_root: impl AsRef<Path>) -> Result<Self> {
        let config = SessionConfig::new()
            .with_target_partitions(num_cpus::get())
            .with_parquet_pruning(true);

        let ctx = SessionContext::new_with_config(config);

        Ok(Self {
            ctx,
            data_root: data_root.as_ref().to_path_buf(),
        })
    }

    /// 注册 Parquet 文件为表
    pub async fn register_table(
        &mut self,
        table_name: &str,
        path: &Path,
    ) -> Result<()> {
        self.ctx
            .register_parquet(table_name, path, ParquetReadOptions::default())
            .await
            .map_err(|e| Error::Storage(format!("Register table: {}", e)))?;
        Ok(())
    }

    /// 查询时间范围内的 Ticks
    ///
    /// # SQL
    /// ```sql
    /// SELECT * FROM <table>
    /// WHERE symbol = '<symbol>'
    ///   AND ts BETWEEN <start> AND <end>
    /// ORDER BY ts
    /// ```
    pub async fn query_ticks(
        &self,
        table_name: &str,
        symbol: &str,
        start_ts: i64,
        end_ts: i64,
    ) -> Result<Vec<Tick>> {
        let sql = format!(
            "SELECT ts, price, volume, bid_vol, ask_vol, bid_px, ask_px
             FROM {}
             WHERE ts >= {} AND ts < {}
             ORDER BY ts",
            table_name, start_ts, end_ts
        );

        let df = self.ctx.sql(&sql).await
            .map_err(|e| Error::Storage(format!("SQL error: {}", e)))?;

        let batches = df.collect().await
            .map_err(|e| Error::Storage(format!("Collect error: {}", e)))?;

        let mut ticks = Vec::new();
        for batch in batches {
            ticks.extend(Tick::from_record_batch(&batch));
        }

        Ok(ticks)
    }

    /// 聚合查询 - 计算 OHLCV
    pub async fn query_ohlcv(
        &self,
        table_name: &str,
        symbol: &str,
        start_ts: i64,
        end_ts: i64,
        bar_size: i64,  // 纳秒
    ) -> Result<Vec<Bar>> {
        let sql = format!(
            "SELECT
                floor(ts / {}) * {} as bar_ts,
                min(price) as open,
                max(price) as high,
                min(price) as low,
                last(price) as close,
                sum(volume) as volume
             FROM {}
             WHERE ts >= {} AND ts < {}
             GROUP BY floor(ts / {})
             ORDER BY bar_ts",
            bar_size, bar_size, table_name, start_ts, end_ts, bar_size
        );

        let df = self.ctx.sql(&sql).await?;
        let batches = df.collect().await?;

        let mut bars = Vec::new();
        for batch in batches {
            let bar_ts = batch.column(0).as_primitive::<Int64Type>();
            let open = batch.column(1).as_primitive::<Float64Type>();
            let high = batch.column(2).as_primitive::<Float64Type>();
            let low = batch.column(3).as_primitive::<Float64Type>();
            let close = batch.column(4).as_primitive::<Float64Type>();
            let volume = batch.column(5).as_primitive::<UInt64Type>();

            for i in 0..batch.num_rows() {
                bars.push(Bar {
                    ts: bar_ts.value(i),
                    open: Price::from_f64(open.value(i)),
                    high: Price::from_f64(high.value(i)),
                    low: Price::from_f64(low.value(i)),
                    close: Price::from_f64(close.value(i)),
                    volume: Quantity::new(volume.value(i)),
                });
            }
        }

        Ok(bars)
    }

    /// 统计查询 - 获取数据量统计
    pub async fn get_stats(
        &self,
        table_name: &str,
        symbol: Option<&str>,
    ) -> Result<DataStats> {
        let where_clause = if let Some(s) = symbol {
            format!(" WHERE symbol = '{}'", s)
        } else {
            String::new()
        };

        let sql = format!(
            "SELECT
                count(*) as total_count,
                min(ts) as min_ts,
                max(ts) as max_ts,
                count(distinct symbol) as symbol_count
             FROM {}{}",
            table_name, where_clause
        );

        let df = self.ctx.sql(&sql).await?;
        let batch = df.collect().await?.into_iter().next()
            .ok_or_else(|| Error::Storage("No results".to_string()))?;

        Ok(DataStats {
            total_count: batch.column(0).as_primitive::<Int64Type>().value(0) as usize,
            min_ts: batch.column(1).as_primitive::<Int64Type>().value(0),
            max_ts: batch.column(2).as_primitive::<Int64Type>().value(0),
            symbol_count: batch.column(3).as_primitive::<Int32Type>().value(0) as usize,
        })
    }
}

/// 数据统计信息
#[derive(Debug, Clone)]
pub struct DataStats {
    pub total_count: usize,
    pub min_ts: i64,
    pub max_ts: i64,
    pub symbol_count: usize,
}
```

---

## 6. 行情数据缓存

### 6.1 LRU Cache 设计

```rust
use std::collections::HashMap;
use std::num::NonZeroUsize;

/// 行情数据缓存 (LRU)
///
/// 缓存最近访问的 Tick 数据，减少重复查询开销
pub struct TickCache {
    /// 内部缓存
    cache: lru::LruCache<CacheKey, Tick>,

    /// 缓存容量
    capacity: usize,

    /// 统计信息
    stats: CacheStats,
}

#[derive(Debug, Clone, Hash, PartialEq, Eq)]
struct CacheKey {
    symbol: Symbol,
    ts: i64,
}

#[derive(Debug, Default, Clone)]
pub struct CacheStats {
    hits: u64,
    misses: u64,
}

impl TickCache {
    pub fn new(capacity: usize) -> Self {
        Self {
            cache: lru::LruCache::new(NonZeroUsize::new(capacity).unwrap()),
            capacity,
            stats: CacheStats::default(),
        }
    }

    /// 获取 Tick
    pub fn get(&mut self, symbol: Symbol, ts: i64) -> Option<Tick> {
        let key = CacheKey { symbol, ts };
        if let Some(tick) = self.cache.get(&key) {
            self.stats.hits += 1;
            Some(*tick)
        } else {
            self.stats.misses += 1;
            None
        }
    }

    /// 插入 Tick
    pub fn put(&mut self, symbol: Symbol, tick: Tick) {
        let key = CacheKey { symbol, ts: tick.ts };
        self.cache.put(key, tick);
    }

    /// 获取最近 N 个 Ticks
    pub fn get_recent(&mut self, symbol: Symbol, n: usize) -> Vec<Tick> {
        self.cache
            .iter()
            .filter(|(k, _)| k.symbol == symbol)
            .take(n)
            .map(|(_, v)| *v)
            .collect()
    }

    /// 清空缓存
    pub fn clear(&mut self) {
        self.cache.clear();
    }

    /// 获取缓存统计
    pub fn stats(&self) -> &CacheStats {
        &self.stats
    }

    /// 计算命中率
    pub fn hit_rate(&self) -> f64 {
        let total = self.stats.hits + self.stats.misses;
        if total == 0 {
            0.0
        } else {
            self.stats.hits as f64 / total as f64
        }
    }
}
```

---

## 7. 数据分区策略

### 7.1 分区管理器

```rust
/// 数据分区管理器
///
/// 负责按时间对 Parquet 文件进行分区
pub struct PartitionManager {
    /// 数据根目录
    root: PathBuf,

    /// 分区粒度
    partition_granularity: PartitionGranularity,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PartitionGranularity {
    /// 按天分区
    Day,
    /// 按小时分区
    Hour,
    /// 按月分区
    Month,
}

impl PartitionManager {
    pub fn new(root: PathBuf, granularity: PartitionGranularity) -> Self {
        Self { root, partition_granularity: granularity }
    }

    /// 获取分区路径
    pub fn get_partition_path(&self, symbol: &str, ts: i64) -> PathBuf {
        let datetime = chrono::DateTime::<chrono::Utc>::from_timestamp_nanos(ts)
            .unwrap();

        let (year, month, day, hour) = match self.partition_granularity {
            PartitionGranularity::Day => (
                datetime.year(),
                datetime.month(),
                datetime.day(),
                0,
            ),
            PartitionGranularity::Hour => (
                datetime.year(),
                datetime.month(),
                datetime.day(),
                datetime.hour(),
            ),
            PartitionGranularity::Month => (
                datetime.year(),
                datetime.month(),
                0,
                0,
            ),
        };

        self.root
            .join("tick")
            .join(symbol)
            .join(format!("{:04}", year))
            .join(format!("{:02}", month))
            .join(format!("{:02}", day))
    }

    /// 获取 Parquet 文件路径
    pub fn get_parquet_path(&self, symbol: &str, ts: i64) -> PathBuf {
        let partition = self.get_partition_path(symbol, ts);
        let filename = format!("{}.parquet", ts / 1_000_000_000); // 按秒命名
        partition.join(filename)
    }

    /// 列出所有分区
    pub fn list_partitions(&self, symbol: &str) -> Result<Vec<PathBuf>> {
        let symbol_dir = self.root.join("tick").join(symbol);

        let mut partitions = Vec::new();
        if symbol_dir.exists() {
            for entry in walkdir::WalkDir::new(&symbol_dir)
                .min_depth(3)
                .max_depth(4)
                .into_iter()
                .filter_map(|e| e.ok())
            {
                if entry.path().is_dir() {
                    partitions.push(entry.path().to_path_buf());
                }
            }
        }

        Ok(partitions)
    }
}
```

---

## 8. 模块导出

```rust
// src/data/mod.rs

pub mod feed;
pub mod websocket;
pub mod rest;
pub mod query;
pub mod cache;

pub use feed::{Feed, FeedConfig, FeedType, FeedManager, FeedId};
pub use websocket::BinanceWebSocketFeed;
pub use rest::RestFeed;
pub use query::{QueryEngine, DataStats};
pub use cache::{TickCache, CacheStats};
```

---

## 9. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| WebSocket 解析速度 | > 100K ticks/s | 批量解析测试 |
| REST 轮询延迟 | < 100ms | 端到端延迟测试 |
| 查询吞吐量 | > 1M ticks/s | DataFusion 查询测试 |
| 缓存命中率 | > 95% | 实盘监控 |
| 内存占用 | < 1GB | 内存分析 |

---

## 10. 未来扩展

- **多数据源聚合**: 同时连接多个交易所
- **数据质量检测**: 检测异常 Tick、漏 Tick
- **数据补全**: 自动补充缺失的历史数据
- **预加载**: 基于策略行为预加载数据
