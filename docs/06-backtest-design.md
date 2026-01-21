# 回测模块设计文档 (backtest)

## 1. 模块概述

### 1.1 职责
- 历史数据回放
- 模拟交易所执行逻辑
- 策略性能指标计算
- 回测报告生成

### 1.2 依赖关系
- **依赖**: `core`, `data`, `strategy`, `execution`
- **被依赖**: 无（顶层模块）

### 1.3 文件结构
```
src/backtest/
├── mod.rs           # 模块导出
├── engine.rs        # 回测引擎
├── replay.rs        # 数据回放
├── execution.rs     # 模拟执行
├── metrics.rs       # 性能指标
└── report.rs        # 报告生成
```

---

## 2. 回测引擎设计

### 2.1 核心架构

```rust
/// 回测引擎
///
/// 基于历史数据模拟策略执行，计算性能指标
pub struct BacktestEngine {
    /// 策略实例
    strategy: Box<dyn Strategy>,

    /// 数据回放器
    replay: DataReplay,

    /// 模拟执行器
    execution: SimulatedExecution,

    /// 回测配置
    config: BacktestConfig,

    /// 当前状态
    state: BacktestState,
}

/// 回测配置
#[derive(Debug, Clone)]
pub struct BacktestConfig {
    /// 开始时间
    pub start_ts: i64,

    /// 结束时间
    pub end_ts: i64,

    /// 初始资金
    pub initial_capital: f64,

    /// 交易手续费率
    pub commission_rate: f64,

    /// 滑点（百分比）
    pub slippage_pct: f64,

    /// 每股最小单位
    pub min_tick_size: f64,

    /// 是否启用详细日志
    pub verbose: bool,
}

impl Default for BacktestConfig {
    fn default() -> Self {
        Self {
            start_ts: 0,
            end_ts: i64::MAX,
            initial_capital: 100_000.0,
            commission_rate: 0.001,
            slippage_pct: 0.0005,
            min_tick_size: 0.01,
            verbose: false,
        }
    }
}

/// 回测状态
#[derive(Debug, Clone)]
pub struct BacktestState {
    /// 当前时间戳
    pub current_ts: i64,

    /// 当前价格
    pub current_price: f64,

    /// 持仓数量
    pub position: f64,

    /// 持仓均价
    pub avg_entry_price: f64,

    /// 可用现金
    pub cash: f64,

    /// 总权益
    pub equity: f64,

    /// 未平仓订单
    pub pending_orders: Vec<Order>,

    /// 已完成订单
    pub completed_orders: Vec<Order>,

    /// 成交记录
    pub trades: Vec<Trade>,

    /// 权益曲线
    pub equity_curve: Vec<(i64, f64)>,
}

impl BacktestState {
    pub fn initial(capital: f64) -> Self {
        Self {
            current_ts: 0,
            current_price: 0.0,
            position: 0.0,
            avg_entry_price: 0.0,
            cash: capital,
            equity: capital,
            pending_orders: Vec::new(),
            completed_orders: Vec::new(),
            trades: Vec::new(),
            equity_curve: Vec::new(),
        }
    }

    /// 计算浮动盈亏
    pub const fn unrealized_pnl(&self) -> f64 {
        if self.position == 0.0 {
            0.0
        } else {
            (self.current_price - self.avg_entry_price) * self.position
        }
    }

    /// 计算已实现盈亏
    pub const fn realized_pnl(&self) -> f64 {
        self.cash + self.position * self.current_price - self.equity_curve.get(0)
            .map(|(_, e)| *e)
            .unwrap_or(0.0)
    }
}

impl BacktestEngine {
    /// 创建新的回测引擎
    pub fn new(
        strategy: Box<dyn Strategy>,
        replay: DataReplay,
        config: BacktestConfig,
    ) -> Self {
        Self {
            strategy,
            replay,
            config,
            state: BacktestState::initial(config.initial_capital),
        }
    }

    /// 运行回测
    pub fn run(&mut self) -> Result<BacktestReport> {
        tracing::info!("Starting backtest from {} to {}",
            self.config.start_ts, self.config.end_ts);

        // 重置状态
        self.state = BacktestState::initial(self.config.initial_capital);

        // 开始回放
        while let Some(tick) = self.replay.next()? {
            if tick.ts < self.config.start_ts {
                continue;
            }
            if tick.ts > self.config.end_ts {
                break;
            }

            self.process_tick(tick)?;
        }

        // 处理剩余的挂单
        self.process_pending_orders()?;

        // 生成报告
        let report = self.generate_report()?;

        Ok(report)
    }

    /// 处理单个 Tick
    fn process_tick(&mut self, tick: Tick) -> Result<()> {
        self.state.current_ts = tick.ts;
        self.state.current_price = tick.price_f64();

        // 更新权益
        self.update_equity();

        // 处理挂单
        self.process_orders(&tick)?;

        // 策略处理
        if let Some(signal) = self.strategy.on_tick(&tick) {
            self.handle_signal(signal, &tick)?;
        }

        Ok(())
    }

    /// 更新权益
    fn update_equity(&mut self) {
        self.state.equity = self.state.cash
            + self.state.position * self.state.current_price;
        self.state.equity_curve.push((self.state.current_ts, self.state.equity));
    }

    /// 处理挂单
    fn process_orders(&mut self, tick: &Tick) -> Result<()> {
        let mut filled = Vec::new();

        for order in self.state.pending_orders.iter_mut() {
            if self.execution.should_fill(order, tick) {
                let fill = self.execution.fill_order(order, tick, &self.config)?;
                filled.push((order.id, fill));
            }
        }

        for (order_id, fill) in filled {
            self.apply_fill(order_id, fill)?;
        }

        // 移除已成交订单
        self.state.pending_orders.retain(|o| !o.is_filled());

        Ok(())
    }

    /// 处理剩余挂单
    fn process_pending_orders(&mut self) -> Result<()> {
        // 取消所有未成交订单
        let order_ids: Vec<_> = self.state.pending_orders.iter()
            .map(|o| o.id)
            .collect();

        for order_id in order_ids {
            self.state.completed_orders.push(
                self.state.pending_orders.remove(
                    self.state.pending_orders.iter()
                        .position(|o| o.id == order_id)
                        .unwrap()
                )
            );
        }

        Ok(())
    }

    /// 应用成交
    fn apply_fill(&mut self, order_id: OrderId, fill: Fill) -> Result<()> {
        // 更新持仓
        let sign = match self.state.pending_orders.iter()
            .find(|o| o.id == order_id)
            .map(|o| o.side) {
            Some(OrderSide::Buy) => 1.0,
            Some(OrderSide::Sell) => -1.0,
            None => return Ok(()),
        };

        let qty = fill.qty.as_f64() * sign;
        let price = fill.price.as_f64();

        self.state.cash -= qty * price * (1.0 + self.config.commission_rate);
        self.state.position += qty;

        // 更新持仓均价
        if self.state.position != 0.0 {
            let old_value = self.state.avg_entry_price * (self.state.position - qty);
            let new_value = price * qty;
            self.state.avg_entry_price = (old_value + new_value) / self.state.position;
        }

        // 记录交易
        self.state.trades.push(Trade {
            ts: fill.ts,
            side: if qty > 0.0 { OrderSide::Buy } else { OrderSide::Sell },
            price: fill.price,
            qty: fill.qty,
            pnl: 0.0, // 需要计算
        });

        Ok(())
    }

    /// 处理信号
    fn handle_signal(&mut self, signal: Signal, tick: &Tick) -> Result<()> {
        match signal.signal_type {
            SignalType::Buy => {
                let qty = self.calculate_order_qty(signal.qty, tick.price_f64())?;
                if qty > 0.0 {
                    let order = Order::new(
                        signal.symbol,
                        OrderSide::Buy,
                        OrderType::Limit(Price::from_f64(tick.price_f64())),
                        Quantity::from_f64(qty),
                        signal.strategy_id,
                    );
                    self.state.pending_orders.push(order);
                }
            }
            SignalType::Sell => {
                let qty = self.state.position.abs().min(signal.qty.as_f64());
                if qty > 0.0 {
                    let order = Order::new(
                        signal.symbol,
                        OrderSide::Sell,
                        OrderType::Limit(Price::from_f64(tick.price_f64())),
                        Quantity::from_f64(qty),
                        signal.strategy_id,
                    );
                    self.state.pending_orders.push(order);
                }
            }
            SignalType::Close | SignalType::CloseLong | SignalType::CloseShort => {
                // 平仓逻辑
                if self.state.position != 0.0 {
                    let order = Order::new(
                        signal.symbol,
                        if self.state.position > 0.0 {
                            OrderSide::Sell
                        } else {
                            OrderSide::Buy
                        },
                        OrderType::Market,
                        Quantity::from_f64(self.state.position.abs()),
                        signal.strategy_id,
                    );
                    self.state.pending_orders.push(order);
                }
            }
        }

        Ok(())
    }

    /// 计算订单数量
    fn calculate_order_qty(&self, qty: Quantity, price: f64) -> Result<f64> {
        let max_qty = self.state.cash / price / (1.0 + self.config.commission_rate);
        let qty = qty.as_f64().min(max_qty);
        Ok(qty)
    }

    /// 生成报告
    fn generate_report(&self) -> Result<BacktestReport> {
        Ok(BacktestReport {
            config: self.config.clone(),
            metrics: MetricsCalculator::calculate(&self.state, &self.config),
            equity_curve: self.state.equity_curve.clone(),
            trades: self.state.trades.clone(),
        })
    }
}
```

