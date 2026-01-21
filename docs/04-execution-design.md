# 执行模块设计文档 (execution)

## 1. 模块概述

### 1.1 职责
- 订单生命周期管理（创建、提交、取消、查询）
- 风控检查（持仓限制、资金限制、亏损限制）
- 交易所 API 集成（订单提交、成交回报）
- 订单簿管理

### 1.2 依赖关系
- **依赖**: `core` (Tick, Event, Error), `strategy` (Signal)
- **被依赖**: `backtest`

### 1.3 文件结构
```
src/execution/
├── mod.rs           # 模块导出
├── order.rs         # 订单定义和状态机
├── oms.rs           # 订单管理系统
├── risk.rs          # 风控引擎
├── client.rs        # 交易所客户端 Trait
└── exchanges/       # 具体交易所实现
    ├── mod.rs
    ├── binance.rs   # Binance
    └── okx.rs       # OKX
```

---

## 2. 订单定义

### 2.1 订单结构

```rust
/// 订单类型
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderType {
    /// 市价单
    Market,
    /// 限价单
    Limit(Price),
    /// 止损单
    Stop(Price),
    /// 止盈限价单
    StopLimit { stop_price: Price, limit_price: Price },
    /// 仅作 Maker 单
    PostOnly(Price),
}

/// 订单方向
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderSide {
    /// 买入
    Buy,
    /// 卖出
    Sell,
}

/// 订单状态
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum OrderStatus {
    /// 新建（本地创建，未提交）
    New,
    /// 已提交
    Submitted,
    /// 部分成交
    PartiallyFilled,
    /// 完全成交
    Filled,
    /// 已取消
    Cancelled,
    /// 已拒绝
    Rejected,
    /// 过期
    Expired,
}

/// 订单
#[derive(Debug, Clone)]
pub struct Order {
    /// 本地订单 ID
    pub id: OrderId,

    /// 交易所订单 ID
    pub exchange_id: Option<String>,

    /// 交易对
    pub symbol: Symbol,

    /// 订单方向
    pub side: OrderSide,

    /// 订单类型
    pub order_type: OrderType,

    /// 订单数量
    pub qty: Quantity,

    /// 订单价格
    pub price: Option<Price>,

    /// 订单状态
    pub status: OrderStatus,

    /// 已成交数量
    pub filled_qty: Quantity,

    /// 平均成交价
    pub avg_fill_price: Price,

    /// 创建时间（纳秒）
    pub created_at: i64,

    /// 更新时间（纳秒）
    pub updated_at: i64,

    /// 来源策略 ID
    pub strategy_id: StrategyId,

    /// 关联信号 ID
    pub signal_id: Option<SignalId>,

    /// 止损价
    pub stop_loss: Option<Price>,

    /// 止盈价
    pub take_profit: Option<Price>,
}

impl Order {
    /// 创建新的订单
    pub fn new(
        symbol: Symbol,
        side: OrderSide,
        order_type: OrderType,
        qty: Quantity,
        strategy_id: StrategyId,
    ) -> Self {
        let now = utils::now_nanos();
        let price = match order_type {
            OrderType::Limit(p) | OrderType::PostOnly(p) => Some(p),
            OrderType::Stop(p) => Some(p),
            OrderType::StopLimit { limit_price, .. } => Some(limit_price),
            OrderType::Market => None,
        };

        Self {
            id: OrderId::new(),
            exchange_id: None,
            symbol,
            side,
            order_type,
            qty,
            price,
            status: OrderStatus::New,
            filled_qty: Quantity::ZERO,
            avg_fill_price: Price::ZERO,
            created_at: now,
            updated_at: now,
            strategy_id,
            signal_id: None,
            stop_loss: None,
            take_profit: None,
        }
    }

    /// 是否已完全成交
    pub const fn is_filled(&self) -> bool {
        self.status == OrderStatus::Filled
    }

    /// 是否处于活跃状态
    pub const fn is_active(&self) -> bool {
        matches!(
            self.status,
            OrderStatus::Submitted | OrderStatus::PartiallyFilled
        )
    }

    /// 是否已终态
    pub const fn is_terminal(&self) -> bool {
        matches!(
            self.status,
            OrderStatus::Filled | OrderStatus::Cancelled
                | OrderStatus::Rejected | OrderStatus::Expired
        )
    }

    /// 剩余数量
    pub const fn remaining_qty(&self) -> Quantity {
        Quantity::new(self.qty.as_u64() - self.filled_qty.as_u64())
    }

    /// 计算订单价值
    pub fn value(&self) -> f64 {
        let price = self.price.unwrap_or(self.avg_fill_price);
        self.qty.as_f64() * price.as_f64()
    }
}

/// 订单请求（用于提交到交易所）
#[derive(Debug, Clone)]
pub struct OrderRequest {
    pub symbol: Symbol,
    pub side: OrderSide,
    pub order_type: OrderType,
    pub qty: Quantity,
    pub time_in_force: TimeInForce,
    pub reduce_only: bool,
}

/// 订单有效期
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TimeInForce {
    /// 立即成交或取消（IOC）
    ImmediateOrCancel,
    /// 成交或取消（FOK）
    FillOrKill,
    /// 当天有效（GTD）
    GoodTillDate(i64),
    /// 撤销前有效（GTC）
    GoodTillCancel,
}
```

