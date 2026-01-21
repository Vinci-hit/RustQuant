# 策略模块设计文档 (strategy)

## 1. 模块概述

### 1.1 职责
- 定义策略抽象接口 (Trait)
- 提供策略引擎用于执行多个策略
- 管理策略生命周期（启动、停止、热重载）
- 策略信号分发和订单路由

### 1.2 依赖关系
- **依赖**: `core` (Tick, Event, Error), `data` (Feed)
- **被依赖**: `execution`, `backtest`

### 1.3 文件结构
```
src/strategy/
├── mod.rs           # 模块导出
├── trait.rs         # 策略 Trait 定义
├── engine.rs        # 策略引擎
├── registry.rs      # 策略注册表
├── context.rs       # 策略上下文
└── examples/        # 示例策略
    ├── mod.rs
    ├── ma_cross.rs  # 均线交叉策略
    └── mean_reversion.rs  # 均值回归策略
```

---

## 2. 策略 Trait 设计

### 2.1 核心接口

```rust
/// 策略 Trait
///
/// 定义了所有策略必须实现的核心接口
///
/// # Thread Safety
/// 所有策略必须是 `Send` 的，因为每个策略运行在独立线程中
///
/// # State Management
/// 策略实例应该维护自己的状态，但不应该共享可变状态
pub trait Strategy: Send + 'static {
    /// 策略配置类型
    type Config: Send + Sync + Clone;

    /// 策略上下文类型
    type Context: Send + Clone;

    /// 策略初始化
    ///
    /// # Parameters
    /// - `config`: 策略配置
    /// - `ctx`: 策略运行时上下文
    ///
    /// # Returns
    /// 初始化后的策略实例
    fn init(config: Self::Config, ctx: Self::Context) -> Self;

    /// 处理 Tick 数据
    ///
    /// # Parameters
    /// - `tick`: 行情 Tick
    ///
    /// # Returns
    /// - `Some(signal)`: 生成交易信号
    /// - `None`: 无信号
    ///
    /// # Performance
    /// 此函数被高频调用，必须保证 O(1) 或 O(log n) 复杂度
    fn on_tick(&mut self, tick: &Tick) -> Option<Signal>;

    /// 处理 Bar 数据（可选）
    ///
    /// 默认实现为空，只有需要 Bar 数据的策略才需要覆盖
    #[allow(unused_variables)]
    fn on_bar(&mut self, bar: &Bar) -> Option<Signal> {
        None
    }

    /// 处理成交回报（可选）
    ///
    /// 默认实现为空，只有需要跟踪成交的策略才需要覆盖
    #[allow(unused_variables)]
    fn on_fill(&mut self, fill: &Fill) {
        // 默认不处理
    }

    /// 处理订单拒绝（可选）
    #[allow(unused_variables)]
    fn on_order_rejected(&mut self, order_id: OrderId, reason: &str) {
        tracing::warn!("Order {} rejected: {}", order_id, reason);
    }

    /// 获取策略名称
    fn name(&self) -> &str;

    /// 获取策略状态
    fn state(&self) -> StrategyState {
        StrategyState::Running
    }

    /// 策略健康检查（可选）
    ///
    /// 返回 `false` 表示策略处于异常状态，可能需要重启
    fn is_healthy(&self) -> bool {
        true
    }

    /// 获取策略统计信息（可选）
    fn stats(&self) -> StrategyStats {
        StrategyStats::default()
    }
}

/// 策略运行时上下文
///
/// 提供给策略访问系统资源的接口
#[derive(Clone)]
pub struct StrategyContext {
    /// 策略 ID
    pub strategy_id: StrategyId,

    /// 交易对
    pub symbol: Symbol,

    /// 当前持仓
    pub position: Quantity,

    /// 未平仓订单
    pub pending_orders: Vec<OrderId>,

    /// 配置
    pub config: StrategyConfig,
}

/// 策略配置
#[derive(Debug, Clone)]
pub struct StrategyConfig {
    /// 最大持仓量
    pub max_position: Quantity,

    /// 单笔订单最大量
    pub max_order_qty: Quantity,

    /// 止损比例（0.01 = 1%）
    pub stop_loss_pct: f64,

    /// 止盈比例
    pub take_profit_pct: f64,
}

impl Default for StrategyConfig {
    fn default() -> Self {
        Self {
            max_position: Quantity::from_f64(1.0),
            max_order_qty: Quantity::from_f64(1.0),
            stop_loss_pct: 0.02,
            take_profit_pct: 0.05,
        }
    }
}

/// 策略状态
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum StrategyState {
    /// 初始化中
    Initializing,
    /// 运行中
    Running,
    /// 暂停
    Paused,
    /// 停止
    Stopped,
    /// 错误
    Error,
}
```