---

## 3. 数据回放

### 3.1 Replay 设计

```rust
/// 数据回放器
///
/// 从数据源按时间顺序读取 Ticks
pub struct DataReplay {
    /// 数据源
    source: DataSource,

    /// 当前缓冲区
    buffer: VecDeque<Tick>,

    /// 读取批次大小
    batch_size: usize,
}

#[derive(Debug)]
pub enum DataSource {
    /// 从 Parquet 文件读取
    Parquet {
        path: PathBuf,
        reader: Option<ParquetReader>,
    },
    /// 从内存数据读取
    Memory(Vec<Tick>),
    /// 从回调函数读取
    Callback(Box<dyn FnMut() -> Option<Tick> + Send>),
}

impl DataReplay {
    /// 从 Parquet 文件创建回放器
    pub fn from_parquet(path: impl AsRef<Path>) -> Result<Self> {
        let path = path.as_ref().to_path_buf();
        let reader = Some(ParquetReader::new(&path)?);

        Ok(Self {
            source: DataSource::Parquet { path, reader },
            buffer: VecDeque::new(),
            batch_size: 10_000,
        })
    }

    /// 从内存数据创建回放器
    pub fn from_memory(ticks: Vec<Tick>) -> Self {
        let mut sorted = ticks;
        sorted.sort_by_key(|t| t.ts);

        Self {
            source: DataSource::Memory(sorted),
            buffer: VecDeque::new(),
            batch_size: 10_000,
        }
    }

    /// 读取下一个 Tick
    pub fn next(&mut self) -> Result<Option<Tick>> {
        // 如果缓冲区为空，从数据源读取更多
        if self.buffer.is_empty() {
            self.refill_buffer()?;
        }

        Ok(self.buffer.pop_front())
    }

    /// 填充缓冲区
    fn refill_buffer(&mut self) -> Result<()> {
        match &mut self.source {
            DataSource::Parquet { reader, .. } => {
                if let Some(r) = reader {
                    let ticks = r.read_next(self.batch_size)?;
                    for tick in ticks {
                        self.buffer.push_back(tick);
                    }
                }
            }
            DataSource::Memory(ticks) => {
                for tick in ticks.drain(..self.batch_size.min(ticks.len())) {
                    self.buffer.push_back(tick);
                }
            }
            DataSource::Callback(cb) => {
                for _ in 0..self.batch_size {
                    if let Some(tick) = cb() {
                        self.buffer.push_back(tick);
                    }
                }
            }
        }
        Ok(())
    }
}

/// Parquet 读取器
struct ParquetReader {
    reader: ParquetRecordBatchReader,
    current_batch: Option<Vec<Tick>>,
    batch_index: usize,
}

impl ParquetReader {
    fn new(path: &Path) -> Result<Self> {
        let file = File::open(path)?;
        let reader = ParquetRecordBatchReader::try_new(file, 1024 * 1024)?;

        Ok(Self {
            reader,
            current_batch: None,
            batch_index: 0,
        })
    }

    fn read_next(&mut self, limit: usize) -> Result<Vec<Tick>> {
        let mut ticks = Vec::new();

        while ticks.len() < limit {
            if self.current_batch.is_none() {
                if let Some(batch) = self.reader.next() {
                    let batch = batch.map_err(|e| Error::Storage(e.to_string()))?;
                    self.current_batch = Some(Tick::from_record_batch(&batch));
                    self.batch_index = 0;
                } else {
                    break;
                }
            }

            if let Some(batch) = &self.current_batch {
                for tick in batch.iter().skip(self.batch_index).take(limit - ticks.len()) {
                    ticks.push(*tick);
                }
                self.batch_index += ticks.len();

                if self.batch_index >= batch.len() {
                    self.current_batch = None;
                }
            }
        }

        Ok(ticks)
    }
}
```

