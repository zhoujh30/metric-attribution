# AB Attribution Skill

A Claude Code skill for diagnosing AB experiment anomalies in e-commerce — specifically built for the classic "traffic went up but GMV went down" problem.

**Trigger phrases:**
- "Help me analyze this AB test data"
- "The experiment result is bad, help me figure out why"
- "Traffic increased but GMV didn't grow"

---

## What It Does

Guides Claude through a structured 5-phase analysis:

| Phase | Name | Description |
|-------|------|-------------|
| 0 | Requirements Alignment | Clarify the research question, analysis dimensions, and action direction **before writing any code** |
| 1 | Data Processing | Compute PV / GMV / GPM / CTR / CO / AOV across experiment vs. control |
| 2 | Four-Quadrant Classification | Classify all SKUs into A (double-up) / B (efficiency gain) / C (double-down) / D (high traffic, low conversion) |
| 3 | Multi-Dimensional Drill-Down | Identify top contributors to negative GMV across category / price tier / user type / seller tier / dimension combos |
| 4 | GPM Mix-Rate Decomposition | Separate structural effects (traffic mix shift) from efficiency effects (same-category conversion drop) |
| 5 | Action Plan | Generate prioritized recommendations: A-class expansion pool + D-class blacklist + P0-P3 priority ranking |

## Output Format

- **Interactive HTML dashboard** — dark theme, sticky tabs, Chart.js charts, insight boxes
- **Excel blacklist** — full D-class SKU list with disposal priority (immediate remove / strongly recommend remove / rate-limit & observe)

## Example Output

See [examples/analysis.html](examples/analysis.html) for a complete dashboard generated from a real anonymized dataset (1,139 SKUs, Exp PV +29.5%, GMV -1.4%, GPM -24%).

## Project Structure

```
ab-attribution/
├── SKILL.md           # Core skill definition (triggers, phases, output specs, checklist)
├── examples/
│   └── analysis.html  # Example interactive dashboard
└── README.md
```

## How to Use

1. Place `SKILL.md` in your Claude Code skills directory (typically `~/.claude/skills/`)
2. Upload your AB experiment data (Excel or CSV with PV / Click / Order / GMV / Subsidy per product per version)
3. Say any of the trigger phrases above
4. Claude will run Phase 0 to align on your research question before generating the analysis

## Roadmap

- [ ] Python analysis template (`templates/analysis.py`) for direct data processing
- [ ] Support for non-e-commerce AB tests (SaaS metrics: activation, retention, revenue)
- [ ] English version of SKILL.md

## Contributing

Issues and PRs are welcome. This skill is designed to be extended — common extensions include:

- Additional drill-down dimensions (city tier, device type, traffic source)
- New action modules (cohort analysis, holdout validation)
- Custom chart themes

## License

MIT — see [LICENSE](LICENSE)

---

## 中文文档

**AB 实验归因分析 Skill** — 专为电商场景下"流量涨但 GMV 没涨"问题设计的 Claude Code 技能。

### 触发方式

直接上传实验数据后说：
- "帮我分析下这个 AB 数据"
- "实验效果不好帮我看看为什么"
- "流量涨了但 GMV 没涨"

### 分析流程

先经过 **Phase 0 需求对齐**（锁定研究问题 + 分析维度 + 行动方向），再进入 **Phase 1-5 分析执行**（指标计算 → 四象限分类 → 多维下钻 → GPM Mix-Rate 拆解 → 行动方案）。

分析执行阶段完全根据 Phase 0 的对齐结果动态调整，不是固定模板。

### 输出

- 纯 HTML 交互看板（无需 React，Chart.js CDN）
- Excel 黑名单清单（含处置优先级分档）

详见 [SKILL.md](SKILL.md)。
