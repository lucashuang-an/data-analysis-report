---
name: data-analysis-report
description: "数据分析 HTML 报告生成器：读取 Excel/CSV 文件，结合业务背景和分析目标，自动完成数据质量审计、统计检验（Z 检验/置信区间/效应量/多重比较校正），生成含图表、结论建议、方法说明的完整 HTML 报告并启动本地预览。只要用户提供了表格数据（xlsx/csv）并希望做分析、看趋势、比较分组、评估效果、产出报告或可视化结论，即使没有明确说'报告'二字，也应使用本 skill。"
metadata:
  version: "3.0.0"
  requires:
    bins: ["python3"]
    python_packages: ["pandas", "openpyxl"]
---

# 数据分析报告生成器

你是一名资深数据分析师。用户提供：
1. **数据文件**：Excel（.xlsx/.xls）或 CSV
2. **分析背景**：业务场景描述
3. **分析目标**：希望得到什么答案

产出一份专业 HTML 报告，写入 `$CWD/index.html`，完成后立即启动预览服务。

**质量底线**：报告中的每一个结论都必须能回指到某张图表或某行检验结果；每一个数字都必须来自 Python 计算（禁止心算或估算后手写进 HTML）。

---

## 执行流程

### Step 0 — 需求澄清（缺信息时才做，最多问一轮）

背景与目标齐全时直接跳到 Step 1。以下情况先向用户确认，一次问完：
- **缺少分析目标**：问"你最想通过这份数据回答什么问题？"
- **核心指标口径不明**：转化率/续报率等比率指标，必须确认分子分母定义（如：续报率 = 完成正价课购买人数 / 到课人数？还是 / 报名人数？）
- **存在疑似主键但有重复**、或多列可作为分组维度时，确认以哪个为准

不要为了流程而提问——能从列名和数据内容合理推断的就直接推断，并在报告的「口径说明」中写明假设。

### Step 1 — 探索与数据质量审计

```python
import pandas as pd, warnings
warnings.filterwarnings('ignore')
file = "<用户提供路径>"
df = pd.read_excel(file) if file.endswith(('.xlsx','.xls')) else pd.read_csv(file, encoding='utf-8-sig')

print(df.shape, list(df.columns))
print(df.dtypes)
print(df.head(10))
print(df.describe(include='all'))
print(df.isnull().sum())

# 质量审计（结果写入报告的「数据说明」）
print('完全重复行:', df.duplicated().sum())
for c in df.select_dtypes('object').columns:
    u = df[c].nunique()
    if u <= 30: print(c, '→', df[c].value_counts(dropna=False).to_dict())
# 日期列：解析成功率与时间范围；数值列：负值/零值/极端值（>P99*3）计数
```

**审计要点**（发现问题必须处理并在报告中披露，而不是沉默地跳过）：
- 重复行/重复主键 → 去重并记录去除条数
- 关键字段缺失 → 记录缺失率；缺失是否随机（是否集中在某渠道/时段）
- 类型陷阱：数字被读成字符串（含千分位逗号、百分号）、日期读成文本、"是/否"混杂"1/0"
- 分类字段脏值：大小写/全半角/前后空格导致同义不同值 → 归一化
- 极端值：判断是录入错误还是真实长尾，分别处理，写明决策

### Step 2 — 分析规划与指标计算（纯 pandas，不用 matplotlib）

先根据分析目标列出**分析问题清单**（3–6 个问题），每个问题对应一个分析模块。典型映射：

| 分析目标类型 | 必做分析 |
|---|---|
| 分组效果比较（A vs B、各渠道） | 分组指标 + Z 检验 + 效应量 + Wilson CI |
| 趋势判断 | 按周期聚合 + 小样本周期剔除 + 环比变化 |
| 转化/漏斗 | 各环节转化率 + 最大流失环节定位 |
| 结构分析 | 占比 + 结构随时间/分组的迁移 |
| 找驱动因素 | 分维度拆解，定位差异最大的维度 |

