# 单机顶配量化交易系统 - 设计文档

## 1. 项目概述

### 1.1 设计理念

本系统定位于 **单机单兵** 顶配方案，核心理念是：**榨干单机性能，消除一切非必要的垃圾回收与序列化开销**。

### 1.2 核心目标

| 目标维度 | 具体指标 |
|---------|---------|
| **吞吐量** | 单秒处理 10万+ Tick |
| **延迟** | P99 < 1ms (非 HFT 级别) |
| **内存效率** | 零拷贝数据链路 |
| **回测性能** | 1秒内扫描 1亿行 Ticks |
| **工程优雅** | 模块化、可测试、可维护 |

### 1.3 技术选型

| 组件 | 选型 | 理由 |
|-----|------|-----|
| 语言 | Rust | 零 GC、内存安全、高性能 |
| 运行时 | Tokio (current_thread) | 精细控制异步调度 |
| 数据存储 | Parquet + DataFusion | 列式存储、向量化查询 |
| 计算引擎 | Polars | SIMD 加速、Arrow 原生 |
| 序列化 | rkyv/zerocopy | 零拷贝序列化 |
| 线程控制 | core_affinity | CPU 亲和性绑定 |
| 消息传递 | SPSC / Disruptor | 无锁队列 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Trading System                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐              │
│  │  Market     │    │   Event     │    │  Strategy   │              │
│  │   Data Feed │───▶│    Bus      │───▶│   Engine    │              │
│  │ (Core 0-1)  │    │ (SPSC Q)    │    │ (Core 2-N)  │              │
│  └─────────────┘    └─────────────┘    └─────────────┘              │
│                             │                     │                  │
│                             ▼                     ▼                  │
│                       ┌─────────────┐    ┌─────────────┐              │
│                       │   Backtest  │    │  Order      │              │
│                       │   Engine    │    │  Manager    │              │
│                       └─────────────┘    └─────────────┘              │
│                             │                     │                  │
│                             └──────────┬──────────┘                  │
│                                        ▼                             │
│                              ┌─────────────┐                        │
│                              │  Storage &  │                        │
│                              │  Monitoring │                        │
│                              │ (Last Core) │                        │
│                              └─────────────┘                        │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 CPU 核心分配策略

```
┌─────────────────────────────────────────────────────────────────┐
│                        CPU Cores Layout                            │
├────────────┬────────────┬────────────┬────────────┬──────────────┤
│  Core 0    │  Core 1    │ Core 2-N   │   ...      │   Last Core  │
├────────────┼────────────┼────────────┼────────────┼──────────────┤
│  WSS Feed  │  REST API  │ Strategy 1  │  ...       │  Parquet     │
│  (Socket)  │  (HTTP)    │  (Actor)    │ Strategy N │  Writer      │
│            │            │             │            │  Monitoring  │
│  SPSC Tx   │  SPSC Rx   │  SPSC Rx    │  SPSC Rx   │  SPSC Rx     │
└────────────┴────────────┴────────────┴────────────┴──────────────┘
```

| 核心范围 | 职责 | 调度模式 |
|---------|------|---------|
| Core 0 | WebSocket 行情接收 | current_thread |
| Core 1 | REST API 服务 | current_thread |
| Core 2-N | 策略执行（每核一个 Actor） | dedicated threads |
| Last Core | 存储 + 监控 | dedicated thread |

---

## 3. 核心模块设计

### 3.1 数据层 (Data Layer)

#### 3.1.1 零拷贝 Tick 结构

```rust
#[repr(C, packed)]
#[derive(Debug, Clone, Copy, zerocopy::FromBytes, zerocopy::AsBytes, zerocopy::Unaligned)]
pub struct Tick {
    /// 纳秒级时间戳 (i64)
    pub ts: i64,
    /// 价格 (固定点数: price / 1e6)
    pub price: i64,
    /// 成交量
    pub volume: u64,
    /// 买一量
    pub bid_vol: u32,
    /// 卖一量
    pub ask_vol: u32,
    /// 买一价 (固定点数)
    pub bid_px: i64,
    /// 卖一价 (固定点数)
    pub ask_px: i64,
}
```

**设计要点：**
- 使用 `#[repr(C, packed)]` 确保内存布局固定
- 固定点数代替浮点数，避免精度问题
- `zerocopy` 支持零拷贝反序列化
- 总大小 48 字节，缓存行友好

#### 3.1.2 Parquet 存储方案