### 2.2 交易信号

```rust
/// 交易信号
#[derive(Debug, Clone)]
pub struct Signal {
    /// 信号 ID
    pub id: SignalId,

    /// 策略 ID
    pub strategy_id: StrategyId,

    /// 交易对
    pub symbol: Symbol,

    /// 信号类型
    pub signal_type: SignalType,

    /// 信号价格
    pub price: Option<Price>,

    /// 信号数量
    pub qty: Quantity,

    /// 信号强度 (0.0 - 1.0)
    pub strength: f64,

    /// 信号原因（用于日志）
    pub reason: String,

    /// 信号元数据
    pub metadata: SignalMetadata,
}

/// 信号类型
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SignalType {
    /// 做多
    Buy,
    /// 做空
    Sell,
    /// 平多
    CloseLong,
    /// 平空
    CloseShort,
    /// 平仓
    Close,
}

/// 信号元数据
#[derive(Debug, Clone)]
pub struct SignalMetadata {
    /// 技术指标值
    pub indicators: HashMap<String, f64>,

    /// 信号置信度
    pub confidence: f64,

    /// 预期持仓时间（秒）
    pub expected_hold_time: Option<u64>,
}

impl Signal {
    /// 创建新的信号
    pub fn new(
        strategy_id: StrategyId,
        symbol: Symbol,
        signal_type: SignalType,
        qty: Quantity,
    ) -> Self {
        Self {
            id: SignalId::new(),
            strategy_id,
            symbol,
            signal_type,
            price: None,
            qty,
            strength: 1.0,
            reason: String::new(),
            metadata: SignalMetadata {
                indicators: HashMap::new(),
                confidence: 1.0,
                expected_hold_time: None,
            },
        }
    }

    /// 设置价格
    pub fn with_price(mut self, price: Price) -> Self {
        self.price = Some(price);
        self
    }

    /// 设置强度
    pub fn with_strength(mut self, strength: f64) -> Self {
        self.strength = strength.clamp(0.0, 1.0);
        self
    }

    /// 设置原因
    pub fn with_reason(mut self, reason: impl Into<String>) -> Self {
        self.reason = reason.into();
        self
    }

    /// 添加指标值
    pub fn with_indicator(mut self, name: impl Into<String>, value: f64) -> Self {
        self.metadata.indicators.insert(name.into(), value);
        self
    }
}

impl SignalId {
    pub fn new() -> Self {
        use std::sync::atomic::{AtomicU64, Ordering};
        static COUNTER: AtomicU64 = AtomicU64::new(1);
        Self(COUNTER.fetch_add(1, Ordering::Relaxed))
    }
}
```

---

## 3. 策略引擎设计

### 3.1 策略引擎架构

