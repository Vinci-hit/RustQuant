# AI Agent 集成模块设计文档 (ai_agent)

## 1. 模块概述

### 1.1 职责
- 与本地 AI Agent 通信
- AI 意图识别和信号生成
- 风险评估和情绪分析
- AI 决策与规则策略的融合

### 1.2 依赖关系
- **依赖**: `core` (Tick, Event, Error), `strategy` (Signal)
- **被依赖**: `strategy`, `execution`

### 1.3 文件结构
```
src/ai_agent/
├── mod.rs           # 模块导出
├── client.rs        # AI Agent 客户端
├── prompt.rs        # Prompt 模板管理
├── analyzer.rs      # AI 分析器
├── fusion.rs        # 信号融合
└── config.rs        # AI 配置
```

---

## 2. AI Agent 客户端

### 2.1 客户端抽象

```rust
/// AI Agent 客户端 Trait
///
/// 抽象不同 AI 模型的接口，支持本地部署的各种模型
#[async_trait::async_trait]
pub trait AiClient: Send + Sync {
    /// 发送文本请求
    async fn chat(&self, messages: &[Message]) -> Result<AiResponse>;

    /// 流式聊天
    async fn chat_stream(
        &self,
        messages: &[Message],
    ) -> Result<Pin<Box<dyn Stream<Item = Result<String>> + Send>>>;

    /// 获取模型名称
    fn model_name(&self) -> &str;

    /// 检查模型可用性
    async fn health_check(&self) -> Result<bool>;
}

/// AI 消息
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct Message {
    pub role: MessageRole,
    pub content: String,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_calls: Option<Vec<ToolCall>>,

    #[serde(skip_serializing_if = "Option::is_none")]
    pub tool_call_id: Option<String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum MessageRole {
    System,
    User,
    Assistant,
    Tool,
}

/// AI 响应
#[derive(Debug, Clone, serde::Deserialize)]
pub struct AiResponse {
    pub id: String,
    pub object: String,
    pub created: u64,
    pub model: String,
    pub choices: Vec<Choice>,

    #[serde(default)]
    pub usage: Option<Usage>,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct Choice {
    pub index: u32,
    pub message: Message,
    pub finish_reason: Option<String>,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct Usage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

/// 工具调用（Function Calling）
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct ToolCall {
    pub id: String,
    #[serde(rename = "type")]
    pub call_type: String,
    pub function: FunctionCall,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct FunctionCall {
    pub name: String,
    pub arguments: String,
}
```

### 2.2 本地 AI 模型实现