---

## 4. 模拟执行

### 4.1 Execution Simulator

```rust
/// 模拟执行器
///
/// 模拟交易所的订单执行逻辑
pub struct SimulatedExecution;

impl SimulatedExecution {
    /// 判断订单是否应该成交
    pub fn should_fill(&self, order: &Order, tick: &Tick) -> bool {
        match order.order_type {
            OrderType::Market => true,
            OrderType::Limit(price) => {
                match order.side {
                    OrderSide::Buy => tick.price_f64() <= price.as_f64(),
                    OrderSide::Sell => tick.price_f64() >= price.as_f64(),
                }
            }
            OrderType::Stop(stop_price) => {
                match order.side {
                    OrderSide::Buy => tick.price_f64() >= stop_price.as_f64(),
                    OrderSide::Sell => tick.price_f64() <= stop_price.as_f64(),
                }
            }
            OrderType::PostOnly(price) => {
                // 检查是否为 Maker
                match order.side {
                    OrderSide::Buy => tick.ask_px_f64() > price.as_f64(),
                    OrderSide::Sell => tick.bid_px_f64() < price.as_f64(),
                }
            }
            OrderType::StopLimit { stop_price, limit_price } => {
                let stop_triggered = match order.side {
                    OrderSide::Buy => tick.price_f64() >= stop_price.as_f64(),
                    OrderSide::Sell => tick.price_f64() <= stop_price.as_f64(),
                };
                stop_triggered && self.should_fill(&Order {
                    order_type: OrderType::Limit(*limit_price),
                    ..order.clone()
                }, tick)
            }
        }
    }

    /// 执行订单
    pub fn fill_order(
        &self,
        order: &mut Order,
        tick: &Tick,
        config: &BacktestConfig,
    ) -> Result<Fill> {
        let qty = order.remaining_qty();
        let mut price = tick.price;

        // 应用滑点
        if let OrderType::Limit(limit_price) = order.order_type {
            price = limit_price;
        } else {
            price = Price::from_f64(
                price.as_f64() * (1.0 + match order.side {
                    OrderSide::Buy => config.slippage_pct,
                    OrderSide::Sell => -config.slippage_pct,
                })
            );
        }

        order.filled_qty = Quantity::new(order.filled_qty.as_u64() + qty.as_u64());
        order.avg_fill_price = Price::from_f64(
            (order.avg_fill_price.as_f64() * order.filled_qty.as_f64()
                + price.as_f64() * qty.as_f64())
                / (order.filled_qty.as_f64())
        );
        order.status = OrderStatus::Filled;

        let fee = order.filled_qty.as_f64() * price.as_f64() * config.commission_rate;

        Ok(Fill {
            order_id: order.id,
            exchange_order_id: None,
            symbol: order.symbol,
            price,
            qty,
            ts: tick.ts,
            fee,
            exchange: "SIMULATED".to_string(),
            is_maker: false,
        })
    }
}
```