---

## 3. 订单管理系统 (OMS)

### 3.1 OMS 架构

```rust
/// 订单管理系统
///
/// 负责订单的生命周期管理、路由到交易所、跟踪订单状态
pub struct OrderManager {
    /// 所有订单
    orders: HashMap<OrderId, Order>,

    /// 活跃订单（未终态）
    active_orders: HashMap<OrderId, Order>,

    /// 按策略分组的订单
    strategy_orders: HashMap<StrategyId, HashSet<OrderId>>,

    /// 按交易对分组的订单
    symbol_orders: HashMap<Symbol, HashSet<OrderId>>,

    /// 交易所客户端
    exchange: Box<dyn ExchangeClient>,

    /// 风控引擎
    risk_engine: RiskEngine,

    /// 成交发送器
    fill_tx: crossbeam::channel::Sender<Fill>,

    /// 事件发送器
    event_tx: crossbeam::channel::Sender<OrderEvent>,

    /// 运行状态
    running: Arc<AtomicBool>,

    /// 统计
    stats: OrderStats,
}

/// 订单统计
#[derive(Debug, Clone, Default)]
pub struct OrderStats {
    pub submitted: u64,
    pub filled: u64,
    pub cancelled: u64,
    pub rejected: u64,
    pub fills: u64,
    pub total_volume: f64,
}

impl OrderManager {
    /// 创建新的订单管理器
    pub fn new(
        exchange: Box<dyn ExchangeClient>,
        risk_engine: RiskEngine,
        fill_tx: crossbeam::channel::Sender<Fill>,
        event_tx: crossbeam::channel::Sender<OrderEvent>,
    ) -> Self {
        Self {
            orders: HashMap::new(),
            active_orders: HashMap::new(),
            strategy_orders: HashMap::new(),
            symbol_orders: HashMap::new(),
            exchange,
            risk_engine,
            fill_tx,
            event_tx,
            running: Arc::new(AtomicBool::new(false)),
            stats: OrderStats::default(),
        }
    }

    /// 提交订单
    pub async fn submit_order(
        &mut self,
        request: OrderRequest,
        strategy_id: StrategyId,
    ) -> Result<OrderId> {
        // 创建订单
        let order = Order::new(
            request.symbol,
            request.side,
            request.order_type,
            request.qty,
            strategy_id,
        );

        let order_id = order.id;

        // 风控检查
        self.risk_engine.check_order(&order, request.time_in_force)?;

        // 本地存储
        self.store_order(order.clone());

        // 发送到交易所
        let exchange_id = self.exchange.submit_order(request.clone()).await
            .map_err(|e| Error::Exchange(format!("Submit order failed: {}", e)))?;

        // 更新状态
        self.update_order_status(order_id, OrderStatus::Submitted);
        self.set_exchange_id(order_id, exchange_id);

        // 发送事件
        let _ = self.event_tx.send(OrderEvent::Submitted { order_id });

        self.stats.submitted += 1;

        Ok(order_id)
    }

    /// 取消订单
    pub async fn cancel_order(&mut self, order_id: OrderId) -> Result<()> {
        let order = self.active_orders.get(&order_id)
            .ok_or_else(|| Error::Validation("Order not found or not active".to_string()))?;

        if let Some(exchange_id) = &order.exchange_id {
            self.exchange.cancel_order(order.symbol, exchange_id.clone()).await?;
        }

        self.update_order_status(order_id, OrderStatus::Cancelled);
        let _ = self.event_tx.send(OrderEvent::Cancelled { order_id });

        self.stats.cancelled += 1;

        Ok(())
    }

    /// 取消策略的所有订单
    pub async fn cancel_strategy_orders(&mut self, strategy_id: StrategyId) -> Result<u32> {
        let orders: Vec<OrderId> = self.strategy_orders
            .get(&strategy_id)
            .map(|ids| ids.iter().copied().collect())
            .unwrap_or_default();

        for order_id in orders {
            let _ = self.cancel_order(order_id).await;
        }

        Ok(orders.len() as u32)
    }

    /// 处理成交回报
    pub fn on_fill(&mut self, fill: Fill) -> Result<()> {
        let order = self.orders.get_mut(&fill.order_id)
            .ok_or_else(|| Error::Validation("Order not found".to_string()))?;

        // 更新成交信息
        order.filled_qty = Quantity::new(order.filled_qty.as_u64() + fill.qty.as_u64());

        let total_value = order.avg_fill_price.as_f64() * order.filled_qty.as_f64();
        let fill_value = fill.price.as_f64() * fill.qty.as_f64();
        order.avg_fill_price = Price::from_f64((total_value + fill_value) / order.filled_qty.as_f64());

        // 更新状态
        order.updated_at = fill.ts;
        if order.filled_qty >= order.qty {
            self.update_order_status(fill.order_id, OrderStatus::Filled);
        } else {
            self.update_order_status(fill.order_id, OrderStatus::PartiallyFilled);
        }

        // 发送成交事件
        self.stats.fills += 1;
        self.stats.total_volume += fill.qty.as_f64();
        let _ = self.fill_tx.send(fill);

        Ok(())
    }

    /// 存储订单
    fn store_order(&mut self, order: Order) {
        let order_id = order.id;

        self.orders.insert(order_id, order.clone());
        self.active_orders.insert(order_id, order.clone());

        self.strategy_orders
            .entry(order.strategy_id)
            .or_default()
            .insert(order_id);

        self.symbol_orders
            .entry(order.symbol)
            .or_default()
            .insert(order_id);
    }

    /// 更新订单状态
    fn update_order_status(&mut self, order_id: OrderId, status: OrderStatus) {
        if let Some(order) = self.orders.get_mut(&order_id) {
            order.status = status;
            order.updated_at = utils::now_nanos();

            if status.is_terminal() {
                self.active_orders.remove(&order_id);
            }

            if status == OrderStatus::Filled {
                self.stats.filled += 1;
            } else if status == OrderStatus::Rejected {
                self.stats.rejected += 1;
            }
        }
    }

    /// 设置交易所订单 ID
    fn set_exchange_id(&mut self, order_id: OrderId, exchange_id: String) {
        if let Some(order) = self.orders.get_mut(&order_id) {
            order.exchange_id = Some(exchange_id);
        }
    }

    /// 获取订单
    pub fn get_order(&self, order_id: OrderId) -> Option<&Order> {
        self.orders.get(&order_id)
    }

    /// 获取策略订单
    pub fn get_strategy_orders(&self, strategy_id: StrategyId) -> Vec<&Order> {
        self.strategy_orders
            .get(&strategy_id)
            .map(|ids| {
                ids.iter()
                    .filter_map(|id| self.orders.get(id))
                    .collect()
            })
            .unwrap_or_default()
    }

    /// 获取活跃订单
    pub fn get_active_orders(&self) -> Vec<&Order> {
        self.active_orders.values().collect()
    }

    /// 获取统计信息
    pub fn stats(&self) -> &OrderStats {
        &self.stats
    }
}
```