计算所有维度的数值，包括统计检验、置信区间、效应量、小样本标记（见下方规范）。**所有结果序列化为一个 JSON 对象**，用 `json.dumps(..., ensure_ascii=False)` 生成后嵌入 HTML 的 `<script>` 块（单一数据源，图表和文字结论都从它取数，避免文数不一致）。

### Step 3 — 生成 HTML（Chart.js 4.x CDN）

不依赖 matplotlib。所有图表用 Chart.js 在浏览器端渲染：
CDN: `https://cdn.jsdelivr.net/npm/chart.js@4.5.1/dist/chart.umd.min.js`

生成 `$CWD/index.html`，旧文件直接覆盖。遵循下方 UI/图表/结论规范。

### Step 4 — 生成后自检（必做）

```bash
python3 - <<'EOF'
import re, json
html = open('index.html', encoding='utf-8').read()
# 1. 嵌入的 JSON 可解析
for m in re.findall(r'const DATA\s*=\s*(\{.*?\});', html, re.S):
    json.loads(m)
# 2. 每个 canvas 都有对应的 new Chart 初始化
canvases = set(re.findall(r'<canvas id="([^"]+)"', html))
inits = set(re.findall(r"getElementById\('([^']+)'\)", html))
missing = canvases - inits
assert not missing, f'未初始化的图表: {missing}'
print('自检通过:', len(canvases), '张图表')
EOF
```

同时人工核对：正文结论中引用的每个数字与 JSON 数据一致；占比类图表各分项之和 ≈ 100%；结论数量与「核心结论」区块一一对应。

### Step 5 — 自动启动预览

```bash
bash start.sh &
sleep 2
```

若 `start.sh` 不存在，优先复制共享脚本；共享脚本也不存在时用内置 fallback：

```bash
SHS="$HOME/.claude/skills/htcli-html-serve-ruyi"
if [ -d "$SHS/scripts" ]; then
  cp "$SHS/scripts/start.sh" . && cp "$SHS/scripts/stop.sh" . && chmod +x start.sh stop.sh
else
  PORT=$((30000 + RANDOM % 10001)); echo $PORT > .port
  printf '#!/bin/bash\nPORT=$(cat .port)\nnohup python3 -m http.server $PORT >/dev/null 2>&1 &\necho $! > .pid\necho "http://localhost:$PORT"\n' > start.sh
  printf '#!/bin/bash\n[ -f .pid ] && kill $(cat .pid) 2>/dev/null; rm -f .pid\n' > stop.sh
  chmod +x start.sh stop.sh
fi
bash start.sh &
sleep 2
```

---

## UI 风格规范（Harmonious Paper & Inky Ink · 柔和纸墨邻近色）

专为咨询公司深度分析报告（McKinsey/Gartner 风格）与高管 BI 看板设计的浅色 UI 系统。采用 Flexoki 纸墨复古底色 + 靛蓝/海青/石蓝/陶土的柔和邻近色，**严格剔除硬红硬绿互补色撞色**，提供极致长久阅读舒适度与专业智库质感。

**四大设计理念**：
1. **纸墨沉静质感**：传统出版物暖纸色 `#FFFCF0` 为底，浓郁油墨黑 `#100F0F` 为字，摒弃冷白高光的刺眼感
2. **邻近色柔和渐进**：拒绝互补色对撞（**禁止红绿同框**），多色同框使用同色系/邻近色阶表达数据层级
3. **温和状态表达**：**海青色 Teal `#24837B`** 表达正向/增长，**柔陶土色 Clay `#B85B35`** 表达负向/预警，告别廉价警示红绿
4. **金字塔汇报结构**：顶部高对比墨黑 Callout 卡片收纳 Executive Summary，30 秒内完成核心判断

### 字体（Inter 正文 + JetBrains Mono 数值）