---

## 5. 性能指标计算

### 5.1 Metrics Calculator

```rust
/// 性能指标计算器
pub struct MetricsCalculator;

impl MetricsCalculator {
    /// 计算所有指标
    pub fn calculate(state: &BacktestState, config: &BacktestConfig) -> BacktestMetrics {
        let total_return = (state.equity - config.initial_capital) / config.initial_capital;

        let annualized_return = if state.equity_curve.len() > 1 {
            let days = (state.equity_curve.last().unwrap().0
                - state.equity_curve.first().unwrap().0) as f64 / (24.0 * 3600.0 * 1e9);
            if days > 0.0 {
                (1.0 + total_return).powf(365.0 / days) - 1.0
            } else {
                0.0
            }
        } else {
            0.0
        };

        let (max_drawdown, max_drawdown_duration) = Self::calculate_max_drawdown(&state.equity_curve);

        let sharpe_ratio = Self::calculate_sharpe_ratio(&state.equity_curve, 0.02);

        let sortino_ratio = Self::calculate_sortino_ratio(&state.equity_curve, 0.02);

        let win_rate = Self::calculate_win_rate(&state.trades);

        let profit_factor = Self::calculate_profit_factor(&state.trades);

        BacktestMetrics {
            initial_capital: config.initial_capital,
            final_equity: state.equity,
            total_return,
            annualized_return,
            max_drawdown,
            max_drawdown_duration,
            sharpe_ratio,
            sortino_ratio,
            win_rate,
            profit_factor,
            total_trades: state.trades.len(),
            avg_trade_duration: Self::calculate_avg_trade_duration(&state.trades),
        }
    }

    /// 计算最大回撤
    fn calculate_max_drawdown(equity_curve: &[(i64, f64)]) -> (f64, i64) {
        let mut peak = 0.0;
        let mut max_dd = 0.0;
        let mut max_dd_duration = 0i64;
        let mut drawdown_start = 0i64;

        for (i, (_, equity)) in equity_curve.iter().enumerate() {
            if *equity > peak {
                peak = *equity;
                drawdown_start = equity_curve[i].0;
            } else {
                let dd = (peak - *equity) / peak;
                if dd > max_dd {
                    max_dd = dd;
                    max_dd_duration = equity_curve[i].0 - drawdown_start;
                }
            }
        }

        (max_dd, max_dd_duration)
    }

    /// 计算 Sharpe 比率
    fn calculate_sharpe_ratio(equity_curve: &[(i64, f64)], risk_free_rate: f64) -> f64 {
        if equity_curve.len() < 2 {
            return 0.0;
        }

        let returns: Vec<f64> = equity_curve.windows(2)
            .map(|w| (w[1].1 - w[0].1) / w[0].1)
            .collect();

        let mean_return: f64 = returns.iter().sum::<f64>() / returns.len() as f64;
        let std_return: f64 = {
            let variance: f64 = returns.iter()
                .map(|r| (r - mean_return).powi(2))
                .sum::<f64>() / returns.len() as f64;
            variance.sqrt()
        };

        if std_return == 0.0 {
            0.0
        } else {
            (mean_return - risk_free_rate) / std_return
        }
    }

    /// 计算 Sortino 比率（使用下行波动率）
    fn calculate_sortino_ratio(equity_curve: &[(i64, f64)], risk_free_rate: f64) -> f64 {
        if equity_curve.len() < 2 {
            return 0.0;
        }

        let returns: Vec<f64> = equity_curve.windows(2)
            .map(|w| (w[1].1 - w[0].1) / w[0].1)
            .collect();

        let mean_return: f64 = returns.iter().sum::<f64>() / returns.len() as f64;
        let downside_std: f64 = {
            let negative_returns: Vec<f64> = returns.iter()
                .filter(|r| **r < risk_free_rate)
                .map(|r| (r - risk_free_rate).powi(2))
                .collect();

            if negative_returns.is_empty() {
                0.0
            } else {
                let variance: f64 = negative_returns.iter().sum::<f64>() / negative_returns.len() as f64;
                variance.sqrt()
            }
        };

        if downside_std == 0.0 {
            0.0
        } else {
            (mean_return - risk_free_rate) / downside_std
        }
    }

    /// 计算胜率
    fn calculate_win_rate(trades: &[Trade]) -> f64 {
        if trades.is_empty() {
            0.0
        } else {
            let wins = trades.iter().filter(|t| t.pnl > 0.0).count();
            wins as f64 / trades.len() as f64
        }
    }

    /// 计算盈亏比
    fn calculate_profit_factor(trades: &[Trade]) -> f64 {
        let gross_profit: f64 = trades.iter().filter(|t| t.pnl > 0.0).map(|t| t.pnl).sum();
        let gross_loss: f64 = trades.iter().filter(|t| t.pnl < 0.0).map(|t| t.pnl.abs()).sum();

        if gross_loss == 0.0 {
            if gross_profit > 0.0 { f64::INFINITY } else { 0.0 }
        } else {
            gross_profit / gross_loss
        }
    }

    /// 计算平均持仓时间
    fn calculate_avg_trade_duration(trades: &[Trade]) -> f64 {
        if trades.len() < 2 {
            0.0
        } else {
            let total_duration: i64 = trades.windows(2)
                .map(|w| w[1].ts - w[0].ts)
                .sum();
            total_duration as f64 / 1e9 // 转换为秒
        }
    }
}

/// 回测性能指标
#[derive(Debug, Clone)]
pub struct BacktestMetrics {
    /// 初始资金
    pub initial_capital: f64,
    /// 最终权益
    pub final_equity: f64,
    /// 总收益率
    pub total_return: f64,
    /// 年化收益率
    pub annualized_return: f64,
    /// 最大回撤
    pub max_drawdown: f64,
    /// 最大回撤持续时间（纳秒）
    pub max_drawdown_duration: i64,
    /// Sharpe 比率
    pub sharpe_ratio: f64,
    /// Sortino 比率
    pub sortino_ratio: f64,
    /// 胜率
    pub win_rate: f64,
    /// 盈亏比
    pub profit_factor: f64,
    /// 总交易次数
    pub total_trades: usize,
    /// 平均持仓时间（秒）
    pub avg_trade_duration: f64,
}
```

