# 核心模块设计文档 (core)

## 1. 模块概述

### 1.1 职责
- 定义系统的核心数据类型
- 定义事件类型系统
- 定义统一的错误类型
- 提供基础工具函数

### 1.2 依赖关系
- **无外部依赖**（仅使用 std 和基础 crate）
- **被以下模块依赖**: data, strategy, execution, storage, backtest, monitor, utils

### 1.3 文件结构
```
src/core/
├── mod.rs           # 模块导出
├── tick.rs          # Tick 数据结构
├── events.rs        # 事件定义
├── error.rs         # 错误类型
└── types.rs         # 基础类型定义
```

---

## 2. Tick 数据结构设计

### 2.1 设计原则
1. **零拷贝**: 使用 zerocopy 支持直接字节转换
2. **缓存友好**: 结构大小 < 64 字节，避免跨缓存行
3. **固定布局**: `#[repr(C, packed)]` 确保内存布局稳定
4. **定点数**: 价格使用定点数避免浮点精度问题

### 2.2 Tick 定义

```rust
/// 行情 Tick 数据结构
///
/// # Memory Layout
/// ```text
/// +--------+--------+--------+--------+--------+--------+--------+--------+
/// | ts (8) | price (8) | volume (8) | bid_vol (4) | ask_vol (4) | bid_px (8) |
/// +--------+--------+--------+--------+--------+--------+--------+--------+
/// | ask_px (8) | padding (8) |
/// +--------+--------+--------+
/// Total: 56 bytes
/// ```
#[repr(C, packed)]
#[derive(Debug, Clone, Copy, zerocopy::FromBytes, zerocopy::AsBytes, zerocopy::Unaligned)]
pub struct Tick {
    /// 纳秒级时间戳 (UTC)
    pub ts: i64,

    /// 价格 (定点数: 实际价格 = price / PRICE_SCALE)
    pub price: i64,

    /// 成交量
    pub volume: u64,

    /// 买一量
    pub bid_vol: u32,

    /// 卖一量
    pub ask_vol: u32,

    /// 买一价 (定点数)
    pub bid_px: i64,

    /// 卖一价 (定点数)
    pub ask_px: i64,

    /// 填充对齐 (暂未使用)
    _padding: u64,
}
```

### 2.3 常量定义

```rust
impl Tick {
    /// 价格缩放因子: 1e6 (6 位小数精度)
    pub const PRICE_SCALE: i64 = 1_000_000;

    /// 最小时间戳 (1970-01-01)
    pub const MIN_TS: i64 = 0;

    /// 最大时间戳 (2262-04-11)
    pub const MAX_TS: i64 = i64::MAX;

    /// 结构体字节大小
    pub const SIZE: usize = 56;
}
```

### 2.4 实现方法

```rust
impl Tick {
    /// 创建新的 Tick
    pub const fn new(ts: i64, price: i64, volume: u64) -> Self {
        Self {
            ts,
            price,
            volume,
            bid_vol: 0,
            ask_vol: 0,
            bid_px: 0,
            ask_px: 0,
            _padding: 0,
        }
    }

    /// 从浮点数价格创建
    pub fn with_f64_price(ts: i64, price: f64, volume: u64) -> Self {
        Self {
            ts,
            price: (price * Self::PRICE_SCALE as f64) as i64,
            volume,
            bid_vol: 0,
            ask_vol: 0,
            bid_px: 0,
            ask_px: 0,
            _padding: 0,
        }
    }

    /// 获取浮点数价格
    #[inline]
    pub const fn price_f64(&self) -> f64 {
        self.price as f64 / Self::PRICE_SCALE as f64
    }

    /// 获取浮点数买一价
    #[inline]
    pub const fn bid_px_f64(&self) -> f64 {
        self.bid_px as f64 / Self::PRICE_SCALE as f64
    }

    /// 获取浮点数卖一价
    #[inline]
    pub const fn ask_px_f64(&self) -> f64 {
        self.ask_px as f64 / Self::PRICE_SCALE as f64
    }

    /// 设置买一卖一
    pub fn with_bid_ask(mut self, bid_px: f64, bid_vol: u32, ask_px: f64, ask_vol: u32) -> Self {
        self.bid_px = (bid_px * Self::PRICE_SCALE as f64) as i64;
        self.bid_vol = bid_vol;
        self.ask_px = (ask_px * Self::PRICE_SCALE as f64) as i64;
        self.ask_vol = ask_vol;
        self
    }

    /// 计算 spread
    #[inline]
    pub const fn spread(&self) -> i64 {
        self.ask_px - self.bid_px
    }

    /// 计算买卖价差百分比
    #[inline]
    pub fn spread_pct(&self) -> f64 {
        if self.mid_px() != 0 {
            self.spread() as f64 / self.mid_px() as f64 * 100.0
        } else {
            0.0
        }
    }

