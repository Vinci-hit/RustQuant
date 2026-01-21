# 监控模块设计文档 (monitor)

## 1. 模块概述

### 1.1 职责
- 系统指标采集和暴露
- 分布式追踪
- 日志管理
- 健康检查和告警

### 1.2 依赖关系
- **依赖**: `core` (Error), 可选地依赖所有其他模块
- **被依赖**: 所有模块（通过宏或全局状态）

### 1.3 文件结构
```
src/monitor/
├── mod.rs           # 模块导出
├── metrics.rs       # 指标采集
├── trace.rs         # tracing 配置
├── health.rs        # 健康检查
├── alert.rs         # 告警系统
└── profiling.rs     # 性能分析
```

---

## 2. 指标采集系统

### 2.1 指标类型

```rust
use std::sync::atomic::{AtomicU64, AtomicF64, Ordering};
use std::sync::Arc;
use std::collections::HashMap;

/// 指标类型
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum MetricType {
    /// 计数器（单调递增）
    Counter,
    /// 仪表盘（可增减）
    Gauge,
    /// 直方图（分布）
    Histogram,
    /// 摘要（统计）
    Summary,
}

/// 指标
pub enum Metric {
    /// 计数器
    Counter(Arc<AtomicU64>),

    /// 仪表盘
    Gauge(Arc<AtomicF64>),

    /// 直方图
    Histogram(Arc<Histogram>),

    /// 摘要
    Summary(Arc<Summary>),
}

impl Metric {
    /// 创建计数器
    pub fn counter() -> Self {
        Self::Counter(Arc::new(AtomicU64::new(0)))
    }

    /// 创建仪表盘
    pub fn gauge() -> Self {
        Self::Gauge(Arc::new(AtomicF64::new(0.0)))
    }

    /// 创建直方图
    pub fn histogram(buckets: Vec<f64>) -> Self {
        Self::Histogram(Arc::new(Histogram::new(buckets)))
    }

    /// 创建摘要
    pub fn summary(quantiles: Vec<f64>) -> Self {
        Self::Summary(Arc::new(Summary::new(quantiles)))
    }

    /// 增加计数
    pub fn inc(&self) {
        if let Self::Counter(c) = self {
            c.fetch_add(1, Ordering::Relaxed);
        }
    }

    /// 增加指定值
    pub fn inc_by(&self, value: u64) {
        if let Self::Counter(c) = self {
            c.fetch_add(value, Ordering::Relaxed);
        }
    }

    /// 设置仪表盘值
    pub fn set(&self, value: f64) {
        if let Self::Gauge(g) = self {
            g.store(value, Ordering::Relaxed);
        }
    }

    /// 记录观测值（直方图/摘要）
    pub fn observe(&self, value: f64) {
        match self {
            Self::Histogram(h) => h.observe(value),
            Self::Summary(s) => s.observe(value),
            _ => {}
        }
    }
}

/// 直方图
pub struct Histogram {
    buckets: Vec<f64>,
    counts: Vec<AtomicU64>,
    sum: AtomicF64,
    count: AtomicU64,
}

impl Histogram {
    fn new(buckets: Vec<f64>) -> Self {
        let counts = (0..=buckets.len())
            .map(|_| AtomicU64::new(0))
            .collect();

        Self {
            buckets,
            counts,
            sum: AtomicF64::new(0.0),
            count: AtomicU64::new(0),
        }
    }

    fn observe(&self, value: f64) {
        self.sum.fetch_add(value, Ordering::Relaxed);
        self.count.fetch_add(1, Ordering::Relaxed);

        // 找到对应的 bucket
        for (i, &bucket) in self.buckets.iter().enumerate() {
            if value <= bucket {
                self.counts[i].fetch_add(1, Ordering::Relaxed);
                return;
            }
        }
        // 超过所有 bucket
        self.counts.last().unwrap().fetch_add(1, Ordering::Relaxed);
    }

    /// 获取 bucket 计数
    pub fn buckets(&self) -> &Vec<f64> {
        &self.buckets
    }

    /// 获取每个 bucket 的计数
    pub fn counts(&self) -> Vec<u64> {
        self.counts.iter().map(|c| c.load(Ordering::Relaxed)).collect()
    }

    /// 获取总和
    pub fn sum(&self) -> f64 {
        self.sum.load(Ordering::Relaxed)
    }

    /// 获取观测次数
    pub fn count(&self) -> u64 {
        self.count.load(Ordering::Relaxed)
    }
}

/// 摘要
pub struct Summary {
    quantiles: Vec<f64>,
    values: parking_lot::Mutex<Vec<f64>>,
    max_age: usize,
}

impl Summary {
    fn new(quantiles: Vec<f64>) -> Self {
        Self {
            quantiles,
            values: parking_lot::Mutex::new(Vec::new()),
            max_age: 10000,
        }
    }

    fn observe(&self, value: f64) {
        let mut values = self.values.lock();
        values.push(value);

        // 限制大小
        if values.len() > self.max_age {
            values.drain(0..values.len() / 2);
        }
    }

    /// 获取分位数
    pub fn quantile(&self, q: f64) -> Option<f64> {
        let values = self.values.lock();
        if values.is_empty() {
            return None;
        }

        let index = ((q.clamp(0.0, 1.0) * (values.len() - 1) as f64) as usize).min(values.len() - 1);
        Some(values[index])
    }

    /// 获取所有分位数
    pub fn quantiles(&self) -> Vec<(f64, Option<f64>)> {
        self.quantiles.iter().map(|&q| (q, self.quantile(q))).collect()
    }
}
```