---

## 4. 风控引擎

### 4.1 风控规则

```rust
/// 风控引擎
///
/// 实时检查订单是否符合风控规则
pub struct RiskEngine {
    /// 最大持仓
    max_position: Quantity,

    /// 最大单笔订单价值
    max_order_value: f64,

    /// 日内最大亏损
    daily_loss_limit: f64,

    /// 当前持仓
    current_position: HashMap<Symbol, Quantity>,

    /// 当前盈亏
    current_pnl: HashMap<Symbol, f64>,

    /// 今日盈亏
    daily_pnl: f64,

    /// 今日开始时间
    day_start: i64,

    /// 订单速率限制
    rate_limiter: RateLimiter,

    /// 风控配置
    config: RiskConfig,
}

/// 风控配置
#[derive(Debug, Clone)]
pub struct RiskConfig {
    /// 最大持仓比例（相对于账户余额）
    pub max_position_ratio: f64,

    /// 最大单笔订单比例
    pub max_order_ratio: f64,

    /// 最大日内亏损比例
    pub max_daily_loss_ratio: f64,

    /// 单位时间最大订单数
    pub max_orders_per_sec: u32,

    /// 最大持仓数
    pub max_positions: usize,
}

/// 风控检查结果
#[derive(Debug, Clone)]
pub enum RiskResult {
    /// 通过
    Passed,
    /// 拒绝
    Rejected(String),
    /// 调整后通过
    Adjusted { new_qty: Quantity },
}

impl RiskEngine {
    /// 创建新的风控引擎
    pub fn new(config: RiskConfig, account_balance: f64) -> Self {
        let day_start = utils::day_start_ts();

        Self {
            max_position: Quantity::from_f64(account_balance * config.max_position_ratio),
            max_order_value: account_balance * config.max_order_ratio,
            daily_loss_limit: account_balance * config.max_daily_loss_ratio,
            current_position: HashMap::new(),
            current_pnl: HashMap::new(),
            daily_pnl: 0.0,
            day_start,
            rate_limiter: RateLimiter::new(config.max_orders_per_sec),
            config,
        }
    }

    /// 检查订单
    pub fn check_order(
        &self,
        order: &Order,
        tif: TimeInForce,
    ) -> Result<()> {
        // 检查订单速率
        if !self.rate_limiter.check() {
            return Err(Error::Risk("Order rate limit exceeded".to_string()));
        }

        // 检查订单价值
        let order_value = order.value();
        if order_value > self.max_order_value {
            return Err(Error::Risk(format!(
                "Order value {} exceeds limit {}",
                order_value, self.max_order_value
            )));
        }

        // 检查持仓限制
        if let Some(current) = self.current_position.get(&order.symbol) {
            let new_position = match order.side {
                OrderSide::Buy => current.as_u64() + order.qty.as_u64(),
                OrderSide::Sell => current.as_u64().saturating_sub(order.qty.as_u64()),
            };

            if new_position > self.max_position.as_u64() {
                return Err(Error::Risk(format!(
                    "Position {} exceeds limit {}",
                    new_position, self.max_position.as_u64()
                )));
            }
        }

        // 检查日内亏损
        if self.daily_pnl < -self.daily_loss_limit {
            return Err(Error::Risk(format!(
                "Daily loss {} exceeds limit {}",
                self.daily_pnl.abs(),
                self.daily_loss_limit
            )));
        }

        Ok(())
    }

    /// 更新持仓
    pub fn update_position(&mut self, symbol: Symbol, delta: Quantity) {
        let entry = self.current_position.entry(symbol).or_default();
        let current = entry.as_u64() as i64;
        let change = delta.as_u64() as i64;
        let new = current.saturating_add(change);
        *entry = Quantity::new(new.max(0) as u64);
    }

    /// 更新盈亏
    pub fn update_pnl(&mut self, symbol: Symbol, pnl: f64) {
        *self.current_pnl.entry(symbol).or_default() += pnl;
        self.daily_pnl += pnl;
    }

    /// 检查止损
    pub fn check_stop_loss(&self, symbol: Symbol) -> Option<OrderRequest> {
        if let Some(pnl) = self.current_pnl.get(&symbol) {
            if *pnl < -self.daily_loss_limit / 10.0 {
                // 触发平仓
                let current = self.current_position.get(&symbol)?;
                return Some(OrderRequest {
                    symbol,
                    side: OrderSide::Sell,
                    order_type: OrderType::Market,
                    qty: *current,
                    time_in_force: TimeInForce::ImmediateOrCancel,
                    reduce_only: true,
                });
            }
        }
        None
    }

    /// 获取当前持仓
    pub fn get_position(&self, symbol: Symbol) -> Quantity {
        *self.current_position.get(&symbol).unwrap_or(&Quantity::ZERO)
    }

    /// 获取所有持仓
    pub fn get_all_positions(&self) -> &HashMap<Symbol, Quantity> {
        &self.current_position
    }
}

/// 订单速率限制器
struct RateLimiter {
    max_per_sec: u32,
    timestamps: VecDeque<i64>,
}

impl RateLimiter {
    fn new(max_per_sec: u32) -> Self {
        Self {
            max_per_sec,
            timestamps: VecDeque::new(),
        }
    }

    fn check(&mut self) -> bool {
        let now = utils::now_nanos();
        let one_sec_ago = now - 1_000_000_000;

        // 移除过期的时间戳
        while let Some(&ts) = self.timestamps.front() {
            if ts < one_sec_ago {
                self.timestamps.pop_front();
            } else {
                break;
            }
        }

        // 检查是否超过限制
        if self.timestamps.len() < self.max_per_sec as usize {
            self.timestamps.push_back(now);
            true
        } else {
            false
        }
    }
}
```

