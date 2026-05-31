# daily_agent_evolution 字段契约

本文件是 [SKILL.md](../SKILL.md) 中 journal / summary / experience_db 的字段真源镜像。Agent 写入时须与本契约一致。

---

## 文件命名

| 文件 | 说明 |
|------|------|
| `YYYY-MM-DD_journal.md` | 当日主记录 |
| `YYYY-MM-DD_summary.txt` | 当日短摘要（≤40 行） |
| `experience_db.json` | 跨日结构化经验库（本地，不入库） |

---

## journal frontmatter

| 字段 | 类型 | 取值 |
|------|------|------|
| `date` | string | `YYYY-MM-DD` |
| `run_status` | enum | `success` \| `skipped_non_trading_day` \| `failed` |
| `run_command` | string | 如 `python main.py`、`python main.py --force-run`、`none` |
| `duration_sec` | number | Phase 1 耗时秒数；未跑则为 `0` |
| `stocks` | list[string] | 自选股代码列表 |
| `report_score` | number | 1–10，个股 report 加权总分 |
| `market_review_score` | number | 1–10；无大盘报告则为 `0` |
| `health` | enum | `healthy` \| `degraded` \| `critical` |
| `git_touch` | string | 默认 `none`；若误改其他路径须如实填写 |

---

## journal 章节（固定顺序，不可删）

1. 运行摘要  
2. 日志审计（Issues 表 + 已忽略噪声）  
3. 报告评价（表 + 亮点/硬伤/明日阅读重点）  
4. 近 5 交易日回溯（表 + 综合）  
5. 明日/下次建议（≤5 条，带 `[config]` / `[process]` / `[human]` 前缀）  
6. experience_db 更新摘要  

---

## Issue 表列

| 列 | 说明 |
|----|------|
| `id` | `{YYYYMMDD}-{序号}`，如 `20260531-01` |
| `severity` | `critical` \| `high` \| `medium` \| `low` |
| `category` | 见 SKILL.md Phase 2 类型表 |
| `evidence` | 日志原文一句 |
| `impact` | 影响范围 |
| `action` | 具体下一步 |

---

## experience_db.json 增量字段

字段说明：

### `daily_runs[]`

```json
{
  "date": "YYYY-MM-DD",
  "run_status": "success",
  "report_score": 7,
  "market_review_score": 6,
  "health": "degraded",
  "top_issue_id": "20260531-01"
}
```

同一 `date` 重复运行：更新该条而非 duplicate。

### `prediction_samples[]`

```json
{
  "date": "YYYY-MM-DD",
  "code": "601138",
  "advice": "持有",
  "trend": "震荡偏多",
  "window_return_pct": 1.2,
  "hit": "true | false | partial | unknown",
  "note": "简短说明"
}
```

### `open_recommendations[]`

```json
{
  "id": "rec-slug",
  "category": "config | process | human",
  "text": "具体建议",
  "first_seen": "YYYY-MM-DD",
  "last_mentioned": "YYYY-MM-DD",
  "status": "open | resolved"
}
```

---

## summary.txt 结构

单行键值 + Top3 issues + 报告分 + 5 日命中率一句 + 最重要建议；总行数 ≤40。

---

## 写入边界

仅允许修改 `daily_agent_evolution/**`。违反时 journal 的 `git_touch` 不得为 `none`。