### 2.2 指标注册表

```rust
/// 指标注册表
///
/// 全局单例，管理所有指标
pub struct MetricsRegistry {
    metrics: parking_lot::RwLock<HashMap<String, MetricInfo>>,

    /// 指标标签
    labels: parking_lot::RwLock<Vec<String>>,
}

#[derive(Debug, Clone)]
pub struct MetricInfo {
    pub name: String,
    pub metric_type: MetricType,
    pub help: String,
    pub metric: Metric,
}

impl MetricsRegistry {
    /// 获取全局注册表
    pub fn global() -> &'static Self {
        use std::sync::OnceLock;
        static REGISTRY: OnceLock<MetricsRegistry> = OnceLock::new();
        REGISTRY.get_or_init(|| {
            Self {
                metrics: parking_lot::RwLock::new(HashMap::new()),
                labels: parking_lot::RwLock::new(Vec::new()),
            }
        })
    }

    /// 注册指标
    pub fn register(&self, name: String, metric_type: MetricType, help: String) -> Arc<Metric> {
        let metric = match metric_type {
            MetricType::Counter => Arc::new(Metric::counter()),
            MetricType::Gauge => Arc::new(Metric::gauge()),
            MetricType::Histogram => {
                Arc::new(Metric::histogram(vec![
                    0.001, 0.005, 0.01, 0.025, 0.05,
                    0.1, 0.25, 0.5, 1.0, 2.5, 5.0,
                    10.0, 25.0, 50.0, 100.0,
                ]))
            }
            MetricType::Summary => {
                Arc::new(Metric::summary(vec![0.5, 0.9, 0.95, 0.99]))
            }
        };

        self.metrics.write().insert(name.clone(), MetricInfo {
            name,
            metric_type,
            help,
            metric: (*metric).clone(),
        });

        metric
    }

    /// 获取指标
    pub fn get(&self, name: &str) -> Option<Arc<Metric>> {
        self.metrics.read().get(name).map(|info| Arc::clone(&Arc::new(info.metric.clone())))
    }

    /// 导出为 Prometheus 格式
    pub fn export_prometheus(&self) -> String {
        let mut output = String::new();

        for info in self.metrics.read().values() {
            output.push_str(&format!("# HELP {} {}\n", info.name, info.help));
            output.push_str(&format!("# TYPE {} {:?}\n", info.name, info.metric_type));

            match &info.metric {
                Metric::Counter(c) => {
                    output.push_str(&format!("{} {}\n", info.name, c.load(Ordering::Relaxed)));
                }
                Metric::Gauge(g) => {
                    output.push_str(&format!("{} {}\n", info.name, g.load(Ordering::Relaxed)));
                }
                Metric::Histogram(h) => {
                    output.push_str(&format!("{}_sum {}\n", info.name, h.sum()));
                    output.push_str(&format!("{}_count {}\n", info.name, h.count()));

                    let counts = h.counts();
                    for (i, (&bucket, &count)) in h.buckets().iter().zip(counts.iter()).enumerate() {
                        output.push_str(&format!("{}_bucket{{le=\"{}\"}} {}\n", info.name, bucket, count));
                    }
                    output.push_str(&format!("{}_bucket{{le=\"+Inf\"}} {}\n", info.name, h.count()));
                }
                Metric::Summary(s) => {
                    output.push_str(&format!("{}_sum {}\n", info.name, 0.0)); // TODO
                    output.push_str(&format!("{}_count {}\n", info.name, 0.0)); // TODO

                    for (q, v) in s.quantiles() {
                        if let Some(v) = v {
                            output.push_str(&format!("{}_quantile{{quantile=\"{}\"}} {}\n", info.name, q, v));
                        }
                    }
                }
            }

            output.push('\n');
        }

        output
    }
}

/// 便捷宏：创建计数器
#[macro_export]
macro_rules! counter {
    ($name:expr, $help:expr) => {
        $crate::monitor::MetricsRegistry::global()
            .register($name.to_string(), MetricType::Counter, $help.to_string())
    };
}

/// 便捷宏：创建仪表盘
#[macro_export]
macro_rules! gauge {
    ($name:expr, $help:expr) => {
        $crate::monitor::MetricsRegistry::global()
            .register($name.to_string(), MetricType::Gauge, $help.to_string())
    };
}

/// 便捷宏：创建直方图
#[macro_export]
macro_rules! histogram {
    ($name:expr, $help:expr) => {
        $crate::monitor::MetricsRegistry::global()
            .register($name.to_string(), MetricType::Histogram, $help.to_string())
    };
}
```