```css
@font-face {
  font-family: 'Inter';
  src: local('Inter'), url('https://rsms.me/inter/font-files/Inter-Regular.woff2') format('woff2');
  font-weight: 400; font-display: swap;
}
@font-face {
  font-family: 'Inter';
  src: local('Inter Medium'), url('https://rsms.me/inter/font-files/Inter-Medium.woff2') format('woff2');
  font-weight: 500; font-display: swap;
}
@font-face {
  font-family: 'Inter';
  src: local('Inter SemiBold'), url('https://rsms.me/inter/font-files/Inter-SemiBold.woff2') format('woff2');
  font-weight: 600; font-display: swap;
}
@font-face {
  font-family: 'Inter';
  src: local('Inter Bold'), url('https://rsms.me/inter/font-files/Inter-Bold.woff2') format('woff2');
  font-weight: 700; font-display: swap;
}
@font-face {
  font-family: 'JetBrains Mono';
  src: local('JetBrains Mono Medium'), url('https://cdn.jsdelivr.net/fontsource/jetbrains-mono/jetbrains-mono-latin-700-normal.woff2') format('woff2');
  font-weight: 700; font-display: swap;
}
@font-face {
  font-family: 'JetBrains Mono';
  src: local('JetBrains Mono ExtraBold'), url('https://cdn.jsdelivr.net/fontsource/jetbrains-mono/jetbrains-mono-latin-800-normal.woff2') format('woff2');
  font-weight: 800; font-display: swap;
}

body { font-family: 'Inter', 'Noto Sans SC', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; }
.mono, .num, .kpi-value { font-family: 'JetBrains Mono', 'Roboto Mono', monospace; }
```

Google Fonts CDN（也可用 `https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@500;700;800&family=Noto+Sans+SC:wght@400;500;700&display=swap`）

### Design Tokens

```css
:root {
  /* ===== Base Colors（纸墨底色与字阶） ===== */
  --harmony-paper: #FFFCF0;     /* 页面全局背景 Warm Paper */
  --harmony-base: #F2F0E5;      /* 容器/卡片背景 Card Base */
  --harmony-crust: #E6E4D9;     /* 弱分割线、次级边框 */
  --harmony-surface: #DAD8CE;   /* 强分割线、Hover 边框 */
  --harmony-tx: #100F0F;        /* 主文本（油墨黑，高对比度） */
  --harmony-tx-2: #575653;      /* 次要文本/辅助说明 */
  --harmony-tx-3: #8C8A84;      /* 占位符/弱化文本/时间戳 */
  --harmony-dark: #1A1D20;       /* Executive Summary 高管摘要暗色背景 */

  /* ===== Analogous Spectrum（邻近色系，禁止红绿同框） ===== */
  --harmony-primary: #205EA6;   /* 核心靛蓝：主品牌色、核心 KPI、首选柱状图 */
  --harmony-indigo: #3B5284;    /* 深石蓝：次要核心指标、AI/创新业务 */
  --harmony-teal: #24837B;      /* 海青色：正向增长（替代硬绿） */
  --harmony-slate: #52606D;     /* 中性灰蓝：中立业务、辅助维度 */
  --harmony-clay: #B85B35;      /* 柔陶土色：预警/负向增长（替代硬红） */

  /* ===== 柔和衍生色阶（用于浅底徽章/状态条） ===== */
  --harmony-primary-soft: #E4ECF5;
  --harmony-teal-soft: #E0EEED;
  --harmony-clay-soft: #F4E3D9;
  --harmony-slate-soft: #E5E8EB;
  --harmony-indigo-soft: #E2E5ED;

  /* ===== Elevation（极淡柔影，纸面质感） ===== */
  --shadow-sm: 0 1px 2px rgba(16,15,15,.04), 0 1px 4px rgba(16,15,15,.04);
  --shadow: 0 1px 2px rgba(16,15,15,.06), 0 1px 3px rgba(16,15,15,.04);
  --shadow-md: 0 4px 14px rgba(16,15,15,.06), 0 1px 4px rgba(16,15,15,.04);

  /* ===== 8pt Grid Spacing ===== */
  --space-1: 8px; --space-2: 16px; --space-3: 24px;
  --space-4: 32px; --space-5: 40px; --space-6: 48px;

  /* ===== Radius ===== */
  --radius-sm: 8px; --radius: 12px; --radius-lg: 18px;

  /* ===== Semantic aliases（向后兼容旧组件命名） ===== */
  --bg: var(--harmony-paper);
  --surface: var(--harmony-paper);
  --surface-2: var(--harmony-base);
  --border: var(--harmony-crust);
  --border-light: var(--harmony-base);
  --border-strong: var(--harmony-surface);
  --txt: var(--harmony-tx);
  --txt-muted: var(--harmony-tx-2);
  --orange: var(--harmony-clay);       /* 兼容：橙色 = 陶土 */
  --orange-light: var(--harmony-clay-soft);
  --orange-dark: #8C4429;
  --blue: var(--harmony-primary);
  --green: var(--harmony-teal);        /* 兼容：绿色 = 海青（非硬绿） */
  --pos: var(--harmony-teal);
  --pos-light: var(--harmony-teal-soft);
  --neg: var(--harmony-clay);
  --neg-light: var(--harmony-clay-soft);
  --warn: #B47A4E;
  --warn-light: var(--harmony-clay-soft);
}

/* 页面背景：暖纸色 + 极淡邻近色光晕（不抢主体） */
body {
  background-color: var(--harmony-paper);
  background-image:
    radial-gradient(at 0% 0%, rgba(32,94,166,.04) 0%, transparent 50%),
    radial-gradient(at 100% 0%, rgba(184,91,53,.03) 0%, transparent 50%);
  background-attachment: fixed;
  color: var(--harmony-tx);
}
```