    /// 计算中间价
    #[inline]
    pub const fn mid_px(&self) -> i64 {
        if self.bid_px > 0 && self.ask_px > 0 {
            (self.bid_px + self.ask_px) / 2
        } else {
            self.price
        }
    }

    /// 验证 Tick 有效性
    pub const fn is_valid(&self) -> bool {
        self.ts > 0
            && self.price > 0
            && self.price < Self::MAX_TS
            && self.volume < u64::MAX
    }
}
```

### 2.5 Arrow 转换

```rust
use arrow::array::*;
use arrow::datatypes::*;

impl Tick {
    /// 定义 Arrow Schema
    pub fn arrow_schema() -> Schema {
        Schema::new(vec![
            Field::new("ts", DataType::Int64, false),
            Field::new("price", DataType::Int64, false),
            Field::new("volume", DataType::UInt64, false),
            Field::new("bid_vol", DataType::UInt32, false),
            Field::new("ask_vol", DataType::UInt32, false),
            Field::new("bid_px", DataType::Int64, false),
            Field::new("ask_px", DataType::Int64, false),
        ])
    }

    /// 批量转换为 Arrow RecordBatch
    pub fn to_record_batch(ticks: &[Self]) -> RecordBatch {
        let len = ticks.len();
        let ts = Int64Array::from_iter(ticks.iter().map(|t| t.ts));
        let price = Int64Array::from_iter(ticks.iter().map(|t| t.price));
        let volume = UInt64Array::from_iter(ticks.iter().map(|t| t.volume));
        let bid_vol = UInt32Array::from_iter(ticks.iter().map(|t| t.bid_vol));
        let ask_vol = UInt32Array::from_iter(ticks.iter().map(|t| t.ask_vol));
        let bid_px = Int64Array::from_iter(ticks.iter().map(|t| t.bid_px));
        let ask_px = Int64Array::from_iter(ticks.iter().map(|t| t.ask_px));

        RecordBatch::try_new(
            Arc::new(Self::arrow_schema()),
            vec![
                Arc::new(ts),
                Arc::new(price),
                Arc::new(volume),
                Arc::new(bid_vol),
                Arc::new(ask_vol),
                Arc::new(bid_px),
                Arc::new(ask_px),
            ],
        ).unwrap()
    }

    /// 从 RecordBatch 转换
    pub fn from_record_batch(batch: &RecordBatch) -> Vec<Self> {
        let len = batch.num_rows();
        let mut ticks = Vec::with_capacity(len);

        let ts = batch.column(0).as_primitive::<Int64Type>();
        let price = batch.column(1).as_primitive::<Int64Type>();
        let volume = batch.column(2).as_primitive::<UInt64Type>();
        let bid_vol = batch.column(3).as_primitive::<UInt32Type>();
        let ask_vol = batch.column(4).as_primitive::<UInt32Type>();
        let bid_px = batch.column(5).as_primitive::<Int64Type>();
        let ask_px = batch.column(6).as_primitive::<Int64Type>();

        for i in 0..len {
            ticks.push(Self {
                ts: ts.value(i),
                price: price.value(i),
                volume: volume.value(i),
                bid_vol: bid_vol.value(i),
                ask_vol: ask_vol.value(i),
                bid_px: bid_px.value(i),
                ask_px: ask_px.value(i),
                _padding: 0,
            });
        }

        ticks
    }
}
```

---

## 3. 事件系统设计

### 3.1 事件类型层次

```rust
/// 所有事件的基础 Trait
pub trait Event: Send + 'static {
    /// 事件发生时间戳
    fn timestamp(&self) -> i64;

    /// 事件类型标识
    fn event_type(&self) -> &'static str;
}

/// 市场事件（行情相关）
#[derive(Debug, Clone)]
pub enum MarketEvent {
    Tick {
        symbol: Symbol,
        tick: Tick,
    },
    Bar {
        symbol: Symbol,
        bar: Bar,
    },
    Trade {
        symbol: Symbol,
        trade: Trade,
    },
}

impl Event for MarketEvent {
    fn timestamp(&self) -> i64 {
        match self {
            Self::Tick { tick, .. } => tick.ts,
            Self::Bar { bar, .. } => bar.ts,
            Self::Trade { trade, .. } => trade.ts,
        }
    }

    fn event_type(&self) -> &'static str {
        match self {
            Self::Tick { .. } => "tick",
            Self::Bar { .. } => "bar",
            Self::Trade { .. } => "trade",
        }
    }
}

/// 系统事件（控制流相关）
#[derive(Debug, Clone)]
pub enum SystemEvent {
    /// 启动
    Start,
    /// 停止
    Stop,
    /// 配置重载
    ConfigReload,
    /// 心跳
    Heartbeat { ts: i64 },
    /// 错误通知
    Error(String),
}