---

## 3. Tracing 配置

### 3.1 日志系统

```rust
use tracing_subscriber::{
    fmt, prelude::*, EnvFilter, Registry,
};
use tracing_appender::{non_blocking, rolling};

/// Tracing 配置
#[derive(Debug, Clone)]
pub struct TracingConfig {
    /// 日志级别
    pub level: String,

    /// 日志文件路径
    pub log_path: Option<String>,

    /// 是否启用 JSON 格式
    pub json_format: bool,

    /// 是否启用 ANSI 颜色
    pub ansi: bool,
}

impl Default for TracingConfig {
    fn default() -> Self {
        Self {
            level: "info".to_string(),
            log_path: Some("./logs".to_string()),
            json_format: false,
            ansi: true,
        }
    }
}

impl TracingConfig {
    /// 初始化 tracing
    pub fn init(&self) -> Result<()> {
        let env_filter = EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new(&self.level));

        let fmt_layer = if self.json_format {
            fmt::layer().json().with_ansi(self.ansi).boxed()
        } else {
            fmt::layer().pretty().with_ansi(self.ansi).boxed()
        };

        let registry = Registry::default().with(env_filter);

        if let Some(log_path) = &self.log_path {
            let file_appender = rolling::daily(log_path, "quant.log");
            let (non_blocking, _guard) = non_blocking(file_appender);

            registry
                .with(fmt_layer)
                .with(fmt::layer()
                    .with_writer(non_blocking)
                    .json()
                    .boxed())
                .init();
        } else {
            registry.with(fmt_layer).init();
        }

        Ok(())
    }
}

/// 嵌入式 span，用于跟踪关键路径
#[macro_export]
macro_rules! timed {
    ($name:expr, $block:block) => {
        {
            let span = tracing::span!(tracing::Level::INFO, $name);
            let _enter = span.enter();
            let result = $block;
            result
        }
    };
}
```

---

## 4. 健康检查

### 4.1 Health Check

