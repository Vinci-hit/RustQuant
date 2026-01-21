# 模块详细设计文档索引

本目录包含系统中每个模块的详细设计文档。

## 文档列表

| 序号 | 文档 | 描述 |
|-----|------|------|
| 01 | [核心模块设计](./01-core-design.md) | Tick、Event、Error 核心数据结构 |
| 02 | [数据模块设计](./02-data-design.md) | Parquet 存储、DataFusion 查询、行情源 |
| 03 | [策略模块设计](./03-strategy-design.md) | 策略 Trait、策略引擎、策略注册 |
| 04 | [执行模块设计](./04-execution-design.md) | 订单管理、风控、成交回报 |
| 05 | [存储模块设计](./05-storage-design.md) | 写入器、读取器、数据分区 |
| 06 | [回测模块设计](./06-backtest-design.md) | 回测引擎、事件回放、报告生成 |
| 07 | [监控模块设计](./07-monitor-design.md) | 指标采集、tracing、告警 |
| 08 | [工具模块设计](./08-utils-design.md) | CPU 亲和性、SPSC 队列、时间工具 |

---

## 模块依赖关系图

```
                    ┌──────────────┐
                    │     core     │
                    │  (无依赖)    │
                    └──────┬───────┘
                           │
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
┌──────────┐        ┌──────────┐         ┌──────────┐
│   data   │        │ strategy │         │ storage  │
└────┬─────┘        └────┬─────┘         └────┬─────┘
     │                   │                    │
     └───────────┬────────┘                    │
                 ▼                             │
          ┌──────────┐                        │
          │ execution│                        │
          └────┬─────┘                        │
               │                             │
               └──────────┬──────────────────┘
                          ▼
                   ┌──────────┐
                   │  monitor │
                   └──────────┘

                 (utils 被所有模块依赖)
```

---

## 服务边界定义

### 核心模块 (core)
- **职责**: 定义系统的基础数据类型和错误类型
- **边界**: 不依赖任何其他业务模块
- **导出**: `Tick`, `MarketEvent`, `SystemEvent`, `Error`

### 数据模块 (data)
- **职责**: 负责行情数据的获取、查询、存储接口
- **边界**: 依赖 core，被 strategy、backtest、execution 依赖
- **导出**: `Feed`, `QueryEngine`, `MarketDataClient`

### 策略模块 (strategy)
- **职责**: 策略的抽象定义、执行引擎、策略注册
- **边界**: 依赖 core、data，被 execution、backtest 依赖
- **导出**: `Strategy`, `StrategyEngine`, `Signal`

### 执行模块 (execution)
- **职责**: 订单管理、风控、与交易所交互
- **边界**: 依赖 core、strategy、data
- **导出**: `OrderManager`, `RiskEngine`, `ExecutionClient`

### 存储模块 (storage)
- **职责**: 高性能 Parquet 写入、数据分区管理
- **边界**: 依赖 core，被 data 使用
- **导出**: `TickWriter`, `DataPartition`

### 回测模块 (backtest)
- **职责**: 历史数据回测、性能指标计算、报告生成
- **边界**: 依赖 core、data、strategy、execution
- **导出**: `BacktestEngine`, `BacktestReport`

### 监控模块 (monitor)
- **职责**: 指标采集、日志、性能分析
- **边界**: 依赖所有模块（可选）
- **导出**: `MetricsCollector`, `TracingConfig`

### 工具模块 (utils)
- **职责**: CPU 亲和性、高性能队列、时间工具
- **边界**: 无业务依赖，被所有模块共享
- **导出**: `CorePinner`, `SPSCQueue`, `TimeUtils`

---

## 消息流设计

### 实盘模式消息流
```
WebSocket Feed
    │
    ├─▶ Tick
    │    │
    │    ├─▶ Storage (持久化)
    │    │
    │    └─▶ Strategy Engine
    │         │
    │         ├─▶ Signal
    │         │    │
    │         │    └─▶ OrderManager
    │         │         │
    │         │         ├─▶ ExecutionClient (交易所)
    │         │         │
    │         │         └─▶ RiskEngine (风控检查)
    │         │
    │         └─▶ Fill (成交回报)
    │
    └─▶ Monitor (指标采集)
```

### 回测模式消息流
```
Data Source (Parquet)
    │
    └─▶ Tick Stream
         │
         └─▶ BacktestEngine
              │
              ├─▶ Strategy
              │    │
              │    └─▶ Signal
              │         │
              │         └─▶ SimulatedExecution (模拟成交)
              │              │
              │              └─▶ Fill
              │
              └─▶ ReportGenerator (性能指标)
```

---

## 线程模型

```
┌───────────────────────────────────────────────────────────────────┐
│                         Main Thread                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Tokio Runtime            │             │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
│                                                                   │
│  spawn_blocking:                     Dedicated Threads:            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Core 0      │  │  Core 2      │  │  Core N      │             │
│  │  WS Feed     │  │  Strategy-1  │  │  Strategy-M  │             │
│  │  (tokio)     │  │  (Actor)     │  │  (Actor)     │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Core 1      │  │  Core 3      │  │  Last Core   │             │
│  │  REST API    │  │  Strategy-2  │  │  Storage     │             │
│  │  (axum)      │  │  (Actor)     │  │  Monitor     │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└───────────────────────────────────────────────────────────────────┘
```

---

## 数据流边界

### 输入边界
- **WebSocket**: 实时行情 Tick
- **REST API**: 控制命令、查询请求
- **Parquet**: 历史数据回放

### 内部边界
- **Event Bus**: 所有跨模块消息统一通过事件总线传递
- **SPSC Queues**: 高频路径使用无锁队列
- **Shared Memory**: 策略内部状态独立，不共享

### 输出边界
- **ExecutionClient**: 订单请求发送到交易所
- **Storage**: Tick 写入 Parquet
- **Monitor**: 指标输出到日志/Prometheus

---

## 配置边界

配置系统按模块划分，每个模块有独立的配置结构：

```toml
[core]
time_source = "system"  # 或 "monotonic"
tick_precision = "nanosecond"

[data]
feed_type = "websocket"  # 或 "rest"
ws_url = "wss://stream.binance.com:9443"
symbols = ["BTCUSDT", "ETHUSDT"]
retry_delay_ms = 1000

[storage]
path = "./data"
compression = "zstd"
partition_by = "day"
flush_interval_ms = 1000
batch_size = 10000

[strategy]
enabled = ["ma_cross", "mean_reversion"]
threads = 4

[execution]
exchange = "binance"
api_key = "${API_KEY}"
api_secret = "${API_SECRET}"
testnet = false

[risk]
max_position = 1.0
max_order_value = 10000.0
daily_loss_limit = 1000.0

[monitor]
metrics_port = 9090
log_level = "info"
log_path = "./logs"
```