---

## 6. 回测报告

### 6.1 报告结构

```rust
/// 回测报告
#[derive(Debug, Clone)]
pub struct BacktestReport {
    /// 回测配置
    pub config: BacktestConfig,

    /// 性能指标
    pub metrics: BacktestMetrics,

    /// 权益曲线
    pub equity_curve: Vec<(i64, f64)>,

    /// 所有交易
    pub trades: Vec<Trade>,
}

impl BacktestReport {
    /// 打印到控制台
    pub fn print(&self) {
        println!("═══════════════════════════════════════════════════");
        println!("                    Backtest Report                  ");
        println!("═══════════════════════════════════════════════════");
        println!();
        println!("Initial Capital:   ${:,.2}", self.metrics.initial_capital);
        println!("Final Equity:     ${:,.2}", self.metrics.final_equity);
        println!("Total Return:     {:.2}%", self.metrics.total_return * 100.0);
        println!("Annual Return:    {:.2}%", self.metrics.annualized_return * 100.0);
        println!();
        println!("Max Drawdown:      {:.2}%", self.metrics.max_drawdown * 100.0);
        println!("Sharpe Ratio:     {:.2}", self.metrics.sharpe_ratio);
        println!("Sortino Ratio:    {:.2}", self.metrics.sortino_ratio);
        println!();
        println!("Win Rate:         {:.2}%", self.metrics.win_rate * 100.0);
        println!("Profit Factor:    {:.2}", self.metrics.profit_factor);
        println!("Total Trades:     {}", self.metrics.total_trades);
        println!("Avg Hold Time:    {:.2}s", self.metrics.avg_trade_duration);
        println!("═══════════════════════════════════════════════════");
    }

    /// 保存为 JSON
    pub fn save_json(&self, path: impl AsRef<Path>) -> Result<()> {
        let json = serde_json::to_string_pretty(self)?;
        std::fs::write(path, json)?;
        Ok(())
    }

    /// 生成 HTML 报告
    pub fn generate_html(&self) -> String {
        let mut html = String::new();
        html.push_str("<!DOCTYPE html>\n<html>\n<head>\n");
        html.push_str("<meta charset='utf-8'>\n");
        html.push_str("<title>Backtest Report</title>\n");
        // 添加 Chart.js 和样式
        html.push_str("<script src='https://cdn.jsdelivr.net/npm/chart.js'></script>\n");
        html.push_str("<style>\n");
        html.push_str("body { font-family: Arial, sans-serif; margin: 20px; }\n");
        html.push_str(".metric { display: inline-block; width: 200px; margin: 10px; }\n");
        html.push_str(".metric-value { font-size: 24px; font-weight: bold; }\n");
        html.push_str(".metric-label { font-size: 14px; color: #666; }\n");
        html.push_str(".positive { color: #00a000; }\n");
        html.push_str(".negative { color: #a00000; }\n");
        html.push_str("</style>\n</head>\n<body>\n");

        html.push_str("<h1>Backtest Report</h1>\n");

        // 指标卡片
        html.push_str("<div class='metrics'>\n");
        self.add_metric_html(&mut html, "Initial Capital", self.metrics.initial_capital, "$", None);
        self.add_metric_html(&mut html, "Final Equity", self.metrics.final_equity, "$", None);
        self.add_metric_html(&mut html, "Total Return", self.metrics.total_return, "%", Some(100.0));
        self.add_metric_html(&mut html, "Annual Return", self.metrics.annualized_return, "%", Some(100.0));
        self.add_metric_html(&mut html, "Max Drawdown", self.metrics.max_drawdown, "%", Some(100.0));
        self.add_metric_html(&mut html, "Sharpe Ratio", self.metrics.sharpe_ratio, "", None);
        self.add_metric_html(&mut html, "Win Rate", self.metrics.win_rate, "%", Some(100.0));
        html.push_str("</div>\n");

        // 权益曲线图表
        html.push_str("<canvas id='equityChart'></canvas>\n");
        html.push_str("<script>\n");
        html.push_str("const ctx = document.getElementById('equityChart').getContext('2d');\n");
        html.push_str("new Chart(ctx, {\n");
        html.push_str("  type: 'line',\n");
        html.push_str("  data: {\n");
        html.push_str("    labels: [");
        for (i, _) in self.equity_curve.iter().enumerate().step_by(100) {
            html.push_str(&format!("'{}',", i));
        }
        html.push_str("],\n");
        html.push_str("    datasets: [{\n");
        html.push_str("      label: 'Equity',\n");
        html.push_str("      data: [");
        for (i, (_, equity)) in self.equity_curve.iter().enumerate().step_by(100) {
            html.push_str(&format!("{},", equity));
        }
        html.push_str("],\n");
        html.push_str("      borderColor: '#4a90e2',\n");
        html.push_str("      tension: 0.1\n");
        html.push_str("    }]\n");
        html.push_str("  }\n");
        html.push_str("});\n");
        html.push_str("</script>\n");

        html.push_str("</body>\n</html>\n");

        html
    }

    fn add_metric_html(&self, html: &mut String, label: &str, value: f64, suffix: &str, scale: Option<f64>) {
        let scaled_value = scale.map_or(value, |s| value * s);
        let class = if value >= 0.0 { "positive" } else { "negative" };

        html.push_str(&format!(
            "<div class='metric'>\n\
              <div class='metric-value {}'>{:.2}{}</div>\n\
              <div class='metric-label'>{}</div>\n\
             </div>\n",
            class, scaled_value, suffix, label
        ));
    }

    /// 保存为 HTML
    pub fn save_html(&self, path: impl AsRef<Path>) -> Result<()> {
        let html = self.generate_html();
        std::fs::write(path, html)?;
        Ok(())
    }
}

/// 交易记录
#[derive(Debug, Clone)]
pub struct Trade {
    pub ts: i64,
    pub side: OrderSide,
    pub price: Price,
    pub qty: Quantity,
    pub pnl: f64,
}
```

---

## 7. 模块导出

```rust
// src/backtest/mod.rs

pub mod engine;
pub mod replay;
pub mod execution;
pub mod metrics;
pub mod report;

pub use engine::{BacktestEngine, BacktestConfig, BacktestState};
pub use replay::{DataReplay, DataSource};
pub use execution::SimulatedExecution;
pub use metrics::{BacktestMetrics, MetricsCalculator};
pub use report::{BacktestReport, Trade};
```

---

## 8. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| 回测速度 | > 10M ticks/s | 批量回测测试 |
| 内存占用 | < 500MB | 回测 1 年数据 |
| 报告生成 | < 1s | HTML 报告生成 |

---

## 9. 未来扩展

- **多策略组合回测**: 多个策略同时回测
- **参数优化**: 遗传算法/贝叶斯优化
- **Walk-Forward 分析**: 滚动窗口验证
- **蒙特卡洛模拟**: 随机扰动测试鲁棒性