```rust
/// 健康状态
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

/// 健康检查项
#[derive(Debug, Clone)]
pub struct HealthCheck {
    name: String,
    status: Arc<AtomicHealthStatus>,
    message: Arc<RwLock<String>>,
    last_check: Arc<AtomicI64>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct AtomicHealthStatus(u8);

impl AtomicHealthStatus {
    const HEALTHY: u8 = 0;
    const DEGRADED: u8 = 1;
    const UNHEALTHY: u8 = 2;

    fn new(status: HealthStatus) -> Self {
        Self(match status {
            HealthStatus::Healthy => Self::HEALTHY,
            HealthStatus::Degraded => Self::DEGRADED,
            HealthStatus::Unhealthy => Self::UNHEALTHY,
        })
    }

    fn get(&self) -> HealthStatus {
        match self.0 {
            Self::HEALTHY => HealthStatus::Healthy,
            Self::DEGRADED => HealthStatus::Degraded,
            Self::UNHEALTHY => HealthStatus::Unhealthy,
            _ => HealthStatus::Unhealthy,
        }
    }

    fn set(&self, status: HealthStatus) {
        self.0.store(match status {
            HealthStatus::Healthy => Self::HEALTHY,
            HealthStatus::Degraded => Self::DEGRADED,
            HealthStatus::Unhealthy => Self::UNHEALTHY,
        }, Ordering::SeqCst);
    }
}

impl HealthCheck {
    /// 创建新的健康检查项
    pub fn new(name: String) -> Self {
        Self {
            name,
            status: Arc::new(AtomicHealthStatus::new(HealthStatus::Healthy)),
            message: Arc::new(RwLock::new(String::new())),
            last_check: Arc::new(AtomicI64::new(0)),
        }
    }

    /// 更新状态
    pub fn update(&self, status: HealthStatus, message: String) {
        self.status.set(status);
        *self.message.write() = message;
        self.last_check.store(utils::now_nanos(), Ordering::SeqCst);
    }

    /// 获取状态
    pub fn status(&self) -> HealthStatus {
        self.status.get()
    }

    /// 获取消息
    pub fn message(&self) -> String {
        self.message.read().clone()
    }

    /// 获取最后检查时间
    pub fn last_check(&self) -> i64 {
        self.last_check.load(Ordering::SeqCst)
    }
}

/// 健康检查器
pub struct HealthChecker {
    checks: HashMap<String, HealthCheck>,
}

impl HealthChecker {
    pub fn new() -> Self {
        Self {
            checks: HashMap::new(),
        }
    }

    /// 注册检查项
    pub fn register(&mut self, name: String) -> HealthCheck {
        let check = HealthCheck::new(name.clone());
        self.checks.insert(name.clone(), check.clone());
        check
    }

    /// 获取检查项
    pub fn get(&self, name: &str) -> Option<&HealthCheck> {
        self.checks.get(name)
    }

    /// 获取整体健康状态
    pub fn overall_status(&self) -> HealthStatus {
        let mut has_unhealthy = false;
        let mut has_degraded = false;

        for check in self.checks.values() {
            match check.status() {
                HealthStatus::Unhealthy => has_unhealthy = true,
                HealthStatus::Degraded => has_degraded = true,
                _ => {}
            }
        }

        if has_unhealthy {
            HealthStatus::Unhealthy
        } else if has_degraded {
            HealthStatus::Degraded
        } else {
            HealthStatus::Healthy
        }
    }

    /// 生成健康报告
    pub fn report(&self) -> HealthReport {
        HealthReport {
            status: self.overall_status(),
            checks: self.checks.values()
                .map(|c| HealthCheckReport {
                    name: c.name.clone(),
                    status: c.status(),
                    message: c.message(),
                    last_check: c.last_check(),
                })
                .collect(),
        }
    }
}

#[derive(Debug, Clone)]
pub struct HealthReport {
    pub status: HealthStatus,
    pub checks: Vec<HealthCheckReport>,
}

#[derive(Debug, Clone)]
pub struct HealthCheckReport {
    pub name: String,
    pub status: HealthStatus,
    pub message: String,
    pub last_check: i64,
}
```

---

## 5. 告警系统

### 5.1 Alert Engine