```rust
/// 策略引擎
///
/// 负责管理多个策略实例，分配 CPU 核心，处理消息分发
pub struct StrategyEngine {
    /// 策略实例（每个策略在独立线程中运行）
    strategies: HashMap<StrategyId, StrategyActor>,

    /// 策略配置
    configs: HashMap<StrategyId, StrategyConfig>,

    /// 事件接收器（来自 Feed）
    event_rx: crossbeam::channel::Receiver<MarketEvent>,

    /// 信号发送器（到执行层）
    signal_tx: crossbeam::channel::Sender<StrategyEvent>,

    /// 成交回报发送器
    fill_tx: crossbeam::channel::Sender<Fill>,

    /// 运行状态
    running: AtomicBool,

    /// 核心分配
    core_pinner: Option<CorePinner>,
}

/// 策略 Actor（运行在独立线程中）
struct StrategyActor {
    /// 策略实例（使用 Box<dyn Strategy>）
    strategy: Box<dyn Strategy>,

    /// 消息队列（SPSC）
    msg_rx: crossbeam::channel::Receiver<StrategyMessage>,

    /// 消息发送器
    msg_tx: crossbeam::channel::Sender<StrategyMessage>,

    /// 线程句柄
    thread: Option<JoinHandle<()>>,

    /// 分配的 CPU 核心
    core_id: Option<usize>,
}

/// 策略内部消息
enum StrategyMessage {
    Tick(Tick),
    Bar(Bar),
    Fill(Fill),
    OrderRejected { order_id: OrderId, reason: String },
    QueryState(crossbeam::channel::Sender<StrategyState>),
    Shutdown,
}

impl StrategyEngine {
    /// 创建新的策略引擎
    pub fn new(
        event_rx: crossbeam::channel::Receiver<MarketEvent>,
        signal_tx: crossbeam::channel::Sender<StrategyEvent>,
        fill_tx: crossbeam::channel::Sender<Fill>,
    ) -> Self {
        Self {
            strategies: HashMap::new(),
            configs: HashMap::new(),
            event_rx,
            signal_tx,
            fill_tx,
            running: AtomicBool::new(false),
            core_pinner: None,
        }
    }

    /// 注册策略
    ///
    /// # Generic Parameters
    /// - `S`: 策略类型，必须实现 `Strategy` trait
    pub fn register_strategy<S: Strategy>(
        &mut self,
        strategy_id: StrategyId,
        config: <S as Strategy>::Config,
        core_id: Option<usize>,
    ) -> Result<()> {
        let ctx = StrategyContext {
            strategy_id,
            symbol: Symbol::from("BTCUSDT"), // TODO: 从配置读取
            position: Quantity::ZERO,
            pending_orders: Vec::new(),
            config: StrategyConfig::default(),
        };

        let strategy = S::init(config, ctx);

        let (msg_tx, msg_rx) = crossbeam::channel::unbounded();

        let actor = StrategyActor {
            strategy: Box::new(strategy),
            msg_rx,
            msg_tx: msg_tx.clone(),
            thread: None,
            core_id,
        };

        self.strategies.insert(strategy_id, actor);
        Ok(())
    }

    /// 启动引擎
    pub fn start(&mut self) -> Result<()> {
        self.running.store(true, Ordering::SeqCst);

        // 启动每个策略线程
        for (id, actor) in self.strategies.iter_mut() {
            self.start_strategy_thread(*id, actor)?;
        }

        // 启动事件分发线程
        let event_rx = self.event_rx.clone();
        let signal_tx = self.signal_tx.clone();
        let fill_tx = self.fill_tx.clone();
        let strategies = self.strategies.clone();
        let running = self.running.clone();

        tokio::spawn(async move {
            while running.load(Ordering::SeqCst) {
                if let Ok(event) = event_rx.recv() {
                    match event {
                        MarketEvent::Tick { symbol, tick } => {
                            // 分发到订阅该交易对的策略
                            for actor in strategies.values() {
                                if let Ok(_) = actor.msg_tx.send(StrategyMessage::Tick(tick)) {
                                    // 策略线程处理
                                }
                            }
                        }
                        MarketEvent::Bar { symbol, bar } => {
                            for actor in strategies.values() {
                                let _ = actor.msg_tx.send(StrategyMessage::Bar(bar));
                            }
                        }
                        _ => {}
                    }
                }
            }
        });

        Ok(())
    }

    /// 启动单个策略线程
    fn start_strategy_thread(
        &mut self,
        strategy_id: StrategyId,
        actor: &mut StrategyActor,
    ) -> Result<()> {
        let msg_rx = actor.msg_rx.clone();
        let signal_tx = self.signal_tx.clone();
        let fill_tx = self.fill_tx.clone();
        let mut strategy = unsafe { std::ptr::read(&actor.strategy) };
        let core_id = actor.core_id;

        let thread = thread::spawn(move || {
            // 绑定 CPU 核心
            if let Some(core) = core_id {
                if let Some(core_ids) = core_affinity::get_core_ids() {
                    if core < core_ids.len() {
                        core_affinity::set_for_current(core_ids[core].clone());
                        tracing::info!("Strategy {} pinned to core {}", strategy_id, core);
                    }
                }
            }

            loop {
                match msg_rx.recv() {
                    Ok(StrategyMessage::Tick(tick)) => {
                        if let Some(signal) = strategy.on_tick(&tick) {
                            let event = StrategyEvent::Signal {
                                strategy_id,
                                signal,
                            };
                            let _ = signal_tx.send(event);
                        }
                    }
                    Ok(StrategyMessage::Bar(bar)) => {
                        if let Some(signal) = strategy.on_bar(&bar) {
                            let event = StrategyEvent::Signal {
                                strategy_id,
                                signal,
                            };
                            let _ = signal_tx.send(event);
                        }
                    }
                    Ok(StrategyMessage::Fill(fill)) => {
                        strategy.on_fill(&fill);
                    }
                    Ok(StrategyMessage::OrderRejected { order_id, reason }) => {
                        strategy.on_order_rejected(order_id, reason);
                    }
                    Ok(StrategyMessage::QueryState(tx)) => {
                        let _ = tx.send(strategy.state());
                    }
                    Ok(StrategyMessage::Shutdown) => {
                        break;
                    }
                    Err(_) => break,
                }
            }
        });

        actor.thread = Some(thread);
        Ok(())
    }

    /// 停止引擎
    pub async fn stop(&mut self) -> Result<()> {
        self.running.store(false, Ordering::SeqCst);

        // 停止所有策略线程
        for (id, actor) in self.strategies.iter_mut() {
            let _ = actor.msg_tx.send(StrategyMessage::Shutdown);
            if let Some(thread) = actor.thread.take() {
                let _ = thread.join();
            }
        }

        Ok(())
    }
}
```

