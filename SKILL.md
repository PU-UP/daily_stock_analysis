---
name: daily-stock-analysis-ops
description: 每日运行 daily_stock_analysis 主流程并做日志/报告/预测复盘。用户说「跑今日分析」「每日复盘」「检查昨天报告有没有用」时使用。
write-scope:
  - daily_agent_evolution/**
---

# Daily Stock Analysis — 每日运维 Skill

本 Skill 定义 **本地每日运维 playbook**：先跑主工程，再由 Agent **独立**完成日志审计、报告评价、近 5 交易日回溯，并写入标准化运行日志。

> **不是** OpenClaw / REST API 集成说明。通过 HTTP 触发分析见 [docs/openclaw-skill-integration.md](docs/openclaw-skill-integration.md)。

---

## 角色原则

1. **执行器 + 复盘者**：不只跑命令，必须质疑报告与流程。
2. **证据优先**：每条 issue 必须有日志/报告/DB 证据，禁止空泛「检查网络」。
3. **宁可低分**：报告评价禁止关键词堆砌充「有价值内容」。
4. **控制 git diff**：默认只写 `daily_agent_evolution/`，不改代码与配置。
5. **不确定就写「不确定」**，不粉饰。

---

## 硬边界（必须遵守）

### 允许写入（仅此范围）

| 路径 | 用途 |
|------|------|
| `daily_agent_evolution/YYYY-MM-DD_journal.md` | 主记录 |
| `daily_agent_evolution/YYYY-MM-DD_summary.txt` | 短摘要（≤40 行） |
| `daily_agent_evolution/experience_db.json` | 结构化经验（增量更新） |

### 禁止（除非用户当次明确授权）

- `src/`、`tests/`、`apps/`、`.github/`、`.env`、`.env.example`、`requirements.txt`、`SKILL.md`（本文件）
- `git commit` / `git push` / 安装依赖 / 改 CI / 重构

### 允许只读

- `logs/stock_analysis_YYYYMMDD.log`、`logs/stock_analysis_debug_YYYYMMDD.log`
- `reports/report_YYYYMMDD.md`、`reports/market_review_YYYYMMDD.md`
- `data/stock_analysis.db`
- `.env`（核对配置，不改）

Journal frontmatter 中 **`git_touch: none`** 为默认值；若违反写入范围，必须在 journal 中如实记录。

---

## 每日工作流总览

```
Phase 0  Pre-flight   → 日历、配置只读检查
Phase 1  Run          → python main.py [ --force-run ]
Phase 2  Log Audit    → 日志审计（区分噪声与真 issue）
Phase 3  Report Review→ 报告价值评价
Phase 4  Retro 5d     → 近 5 交易日回溯与预测校准
Write    Journal      → 仅写 daily_agent_evolution/
```

---

## Phase 0 — Pre-flight

1. **只读** `.env` 关键项：`STOCK_LIST`、`MARKET_REVIEW_ENABLED`、`MAX_WORKERS`、搜索（如 `MINIMAX_API_KEYS`）、LLM 渠道、通知（如 `FEISHU_WEBHOOK_URL`）。
2. **判断是否 A 股交易日**（cn 市场）：

```bash
python -c "from src.core.trading_calendar import get_open_markets_today; print('cn' in get_open_markets_today())"
```

- 输出 `False`：**默认不跑** `main.py`，仍完成 Phase 2–4（若有历史报告/日志）并写 journal，`run_status: skipped_non_trading_day`。
- 用户明确要求 force / 补跑：Phase 1 使用 `python main.py --force-run`。
3. 记录 `run_id`：`YYYY-MM-DDTHH:MM`（本地时间）。

---

## Phase 1 — Run

```bash
# 交易日默认
python main.py

# 仅 Pre-flight 允许时
python main.py --force-run
```

**成功标准**

- `exit code == 0`
- 日志出现 `成功: N, 失败: 0`（N 应等于 `STOCK_LIST` 数量）
- 生成 `reports/report_YYYYMMDD.md`；若 `MARKET_REVIEW_ENABLED=true`，另有 `reports/market_review_YYYYMMDD.md`

**超时预期**：`MAX_WORKERS=1` 且 5 只股票时约 5–10 分钟；不要中途杀进程。

**失败时**：仍进入 Phase 2–4，journal 中 `run_status: failed`，重点审计失败原因。

---

## Phase 2 — Log Audit

**必读**：`logs/stock_analysis_{YYYYMMDD}.log` 中**本次运行**段落（从 `A股自选股智能分析系统 启动` 到 `程序执行完成`）。若同日多次运行，以**最后一次**为准并注明。

### 噪声清单（默认不计入 issue）

| 噪声 | 说明 |
|------|------|
| `UnicodeEncodeError` / `gbk` | Windows 控制台 emoji，文件日志通常正常 |
| LiteLLM model cost map 超时 | 已 fallback 本地 |
| 缺少 `botocore` | 未用 AWS/Bedrock 可忽略 |
| Tushare `trade_cal` / `cyq_chips` 限流 | 低积分账号预期行为；若 `ENABLE_CHIP_DISTRIBUTION=false` 仍出现 cyq 调用 → 记 **config** issue |

### Issue 类型（必须记录的真问题）

| category | 严重度 | 示例 |
|----------|--------|------|
| `run_failed` | critical | exit_code≠0 或 成功数 < 自选股数 |
| `llm_failed` | high | `[LLM调用]` 失败、无响应 |
| `search_degraded` | high | 所有搜索 provider 不可用 |
| `data_gap` | medium | 个股缺 K 线/实时价导致分析降级 |
| `notify_failed` | medium | 飞书/Webhook 发送失败 |
| `market_review_degraded` | medium | 大盘复盘缺涨跌/板块（多源均失败） |
| `config_drift` | medium | 日志行为与 `.env` 声明不一致 |

每条 issue 字段：`id`（如 `20260531-01`）、`severity`、`category`、`evidence`（日志原文一句）、`impact`、`action`（具体：改哪项 env / 等下一交易日 / 需人工）。

---

## Phase 3 — Report Review

**输入**

- `reports/report_YYYYMMDD.md`
- `reports/market_review_YYYYMMDD.md`（若存在）

**五维 rubric**（各 1–5 分，加权总分 1–10）

| 维度 | 权重 | 看什么 |
|------|------|--------|
| 数据锚定 | 25% | 价/MA/量比与日志技术面一致；非交易日是否说明用最近收盘 |
| 可执行性 | 25% | 观望/持有/介入 + 价位或事件触发，非空话 |
| 情报质量 | 20% | 搜索是否 direct 个股新闻，非 macro 灌水 |
| 自洽性 | 20% | 结论 vs 乖离率/趋势/量能是否矛盾 |
| 复盘增量 | 10% | 相对昨日是否模板复读 |

**输出表格**（写入 journal §3）

```markdown
| 报告 | 总分 | 信任度 | 最有用 1 条 | 最大问题 1 条 |
|------|------|--------|-------------|---------------|
| report_YYYYMMDD.md | x/10 | 高/中/低 | ... | ... |
| market_review_YYYYMMDD.md | x/10 | 高/中/低 | ... | ... |
```

各报告再写 **3 条** bullet：亮点 / 硬伤 / 明日阅读重点。禁止把「包含技术分析字样」当亮点。

---

## Phase 4 — 近 5 个交易日回溯

### 取样

1. 在 `reports/` 下列出 `report_*.md`，按日期降序。
2. 取最近 **5 个有 report 文件的日期**（不要求连续日历日；无 report 的日期跳过）。
3. 对每个日期、每只自选股提取：`operation_advice`、`trend_prediction`、`sentiment_score`、理想买/止损/目标（来自 report 或 DB）。

### 只读 DB 查询（预测 + 涨跌）

在项目根目录执行：

```bash
python -c "
from datetime import datetime, timedelta
from src.storage import DatabaseManager

db = DatabaseManager.get_instance()
code = '601138'  # 替换
rows = db.get_analysis_history(code=code, days=30, limit=10)
for r in rows:
    print(r.created_at.date(), r.operation_advice, r.trend_prediction, r.sentiment_score)

bars = db.get_latest_data(code, days=10)
for b in bars:
    print(b.date, b.close, b.pct_chg)
"
```

> 日线方法为 `DatabaseManager.get_latest_data(code, days=N)` 或 `get_data_range(code, start, end)`，以 [src/storage.py](src/storage.py) 为准；**只读**，不改 DB。

### 方向命中（粗判，写入 note）

- 建议含「看多/持有/介入」且后续 5 日内跌幅 >3% → `hit: false`（方向偏乐观）
- 建议含「观望/看空」且后续 5 日内涨幅 >5% → `hit: false`（漏机会）
- 其余 → `hit: partial` 或 `true`，并在 note 说明

### 输出表格（journal §4）

```markdown
| 交易日 | 代码 | 当时建议 | 5日内涨跌 | 方向命中 | 复盘价值(1-5) | 若是我会怎么改 |
|--------|------|----------|-----------|----------|---------------|----------------|
| 2026-05-29 | 601138 | 持有 | +1.2% | partial | 4 | ... |
```

**综合段（必填）**

- 5 次运行中 **重复出现的流程 issue**（引用 issue id）
- 报告评分趋势（升/降/平）
- 预测命中率：`命中数 / 可评估数`（样本不足写「样本不足」）
- **仅** 配置 / 流程 / 阅读重点建议；**不写** 代码改动 PR

---

## 写入产物

### 1. `daily_agent_evolution/YYYY-MM-DD_journal.md`

完整模板见下文 [Journal 模板](#journal-模板)。Frontmatter 字段说明见 [daily_agent_evolution/_SCHEMA.md](daily_agent_evolution/_SCHEMA.md)。

### 2. `daily_agent_evolution/YYYY-MM-DD_summary.txt`

≤40 行，结构：

```
日期 / run_status / health / 耗时
Top3 issues（id + 一行）
报告分 report x/10 | market_review x/10
5日回溯：命中率 + 一句结论
最重要 1 条建议
```

### 3. `daily_agent_evolution/experience_db.json`

**增量更新**（本地文件，见 `.gitignore`）。维护：

```json
{
  "daily_runs": [
    {
      "date": "2026-05-31",
      "run_status": "success",
      "report_score": 7,
      "market_review_score": 6,
      "health": "degraded",
      "top_issue_id": "20260531-01"
    }
  ],
  "prediction_samples": [
    {
      "date": "2026-05-29",
      "code": "601138",
      "advice": "持有",
      "trend": "震荡偏多",
      "window_return_pct": 1.2,
      "hit": "partial",
      "note": "..."
    }
  ],
  "open_recommendations": [
    {
      "id": "rec-tushare-rate-limit",
      "category": "config",
      "text": "低积分 Tushare 大盘 trade_cal 限流，复盘依赖 efinance fallback",
      "first_seen": "2026-05-31",
      "last_mentioned": "2026-05-31",
      "status": "open"
    }
  ]
}
```

更新规则：读 JSON → 仅 append/更新对应条目 → 写回；**禁止** 为格式化而重写整个仓库其他文件。

---

## Journal 模板

复制并填充，勿删章节标题：

```markdown
---
date: YYYY-MM-DD
run_status: success | skipped_non_trading_day | failed
run_command: python main.py
duration_sec: 0
stocks: []
report_score: 0
market_review_score: 0
health: healthy | degraded | critical
git_touch: none
---

# 每日运行日志

## 1. 运行摘要

（3–5 句：是否跑分析、成败、报告是否生成、通知是否成功）

## 2. 日志审计

### 2.1 Issues

| id | severity | category | evidence | impact | action |
|----|----------|----------|----------|--------|--------|

### 2.2 已忽略噪声

- ...

## 3. 报告评价

| 报告 | 总分 | 信任度 | 最有用 1 条 | 最大问题 1 条 |
|------|------|--------|-------------|---------------|

**亮点**
- ...

**硬伤**
- ...

**明日阅读重点**
- ...

## 4. 近 5 交易日回溯

| 交易日 | 代码 | 当时建议 | 5日内涨跌 | 方向命中 | 复盘价值(1-5) | 若是我会怎么改 |
|--------|------|----------|-----------|----------|---------------|----------------|

**综合**
- 重复 issue：...
- 评分趋势：...
- 预测命中率：...
- 优化建议（非代码）：...

## 5. 明日/下次建议（最多 5 条）

- [config] ...
- [process] ...
- [human] ...

## 6. experience_db 更新摘要

- daily_runs: +1 ...
- prediction_samples: +N ...
- open_recommendations: 更新 rec-...
```

---

## Summary 模板

```
============================================================
【每日股票分析 — 运维摘要】
============================================================
日期: YYYY-MM-DD
运行: success | skipped | failed
健康: healthy | degraded | critical
耗时: N 秒

Issues (Top 3):
1. [severity] id — 一句话

报告: report x/10 | market_review x/10

5日回溯: 命中率 X/Y — 一句话

最重要建议:
- ...
============================================================
```

---

## 仅复盘模式（未跑 main.py）

当 `run_status: skipped_non_trading_day` 或用户只要「复盘」时：

- Phase 1 跳过
- Phase 2 读最近一份完整日志（若有）
- Phase 3–4 仍对最新 report 与近 5 交易日执行
- journal 中明确 `run_command: none`

---

## 完成自检

- [ ] `git status` 无意外变更（journal/summary/experience_db 已 gitignore；仅编辑 `_SCHEMA.md` 或 `SKILL.md` 时才会有 diff）
- [ ] journal 含 Issues 证据表 + 5 日回溯表 + 综合段
- [ ] 报告评价无关键词堆砌
- [ ] summary.txt ≤40 行
- [ ] experience_db.json 仅增量更新
- [ ] 未修改 `src/`、`.env` 等禁止路径

---

## 相关入口（只读参考）

| 入口 | 用途 |
|------|------|
| [main.py](main.py) | 每日分析主入口 |
| [src/core/trading_calendar.py](src/core/trading_calendar.py) | 交易日判断 |
| [src/storage.py](src/storage.py) | `get_analysis_history`、日线数据 |
| [docs/openclaw-skill-integration.md](docs/openclaw-skill-integration.md) | OpenClaw / API 集成 |
| [daily_agent_evolution/_SCHEMA.md](daily_agent_evolution/_SCHEMA.md) | journal / JSON 字段契约 |