impl Event for SystemEvent {
    fn timestamp(&self) -> i64 {
        match self {
            Self::Start | Self::Stop | Self::ConfigReload => utils::now_nanos(),
            Self::Heartbeat { ts } => *ts,
            Self::Error(_) => utils::now_nanos(),
        }
    }

    fn event_type(&self) -> &'static str {
        match self {
            Self::Start => "start",
            Self::Stop => "stop",
            Self::ConfigReload => "config_reload",
            Self::Heartbeat { .. } => "heartbeat",
            Self::Error(_) => "error",
        }
    }
}

/// 策略事件（策略输出）
#[derive(Debug, Clone)]
pub enum StrategyEvent {
    /// 交易信号
    Signal {
        strategy_id: StrategyId,
        symbol: Symbol,
        signal: Signal,
    },
    /// 订单请求
    OrderRequest {
        strategy_id: StrategyId,
        order: OrderRequest,
    },
    /// 风险告警
    RiskAlert {
        strategy_id: StrategyId,
        alert: RiskAlert,
    },
}

impl Event for StrategyEvent {
    fn timestamp(&self) -> i64 {
        utils::now_nanos()
    }

    fn event_type(&self) -> &'static str {
        match self {
            Self::Signal { .. } => "signal",
            Self::OrderRequest { .. } => "order_request",
            Self::RiskAlert { .. } => "risk_alert",
        }
    }
}

/// 订单事件（执行相关）
#[derive(Debug, Clone)]
pub enum OrderEvent {
    /// 订单已提交
    Submitted { order_id: OrderId },
    /// 订单已成交
    Filled { fill: Fill },
    /// 订单已拒绝
    Rejected { order_id: OrderId, reason: String },
    /// 订单已取消
    Cancelled { order_id: OrderId },
}

impl Event for OrderEvent {
    fn timestamp(&self) -> i64 {
        match self {
            Self::Submitted { .. } | Self::Rejected { .. } | Self::Cancelled { .. } => {
                utils::now_nanos()
            }
            Self::Filled { fill } => fill.ts,
        }
    }

    fn event_type(&self) -> &'static str {
        match self {
            Self::Submitted { .. } => "submitted",
            Self::Filled { .. } => "filled",
            Self::Rejected { .. } => "rejected",
            Self::Cancelled { .. } => "cancelled",
        }
    }
}

/// 统一事件包装器
#[derive(Debug, Clone)]
pub enum UnifiedEvent {
    Market(MarketEvent),
    System(SystemEvent),
    Strategy(StrategyEvent),
    Order(OrderEvent),
}

impl From<MarketEvent> for UnifiedEvent {
    fn from(e: MarketEvent) -> Self { Self::Market(e) }
}

impl From<SystemEvent> for UnifiedEvent {
    fn from(e: SystemEvent) -> Self { Self::System(e) }
}

impl From<StrategyEvent> for UnifiedEvent {
    fn from(e: StrategyEvent) -> Self { Self::Strategy(e) }
}

impl From<OrderEvent> for UnifiedEvent {
    fn from(e: OrderEvent) -> Self { Self::Order(e) }
}
```

---

## 4. 错误类型设计

### 4.1 错误分类

```rust
/// 系统统一错误类型
#[derive(Debug, thiserror::Error)]
pub enum Error {
    // IO 相关
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    // 序列化/反序列化
    #[error("Serialization error: {0}")]
    Serialization(String),

    // 网络
    #[error("Network error: {0}")]
    Network(String),

    // 数据验证
    #[error("Data validation error: {0}")]
    Validation(String),

    // 交易所 API
    #[error("Exchange error: {0}")]
    Exchange(String),

    // 存储
    #[error("Storage error: {0}")]
    Storage(String),

    // 策略
    #[error("Strategy error: {0}")]
    Strategy(String),

    // 风控
    #[error("Risk check failed: {0}")]
    Risk(String),

    // 配置
    #[error("Config error: {0}")]
    Config(String),

    // 超时
    #[error("Timeout: {0}")]
    Timeout(String),

    // 未知错误
    #[error("Unknown error: {0}")]
    Unknown(String),
}

/// 结果类型别名
pub type Result<T> = std::result::Result<T, Error>;
```

### 4.2 错误上下文

```rust
impl Error {
    /// 添加上下文信息
    pub fn context(mut self, ctx: &str) -> Self {
        match &mut self {
            Self::Network(s) | Self::Exchange(s) | Self::Storage(s) |
            Self::Strategy(s) | Self::Risk(s) | Self::Config(s) => {
                *s = format!("{}: {}", ctx, s);
            }
            _ => {}
        }
        self
    }
}