---

## 4. 策略注册表

### 4.1 动态策略注册

```rust
/// 策略工厂 Trait
///
/// 用于动态创建策略实例
pub trait StrategyFactory: Send + Sync {
    /// 创建策略实例
    fn create(&self, config: serde_json::Value, ctx: StrategyContext) -> Result<Box<dyn Strategy>>;

    /// 策略名称
    fn name(&self) -> &str;

    /// 验证配置
    fn validate_config(&self, config: &serde_json::Value) -> Result<()>;
}

/// 策略注册表
///
/// 支持通过名称动态加载策略
pub struct StrategyRegistry {
    factories: HashMap<String, Arc<dyn StrategyFactory>>,
}

impl StrategyRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            factories: HashMap::new(),
        };

        // 注册内置策略
        registry.register_builtin_strategies();

        registry
    }

    /// 注册策略工厂
    pub fn register(&mut self, name: String, factory: Arc<dyn StrategyFactory>) {
        self.factories.insert(name, factory);
    }

    /// 创建策略实例
    pub fn create_strategy(
        &self,
        name: &str,
        config: serde_json::Value,
        ctx: StrategyContext,
    ) -> Result<Box<dyn Strategy>> {
        let factory = self.factories.get(name)
            .ok_or_else(|| Error::Strategy(format!("Strategy '{}' not found", name)))?;

        factory.validate_config(&config)?;
        factory.create(config, ctx)
    }

    /// 获取所有已注册策略
    pub fn list_strategies(&self) -> Vec<String> {
        self.factories.keys().cloned().collect()
    }

    /// 注册内置策略
    fn register_builtin_strategies(&mut self) {
        self.register(
            "ma_cross".to_string(),
            Arc::new(MACrossFactory),
        );
        self.register(
            "mean_reversion".to_string(),
            Arc::new(MeanReversionFactory),
        );
    }
}

impl Default for StrategyRegistry {
    fn default() -> Self {
        Self::new()
    }
}
```

---

## 5. 策略统计

### 5.1 统计数据结构

