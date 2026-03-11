# Metric Attribution Skill

A Claude Code skill for diagnosing **why a metric changed** — built for e-commerce, extensible to other domains.

Handles two scenarios:
- **AB Attribution**: experiment group vs. control group (e.g., "traffic went up but GMV went down")
- **AA Attribution**: no explicit control group, construct a reference via period comparison, peer group, DiD, or synthetic control (e.g., "why did GMV drop this week?")

**Trigger phrases:**
- "Help me analyze this AB test data"
- "Why did GMV drop?"
- "Traffic increased but GMV didn't grow"
- "I want to do an attribution analysis"
- "Help me figure out why this metric changed"

---

## What It Does

### Phase 0: Requirements First
Before any code, Claude asks 4 structured questions (background / observations / action intent / data availability), determines whether this is AB or AA, and for AA recommends the best comparison method. Analysis only starts after a confirmed plan.

### Phase 1–5: Adaptive Analysis Pipeline

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Data Processing | Compute PV / GMV / GPM / CTR / CO / AOV for both groups |
| 2 | Adaptive Classification | **2 metrics → 4 quadrants** (A/B/C/D); **1 metric → 2 segments** (positive / negative contribution) |
| 3 | Multi-Dimensional Drill-Down | Rank contributors by dimension: category (L1→L2→L3) / price tier / user type / seller tier / combos |
| 4 | Effect Decomposition | **4-quadrant**: GPM Mix-Rate decomposition; **2-segment**: contribution ranking + structure/efficiency split on top negatives |
| 5 | Action Plan | Modular output: expansion pool / blacklist / algorithm tuning / traffic reallocation — P0–P3 priority ranking |

## Output Format

- **Interactive HTML dashboard** — shareable via GitHub Secret Gist link (not searchable, accessible by link only)
- **Excel blacklist** — full negative SKU list with disposal tiers (immediate remove / strongly recommend remove / rate-limit & observe) + common trait summary for future sourcing rules

## Example Output

See [examples/analysis.html](examples/analysis.html) — complete dashboard from a real anonymized dataset (1,139 SKUs, Exp PV +29.5%, GMV -1.4%, GPM -24%).

## Project Structure

```
metric-attribution/
├── SKILL.md           # Core skill: phases, methods, output specs, checklist
├── examples/
│   └── analysis.html  # Example interactive dashboard
└── README.md
```

## How to Use

1. Place `SKILL.md` in your Claude Code skills directory (typically `~/.claude/skills/`)
2. Say any trigger phrase — Claude will ask 4 questions before requesting data
3. Upload your data when prompted (Excel / CSV with per-product metrics per group/period)
4. Confirm the analysis plan, then Claude executes Phase 1–5

**Optional — enable shareable Gist links:**
Set a GitHub personal access token (`gist` scope) in your environment:
```bash
export GITHUB_TOKEN=ghp_your_token_here
```

## Roadmap

- [ ] Python analysis template (`templates/analysis.py`)
- [ ] AA method: synthetic control implementation
- [ ] Support for non-e-commerce metrics (activation, retention, revenue per user)
- [ ] English version of SKILL.md

## Contributing

Issues and PRs are welcome. Common extensions:
- Additional drill-down dimensions (city tier, device type, traffic source)
- New action modules (cohort analysis, holdout validation)
- Industry-specific metric presets

## License

MIT — see [LICENSE](LICENSE)

---

## 中文文档

**指标归因分析 Skill** — 定位"某指标为什么变了"，电商场景为主，方法论可扩展至其他行业。

### 触发方式

说出意图即可开始，Claude 会先问 4 个问题再要数据：
- "帮我分析下这个 AB 数据"
- "为什么 GMV 跌了"
- "流量涨了但 GMV 没涨"
- "我想做个归因分析"

### 两种分析模式

- **AB 归因**：有显性实验组/对照组，2 指标→4 象限（ABCD），GPM Mix-Rate 拆解
- **AA 归因**：无显性对照组，构造参照（周期对比/横向对照/DiD/合成控制），1 指标→正向/负向贡献分段

### 输出

- HTML 交互看板，通过 GitHub Secret Gist 生成可分享链接（不可被搜索）
- Excel 黑名单清单（含处置优先级分档 + 共性特征总结）

详见 [SKILL.md](SKILL.md)。