/// 错误的 From 实现简化
impl From<serde_json::Error> for Error {
    fn from(e: serde_json::Error) -> Self {
        Self::Serialization(e.to_string())
    }
}

impl From<reqwest::Error> for Error {
    fn from(e: reqwest::Error) -> Self {
        if e.is_timeout() {
            Self::Timeout(e.to_string())
        } else if e.is_connect() {
            Self::Network(format!("Connect error: {}", e))
        } else {
            Self::Network(e.to_string())
        }
    }
}
```

---

## 5. 基础类型定义

### 5.1 标识符类型

```rust
/// 交易对标识符（使用 interned string）
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct Symbol(pub u32);

impl Symbol {
    pub const MAX: Symbol = Symbol(u32::MAX);
}

/// 策略标识符
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct StrategyId(pub u32);

impl StrategyId {
    pub const SYSTEM: StrategyId = StrategyId(0);
    pub const USER_START: StrategyId = StrategyId(1);
}

/// 订单标识符
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct OrderId(pub u64);

impl OrderId {
    pub fn new() -> Self {
        use std::sync::atomic::{AtomicU64, Ordering};
        static COUNTER: AtomicU64 = AtomicU64::new(1);
        Self(COUNTER.fetch_add(1, Ordering::Relaxed))
    }
}

/// 信号标识符
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct SignalId(pub u64);
```

### 5.2 价格和数量类型

```rust
/// 价格类型（定点数）
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Price(i64);

impl Price {
    pub const ZERO: Self = Self(0);
    pub const SCALE: i64 = 1_000_000;

    pub const fn new(value: i64) -> Self {
        Self(value)
    }

    pub fn from_f64(value: f64) -> Self {
        Self((value * Self::SCALE as f64) as i64)
    }

    pub const fn as_i64(&self) -> i64 {
        self.0
    }

    pub fn as_f64(&self) -> f64 {
        self.0 as f64 / Self::SCALE as f64
    }
}

/// 数量类型（使用 u64，精度 1e-8）
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Quantity(u64);

impl Quantity {
    pub const ZERO: Self = Self(0);
    pub const SCALE: u64 = 100_000_000;

    pub const fn new(value: u64) -> Self {
        Self(value)
    }

    pub fn from_f64(value: f64) -> Self {
        Self((value * Self::SCALE as f64) as u64)
    }

    pub const fn as_u64(&self) -> u64 {
        self.0
    }

    pub fn as_f64(&self) -> f64 {
        self.0 as f64 / Self::SCALE as f64
    }
}
```

---

## 6. 模块导出

```rust
// src/core/mod.rs

pub mod tick;
pub mod events;
pub mod error;
pub mod types;

// 重新导出常用类型
pub use tick::{Tick, Bar, Trade};
pub use events::{
    Event, MarketEvent, SystemEvent, StrategyEvent, OrderEvent, UnifiedEvent,
    Signal, OrderRequest, RiskAlert, Fill,
};
pub use error::{Error, Result};
pub use types::{Symbol, StrategyId, OrderId, Price, Quantity};
```

---

## 7. 性能优化说明

### 7.1 Tick 结构优化
- **大小**: 56 字节，接近 64 字节缓存行，浪费最小
- **对齐**: `packed` 属性确保结构紧凑，但可能影响访问速度
- **权衡**: 在极高频场景可考虑使用 `#[repr(C)]` 并手动对齐

### 7.2 事件传递优化
- **事件复制**: 事件结构使用 `Clone` trait，SPSC 场景下无堆分配
- **枚举大小**: `UnifiedEvent` 使用 `Box<dyn Event>` 可能增加开销
- **建议**: 高频路径使用具体事件类型，不经过 `UnifiedEvent`

### 7.3 错误处理优化
- **避免堆分配**: 错误消息使用 `String` 可能导致分配
- **改进方向**: 考虑使用 `&'static str` + 错误码减少分配

---

## 8. 测试策略

### 8.1 单元测试覆盖
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_tick_price_conversion() {
        let tick = Tick::with_f64_price(0, 12345.6789, 100);
        assert!((tick.price_f64() - 12345.6789).abs() < 1e-6);
    }

    #[test]
    fn test_tick_validation() {
        let invalid = Tick::new(-1, 0, 0);
        assert!(!invalid.is_valid());
    }
}
```

### 8.2 性能测试
- Tick 解析速度: 目标 > 10M ops/s
- 事件克隆速度: 目标 > 5M ops/s
- Arrow 转换速度: 目标 > 1M ticks/s

---

## 9. 未来扩展

- **Tick V2**: 支持更多订单簿深度
- **事件压缩**: 对高频事件进行压缩传输
- **类型擦除**: 使用 `dyn Event` 支持运行时事件注册