```rust
/// 策略统计信息
#[derive(Debug, Clone, Default)]
pub struct StrategyStats {
    /// 生成的信号数量
    pub signals_generated: u64,

    /// 执行的订单数量
    pub orders_executed: u64,

    /// 获胜次数
    pub wins: u64,

    /// 失败次数
    pub losses: u64,

    /// 总盈亏
    pub total_pnl: f64,

    /// 最大回撤
    pub max_drawdown: f64,

    /// 夏普比率
    pub sharpe_ratio: Option<f64>,

    /// 平均持仓时间（秒）
    pub avg_hold_time_secs: f64,
}

impl StrategyStats {
    /// 胜率
    pub fn win_rate(&self) -> f64 {
        let total = self.wins + self.losses;
        if total == 0 {
            0.0
        } else {
            self.wins as f64 / total as f64
        }
    }

    /// 盈亏比
    pub fn profit_factor(&self) -> f64 {
        // TODO: 计算盈亏比
        1.0
    }
}

/// 策略性能追踪器
pub struct PerformanceTracker {
    stats: StrategyStats,
    entry_prices: HashMap<OrderId, (Price, i64)>,
    pnl_history: Vec<(i64, f64)>,
}

impl PerformanceTracker {
    pub fn new() -> Self {
        Self {
            stats: StrategyStats::default(),
            entry_prices: HashMap::new(),
            pnl_history: Vec::new(),
        }
    }

    /// 记录开仓
    pub fn record_entry(&mut self, order_id: OrderId, price: Price, ts: i64) {
        self.entry_prices.insert(order_id, (price, ts));
    }

    /// 记录平仓
    pub fn record_exit(&mut self, order_id: OrderId, price: Price, ts: i64) {
        if let Some((entry_price, entry_ts)) = self.entry_prices.remove(&order_id) {
            let pnl = (price.as_f64() - entry_price.as_f64()) * 100.0; // 假设数量为 100
            self.pnl_history.push((ts, pnl));

            if pnl > 0.0 {
                self.stats.wins += 1;
            } else if pnl < 0.0 {
                self.stats.losses += 1;
            }

            self.stats.total_pnl += pnl;

            // 更新最大回撤
            self.update_max_drawdown();
        }
    }

    fn update_max_drawdown(&mut self) {
        let mut peak = 0.0;
        let mut max_dd = 0.0;
        let mut cumulative = 0.0;

        for (_, pnl) in &self.pnl_history {
            cumulative += pnl;
            peak = peak.max(cumulative);
            let dd = peak - cumulative;
            max_dd = max_dd.max(dd);
        }

        self.stats.max_drawdown = max_dd;
    }

    pub fn stats(&self) -> &StrategyStats {
        &self.stats
    }
}
```

---

## 6. 示例策略

### 6.1 均线交叉策略

```rust
/// 双均线交叉策略
///
/// 当短期均线上穿长期均线时做多，下穿时做空
pub struct MACrossStrategy {
    /// 短期均线周期
    short_period: usize,

    /// 长期均线周期
    long_period: usize,

    /// 价格历史
    price_history: VecDeque<f64>,

    /// 当前持仓方向
    position: Option<Position>,

    /// 策略 ID
    strategy_id: StrategyId,

    /// 交易对
    symbol: Symbol,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Position {
    Long,
    Short,
    Flat,
}

pub struct MACrossConfig {
    pub short_period: usize,
    pub long_period: usize,
    pub qty: f64,
}

impl Strategy for MACrossStrategy {
    type Config = MACrossConfig;
    type Context = StrategyContext;

    fn init(config: Self::Config, ctx: Self::Context) -> Self {
        Self {
            short_period: config.short_period,
            long_period: config.long_period,
            price_history: VecDeque::with_capacity(config.long_period),
            position: Some(Position::Flat),
            strategy_id: ctx.strategy_id,
            symbol: ctx.symbol,
        }
    }

    fn on_tick(&mut self, tick: &Tick) -> Option<Signal> {
        let price = tick.price_f64();
        self.price_history.push_back(price);

        if self.price_history.len() < self.long_period {
            return None;
        }

        // 计算均线
        let short_ma: f64 = self.price_history
            .iter()
            .rev()
            .take(self.short_period)
            .sum::<f64>()
            / self.short_period as f64;

        let long_ma: f64 = self.price_history
            .iter()
            .rev()
            .take(self.long_period)
            .sum::<f64>()
            / self.long_period as f64;

        // 检查交叉
        let current_diff = short_ma - long_ma;

        // 获取上一次的差异（从历史中计算）
        let prev_short_ma: f64 = self.price_history
            .iter()
            .rev()
            .skip(1)
            .take(self.short_period)
            .sum::<f64>()
            / self.short_period as f64;

        let prev_long_ma: f64 = self.price_history
            .iter()
            .rev()
            .skip(1)
            .take(self.long_period)
            .sum::<f64>()
            / self.long_period as f64;

        let prev_diff = prev_short_ma - prev_long_ma;

        // 金叉 - 做多
        if prev_diff <= 0.0 && current_diff > 0.0 && self.position != Some(Position::Long) {
            self.position = Some(Position::Long);
            return Some(Signal::new(
                self.strategy_id,
                self.symbol,
                SignalType::Buy,
                Quantity::from_f64(0.1),
            )
            .with_price(Price::from_f64(price))
            .with_reason("Golden cross detected")
            .with_indicator("short_ma", short_ma)
            .with_indicator("long_ma", long_ma));
        }

        // 死叉 - 做空
        if prev_diff >= 0.0 && current_diff < 0.0 && self.position != Some(Position::Short) {
            self.position = Some(Position::Short);
            return Some(Signal::new(
                self.strategy_id,
                self.symbol,
                SignalType::Sell,
                Quantity::from_f64(0.1),
            )
            .with_price(Price::from_f64(price))
            .with_reason("Death cross detected")
            .with_indicator("short_ma", short_ma)
            .with_indicator("long_ma", long_ma));
        }

        None
    }

    fn name(&self) -> &str {
        "ma_cross"
    }
}
```