```rust
/// Ollama 客户端
///
/// 支持 Ollama 本地部署的各种模型（Llama, Qwen, DeepSeek 等）
pub struct OllamaClient {
    /// 基础 URL
    base_url: String,

    /// 模型名称
    model: String,

    /// HTTP 客户端
    client: reqwest::Client,

    /// 超时时间
    timeout: Duration,
}

impl OllamaClient {
    /// 创建新的 Ollama 客户端
    pub fn new(base_url: String, model: String) -> Self {
        let timeout = Duration::from_secs(30);

        Self {
            base_url: base_url.trim_end_matches('/').to_string(),
            model,
            client: reqwest::Client::builder()
                .timeout(timeout)
                .build()
                .unwrap(),
            timeout,
        }
    }

    /// 聊天补全端点
    fn chat_url(&self) -> String {
        format!("{}/api/chat", self.base_url)
    }

    /// 模型列表端点
    fn list_url(&self) -> String {
        format!("{}/api/tags", self.base_url)
    }

    /// 检查模型是否存在
    pub async fn model_exists(&self) -> Result<bool> {
        let url = self.list_url();
        let response = self.client.get(&url).send().await?;

        if response.status().is_success() {
            let models: serde_json::Value = response.json().await?;
            if let Some(models_array) = models["models"].as_array() {
                for model in models_array {
                    if let Some(name) = model["name"].as_str() {
                        if name.starts_with(&self.model) {
                            return Ok(true);
                        }
                    }
                }
            }
        }

        Ok(false)
    }

    /// 拉取模型
    pub async fn pull_model(&self) -> Result<()> {
        tracing::info!("Pulling model: {}", self.model);

        let url = format!("{}/api/pull", self.base_url);
        let response = self.client
            .post(&url)
            .json(&serde_json::json!({ "name": self.model }))
            .send()
            .await?;

        if response.status().is_success() {
            tracing::info!("Model pulled successfully");
            Ok(())
        } else {
            Err(Error::Ai(format!("Failed to pull model: {}", response.status())))
        }
    }
}

#[async_trait::async_trait]
impl AiClient for OllamaClient {
    async fn chat(&self, messages: &[Message]) -> Result<AiResponse> {
        let request_body = serde_json::json!({
            "model": self.model,
            "messages": messages,
            "stream": false,
            "options": {
                "temperature": 0.3,
                "top_p": 0.9,
                "num_ctx": 4096,
            }
        });

        let response = self.client
            .post(&self.chat_url())
            .json(&request_body)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(Error::Ai(format!(
                "Chat request failed: {}",
                response.status()
            )));
        }

        let ai_response: AiResponse = response.json().await
            .map_err(|e| Error::Ai(format!("Parse response: {}", e)))?;

        Ok(ai_response)
    }

    async fn chat_stream(
        &self,
        messages: &[Message],
    ) -> Result<Pin<Box<dyn Stream<Item = Result<String>> + Send>>> {
        let request_body = serde_json::json!({
            "model": self.model,
            "messages": messages,
            "stream": true,
        });

        let url = self.chat_url();
        let response = self.client
            .post(&url)
            .json(&request_body)
            .send()
            .await?;

        let stream = response.bytes_stream()
            .map(|chunk| {
                chunk.map_err(|e| Error::Ai(e.to_string()))
                    .and_then(|bytes| {
                        let text = String::from_utf8_lossy(&bytes);
                        // 解析 SSE 格式
                        for line in text.lines() {
                            if line.starts_with("data: ") {
                                let json = &line[6..];
                                if json.trim() == "[DONE]" {
                                    return Ok(None);
                                }
                                if let Ok(value) = serde_json::from_str::<serde_json::Value>(json) {
                                    if let Some(content) = value["message"]["content"].as_str() {
                                        return Ok(Some(content.to_string()));
                                    }
                                }
                            }
                        }
                        Ok(None)
                    })
                    .transpose()
            })
            .filter_map(|result| async move { result.transpose() });

        Ok(Box::pin(stream))
    }

    fn model_name(&self) -> &str {
        &self.model
    }

    async fn health_check(&self) -> Result<bool> {
        let response = self.client
            .get(&format!("{}/api/tags", self.base_url))
            .timeout(Duration::from_secs(5))
            .send()
            .await;

        Ok(response.is_ok_and(|r| r.status().is_success()))
    }
}

/// vLLM 客户端
///
/// 支持 vLLM 部署的各种高性能模型
pub struct VLLMClient {
    base_url: String,
    model: String,
    client: reqwest::Client,
    timeout: Duration,
}

impl VLLMClient {
    pub fn new(base_url: String, model: String) -> Self {
        Self {
            base_url,
            model,
            client: reqwest::Client::new(),
            timeout: Duration::from_secs(30),
        }
    }
}

#[async_trait::async_trait]
impl AiClient for VLLMClient {
    async fn chat(&self, messages: &[Message]) -> Result<AiResponse> {
        let request_body = serde_json::json!({
            "model": self.model,
            "messages": messages,
            "max_tokens": 2048,
            "temperature": 0.3,
        });

        let url = format!("{}/v1/chat/completions", self.base_url);
        let response = self.client
            .post(&url)
            .json(&request_body)
            .send()
            .await?;

        if !response.status().is_success() {
            return Err(Error::Ai(format!(
                "vLLM request failed: {}",
                response.status()
            )));
        }

        let ai_response: AiResponse = response.json().await
            .map_err(|e| Error::Ai(format!("Parse response: {}", e)))?;

        Ok(ai_response)
    }

    async fn chat_stream(
        &self,
        messages: &[Message],
    ) -> Result<Pin<Box<dyn Stream<Item = Result<String>> + Send>>> {
        // 实现流式响应
        todo!("Implement stream for vLLM")
    }

    fn model_name(&self) -> &str {
        &self.model
    }

    async fn health_check(&self) -> Result<bool> {
        let url = format!("{}/health", self.base_url);
        let response = self.client.get(&url).send().await?;
        Ok(response.status().is_success())
    }
}
```

---

## 3. Prompt 模板管理

### 3.1 Prompt 模板

