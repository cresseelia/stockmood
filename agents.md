# 系统设计详细规范

> 本文档是 [CLAUDE.md](./CLAUDE.md) 的详细展开，阅读本文档前请先阅读 CLAUDE.md 了解项目定位和整体架构。

## 核心设计决策及原因

### 为什么是"消息驱动发现"而非扫描全量股票

全A股约5000只，逐一分析成本高且噪音多。消息/情绪/涨停本身就是市场的筛选结果——有人讨论、有人买入才会产生信号。因此候选池由消息面自然生成，技术形态只做二次确认。

典型案例：湖北省经信厅发文华工科技春节满产满销 → 当日涨停 → 股吧热度增强 → K线小阳堆积。这类消息来自政府网站，不可能通过预设规则覆盖，必须依赖多渠道广撒网。

### Pipeline 和 LLM 的职责划分

Pipeline 做确定性的批量采集和预处理，LLM 做推理判断并可按需补充信息：

- Pipeline 保证基础上下文的完整性和结构化，是 LLM 分析的起点
- LLM 在推理过程中，如发现上下文不足，可自主调用工具获取额外数据（如深入搜索某条新闻的原文、查询某公司公告详情）
- 这种设计兼顾了批量处理的效率（Pipeline 统一采集）和分析深度（LLM 按需深挖）

结论：**Pipeline 负责确定性的批量采集，LLM 负责推理并可自主决定是否补充调查。**

### 为什么不用海外搜索工具（Tavily/Serper）

底层是 Google，对中文金融内容覆盖不足：股吧帖子基本不收录，省级政府网站收录延迟，雪球/同花顺实时资讯不全。A股的信息生态在中文互联网，必须用针对中文平台的定向采集。

### 为什么涨停板只是入口之一

涨停是价格已经验证的滞后信号。系统真正的价值在于发现**先于价格的信号**：大V发文、股吧热度异动、新闻消息。多入口并行，同一只股票被多个入口命中是更强的信号。

### 上下文结构和 Prompt 是核心竞争力

不在设计阶段固化。需要用真实案例跑通后，通过对比 LLM 输出质量来反向迭代。`context_builder` 和 `llm/prompts/` 是持续演进的模块。

---

## 一、数据采集层（Collectors）

每个 collector 独立运行，输出统一的信号格式：

```python
Signal = {
    "stock_code": str,       # 股票代码，如 "300477"
    "stock_name": str,       # 股票名称，如 "华工科技"
    "source": str,           # 来源标识，如 "limit_up" / "guba" / "weibo"
    "raw_content": str,      # 原始文本内容
    "signal_time": datetime, # 信号产生时间
    "metadata": dict,        # 来源特有字段（热度、转发数等）
}
```

### 1.1 涨停板 (limit_up.py)

- 数据源：AKShare `stock_zt_pool_em`
- 采集时间：盘后（15:30后）
- 输出：当日所有涨停股票列表

### 1.2 东方财富股吧 (guba.py)

- 数据源：AKShare `stock_bar_em` 或直接爬取
- 采集内容：热帖标题、评论数、阅读量、发帖时间
- 采集频率：每小时或盘后
- 重点信号：热度异动（短期评论量突增）

### 1.3 雪球 (xueqiu.py)

- 采集内容：热门讨论、关注度变化
- 重点关注：认证用户（大V）发文提及个股

### 1.4 微博 (weibo.py)

- 复用已有项目的采集能力
- 重点：财经大V对个股的点评

### 1.5 财经新闻 (news.py)

- 数据源：AKShare `stock_news_em`（东方财富个股新闻）
- 补充：百度搜索（覆盖政府网站、行业媒体等非财经平台来源）
- 重点：识别文中提及的具体公司/股票

---

## 二、Pipeline 处理层

### 2.1 信号聚合 (aggregator.py)

输入：所有 collector 产出的 Signal 列表

处理逻辑：
1. 按 `stock_code` 分组
2. 统计每只股票的信号数量和来源分布
3. 提取去重后的原始内容列表
4. 输出候选股票列表，按信号数降序排列

输出格式：
```python
Candidate = {
    "stock_code": str,
    "stock_name": str,
    "signal_count": int,
    "sources": list[str],          # 去重后的来源列表
    "raw_signals": list[Signal],   # 原始信号列表
}
```

### 2.2 技术形态计算 (technical.py)

输入：候选股票代码列表

数据源：AKShare `stock_zh_a_hist`（日线行情）

计算指标：
- MA5、MA10、MA20 及其排列关系
- 量比（今日成交量 / N日均量）
- 价格相对近期高低点的位置
- 涨停次日缩量整理信号（小阳堆积）
- 简单形态标记：均线多头排列、放量突破、缩量整理