```rust
/// 告警级别
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum AlertLevel {
    Info,
    Warning,
    Error,
    Critical,
}

/// 告警
#[derive(Debug, Clone)]
pub struct Alert {
    pub id: String,
    pub level: AlertLevel,
    pub title: String,
    pub message: String,
    pub ts: i64,
    pub resolved: bool,
    pub resolved_at: Option<i64>,
}

/// 告警规则
#[derive(Debug, Clone)]
pub struct AlertRule {
    pub name: String,
    pub condition: AlertCondition,
    pub level: AlertLevel,
    pub cooldown: i64, // 纳秒
}

pub enum AlertCondition {
    /// 指标大于阈值
    MetricAbove { metric: String, threshold: f64 },
    /// 指标小于阈值
    MetricBelow { metric: String, threshold: f64 },
    /// 指标变化率
    MetricRate { metric: String, threshold: f64 },
    /// 自定义条件
    Custom(Box<dyn Fn() -> bool + Send + Sync>),
}

/// 告警引擎
pub struct AlertEngine {
    rules: Vec<AlertRule>,
    alerts: parking_lot::Mutex<Vec<Alert>>,
    last_triggered: HashMap<String, i64>,
}

impl AlertEngine {
    pub fn new() -> Self {
        Self {
            rules: Vec::new(),
            alerts: parking_lot::Mutex::new(Vec::new()),
            last_triggered: HashMap::new(),
        }
    }

    /// 添加规则
    pub fn add_rule(&mut self, rule: AlertRule) {
        self.rules.push(rule);
    }

    /// 检查规则
    pub fn check(&mut self) -> Vec<Alert> {
        let now = utils::now_nanos();
        let mut new_alerts = Vec::new();

        for rule in &self.rules {
            let last_triggered = self.last_triggered.get(&rule.name).copied().unwrap_or(0);

            // 检查冷却时间
            if now - last_triggered < rule.cooldown {
                continue;
            }

            let triggered = match &rule.condition {
                AlertCondition::MetricAbove { metric, threshold } => {
                    if let Some(metric) = MetricsRegistry::global().get(metric) {
                        match &*metric {
                            Metric::Gauge(g) => g.load(Ordering::Relaxed) > *threshold,
                            _ => false,
                        }
                    } else {
                        false
                    }
                }
                AlertCondition::MetricBelow { metric, threshold } => {
                    if let Some(metric) = MetricsRegistry::global().get(metric) {
                        match &*metric {
                            Metric::Gauge(g) => g.load(Ordering::Relaxed) < *threshold,
                            _ => false
                        }
                    } else {
                        false
                    }
                }
                AlertCondition::Custom(f) => f(),
                _ => false,
            };

            if triggered {
                let alert = Alert {
                    id: format!("{}-{}", rule.name, now),
                    level: rule.level,
                    title: rule.name.clone(),
                    message: format!("Rule '{}' triggered", rule.name),
                    ts: now,
                    resolved: false,
                    resolved_at: None,
                };

                new_alerts.push(alert.clone());
                self.alerts.lock().push(alert);
                self.last_triggered.insert(rule.name.clone(), now);
            }
        }

        new_alerts
    }

    /// 获取活跃告警
    pub fn active_alerts(&self) -> Vec<Alert> {
        self.alerts.lock()
            .iter()
            .filter(|a| !a.resolved)
            .cloned()
            .collect()
    }

    /// 解决告警
    pub fn resolve(&self, id: &str) {
        if let Some(alert) = self.alerts.lock().iter_mut().find(|a| a.id == id) {
            alert.resolved = true;
            alert.resolved_at = Some(utils::now_nanos());
        }
    }
}
```

---

## 6. 性能分析

### 6.1 Profiling