```rust
/// Prompt 模板
#[derive(Debug, Clone)]
pub struct PromptTemplate {
    /// 系统提示词
    system_prompt: String,

    /// 用户提示词模板
    user_template: String,

    /// 可用的工具
    tools: Vec<Tool>,

    /// 温度
    temperature: f64,
}

/// AI 工具定义
#[derive(Debug, Clone, serde::Serialize)]
pub struct Tool {
    #[serde(rename = "type")]
    pub tool_type: String,

    pub function: Function,
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct Function {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,
}

impl PromptTemplate {
    /// 创建市场分析 Prompt 模板
    pub fn market_analysis() -> Self {
        let system_prompt = r#"
你是一个专业的金融分析师和交易顾问。你的任务是分析市场数据并提供交易建议。

## 分析框架
1. 技术分析：基于价格、成交量、技术指标
2. 趋势判断：识别上升、下降或横盘趋势
3. 支撑阻力位：识别关键价格水平
4. 风险评估：评估当前市场风险水平

## 输出格式
- 趋势方向：强烈看涨 / 看涨 / 中性 / 看跌 / 强烈看跌
- 建议操作：买入 / 卖出 / 观望
- 目标价格：预期的目标价位
- 止损价格：建议的止损位
- 风险等级：低 / 中 / 高
- 置信度：0.0 - 1.0

## 注意事项
- 只提供客观的分析建议
- 明确说明不确定性
- 风险提示是必需的
"#;

        let user_template = r#"
## 市场数据
交易对: {{symbol}}
当前价格: {{price}}
24h 涨跌: {{change_24h}}%
24h 成交量: {{volume_24h}}

## 技术指标
- MA7: {{ma7}}
- MA25: {{ma25}}
- MA50: {{ma50}}
- RSI(14): {{rsi}}
- MACD: {{macd}}

## 订单簿
买一: {{bid1}} @ {{bid1_vol}}
卖一: {{ask1}} @ {{ask1_vol}}

## 市场新闻
{{news}}

请基于以上数据进行综合分析，给出你的交易建议。
"#;

        Self {
            system_prompt: system_prompt.to_string(),
            user_template: user_template.to_string(),
            tools: vec![
                Self::trade_signal_tool(),
                Self::risk_assessment_tool(),
            ],
            temperature: 0.3,
        }
    }

    /// 创建交易信号工具
    fn trade_signal_tool() -> Tool {
        Tool {
            tool_type: "function".to_string(),
            function: Function {
                name: "generate_signal".to_string(),
                description: "根据分析结果生成交易信号".to_string(),
                parameters: serde_json::json!({
                    "type": "object",
                    "properties": {
                        "signal_type": {
                            "type": "string",
                            "enum": ["BUY", "SELL", "HOLD"],
                            "description": "交易信号类型"
                        },
                        "strength": {
                            "type": "number",
                            "minimum": 0.0,
                            "maximum": 1.0,
                            "description": "信号强度 (0-1)"
                        },
                        "target_price": {
                            "type": "number",
                            "description": "目标价格"
                        },
                        "stop_loss": {
                            "type": "number",
                            "description": "止损价格"
                        },
                        "reasoning": {
                            "type": "string",
                            "description": "生成信号的理由"
                        }
                    },
                    "required": ["signal_type", "strength", "reasoning"]
                }),
            },
        }
    }

    /// 创建风险评估工具
    fn risk_assessment_tool() -> Tool {
        Tool {
            tool_type: "function".to_string(),
            function: Function {
                name: "assess_risk".to_string(),
                description: "评估当前市场风险水平".to_string(),
                parameters: serde_json::json!({
                    "type": "object",
                    "properties": {
                        "risk_level": {
                            "type": "string",
                            "enum": ["LOW", "MEDIUM", "HIGH", "EXTREME"],
                            "description": "风险等级"
                        },
                        "factors": {
                            "type": "array",
                            "items": {
                                "type": "object",
                                "properties": {
                                    "name": {"type": "string"},
                                    "impact": {"type": "string", "enum": ["POSITIVE", "NEUTRAL", "NEGATIVE"]},
                                    "description": {"type": "string"}
                                }
                            }
                        },
                        "recommendation": {
                            "type": "string",
                            "description": "风险建议"
                        }
                    },
                    "required": ["risk_level", "factors", "recommendation"]
                }),
            },
        }
    }

    /// 构建 Prompt
    pub fn build(
        &self,
        variables: &std::collections::HashMap<String, String>,
    ) -> Vec<Message> {
        let mut user_prompt = self.user_template.clone();
        for (key, value) in variables {
            user_prompt = user_prompt.replace(&format!("{{{{{}}}}}", key), value);
        }

        vec![
            Message {
                role: MessageRole::System,
                content: self.system_prompt.clone(),
                tool_calls: None,
                tool_call_id: None,
            },
            Message {
                role: MessageRole::User,
                content: user_prompt,
                tool_calls: None,
                tool_call_id: None,
            },
        ]
    }

    /// 获取工具定义
    pub fn tools(&self) -> &[Tool] {
        &self.tools
    }
}

/// Prompt 变量
#[derive(Debug, Clone)]
pub struct MarketPromptVars {
    pub symbol: String,
    pub price: String,
    pub change_24h: String,
    pub volume_24h: String,
    pub ma7: String,
    pub ma25: String,
    pub ma50: String,
    pub rsi: String,
    pub macd: String,
    pub bid1: String,
    pub bid1_vol: String,
    pub ask1: String,
    pub ask1_vol: String,
    pub news: String,
}

impl MarketPromptVars {
    /// 从 Tick 和技术指标构建变量
    pub fn from_data(
        symbol: &str,
        tick: &Tick,
        indicators: &TechnicalIndicators,
    ) -> Self {
        Self {
            symbol: symbol.to_string(),
            price: tick.price_f64().to_string(),
            change_24h: indicators.change_24h.to_string(),
            volume_24h: indicators.volume_24h.to_string(),
            ma7: indicators.ma7.to_string(),
            ma25: indicators.ma25.to_string(),
            ma50: indicators.ma50.to_string(),
            rsi: indicators.rsi.to_string(),
            macd: format!("{:.4}", indicators.macd),
            bid1: tick.bid_px_f64().to_string(),
            bid1_vol: tick.bid_vol.to_string(),
            ask1: tick.ask_px_f64().to_string(),
            ask1_vol: tick.ask_vol.to_string(),
            news: indicators.latest_news.clone(),
        }
    }
}

/// 技术指标
#[derive(Debug, Clone)]
pub struct TechnicalIndicators {
    pub change_24h: f64,
    pub volume_24h: f64,
    pub ma7: f64,
    pub ma25: f64,
    pub ma50: f64,
    pub rsi: f64,
    pub macd: f64,
    pub latest_news: String,
}
```

