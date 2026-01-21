# 单机顶配量化交易系统 - 设计文档索引

本文档目录包含系统各模块的详细设计文档。

## 文档导航

### [总体设计文档 (README.md)](./README.md)
系统的整体架构、技术选型和开发路线图

### [模块概览 (00-design-overview.md)](./00-design-overview.md)
模块服务边界、消息流设计、线程模型

---

## 模块设计文档

| 序号 | 模块 | 文档 | 描述 |
|:----:|------|------|------|
| 01 | [core](./01-core-design.md) | 核心数据结构 | Tick、Event、Error、基础类型定义 |
| 02 | [data](./02-data-design.md) | 数据模块 | WebSocket/REST 行情源、DataFusion 查询引擎 |
| 03 | [strategy](./03-strategy-design.md) | 策略模块 | Strategy Trait、策略引擎、示例策略 |
| 04 | [execution](./04-execution-design.md) | 执行模块 | 订单管理(OMS)、风控、交易所集成 |
| 05 | [storage](./05-storage-design.md) | 存储模块 | Parquet 读写、分区管理、内存映射 |
| 06 | [backtest](./06-backtest-design.md) | 回测模块 | 历史数据回放、模拟执行、性能指标 |
| 07 | [monitor](./07-monitor-design.md) | 监控模块 | 指标采集、tracing、健康检查、告警 |
| 08 | [utils](./08-utils-design.md) | 工具模块 | CPU 亲和性、SPSC 队列、时间工具 |
| 09 | [ai_agent](./09-ai-agent-design.md) | AI 集成 | 本地 AI Agent 通信、信号融合 |

---

## 模块依赖关系图

```
                 ┌─────────┐
                 │  core   │  (无依赖)
                 └────┬────┘
                      │
      ┌───────────────┼───────────────┐
      ▼               ▼               ▼
 ┌─────────┐     ┌─────────┐     ┌─────────┐
 │  data   │     │strategy │     │ storage │
 └────┬────┘     └────┬────┘     └────┬────┘
      │               │               │
      └───────┬───────┴───────┬─────┘
              │               │
              ▼               ▼
        ┌───────────┐   ┌───────────┐
        │execution  │   │ backtest  │
        └─────┬─────┘   └─────┬─────┘
              │               │
              └───────┬───────┘
                      ▼
                ┌───────────┐
                │  monitor  │
                └───────────┘

              (utils 被所有模块依赖)
              (ai_agent 依赖 core, strategy, data)
```

---

## 服务边界

### 核心模块 (core)
- **职责**: 系统基础数据类型和错误定义
- **边界**: 无外部依赖
- **导出**: `Tick`, `Event`, `Error`, `Symbol`, `Price`, `Quantity`

### 数据模块 (data)
- **职责**: 行情数据获取和查询
- **边界**: 依赖 core，被 strategy、backtest、execution 依赖
- **导出**: `Feed`, `QueryEngine`, `BinanceWebSocketFeed`, `TickCache`

### 策略模块 (strategy)
- **职责**: 策略抽象和执行
- **边界**: 依赖 core、data，被 execution、backtest 依赖
- **导出**: `Strategy`, `StrategyEngine`, `Signal`, `StrategyRegistry`

### 执行模块 (execution)
- **职责**: 订单管理和风控
- **边界**: 依赖 core、strategy、data
- **导出**: `OrderManager`, `RiskEngine`, `ExchangeClient`

### 存储模块 (storage)
- **职责**: 高性能 Parquet 存储和读取
- **边界**: 依赖 core
- **导出**: `TickWriter`, `MmapTickReader`, `PartitionManager`

### 回测模块 (backtest)
- **职责**: 历史数据回测和性能分析
- **边界**: 依赖 core、data、strategy、execution
- **导出**: `BacktestEngine`, `BacktestReport`, `MetricsCalculator`

### 监控模块 (monitor)
- **职责**: 指标采集和系统监控
- **边界**: 依赖所有模块（可选）
- **导出**: `MetricsRegistry`, `HealthChecker`, `AlertEngine`

### 工具模块 (utils)
- **职责**: CPU 亲和性、高性能队列、时间工具
- **边界**: 无业务依赖，被所有模块共享
- **导出**: `CorePinner`, `SpscQueue`, `PrecisionTimer`

### AI 集成模块 (ai_agent)
- **职责**: 本地 AI Agent 通信、信号生成与融合
- **边界**: 依赖 core、strategy、data
- **导出**: `AiClient`, `AiMarketAnalyzer`, `SignalFusion`

---

## 线程分配策略

| 核心范围 | 职责 | 调度模式 | 对应模块 |
|:---------:|------|:---------:|---------|
| Core 0 | WebSocket 行情接收 | current_thread | data::websocket |
| Core 1 | REST API 服务 | current_thread | (future) |
| Core 2-N | 策略执行 | dedicated threads | strategy::engine |
| Last Core | 存储 + 监控 | dedicated thread | storage, monitor |

---

## 消息流

### 实盘模式
```
WebSocket Feed → Tick → [事件总线]
                              ↓
                          策略引擎 → 信号
                              ↓
                          订单管理 → 交易所
                              ↓
                          成交回报 → 策略/风控
                              ↓
                          Parquet 写入 (持久化)
```

### 回测模式
```
Parquet → DataReplay → Tick → 回测引擎
                              ↓
                          策略执行 → 信号
                              ↓
                          模拟执行 → 成交
                              ↓
                          性能指标计算
                              ↓
                          报告生成
```

### AI Agent 集成
```
市场数据 + 技术指标 → Prompt 构建
                              ↓
                          AI Client (Ollama/vLLM)
                              ↓
                          AI 响应解析 → 信号
                              ↓
                          信号融合 (AI + 规则)
                              ↓
                          最终决策
```

---

## 性能目标

| 维度 | 目标 | 实现方式 |
|-----|------|---------|
| 吞吐量 | > 10万 ticks/s | 零拷贝、SPSC 队列 |
| 延迟 | P99 < 1ms | 核心绑定、无锁队列 |
| 回测速度 | > 10M ticks/s | Polars 向量化、Arrow 内存格式 |
| 内存占用 | < 2GB | 零拷贝、高效内存管理 |
| 压缩比 | > 10:1 | Parquet + ZSTD |

---

## 技术栈总结

| 组件 | 选型 | 理由 |
|-----|------|------|
| 语言 | Rust | 零 GC、内存安全 |
| 运行时 | Tokio (current_thread) | 精细控制异步 |
| 数据存储 | Parquet + DataFusion | 列式、向量化查询 |
| 计算引擎 | Polars | SIMD 加速 |
| 序列化 | rkyv/zerocopy | 零拷贝 |
| 消息传递 | SPSC / Disruptor | 无锁队列 |
| 时间序列 | Arrow | 零拷贝内存格式 |
| AI 集成 | Ollama/vLLM | 本地部署、可定制 |

---

## 开发进度

- [x] 设计文档
- [ ] 核心模块实现
- [ ] 数据模块实现
- [ ] 策略模块实现
- [ ] 执行模块实现
- [ ] 存储模块实现
- [ ] 回测模块实现
- [ ] 监控模块实现
- [ ] 工具模块实现
- [ ] AI Agent 集成
- [ ] 集成测试
- [ ] 性能测试

---

## 快速开始

```bash
# 克隆项目
git clone <repo>

# 查看设计文档
cd quant/docs
ls -la

# 构建项目
cd ..
cargo build --release

# 运行回测
cargo run --release --bin backtest

# 启动实盘
cargo run --release --bin live
```
