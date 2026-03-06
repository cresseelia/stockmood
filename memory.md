# 项目记忆 - StockMood

AI 编码助手（Claude Code、Codex 等）应在每次会话开始时读取本文件，在会话结束或有重要发现时更新本文件。

---

## 项目状态

- 阶段：设计完成，尚未开始编码
- 核心文档：CLAUDE.md（项目概览）、agents.md（详细设计）

---

## 已确认的关键决策

| 决策 | 结论 | 原因 |
|------|------|------|
| 选股范围 | 消息驱动发现，不扫描全量 | 候选池自然聚焦有催化剂的标的 |
| LLM角色 | 推理为主，可按需调工具 | Pipeline是起点不是限制，LLM可自主深挖 |
| 搜索工具 | 不用Tavily/Serper | Google对中文金融内容覆盖不足 |
| 技术形态 | 简单形态做二次确认 | 消息面是主驱动，形态是验证 |
| 存储 | SQLite | 个人+少数朋友使用，够用 |
| 展示 | Streamlit | 快速，支持小范围访问 |
| 调度策略 | 纯盘后批处理 | 07:00采消息，15:30拉行情，16:00运行分析 |
| 实体识别 | 独立层，采集和聚合之间 | 公司简称→代码映射是核心，不可省略 |
| metadata落库 | 必须存，不可丢弃 | 评论数/转发量等热度数据是复盘的关键证据 |
| candidates可复现 | 存context_block/prompt_version/model/run_id | 必须能解释"为什么那天选了它" |

---

## 待实现模块优先级

1. collectors/ 中的 1-2 个（优先 news + guba，验证数据质量）
2. entity_mapper.py（公司简称→代码映射）
3. aggregator.py（信号聚合）
4. technical.py（K线/技术形态计算）
5. storage/db.py（SQLite读写，含完整schema）
6. context_builder.py（LLM上下文编译，需迭代）
7. llm/analyst.py（LLM分析调用）
8. web/app.py（Streamlit展示，最后做）

---

## 已知的迭代优化点

- `entity_mapper.py`：初版用词典精确匹配，后续处理模糊指代和多公司文章
- `context_builder.py`：上下文格式未固化，需真实数据跑通后迭代
- `llm/prompts/`：Prompt质量是核心竞争力，用真实案例（华工科技等）校准
- `aggregator.py`：初版按来源类型去重，后续加事件主题聚类防转载虚高
- 技术形态定义：初版先实现简单信号，后续根据实际效果调整

---

## 校准基准案例

**华工科技（300477）**
- 事件：2025年2月24日，湖北省经信厅发文，华工科技春节不停工满产满销
- 结果：当日涨停，次日起小阳堆积
- 意义：消息来自政府网站（非财经媒体），体现了多渠道采集+实体识别的必要性
- 用途：系统上线后做端到端回放，验证此类案例能否被正确识别

---

## 重要约定

- 微博采集：用户在另一个项目已有现成实现，接入时复用，不重写
- 不预设信息源优先级权重，让 LLM 从内容本身判断质量
- 公告分类（利好/利空）由 LLM 判断，不做规则硬编码
- 所有 Prompt 模板存文件（`llm/prompts/`），不硬编码在 Python 中

---

## 会话记录

### 2026-03-06（第一次会话）
- 完成整体系统规划
- 确认架构：多入口 → Pipeline → LLM推理 → 报告
- 写完 CLAUDE.md、agents.md、memory.md

### 2026-03-06（第二次会话，Codex代码审查后迭代）
- 根据Codex对文档的分析，进行设计迭代
- 新增实体识别层（entity_mapper.py），确认为系统发现能力的关键环节
- Signal.metadata 明确必须落库（含comment_count/repost_count等）
- candidates表新增 context_block/prompt_version/model/run_id，支持可复现分析
- performance表新增，支持复盘对比（D1/D3/D5/D10涨跌）
- 调度策略从"小时级或天级"统一为纯盘后批处理
- CLAUDE.md 修正LLM边界描述（之前有"不调用工具"的错误表述）