| 字段 | 类型 | 压缩方式 |
|-----|------|---------|
| ts | INT64 | Delta encoding |
| price | INT64 | Dictionary |
| volume | UINT64 | Dictionary |
| ... | ... | ... |

**数据目录结构：**
```
data/
├── tick/
│   ├── BTCUSDT/
│   │   ├── 2024/
│   │   │   ├── 01/
│   │   │   │   ├── 01.parquet
│   │   │   │   └── ...
│   │   │   └── ...
│   │   └── ...
│   └── ...
└── order/
    └── ...
```

---

### 3.2 事件层 (Event Layer)

#### 3.2.1 事件总线设计

```rust
pub enum MarketEvent {
    Tick(Tick),
    Bar(Bar),
    Trade(Trade),
}

pub enum SystemEvent {
    Shutdown,
    ConfigReload,
    Heartbeat,
}

pub enum StrategyEvent {
    Signal(Signal),
    OrderRequest(OrderRequest),
    RiskAlert(RiskAlert),
}

pub enum OrderEvent {
    Submitted(OrderId),
    Filled(Fill),
    Rejected(OrderId),
    Cancelled(OrderId),
}
```

#### 3.2.2 消息队列方案

| 场景 | 选型 | 理由 |
|-----|------|-----|
| Market Feed → Strategy | SPSC (crossbeam) | 单生产者单消费者，最高性能 |
| Strategy → Order Manager | MPSC (disruptor) | 多策略单消费者，LMAX 模式 |
| Internal → Storage | Bounded MPSC | 背压控制 |

---

### 3.3 策略层 (Strategy Layer)

#### 3.3.1 策略 Trait 定义

```rust
pub trait Strategy: Send + 'static {
    type Config: Send + Sync;
    type Context: Send;

    /// 策略初始化
    fn init(config: Self::Config, ctx: Self::Context) -> Self;

    /// 处理 Tick
    fn on_tick(&mut self, tick: &Tick) -> Option<Signal>;

    /// 处理 Bar
    fn on_bar(&mut self, bar: &Bar) -> Option<Signal>;

    /// 处理订单回调
    fn on_fill(&mut self, fill: &Fill);

    /// 获取策略名称
    fn name(&self) -> &str;
}
```

#### 3.3.2 策略注册机制

```rust
pub struct StrategyRegistry {
    strategies: Vec<Box<dyn Strategy>>,
    // 编译时多态，运行时少量 Box<dyn>
}

impl StrategyRegistry {
    pub fn register<S: Strategy>(&mut self, strategy: S) {
        self.strategies.push(Box::new(strategy));
    }
}
```

---

### 3.4 执行层 (Execution Layer)

#### 3.4.1 订单管理 (OMS)

```rust
pub enum OrderSide {
    Buy,
    Sell,
}

pub enum OrderType {
    Market,
    Limit(f64),
    Stop(f64),
    StopLimit(f64, f64),
}

pub struct Order {
    pub id: OrderId,
    pub symbol: Symbol,
    pub side: OrderSide,
    pub order_type: OrderType,
    pub qty: Decimal,
    pub status: OrderStatus,
    pub created_at: i64,
    pub filled_qty: Decimal,
    pub avg_fill_px: f64,
}

pub struct OrderManager {
    pending_orders: HashMap<OrderId, Order>,
    risk_engine: RiskEngine,
}
```

#### 3.4.2 风控模块

```rust
pub struct RiskEngine {
    max_position: Decimal,
    max_order_value: Decimal,
    daily_loss_limit: Decimal,
    current_pnl: Decimal,
}

impl RiskEngine {
    pub fn check_order(&self, order: &Order) -> RiskResult {
        // 1. 检查持仓限制
        // 2. 检查订单价值
        // 3. 检查日内亏损
    }
}
```

---

### 3.5 存储层 (Storage Layer)

#### 3.5.1 基于 DataFusion 的查询引擎

```rust
pub struct TickStore {
    ctx: SessionContext,
}

impl TickStore {
    pub async fn query_range(
        &self,
        symbol: &str,
        start: i64,
        end: i64,
    ) -> Result<RecordBatch> {
        let sql = format!(
            "SELECT * FROM 'data/tick/{}/{}/*.parquet' WHERE ts BETWEEN {} AND {}",
            symbol, year, start, end
        );
        self.ctx.sql(sql.as_str()).await?.collect().await
    }
}
```

#### 3.5.2 高速写入