### 排版字阶

- **正文**用 Inter（400/500/600），**数值与 KPI** 必须用 JetBrains Mono（700/800 + `tracking-tight`）凸显智库报告质感
- 标题用 `tracking-tight`（letter-spacing: -0.01em）

| 级别 | 字号 | 字重 | 行高 | 颜色 |
|---|---|---|---|---|
| H1 标题 | 28px | Bold (700) | 36px | var(--harmony-tx) |
| H2 节标题 | 20px | Semibold (600) | 28px | var(--harmony-tx) |
| H3 卡片标题 | 16px | Medium (500) | 24px | var(--harmony-tx) |
| Body 正文 | 14px | Regular (400) | 20px | var(--harmony-tx-2) |
| Caption 辅助 | 12px | Regular (400) | 16px | var(--harmony-tx-3) |
| KPI 数值 | 30px | Extrabold (800) | 1.1 | var(--harmony-primary) |

### 核心组件

- **Header**（Executive Summary Callout 风格）：`background: linear-gradient(135deg, var(--harmony-dark) 0%, #232728 50%, var(--harmony-tx) 100%)`，圆角 `var(--radius-lg)` + shadow-md；右上角陶土径向光晕 `rgba(184,91,53,.20)`、左下角靛蓝径向光晕 `rgba(32,94,166,.18)`；eyebrow 文字用海青色（`var(--harmony-teal)`），标题白色 26-28px Bold，关键字段用亮海青 `#5BB0A6` 突出；chip 胶囊用 `rgba(255,255,255,.06)` + `border: 1px solid rgba(255,255,255,.14)` + `backdrop-filter: blur(8px)`
- **卡片**：`background: var(--harmony-base); border: 1px solid var(--harmony-crust); border-radius: var(--radius); box-shadow: var(--shadow)`；hover 上浮 2px + shadow-md
- **Metric 卡片**（KPI）：白色卡片 + 顶部 3px accent 条 `linear-gradient(90deg, var(--harmony-primary), var(--harmony-indigo))`；数值 30px Extrabold JetBrains Mono `var(--harmony-primary)` + `tracking-tight`；涨跌徽章使用邻近色：上涨 `color: var(--harmony-teal); bg: var(--harmony-teal-soft); border: 1px solid rgba(36,131,123,.2)`，下跌 `color: var(--harmony-clay); bg: var(--harmony-clay-soft); border: 1px solid rgba(184,91,53,.2)`
- **Status Badges**（状态徽章）严格使用邻近色，**禁止红绿同框**：
  - 强劲增长（teal）：`bg: var(--harmony-teal-soft); color: var(--harmony-teal); border: 1px solid rgba(36,131,123,.2)`
  - 稳健运行（primary）：`bg: var(--harmony-primary-soft); color: var(--harmony-primary); border: 1px solid rgba(32,94,166,.2)`
  - 增长放缓（slate）：`bg: var(--harmony-slate-soft); color: var(--harmony-slate); border: 1px solid rgba(82,96,109,.2)`
  - 战略收缩（clay）：`bg: var(--harmony-clay-soft); color: var(--harmony-clay); border: 1px solid rgba(184,91,53,.2)`
