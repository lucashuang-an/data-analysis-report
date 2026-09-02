# data-analysis-report

[中文](#chinese) &nbsp;|&nbsp; [English](#english)

---

<a id="chinese"></a>
## 中文

一个 Claude Skill：读取 Excel/CSV 数据文件，结合业务背景和分析目标，自动完成数据质量审计、统计检验（Z 检验 / 置信区间 / 效应量 / 多重比较校正），并生成一份包含图表、结论建议、方法说明的完整 HTML 分析报告。

### 这个 Skill 做什么

只要你提供表格数据（xlsx/csv）并希望做分析、看趋势、比较分组、评估效果、产出报告或可视化结论，这个 Skill 就会被触发——即使你没有明确说"报告"两个字。

核心流程：

1. **需求澄清**（信息缺失时才问，最多一轮）——确认分析目标、核心指标口径
2. **数据质量审计**——重复行、缺失率、类型陷阱、脏值归一化、极端值处理，全部披露而非静默跳过
3. **分析规划与指标计算**——纯 pandas 计算，覆盖分组比较、趋势、转化/漏斗、结构分析、驱动因素定位等场景，包含统计检验、置信区间、效应量、小样本标记
4. **生成 HTML 报告**——用 Chart.js 4.x 在浏览器端渲染图表，遵循统一的视觉规范（"柔和纸墨邻近色" Harmonious Paper & Inky Ink 设计系统）
5. **生成后自检**——确保每个结论都能回指到具体图表或检验结果，每个数字都来自 Python 计算
6. **自动启动本地预览**

### 质量底线

- 报告中的每一个结论都必须能回指到某张图表或某行检验结果
- 每一个数字都必须来自 Python 计算，禁止心算或估算后手写进 HTML

### 依赖

- `python3`
- Python 包：`pandas`, `openpyxl`

### 使用方法

将本仓库的 `SKILL.md` 放入 Claude 的 skills 目录（例如 `~/.claude/skills/data-analysis-report/SKILL.md`，具体路径以你所用的 Claude 客户端/环境为准），Claude 在检测到相关数据分析需求时会自动加载并遵循其中的执行规范。

> `SKILL.md` 的执行指令主体是中文（开发与测试语言），生成的**报告本身**语言不受限——见下方示例中的英文版报告。

### 示例

[`examples/telco-churn-report/`](./examples/telco-churn-report/) —— 基于公开的 **IBM Telco Customer Churn** 数据集生成的完整英文报告，展示了这个 skill 的核心能力：Z 检验 + Wilson 置信区间、Benjamini–Hochberg 多重比较校正、"效率 + 结构"双面板图表（避免辛普森悖论式误读）、混淆因素识别与谨慎因果表述。

👉 [直接查看报告](https://lucashuang-an.github.io/data-analysis-report/examples/telco-churn-report/index.html)

### 目录结构

```
data-analysis-report/
├── SKILL.md              # skill 的完整定义与执行规范
├── README.md
└── examples/
    └── telco-churn-report/
        ├── index.html            # 示例报告（英文）
        ├── telco-customer-churn.csv
        └── README.md
```

---

<a id="english"></a>
## English

A Claude Skill that reads Excel/CSV files and, combined with business context and an analysis goal, automatically performs data-quality auditing, statistical testing (Z-tests, confidence intervals, effect sizes, multiple-comparison correction), and produces a complete HTML analysis report with charts, findings, and a methodology note.

### What this skill does

Triggers whenever you provide tabular data (xlsx/csv) and want analysis, trend-spotting, group comparisons, impact evaluation, a report, or visualized conclusions — even if you never say the word "report".

Core pipeline:

1. **Clarification** (only when information is missing, at most one round) — confirms the analysis goal and how core metrics are defined
2. **Data-quality audit** — duplicate rows, missing rates, type traps, dirty categorical values, outliers; all issues are disclosed, never silently skipped
3. **Analysis planning & metric computation** — pure pandas, covering group comparisons, trends, conversion/funnels, structural analysis, and driver identification, with statistical tests, confidence intervals, effect sizes, and small-sample flags
4. **HTML report generation** — charts rendered client-side with Chart.js 4.x, following a consistent visual system ("Harmonious Paper & Inky Ink")
5. **Post-generation self-check** — every conclusion must trace back to a chart or test result; every number must come from Python computation
6. **Automatic local preview**

### Quality bar

- Every conclusion in the report must be traceable to a specific chart or test result
- Every number must come from Python computation — no mental math or hand-written estimates

### Requirements

- `python3`
- Python packages: `pandas`, `openpyxl`

### Usage

Place this repository's `SKILL.md` into Claude's skills directory (e.g. `~/.claude/skills/data-analysis-report/SKILL.md`, the exact path depends on your Claude client/environment). Claude will automatically load and follow it when it detects a relevant data-analysis request.

> The operating instructions inside `SKILL.md` are written in Chinese (the language it was developed and tested in). The language of the **generated report** is not constrained by that — see the English example below.

### Example

[`examples/telco-churn-report/`](./examples/telco-churn-report/) — a full English-language report generated on the public **IBM Telco Customer Churn** dataset, showcasing the skill's core capabilities: two-proportion Z-tests with Wilson confidence intervals, Benjamini–Hochberg correction for multiple comparisons, a dual-panel "efficiency + structure" chart (to avoid a Simpson's-paradox-style misread), and explicit confounder callouts with cautious causal language.

👉 [View the report](https://lucashuang-an.github.io/data-analysis-report/examples/telco-churn-report/index.html)

### Directory structure

```
data-analysis-report/
├── SKILL.md              # full skill definition and execution rules
├── README.md
└── examples/
    └── telco-churn-report/
        ├── index.html            # example report (English)
        ├── telco-customer-churn.csv
        └── README.md
```