---

## 5. 交易所客户端

### 5.1 ExchangeClient Trait

```rust
/// 交易所客户端 Trait
///
/// 抽象不同交易所的接口
#[async_trait::async_trait]
pub trait ExchangeClient: Send + Sync {
    /// 提交订单
    async fn submit_order(&self, request: OrderRequest) -> Result<String>;

    /// 取消订单
    async fn cancel_order(&self, symbol: Symbol, order_id: String) -> Result<()>;

    /// 查询订单状态
    async fn query_order(
        &self,
        symbol: Symbol,
        order_id: String,
    ) -> Result<OrderStatus>;

    /// 查询账户余额
    async fn query_balance(&self) -> Result<HashMap<String, f64>>;

    /// 查询持仓
    async fn query_positions(&self) -> Result<HashMap<Symbol, Quantity>>;

    /// 获取订单簿
    async fn get_orderbook(&self, symbol: Symbol, depth: usize) -> Result<OrderBook>;

    /// 获取交易所名称
    fn name(&self) -> &str;
}

/// 订单簿
#[derive(Debug, Clone)]
pub struct OrderBook {
    /// 交易对
    pub symbol: Symbol,

    /// 时间戳
    pub ts: i64,

    /// 买盘 (价格 -> 数量)
    pub bids: Vec<(Price, Quantity)>,

    /// 卖盘 (价格 -> 数量)
    pub asks: Vec<(Price, Quantity)>,
}

impl OrderBook {
    /// 获取中间价
    pub fn mid_price(&self) -> Option<Price> {
        let best_bid = self.bids.first().map(|(p, _)| p);
        let best_ask = self.asks.first().map(|(p, _)| p);

        match (best_bid, best_ask) {
            (Some(bid), Some(ask)) => Some(Price::new((bid.as_i64() + ask.as_i64()) / 2)),
            _ => None,
        }
    }

    /// 获取买卖价差
    pub fn spread(&self) -> Option<i64> {
        let best_bid = self.bids.first().map(|(p, _)| p.as_i64());
        let best_ask = self.asks.first().map(|(p, _)| p.as_i64());

        match (best_bid, best_ask) {
            (Some(bid), Some(ask)) => Some(ask - bid),
            _ => None,
        }
    }
}
```