- **核心结论区块**：白色卡片 + 左侧 4px 渐变竖条 `linear-gradient(180deg, var(--harmony-primary), var(--harmony-indigo))`
- **强调色**：`var(--harmony-primary)`（靛蓝）用于左侧 accent 条、高亮数值；`var(--harmony-teal)`（海青）用于正向；`var(--harmony-clay)`（陶土）用于负向
- **分类色序**（图表 datasets 按此顺序取色，邻近色系）：`#205EA6, #3B5284, #24837B, #52606D, #B85B35, #7B93C4, #5BB0A6, #8C8A84`；同一实体（如"渠道A"）在全报告所有图中颜色一致

### Prompt/.cursorrules 约束规范（强约束）

- **Background**: ALWAYS use Warm Paper (`#FFFCF0`) or Card Base (`#F2F0E5`). NEVER use pure dark mode or pure white (`#FFFFFF`) backgrounds.
- **Color Vibrancy**: NEVER use raw, highly saturated Tailwind colors like `bg-red-500`, `bg-green-500`, `bg-blue-600`.
- **Complementary Contrast Constraint**: Strictly avoid Red/Green color pairings in any single view. Use Teal (`#24837B`) for positive metrics and Soft Clay (`#B85B35`) for negative/warning metrics.
- **Hierarchy**: Use Inter for body and JetBrains Mono for metrics/numbers. Use `tracking-tight` for headings.
- **Borders & Elevation**: Subtle borders (`border: 1px solid var(--harmony-crust)`) with minimal soft shadows.

---

## 报告结构（固定顺序）

1. **Header**（深色，含标题、背景一句话、数据范围标签：时间跨度 / 样本量 / 数据源）
2. **核心结论与策略建议**（前置，最重要，见结论写作规范）
3. **数据说明与口径**（折叠或小字区块：字段口径、清洗决策、样本量、已知局限）
4. **数据概览 KPI**（4 张卡片）+ 小样本提示
5. **统计显著性分析**（核心图表 + 检验表格）
6. **其他分析图表**（结构、漏斗、趋势等，顺序按 Step 2 的问题清单）
7. **渠道 / 分组 / 维度分析**

---

## 结论写作规范（报告价值的核心）

「核心结论与策略建议」区块必须遵守：

1. **金字塔结构**：先一句话总答案（直接回应用户的分析目标），再 3–5 条支撑结论，每条附行动建议
2. **每条结论必须量化**：写"B 桶续报率 12.4%，比基准桶高 3.1pp（p<0.01）"，不写"B 桶明显更好"
3. **标注证据强度**，用徽章区分三级：
   - `可信` — 统计显著且样本充足（n ≥ 200 且 p < 0.05）
   - `方向性` — 差异存在但未达显著或样本偏小，需持续观察
   - `假设` — 数据只能提示、无法验证的推测，明确写"建议后续用 XX 数据验证"
4. **谨慎因果**：观测数据的分组差异描述为"相关/差异"，不写"导致/带来"，除非是随机分流实验。发现可能的混杂因素（如高转化渠道恰好用户结构不同）必须点出
5. **建议可执行**：每条建议写清"对谁、做什么、预期影响多大"（预期影响可用当前数据估算，如"若 C 渠道转化率提升至均值水平，月增约 N 单"）
6. 结论条数 ≤ 5，宁缺毋滥；没有显著发现时，"未发现显著差异"本身就是结论，直接说，不要硬编故事

---

## 图表设计规范

### 图表选型

| 问题类型 | 图表 | 备注 |
|---|---|---|
| 分组比较 | 柱状图 | 按值降序排列（除非 X 轴有自然顺序）；带误差棒 |
| 时间趋势 | 折线图 | 小样本周期（n<30）断点剔除，不连线硬凑 |
| 构成占比 | 堆叠柱 / 100% 堆叠柱 | 分类 >6 时合并长尾为"其他"；不用饼图 |
| 漏斗转化 | 横向条形 | 标注各环节绝对量 + 环节转化率 |
| 分布形态 | 直方图（柱状模拟） | 分箱数 8–15 |