输出格式：
```python
TechnicalData = {
    "stock_code": str,
    "ma_trend": str,          # "bullish" / "bearish" / "neutral"
    "volume_ratio": float,    # 量比
    "price_position": str,    # "near_high" / "mid" / "near_low"
    "patterns": list[str],    # 命中的形态标签
    "recent_klines": list,    # 最近N根K线数据（注入上下文用）
}
```

### 2.3 上下文编译 (context_builder.py)

输入：Candidate 列表 + TechnicalData 字典

职责：将每只股票的所有信息编译成供LLM消费的结构化文本块。

**注意**：具体上下文格式和字段选取是需要持续迭代优化的核心内容，此处仅定义接口契约：

- 输入：一只股票的所有采集数据
- 输出：一个文本块，包含该股票足以支撑LLM推理的全部信息
- 原则：信息密度高，冗余低，结构清晰

输出：
```python
ContextPackage = {
    "date": str,
    "candidates": [
        {
            "stock_code": str,
            "stock_name": str,
            "context_block": str,   # 注入LLM的文本
        }
    ]
}
```

---

## 三、LLM 分析层

### 3.1 设计原则

- Pipeline 注入的上下文是**起点**，不是限制
- LLM 可以基于上下文自主判断是否需要补充信息，并调用工具获取
- LLM的核心任务：判断信号质量、识别逻辑链条、给出候选结论和理由
- 工具调用由 LLM 按需发起，而非预设固定流程

### 3.2 输入/输出契约

**输入**：ContextPackage（由 context_builder 产出）

**输出**：
```python
AnalysisResult = {
    "stock_code": str,
    "decision": str,          # "candidate" / "watch" / "exclude"
    "confidence": str,        # "high" / "medium" / "low"
    "logic_summary": str,     # 推理摘要，一两句话
    "key_catalyst": str,      # 核心催化事件
    "risk_notes": str,        # 风险提示（如有）
    "technical_comment": str, # 对技术形态的判断
}
```

### 3.3 Prompt 管理

- Prompt 模板存放在 `llm/prompts/` 目录下，`.txt` 或 `.md` 文件
- 不硬编码在 Python 代码中
- 需要迭代优化，用真实案例（如华工科技）校准输出质量
- System prompt 和 user prompt 分开管理

### 3.4 处理方式

每只候选股票独立调用一次LLM分析，避免上下文混淆。
如候选数量较多（>30只），先用规则做粗筛（信号数 >= 2），再调LLM精析。

---

## 四、存储层 (db.py)

SQLite，三张核心表：

**signals**：原始信号记录
```sql
CREATE TABLE signals (
    id INTEGER PRIMARY KEY,
    stock_code TEXT,
    stock_name TEXT,
    source TEXT,
    raw_content TEXT,
    signal_time DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**candidates**：每日选股结果
```sql
CREATE TABLE candidates (
    id INTEGER PRIMARY KEY,
    date TEXT,
    stock_code TEXT,
    stock_name TEXT,
    decision TEXT,
    confidence TEXT,
    logic_summary TEXT,
    key_catalyst TEXT,
    risk_notes TEXT,
    technical_comment TEXT,
    signal_count INTEGER,
    sources TEXT,              -- JSON array
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**kline_cache**：行情数据缓存（避免重复拉取）
```sql
CREATE TABLE kline_cache (
    stock_code TEXT,
    date TEXT,
    open REAL, high REAL, low REAL, close REAL, volume REAL,
    PRIMARY KEY (stock_code, date)
);
```

---

## 五、输出层

### 5.1 Web 页面 (Streamlit)

页面结构：
- **今日候选**：按 confidence 排序的候选股票表格，点击展开推理详情
- **历史记录**：按日期浏览历史选股，可查看当时的推理原因
- **复盘对比**：选股后N日涨跌幅展示（用于评估系统效果）

### 5.2 报告格式

每只候选股票展示：
- 股票代码 + 名称
- 核心催化事件（一句话）
- 推理摘要
- 技术形态评价
- 信号来源列表
- 风险提示

---

## 六、配置 (config.py)

```python
# 通过环境变量注入，或 .env 文件
ANTHROPIC_API_KEY = ...
BAIDU_SEARCH_API_KEY = ...   # 可选

# 运行参数
SIGNAL_LOOKBACK_HOURS = 24   # 信号时间窗口
MIN_SIGNAL_COUNT = 1         # 进入LLM分析的最低信号数
MAX_CANDIDATES_PER_RUN = 50  # 单次最多分析股票数
KLINE_LOOKBACK_DAYS = 30     # 技术形态计算窗口
```

---

## 七、迭代优化方向

以下模块在初版实现后需要持续迭代：

1. **context_builder**：上下文信息密度、字段选取、格式
2. **Prompt**：LLM的分析质量、输出一致性
3. **技术形态**：信号定义的准确性
4. **校准基准**：用历史真实案例（华工科技等）验证系统输出是否符合预期