---

## 4. AI 分析器

### 4.1 市场分析

```rust
/// AI 市场分析器
pub struct AiMarketAnalyzer {
    /// AI 客户端
    client: Box<dyn AiClient>,

    /// Prompt 模板
    template: PromptTemplate,

    /// 历史数据窗口
    price_history: VecDeque<f64>,

    /// 分析缓存
    cache: parking_lot::RwLock<LruCache<i64, MarketAnalysis>>,
}

/// 市场分析结果
#[derive(Debug, Clone)]
pub struct MarketAnalysis {
    /// 趋势方向
    pub trend: TrendDirection,

    /// 交易信号
    pub signal: Option<AiSignal>,

    /// 风险评估
    pub risk: RiskAssessment,

    /// 分析时间戳
    pub ts: i64,

    /// AI 推理过程
    pub reasoning: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TrendDirection {
    StrongBullish,
    Bullish,
    Neutral,
    Bearish,
    StrongBearish,
}

#[derive(Debug, Clone)]
pub struct AiSignal {
    pub signal_type: SignalType,
    pub strength: f64,
    pub target_price: Option<Price>,
    pub stop_loss: Option<Price>,
    pub reasoning: String,
    pub confidence: f64,
}

#[derive(Debug, Clone)]
pub struct RiskAssessment {
    pub level: RiskLevel,
    pub factors: Vec<RiskFactor>,
    pub recommendation: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RiskLevel {
    Low,
    Medium,
    High,
    Extreme,
}

#[derive(Debug, Clone)]
pub struct RiskFactor {
    pub name: String,
    pub impact: Impact,
    pub description: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Impact {
    Positive,
    Neutral,
    Negative,
}

impl AiMarketAnalyzer {
    /// 创建新的 AI 分析器
    pub fn new(client: Box<dyn AiClient>, template: PromptTemplate) -> Self {
        Self {
            client,
            template,
            price_history: VecDeque::with_capacity(100),
            cache: parking_lot::RwLock::new(LruCache::new(100)),
        }
    }

    /// 分析市场
    pub async fn analyze(
        &self,
        symbol: &str,
        tick: &Tick,
    ) -> Result<MarketAnalysis> {
        // 计算技术指标
        let indicators = self.calculate_indicators(tick);

        // 构建 Prompt 变量
        let vars = MarketPromptVars::from_data(symbol, tick, &indicators);

        let variables = serde_json::to_value(vars)
            .map_err(|e| Error::Ai(format!("Serialize vars: {}", e)))?;

        let mut variable_map = std::collections::HashMap::new();
        if let serde_json::Value::Object(obj) = variables {
            for (k, v) in obj {
                if let serde_json::Value::String(s) = v {
                    variable_map.insert(k, s);
                }
            }
        }

        // 构建消息
        let messages = self.template.build(&variable_map);

        // 调用 AI
        let response = self.client.chat(&messages).await?;

        // 解析响应
        self.parse_response(response, tick.ts)
    }

    /// 计算技术指标
    fn calculate_indicators(&mut self, tick: &Tick) -> TechnicalIndicators {
        let price = tick.price_f64();
        self.price_history.push_back(price);

        let len = self.price_history.len();

        let ma7 = if len >= 7 {
            self.price_history.iter().rev().take(7).sum::<f64>() / 7.0
        } else { price };

        let ma25 = if len >= 25 {
            self.price_history.iter().rev().take(25).sum::<f64>() / 25.0
        } else { price };

        let ma50 = if len >= 50 {
            self.price_history.iter().rev().take(50).sum::<f64>() / 50.0
        } else { price };

        let rsi = self.calculate_rsi(14);
        let macd = self.calculate_macd();

        TechnicalIndicators {
            change_24h: 0.0, // 需要从历史数据计算
            volume_24h: tick.volume as f64,
            ma7,
            ma25,
            ma50,
            rsi,
            macd,
            latest_news: String::new(),
        }
    }

    /// 计算 RSI
    fn calculate_rsi(&self, period: usize) -> f64 {
        // 简化实现
        50.0
    }

    /// 计算 MACD
    fn calculate_macd(&self) -> f64 {
        // 简化实现
        0.0
    }

    /// 解析 AI 响应
    fn parse_response(&self, response: AiResponse, ts: i64) -> Result<MarketAnalysis> {
        let content = response.choices.get(0)
            .and_then(|c| c.message.content.as_ref())
            .ok_or_else(|| Error::Ai("No content in response".to_string()))?;

        // 解析工具调用
        let signal = response.choices.get(0)
            .and_then(|c| c.message.tool_calls.as_ref())
            .and_then(|calls| calls.first())
            .and_then(|call| self.parse_tool_call(&call.function));

        Ok(MarketAnalysis {
            trend: self.extract_trend(content),
            signal,
            risk: RiskAssessment {
                level: RiskLevel::Medium,
                factors: Vec::new(),
                recommendation: String::new(),
            },
            ts,
            reasoning: content.clone(),
        })
    }

    /// 从内容提取趋势
    fn extract_trend(&self, content: &str) -> TrendDirection {
        let lower = content.to_lowercase();

        if lower.contains("强烈看涨") || lower.contains("strongly bullish") {
            TrendDirection::StrongBullish
        } else if lower.contains("看涨") || lower.contains("bullish") {
            TrendDirection::Bullish
        } else if lower.contains("强烈看跌") || lower.contains("strongly bearish") {
            TrendDirection::StrongBearish
        } else if lower.contains("看跌") || lower.contains("bearish") {
            TrendDirection::Bearish
        } else {
            TrendDirection::Neutral
        }
    }

    /// 解析工具调用
    fn parse_tool_call(&self, call: &FunctionCall) -> Option<AiSignal> {
        if call.name != "generate_signal" {
            return None;
        }

        let args: serde_json::Value = serde_json::from_str(&call.arguments).ok()?;

        let signal_type = match args["signal_type"].as_str()? {
            "BUY" => SignalType::Buy,
            "SELL" => SignalType::Sell,
            "HOLD" => return None,
            _ => return None,
        };

        Some(AiSignal {
            signal_type,
            strength: args["strength"].as_f64().unwrap_or(0.5),
            target_price: args["target_price"].as_f64().map(Price::from_f64),
            stop_loss: args["stop_loss"].as_f64().map(Price::from_f64),
            reasoning: args["reasoning"].as_str().unwrap_or("").to_string(),
            confidence: args["strength"].as_f64().unwrap_or(0.5),
        })
    }
}
```