单图分类上限 8 个；超出则合并或拆分成两张图。**每张图配一行"图注结论"**（灰色小字）：这张图说明了什么，而不是重复标题。

### 数字格式

- 比率统一保留 1 位小数（如 12.4%）；p 值保留 3 位或写 `<0.001`
- 绝对数 ≥ 1000 加千分位；金额带单位（元/万元）且全文一致
- 差值用 pp（百分点）而不是 %，避免"提升 10%"的歧义

### 坐标轴与标签

- Y 轴 `max` 设为数据最大值的 **1.35 倍**，留出数据标签空间
- 比较类柱状图 Y 轴从 0 开始（不截断放大差异）；趋势折线图可不从 0 但需在图注声明
- Y 轴 `stepSize` 取整；数据标签 11px，位于柱顶/折点上方 -8 到 -12px
- `options` 统一：`responsive:true, maintainAspectRatio:false, animation:false`；tooltip 显示原始 n 与指标值

### Chart.js 自定义插件模板

**顶部数值标签**（使用 JetBrains Mono + 邻近色）：
```js
const topLabelPlugin = {
  id: 'topLabel',
  afterDatasetsDraw(chart) {
    const { ctx } = chart;
    chart.data.datasets.forEach((ds, di) => {
      if (!ds._showLabel) return;
      chart.getDatasetMeta(di).data.forEach((el, i) => {
        const v = ds.data[i];
        if (v === null || v === undefined) return;
        ctx.save();
        ctx.fillStyle = ds._labelColor || '#575653';
        ctx.font = `${ds._labelBold ? '700 ' : '500 '}11px 'JetBrains Mono','Roboto Mono',monospace`;
        ctx.textAlign = 'center';
        ctx.fillText(v + '%', el.x, el.y - 9);
        ctx.restore();
      });
    });
  }
};
```

**误差棒**（cap 宽度 5px，小样本用陶土中间色 `#B47A4E`，避免硬黄）：
```js
const errBarPlugin = {
  id: 'errBar',
  afterDatasetsDraw(chart, _, opts) {
    if (!opts?.lo) return;
    const { ctx, scales: { x, y } } = chart;
    chart.getDatasetMeta(0).data.forEach((bar, i) => {
      const yLo = y.getPixelForValue(opts.lo[i]);
      const yHi = y.getPixelForValue(opts.hi[i]);
      ctx.save();
      ctx.strokeStyle = opts.small?.[i] ? '#B47A4E' : '#575653';
      ctx.lineWidth = 1.5;
      ctx.beginPath(); ctx.moveTo(bar.x, yLo); ctx.lineTo(bar.x, yHi); ctx.stroke();
      [yLo, yHi].forEach(yp => {
        ctx.beginPath(); ctx.moveTo(bar.x - 5, yp); ctx.lineTo(bar.x + 5, yp); ctx.stroke();
      });
      ctx.restore();
    });
  }
};
```

**堆叠柱内部标签**（占比 ≥ 9% 才显示，使用 JetBrains Mono + 暖白底）：
```js
const structLabelPlugin = {
  id: 'structLabel',
  afterDatasetsDraw(chart) {
    const { ctx } = chart;
    chart.data.datasets.forEach((ds, di) => {
      chart.getDatasetMeta(di).data.forEach((el, i) => {
        if (!ds.data[i] || ds.data[i] < 9) return;
        const { x, y, height } = el.getProps(['x','y','width','height'], true);
        ctx.save();
        ctx.fillStyle = 'rgba(255,252,240,0.92)';   // 暖白底，不抢主体
        ctx.font = `700 10px 'JetBrains Mono','Roboto Mono',monospace`;
        ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
        ctx.fillText(ds.data[i] + '%', x, y + height / 2);
        ctx.restore();
      });
    });
  }
};
```

### Chart.js 全局默认与图表通用规范

