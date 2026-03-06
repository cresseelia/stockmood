# StockMood - A股情绪选股系统

## 项目定位

通过消息面情绪 + 简单技术形态，筛选出短期（几天到几周）有潜力的A股标的，给出带推理理由的候选股票列表。

**不做的事**：自动下单、深度基本面分析、量化回测框架。

## 核心架构

```
多数据源（涨停/股吧/雪球/微博/新闻）
    ↓
实体识别层（公司简称/名称 → 股票代码映射）
    ↓
Pipeline（信号聚合、技术计算，确定性处理）
    ↓ 编译结构化上下文
LLM分析层（Claude，推理为主，可按需调工具补充调查）
    ↓ 产出结构化结论
报告输出（Web页面 + SQLite历史）
```

关键设计原则：
- Pipeline负责采集、实体识别、聚合、技术计算，输出结构化数据包
- LLM以注入上下文为起点做推理，发现信息不足时可自主调工具深挖
- 上下文结构和Prompt是核心竞争力，需持续迭代优化
- 调度策略：**纯盘后批处理**（07:00采集消息，15:30拉行情，16:00运行分析）

详细设计见 [agents.md](./agents.md)。

## 技术栈

| 层 | 技术 |
|----|------|
| 数据采集 | Python + AKShare + 定向爬虫 |
| 存储 | SQLite |
| LLM | Claude API (claude-sonnet-4-6 或以上) |
| 调度 | APScheduler 或 cron |
| Web展示 | Streamlit |

## 项目结构

```
stockmood/
├── CLAUDE.md
├── agents.md
├── pipeline/
│   ├── collectors/        # 各数据源采集模块
│   │   ├── limit_up.py    # 涨停板
│   │   ├── guba.py        # 东方财富股吧
│   │   ├── xueqiu.py      # 雪球
│   │   ├── weibo.py       # 微博
│   │   └── news.py        # 财经新闻
│   ├── entity_mapper.py   # 公司名/简称 → 股票代码映射
│   ├── aggregator.py      # 信号聚合、去重
│   ├── technical.py       # 技术形态计算
│   └── context_builder.py # 编译LLM上下文包
├── llm/
│   ├── analyst.py         # LLM分析入口
│   └── prompts/           # Prompt模板（迭代优化）
├── storage/
│   └── db.py              # SQLite读写
├── web/
│   └── app.py             # Streamlit页面
├── scheduler.py           # 定时任务入口
└── config.py              # 配置（API Keys、参数）
```

## 运行时序

```
07:00  拉取昨夜/今晨消息（公告、雪球、股吧、微博）
15:30  盘后拉取当日行情数据
16:00  Pipeline运行：信号聚合 + 技术形态计算
16:30  LLM分析：生成候选报告
       次日开盘前人工参考
```

## 开发约定

- Python 3.11+
- 依赖管理用 `uv`
- 每个 collector 独立，互不依赖，单独可测试
- Pipeline输出格式保持稳定（LLM的契约边界）
- Prompt模板存文件，不硬编码在Python里，方便迭代
- 所有API Key通过环境变量或 `.env` 文件注入，不提交到版本控制