---

## 5. 信号融合

### 5.1 多信号融合

```rust
/// 信号融合器
///
/// 将 AI 信号与规则策略信号进行融合
pub struct SignalFusion {
    /// 融合策略
    strategy: FusionStrategy,

    /// 权重配置
    weights: FusionWeights,
}

/// 融合策略
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FusionStrategy {
    /// 只使用 AI 信号
    AiOnly,

    /// 只使用规则信号
    RuleOnly,

    /// 信号加权平均
    WeightedAverage,

    /// 信号投票
    Voting,

    /// AI 拥有否决权
    AiVeto,

    /// 规则优先，AI 作为增强
    RulePrimary,
}

/// 融合权重
#[derive(Debug, Clone)]
pub struct FusionWeights {
    /// AI 信号权重
    pub ai: f64,

    /// 规则信号权重
    pub rule: f64,

    /// 技术指标权重
    pub technical: f64,
}

impl Default for FusionWeights {
    fn default() -> Self {
        Self {
            ai: 0.4,
            rule: 0.4,
            technical: 0.2,
        }
    }
}

/// 融合后的信号
#[derive(Debug, Clone)]
pub struct FusedSignal {
    /// 最终信号类型
    pub signal_type: SignalType,

    /// 信号强度
    pub strength: f64,

    /// 信号来源
    pub sources: Vec<String>,

    /// 融合置信度
    pub confidence: f64,

    /// AI 推理
    pub ai_reasoning: Option<String>,

    /// 规则信号信息
    pub rule_signals: Vec<Signal>,
}

impl SignalFusion {
    pub fn new(strategy: FusionStrategy, weights: FusionWeights) -> Self {
        Self {
            strategy,
            weights,
        }
    }

    /// 融合信号
    pub fn fuse(
        &self,
        ai_signal: Option<AiSignal>,
        rule_signals: &[Signal],
        market_context: &MarketContext,
    ) -> Option<FusedSignal> {
        match self.strategy {
            FusionStrategy::AiOnly => {
                ai_signal.map(|s| FusedSignal {
                    signal_type: s.signal_type,
                    strength: s.strength,
                    sources: vec!["AI".to_string()],
                    confidence: s.confidence,
                    ai_reasoning: Some(s.reasoning),
                    rule_signals: Vec::new(),
                })
            }
            FusionStrategy::RuleOnly => {
                self.select_best_rule(rule_signals).map(|s| FusedSignal {
                    signal_type: s.signal_type,
                    strength: s.strength,
                    sources: vec!["Rule".to_string()],
                    confidence: s.metadata.confidence,
                    ai_reasoning: None,
                    rule_signals: vec![s.clone()],
                })
            }
            FusionStrategy::WeightedAverage => {
                self.weighted_fusion(ai_signal, rule_signals, market_context)
            }
            FusionStrategy::Voting => {
                self.voting_fusion(ai_signal, rule_signals)
            }
            FusionStrategy::AiVeto => {
                self.ai_veto_fusion(ai_signal, rule_signals)
            }
            FusionStrategy::RulePrimary => {
                self.rule_primary_fusion(ai_signal, rule_signals, market_context)
            }
        }
    }

    /// 选择最佳规则信号
    fn select_best_rule(&self, signals: &[Signal]) -> Option<Signal> {
        signals.iter()
            .max_by(|a, b| {
                a.strength.partial_cmp(&b.strength)
                    .unwrap_or(std::cmp::Ordering::Equal)
            })
            .cloned()
    }

    /// 加权融合
    fn weighted_fusion(
        &self,
        ai_signal: Option<AiSignal>,
        rule_signals: &[Signal],
        context: &MarketContext,
    ) -> Option<FusedSignal> {
        let rule = self.select_best_rule(rule_signals)?;

        let ai_weight = ai_signal.as_ref().map_or(0.0, |s| s.strength * self.weights.ai);
        let rule_weight = rule.strength * self.weights.rule;
        let technical_weight = context.trend_strength * self.weights.technical;

        let total_weight = self.weights.ai + self.weights.rule + self.weights.technical;
        let combined_strength = (ai_weight + rule_weight + technical_weight) / total_weight;

        let signal_type = if combined_strength > 0.5 {
            if rule.signal_type == SignalType::Buy || ai_signal.as_ref().map_or(false, |s| s.signal_type == SignalType::Buy) {
                SignalType::Buy
            } else {
                SignalType::Sell
            }
        } else {
            SignalType::Close
        };

        Some(FusedSignal {
            signal_type,
            strength: combined_strength,
            sources: vec![
                "AI".to_string(),
                "Rule".to_string(),
                "Technical".to_string(),
            ],
            confidence: combined_strength,
            ai_reasoning: ai_signal.map(|s| s.reasoning),
            rule_signals: vec![rule.clone()],
        })
    }

    /// 投票融合
    fn voting_fusion(
        &self,
        ai_signal: Option<AiSignal>,
        rule_signals: &[Signal],
    ) -> Option<FusedSignal> {
        let mut buy_votes = 0;
        let mut sell_votes = 0;
        let mut sources = Vec::new();

        if let Some(ref s) = ai_signal {
            match s.signal_type {
                SignalType::Buy => buy_votes += 1,
                SignalType::Sell => sell_votes += 1,
                _ => {}
            }
            sources.push("AI".to_string());
        }

        for signal in rule_signals {
            match signal.signal_type {
                SignalType::Buy => buy_votes += 1,
                SignalType::Sell => sell_votes += 1,
                _ => {}
            }
            sources.push(format!("Rule({})", signal.reason));
        }

        if buy_votes == sell_votes {
            return None;
        }

        let signal_type = if buy_votes > sell_votes {
            SignalType::Buy
        } else {
            SignalType::Sell
        };

        Some(FusedSignal {
            signal_type,
            strength: ((buy_votes.max(sell_votes) as f64) / (buy_votes + sell_votes) as f64),
            sources,
            confidence: ((buy_votes.max(sell_votes) as f64) / (buy_votes + sell_votes) as f64),
            ai_reasoning: ai_signal.map(|s| s.reasoning),
            rule_signals: rule_signals.to_vec(),
        })
    }

    /// AI 否决权融合
    fn ai_veto_fusion(
        &self,
        ai_signal: Option<AiSignal>,
        rule_signals: &[Signal],
    ) -> Option<FusedSignal> {
        // AI 拥有否决权，如果 AI 认为高风险，则不执行
        if let Some(ref ai) = ai_signal {
            if ai.confidence < 0.3 {
                tracing::warn!("AI vetoed due to low confidence: {}", ai.reasoning);
                return None;
            }
        }

        self.weighted_fusion(ai_signal, rule_signals, &MarketContext::default())
    }

    /// 规则优先融合
    fn rule_primary_fusion(
        &self,
        ai_signal: Option<AiSignal>,
        rule_signals: &[Signal],
        context: &MarketContext,
    ) -> Option<FusedSignal> {
        let rule = self.select_best_rule(rule_signals)?;

        // AI 可以调整止损和止盈
        let stop_loss = ai_signal.as_ref()
            .and_then(|s| s.stop_loss)
            .or_else(|| {
                // 基于规则信号的元数据计算
                None
            });

        let target_price = ai_signal.as_ref()
            .and_then(|s| s.target_price);

        Some(FusedSignal {
            signal_type: rule.signal_type,
            strength: rule.strength,
            sources: vec!["Rule(primary)".to_string(), "AI(enhance)".to_string()],
            confidence: rule.metadata.confidence,
            ai_reasoning: ai_signal.map(|s| s.reasoning),
            rule_signals: vec![rule.clone()],
        })
    }
}

/// 市场上下文
#[derive(Debug, Clone, Default)]
pub struct MarketContext {
    /// 趋势强度
    pub trend_strength: f64,

    /// 波动率
    pub volatility: f64,

    /// 流动性
    pub liquidity: f64,

    /// 市场情绪
    pub sentiment: f64,
}
```