```rust
pub struct TickWriter {
    arrow_writer: ArrowWriter<BufWriter<File>>,
    buffer: Vec<Tick>,
    flush_threshold: usize,
}
```

---

### 3.6 监控层 (Monitoring Layer)

| 指标 | 类型 | 采集方式 |
|-----|------|---------|
| Tick/s | Counter | 原子计数器 |
| Strategy Latency | Histogram | HdrHistogram |
| Memory Usage | Gauge | `/proc/self/status` |
| CPU Usage | Gauge | `libc::getrusage` |

---

## 4. 性能优化策略

### 4.1 内存优化

| 技术 | 效果 |
|-----|------|
| 零拷贝 | 减少 50%+ 内存分配 |
| 内存池 | 避免频繁分配 |
| Huge Pages | 减少 TLB miss |

### 4.2 CPU 优化

| 技术 | 效果 |
|-----|------|
| 核心绑定 | 减少 Cache Miss |
| SIMD 向量化 | Polars 自动优化 |
| NUMA 感知 | 跨内存带宽优化 |

### 4.3 I/O 优化

| 技术 | 效果 |
|-----|------|
| io_uring | 异步 I/O 无阻塞 |
| NVMe 直写 | 减少系统调用 |
| 批量写入 | 提升吞吐 10x |

---

## 5. 开发路线图

### 阶段一：数据层 (2 周)
- [x] 设计 Tick 结构
- [ ] 实现 Parquet 读写
- [ ] 实现 DataFusion 查询
- [ ] 性能测试：1 秒扫描 1 亿行

### 阶段二：回测引擎 (2 周)
- [ ] 实现 Event Loop
- [ ] 实现 Bar 聚合
- [ ] 实现策略框架
- [ ] 回测报告生成

### 阶段三：实盘行情 (1 周)
- [ ] WebSocket 接入
- [ ] 重连机制
- [ ] 数据校验

### 阶段四：执行系统 (2 周)
- [ ] OMS 实现
- [ ] 风控模块
- [ ] API 对接

### 阶段五：监控与运维 (1 周)
- [ ] Metrics 暴露
- [ ] 日志系统
- [ ] 热重载配置

---

## 6. 目录结构

```
quant/
├── Cargo.toml
├── docs/
│   ├── README.md          # 本文档
│   ├── api.md             # API 文档
│   └── performance.md     # 性能分析
├── data/                  # 数据目录（gitignore）
├── config/                # 配置文件
├── src/
│   ├── main.rs
│   ├── core/
│   │   ├── mod.rs
│   │   ├── tick.rs        # Tick 定义
│   │   ├── events.rs      # 事件定义
│   │   └── error.rs       # 错误类型
│   ├── data/
│   │   ├── mod.rs
│   │   ├── parquet.rs     # Parquet 存储
│   │   ├── query.rs       # DataFusion 查询
│   │   └── feed.rs        # 行情源
│   ├── strategy/
│   │   ├── mod.rs
│   │   ├── trait.rs       # Strategy trait
│   │   ├── engine.rs      # 策略引擎
│   │   └── registry.rs    # 策略注册
│   ├── execution/
│   │   ├── mod.rs
│   │   ├── order.rs       # 订单管理
│   │   ├── risk.rs        # 风控
│   │   └── fill.rs        # 成交回报
│   ├── storage/
│   │   ├── mod.rs
│   │   ├── writer.rs      # 写入器
│   │   └── reader.rs      # 读取器
│   ├── backtest/
│   │   ├── mod.rs
│   │   ├── engine.rs      # 回测引擎
│   │   └── report.rs      # 报告生成
│   ├── monitor/
│   │   ├── mod.rs
│   │   ├── metrics.rs     # 指标采集
│   │   └── trace.rs       # tracing 配置
│   └── utils/
│       ├── mod.rs
│       ├── affinity.rs    # CPU 绑定
│       ├── spsc.rs        # SPSC 队列
│       └── time.rs        # 时间工具
└── benches/               # 基准测试
    ├── tick_processing.rs
    └── event_bus.rs
```

---

## 7. 参考资料

- [LMAX Disruptor](https://lmax-exchange.github.io/disruptor/) - 无锁高性能消息传递
- [Apache Arrow](https://arrow.apache.org/) - 零拷贝列式内存格式
- [DataFusion](https://arrow.apache.org/datafusion/) - Rust SQL 查询引擎
- [Polars](https://pola.rs/) - 高性能 DataFrame 库
- [rkyv](https://docs.rs/rkyv/) - 零拷贝序列化