---

## 6. 成交回报

### 6.1 Fill 定义

```rust
/// 成交回报
#[derive(Debug, Clone)]
pub struct Fill {
    /// 订单 ID
    pub order_id: OrderId,

    /// 交易所订单 ID
    pub exchange_order_id: Option<String>,

    /// 交易对
    pub symbol: Symbol,

    /// 成交价格
    pub price: Price,

    /// 成交数量
    pub qty: Quantity,

    /// 成交时间（纳秒）
    pub ts: i64,

    /// 手续费
    pub fee: f64,

    /// 交易所
    pub exchange: String,

    /// 是否为 Maker
    pub is_maker: bool,
}

impl Fill {
    /// 计算成交价值
    pub fn value(&self) -> f64 {
        self.qty.as_f64() * self.price.as_f64()
    }
}
```

---

## 7. 模块导出

```rust
// src/execution/mod.rs

pub mod order;
pub mod oms;
pub mod risk;
pub mod client;
pub mod exchanges;

pub use order::{
    Order, OrderType, OrderSide, OrderStatus,
    OrderRequest, TimeInForce,
};
pub use oms::{OrderManager, OrderStats};
pub use risk::{RiskEngine, RiskConfig, RiskResult};
pub use client::{ExchangeClient, OrderBook};
pub use exchanges::{BinanceClient, OKXClient};
```

---

## 8. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| 订单提交延迟 | < 10ms | 端到端延迟测试 |
| 订单取消延迟 | < 5ms | 端到端延迟测试 |
| 风控检查延迟 | < 1μs | 单次检查耗时 |
| 成交处理延迟 | < 100μs | 消息处理耗时 |

---

## 9. 未来扩展

- **智能路由**: 多交易所最优价格路由
- **算法订单**: TWAP/VWAP 算法
- **冰山订单**: 大单拆分
- **套利模块**: 跨交易所套利