---

## 6. AI 策略

### 6.1 AI 驱动策略

```rust
/// AI 驱动的交易策略
pub struct AiStrategy {
    /// AI 分析器
    analyzer: Arc<AiMarketAnalyzer>,

    /// 信号融合器
    fusion: SignalFusion,

    /// 策略上下文
    ctx: StrategyContext,

    /// 策略 ID
    strategy_id: StrategyId,

    /// 交易对
    symbol: Symbol,

    /// 当前持仓
    position: f64,

    /// 最后分析时间
    last_analysis_ts: Option<i64>,

    /// 分析间隔（纳秒）
    analysis_interval: i64,
}

#[derive(Debug, Clone)]
pub struct AiStrategyConfig {
    /// AI 客户端配置
    pub ai_client: AiClientConfig,

    /// 融合策略
    pub fusion_strategy: FusionStrategy,

    /// 分析间隔（秒）
    pub analysis_interval_secs: u64,
}

#[derive(Debug, Clone)]
pub enum AiClientConfig {
    Ollama {
        base_url: String,
        model: String,
    },
    VLLM {
        base_url: String,
        model: String,
    },
}

impl AiStrategy {
    /// 创建新的 AI 策略
    pub async fn new(
        strategy_id: StrategyId,
        symbol: Symbol,
        config: AiStrategyConfig,
        ctx: StrategyContext,
    ) -> Result<Self> {
        let client: Box<dyn AiClient> = match config {
            AiClientConfig::Ollama { base_url, model } => {
                Box::new(OllamaClient::new(base_url, model))
            }
            AiClientConfig::VLLM { base_url, model } => {
                Box::new(VLLMClient::new(base_url, model))
            }
        };

        let template = PromptTemplate::market_analysis();
        let analyzer = Arc::new(AiMarketAnalyzer::new(client, template));

        let fusion = SignalFusion::new(
            config.fusion_strategy,
            FusionWeights::default(),
        );

        Ok(Self {
            analyzer,
            fusion,
            ctx,
            strategy_id,
            symbol,
            position: 0.0,
            last_analysis_ts: None,
            analysis_interval: config.analysis_interval_secs as i64 * 1_000_000_000,
        })
    }

    /// 转换为 Strategy Trait
    pub fn into_strategy(self) -> Box<dyn Strategy> {
        Box::new(self)
    }
}

#[async_trait::async_trait]
impl Strategy for AiStrategy {
    type Config = AiStrategyConfig;
    type Context = StrategyContext;

    async fn init(config: Self::Config, ctx: Self::Context) -> Self {
        // 实际实现中需要处理 Result
        unimplemented!()
    }

    fn on_tick(&mut self, tick: &Tick) -> Option<Signal> {
        // 检查是否需要分析
        let now = tick.ts;
        if let Some(last) = self.last_analysis_ts {
            if now - last < self.analysis_interval {
                return None;
            }
        }

        self.last_analysis_ts = Some(now);

        // 异步分析需要在 Tokio 运行时中执行
        // 这里简化为同步返回
        let analysis = tokio::runtime::Handle::try_current()
            .and_then(|h| h.block_on(self.analyzer.analyze(&self.symbol.to_string(), tick)))
            .ok()?;

        // 这里需要将 AiSignal 转换为 Signal
        // 简化实现
        None
    }

    fn name(&self) -> &str {
        "ai_strategy"
    }
}
```