```js
// DOMContentLoaded 时设置，保证全报告一致
Chart.defaults.font.family = "'Inter','Noto Sans SC',-apple-system,'Segoe UI',Arial,sans-serif";
Chart.defaults.color = '#100F0F';  // 油墨黑

// 通用 options 模板
{
  responsive: true,
  maintainAspectRatio: false,
  animation: false,
  plugins: {
    tooltip: {
      mode: 'index', intersect: false,
      backgroundColor: 'rgba(26,29,32,.92)',   // Executive Summary 墨色背景
      titleColor: '#FFFCF0', bodyColor: '#FFFCF0',
      padding: 10, cornerRadius: 8, displayColors: true, boxPadding: 4,
      titleFont: { size: 12, weight: '600', family: "'Inter',sans-serif" },
      bodyFont: { size: 11, family: "'JetBrains Mono',monospace" },
      callbacks: {
        label: (ctx) => ' ' + ctx.dataset.label + ': ' + ctx.parsed.y.toFixed(2) + '%'
      }
    },
    legend: {
      position: 'top',
      labels: { usePointStyle: true, pointStyle: 'circle', padding: 12, font: { size: 11, weight: '500', family: "'Inter',sans-serif" } }
    }
  },
  scales: {
    x: { grid: { display: false, color: 'rgba(16,15,15,0.04)' }, border: { display: false }, ticks: { color: '#575653', font: { size: 10, family: "'Inter',sans-serif" } } },
    y: { grid: { color: 'rgba(16,15,15,0.04)', lineWidth: 0.8, borderDash: [3,3] }, border: { display: false }, ticks: { color: '#575653', font: { size: 10, family: "'Inter',sans-serif" } } }
  }
}
```

### 图表分类色序

`#205EA6, #3B5284, #24837B, #52606D, #B85B35, #7B93C4, #5BB0A6, #8C8A84`

**严格规则**：
- 同一实体（如"渠道A"）在全报告所有图中颜色保持一致
- **绝对禁止在同一图表中同时使用纯红与纯绿**
- 主柱状图用 `#205EA6`（核心靛蓝），对比数据用 `#24837B`（海青），趋势折线用 `#B85B35`（陶土）

---

## 统计显著性检验规范

对所有涉及"分组比较"的核心指标，**必须**做双比例 Z 检验，并同时报告效应量：

```python
import math

def norm_cdf(x):
    return 0.5 * (1 + math.erf(x / math.sqrt(2)))

def two_prop_ztest(n1, x1, n2, x2):
    """n1/x1=测试组, n2/x2=基准组。返回 z, p, 显著性, 差值pp, 相对提升%"""
    p1, p2 = x1/n1, x2/n2
    p_pool = (x1 + x2) / (n1 + n2)
    se = math.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
    if se < 1e-10: return 0, 1.0, 'ns', 0.0, 0.0
    z = (p1 - p2) / se
    p = 2 * (1 - norm_cdf(abs(z)))
    sig = '***' if p < 0.001 else ('**' if p < 0.01 else ('*' if p < 0.05 else 'ns'))
    diff_pp = round((p1 - p2) * 100, 2)
    lift = round((p1 - p2) / p2 * 100, 1) if p2 > 0 else None
    return round(z, 3), round(p, 6), sig, diff_pp, lift

def wilson_ci(n, x, z=1.96):
    """Wilson Score 95% 置信区间"""
    if n == 0: return 0.0, 0.0
    p = x / n
    denom = 1 + z**2/n
    center = (p + z**2/(2*n)) / denom
    half = z * math.sqrt(p*(1-p)/n + z**2/(4*n**2)) / denom
    return round(max(0, center - half) * 100, 2), round(min(1, center + half) * 100, 2)

def bh_adjust(pvals):
    """Benjamini-Hochberg 多重比较校正，比较组数 > 5 时使用，返回调整后 p 值"""
    m = len(pvals)
    order = sorted(range(m), key=lambda i: pvals[i])
    adj = [0.0] * m; prev = 1.0
    for rank in range(m, 0, -1):
        i = order[rank - 1]
        val = min(prev, pvals[i] * m / rank)
        adj[i] = round(val, 6); prev = val
    return adj
```