### 6.2 均值回归策略

```rust
/// 均值回归策略
///
/// 当价格偏离均值一定标准差时，做反向交易
pub struct MeanReversionStrategy {
    /// 回归周期
    period: usize,

    /// 标准差倍数
    std_multiplier: f64,

    /// 价格历史
    price_history: VecDeque<f64>,

    /// 策略 ID
    strategy_id: StrategyId,

    /// 交易对
    symbol: Symbol,

    /// 当前仓位
    position: f64,
}

pub struct MeanReversionConfig {
    pub period: usize,
    pub std_multiplier: f64,
    pub qty: f64,
}

impl Strategy for MeanReversionStrategy {
    type Config = MeanReversionConfig;
    type Context = StrategyContext;

    fn init(config: Self::Config, ctx: Self::Context) -> Self {
        Self {
            period: config.period,
            std_multiplier: config.std_multiplier,
            price_history: VecDeque::with_capacity(config.period),
            strategy_id: ctx.strategy_id,
            symbol: ctx.symbol,
            position: 0.0,
        }
    }

    fn on_tick(&mut self, tick: &Tick) -> Option<Signal> {
        let price = tick.price_f64();
        self.price_history.push_back(price);

        if self.price_history.len() < self.period {
            return None;
        }

        // 计算均值和标准差
        let mean: f64 = self.price_history.iter().sum::<f64>() / self.period as f64;
        let variance: f64 = self.price_history
            .iter()
            .map(|p| (p - mean).powi(2))
            .sum::<f64>()
            / self.period as f64;
        let std = variance.sqrt();

        let z_score = (price - mean) / std;

        // 价格偏离均值过多，做空
        if z_score > self.std_multiplier && self.position > -1.0 {
            self.position = -1.0;
            return Some(Signal::new(
                self.strategy_id,
                self.symbol,
                SignalType::Sell,
                Quantity::from_f64(0.1),
            )
            .with_price(Price::from_f64(price))
            .with_reason(format!("Price {} stddev above mean", z_score))
            .with_indicator("z_score", z_score));
        }

        // 价格低于均值过多，做多
        if z_score < -self.std_multiplier && self.position < 1.0 {
            self.position = 1.0;
            return Some(Signal::new(
                self.strategy_id,
                self.symbol,
                SignalType::Buy,
                Quantity::from_f64(0.1),
            )
            .with_price(Price::from_f64(price))
            .with_reason(format!("Price {} stddev below mean", z_score.abs()))
            .with_indicator("z_score", z_score));
        }

        None
    }

    fn name(&self) -> &str {
        "mean_reversion"
    }
}
```

---

## 7. 模块导出

```rust
// src/strategy/mod.rs

pub mod trait_;
pub mod engine;
pub mod registry;
pub mod context;
pub mod examples;

pub use trait_::{Strategy, Signal, SignalType, SignalMetadata};
pub use engine::{StrategyEngine, StrategyActor};
pub use registry::{StrategyRegistry, StrategyFactory};
pub use context::{StrategyContext, StrategyConfig};
pub use examples::{MACrossStrategy, MeanReversionStrategy};
```

---

## 8. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| 策略执行延迟 | < 100μs | 单 Tick 处理时间 |
| 信号生成延迟 | < 1μs | 无信号情况下的处理时间 |
| 内存占用 | < 100MB/策略 | 内存分析 |
| 跨核通信延迟 | < 10μs | SPSC 队列延迟 |

---

## 9. 未来扩展

- **策略组合**: 多个策略信号融合
- **动态参数优化**: 自动调参
- **策略回测专用模式**: 无需网络连接的回测
- **策略热重载**: 运行时替换策略逻辑