---

## 7. 模块导出

```rust
// src/ai_agent/mod.rs

pub mod client;
pub mod prompt;
pub mod analyzer;
pub mod fusion;
pub mod config;

pub use client::{AiClient, OllamaClient, VLLMClient};
pub use client::{Message, MessageRole, AiResponse};
pub use prompt::{PromptTemplate, Tool, MarketPromptVars};
pub use analyzer::{AiMarketAnalyzer, MarketAnalysis, AiSignal};
pub use analyzer::{TrendDirection, RiskAssessment, RiskLevel};
pub use fusion::{SignalFusion, FusionStrategy, FusedSignal};
pub use fusion::{FusionWeights, MarketContext};
pub use config::{AiStrategy, AiStrategyConfig, AiClientConfig};
```

---

## 8. 性能指标

| 指标 | 目标 | 测试方法 |
|-----|------|---------|
| AI 响应时间 | < 500ms | 端到端延迟 |
| 信号生成延迟 | < 1s | Tick 到信号 |
| Token 使用 | < 1K tokens | 每次分析 |
| 内存占用 | < 200MB | 模型加载后 |

---

## 9. 未来扩展

- **多模型集成**: 同时使用多个 AI 模型进行对比
- **RAG (检索增强)**: 结合历史数据增强 AI 分析
- **Fine-tuning**: 针对特定市场微调模型
- **强化学习**: AI 自主优化策略
- **多模态输入**: 结合新闻、社交媒体等