```rust
/// 性能统计
#[derive(Debug, Clone)]
pub struct PerfStat {
    /// 调用次数
    pub calls: u64,

    /// 总耗时（纳秒）
    pub total_ns: u64,

    /// 最小耗时（纳秒）
    pub min_ns: u64,

    /// 最大耗时（纳秒）
    pub max_ns: u64,

    /// 平均耗时（纳秒）
    pub avg_ns: f64,
}

impl PerfStat {
    fn new() -> Self {
        Self {
            calls: 0,
            total_ns: 0,
            min_ns: u64::MAX,
            max_ns: 0,
            avg_ns: 0.0,
        }
    }

    fn record(&mut self, elapsed_ns: u64) {
        self.calls += 1;
        self.total_ns += elapsed_ns;
        self.min_ns = self.min_ns.min(elapsed_ns);
        self.max_ns = self.max_ns.max(elapsed_ns);
        self.avg_ns = self.total_ns as f64 / self.calls as f64;
    }
}

/// 性能分析器
pub struct Profiler {
    stats: parking_lot::RwLock<HashMap<String, PerfStat>>,
}

impl Profiler {
    /// 获取全局分析器
    pub fn global() -> &'static Self {
        use std::sync::OnceLock;
        static PROFILER: OnceLock<Profiler> = OnceLock::new();
        PROFILER.get_or_init(|| {
            Self {
                stats: parking_lot::RwLock::new(HashMap::new()),
            }
        })
    }

    /// 记录函数调用
    pub fn record(&self, name: &str, elapsed_ns: u64) {
        let mut stats = self.stats.write();
        stats.entry(name.to_string())
            .or_insert_with(PerfStat::new)
            .record(elapsed_ns);
    }

    /// 获取所有统计
    pub fn stats(&self) -> Vec<(String, PerfStat)> {
        let mut stats: Vec<_> = self.stats.read()
            .iter()
            .map(|(k, v)| (k.clone(), v.clone()))
            .collect();

        stats.sort_by(|a, b| b.1.total_ns.cmp(&a.1.total_ns));
        stats
    }

    /// 打印统计报告
    pub fn print_report(&self) {
        let stats = self.stats();
        println!("═══════════════════════════════════════════════════");
        println!("              Performance Report                    ");
        println!("═══════════════════════════════════════════════════");
        println!("{:<30} {:>10} {:>12} {:>12} {:>12} {:>12}",
            "Function", "Calls", "Total(ms)", "Avg(μs)", "Min(μs)", "Max(μs)");
        println!("──────────────────────────────────────────────────────");

        for (name, stat) in stats {
            println!("{:<30} {:>10} {:>12.2} {:>12.2} {:>12.2} {:>12.2}",
                name,
                stat.calls,
                stat.total_ns as f64 / 1_000_000.0,
                stat.avg_ns / 1000.0,
                stat.min_ns as f64 / 1000.0,
                stat.max_ns as f64 / 1000.0,
            );
        }

        println!("═══════════════════════════════════════════════════");
    }
}

/// 计时器
#[must_use]
pub struct Timer<'a> {
    name: &'a str,
    start: std::time::Instant,
}

impl<'a> Timer<'a> {
    /// 开始计时
    pub fn start(name: &'a str) -> Self {
        Self {
            name,
            start: std::time::Instant::now(),
        }
    }
}

impl<'a> Drop for Timer<'a> {
    fn drop(&mut self) {
        let elapsed = self.start.elapsed().as_nanos() as u64;
        Profiler::global().record(self.name, elapsed);
    }
}

/// 便捷宏：计时函数
#[macro_export]
macro_rules! timeit {
    ($name:expr, $block:block) => {
        {
            let _timer = $crate::monitor::Timer::start($name);
            $block
        }
    };
}
```

---

## 7. 模块导出

```rust
// src/monitor/mod.rs

pub mod metrics;
pub mod trace;
pub mod health;
pub mod alert;
pub mod profiling;

pub use metrics::{
    Metric, MetricType, MetricsRegistry,
    Histogram, Summary,
};
pub use trace::{TracingConfig};
pub use health::{HealthCheck, HealthChecker, HealthStatus, HealthReport};
pub use alert::{AlertEngine, AlertRule, Alert, AlertLevel};
pub use profiling::{Profiler, PerfStat, Timer};
```

---

## 8. 未来扩展

- **Tracy 集成**: 更精确的性能分析
- **OpenTelemetry**: 分布式追踪标准
- **Web UI**: 监控面板
- **告警推送**: 邮件/钉钉/企业微信