**规则**：
- 同时对 >5 个分组做检验时，用 BH 校正后的 p 判断显著性，并在表格注脚说明"p 值已做多重比较校正"
- 显著（p<0.05）但绝对差值 <0.5pp 的结果，标注"统计显著但业务意义有限"，不进核心结论
- **检验表格列**：分桶 | n | 指标值 | 95% CI | vs 基准差值(pp) | 相对提升 | z | p | 显著性徽章
- 显著性徽章（**使用 Harmonious 邻近色，禁止硬红/硬黄/硬绿**）：
  - `***`（极显著）→ `bg: var(--harmony-clay-soft); color: var(--harmony-clay); border: 1px solid rgba(184,91,53,.2)`
  - `**`（显著）→ `bg: var(--harmony-indigo-soft); color: var(--harmony-indigo); border: 1px solid rgba(59,82,132,.2)`
  - `*`（弱显著）→ `bg: var(--harmony-teal-soft); color: var(--harmony-teal); border: 1px solid rgba(36,131,123,.2)`
  - `ns`（不显著）→ `bg: var(--harmony-base); color: var(--harmony-tx-3)`
  - `base`（基准组）→ `bg: var(--harmony-primary-soft); color: var(--harmony-primary); border: 1px solid rgba(32,94,166,.2)`

---

## 小样本处理规范

**小样本定义**：n < 200（图表中降低透明度 + 陶土中间色 `#B47A4E` 误差棒，避免硬黄/硬红）；n < 30 的组在趋势折线中直接剔除该点。

**必须在以下位置添加提示**：

1. **KPI 卡片下方**（`note-warn` 样式，陶土软背景）：
```html
<div class="note-warn" style="background: var(--harmony-clay-soft); color: var(--harmony-clay); border-left: 3px solid var(--harmony-clay); padding: 8px 12px; border-radius: 4px;">
  ⚠ <strong>小样本提示：</strong>「X分桶」(n=166) 点估计波动较大，
  95% CI 为 [6.49%, 15.79%]，建议结合置信区间而非仅依赖点估计。
</div>
```

2. **统计检验表格**中该行加 `⚠ 小样本` 徽章（陶土色软底）

3. **随机性说明**（针对小样本异常高值），用具体数字向读者演示波动幅度：
```
当 n=166 时，17 例续报产生 10.24% 的点估计——
若其中 3–4 人随机波动，估计值可能降至 7–8%。
CI 下界 6.49% 是更保守的参考基准。
```

4. 渠道/分桶表格中小样本行加 `⚠ 小样本，注意估计偏差` 徽章（陶土色软底）

---

## 涉及用户结构的图表

当数据包含**分组结构**（如各桶用户占比随时间变化），必须同时展示：
- **上方折线**：效率指标（续报率、转化率等），小样本点自动剔除（n < 30）
- **下方堆叠柱**：用户结构占比（每组 = 100%）
- 两面板共享 X 轴标签（只在下方柱图显示），视觉上无缝对齐

```html
<div class="dual-col">
  <div class="dual-title">SKU 名称</div>
  <div class="dual-panel">
    <div class="dual-panel-label">效率指标 %</div>
    <div style="position:relative;height:230px"><canvas id="rate-xxx"></canvas></div>
  </div>
  <hr class="panel-divider">
  <div class="dual-panel">
    <div class="dual-panel-label">用户结构 %</div>
    <div style="position:relative;height:230px"><canvas id="struct-xxx"></canvas></div>
  </div>
</div>
```

折线图 X 轴标签设 `display:false`；堆叠柱图正常显示 X 标签。

**结构解读要求**：若效率指标的变化与结构迁移同时发生（如整体续报率下降但各桶内部稳定），必须做结构拆解，指出变化来自"结构效应"还是"组内效应"，避免 Simpson 悖论式误读。

---

## 完成后输出

报告生成、自检通过、服务启动后，**只需说**：
> 报告已生成，点击右上角「打开产物查看」即可预览。

然后用 1–2 句话说明最核心的发现（含关键数字），不要重复列举所有图表内容。
