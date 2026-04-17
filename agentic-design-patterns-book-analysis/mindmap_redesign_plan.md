# SVG 脑图详细改造方案

> 角色定位：P8 方案负责人。本文件只给出可实施方案，不修改 `mindmap.html`、`analyze.md` 或其他实现文件。
> 目标文件：`agentic-design-patterns-book-analysis/mindmap.html`
> 内容源：`agentic-design-patterns-book-analysis/analyze.md`

## A. 目标与边界

### A1. 改造目标

1. **增强分块感**：4 个 Part 在颜色、背景分区、路径、节点形态上形成稳定视觉分组，避免相邻章节“看起来都差不多”。
2. **增强层级感**：中心主题、Part、章节、组件、技术、记忆口诀、风险边界要有明确层级，不再只靠节点大小和线条粗细区分。
3. **减少图上与右侧重复**：图上承担“背诵骨架”，右侧承担“补充说明 / 复盘检查 / 证据详情”，两者分工明确。
4. **让图可直接记忆**：默认总览时，用户不点击右侧也能看见每章的关键记忆内容，至少能背出“章名、口诀、组件类目、技术类目、风险提醒”。
5. **保留真正 SVG 脑图结构**：仍然是中心主题 → 4 个 Part 主干 → 21 章分支 → 组件 / 技术 / 记忆节点细枝，不能变成卡片墙、表格或静态海报。
6. **保留现有基础能力**：继续保留 4 个 Part、21 章、搜索、缩放、横向滚动、右侧详情、图上点击、键盘可访问。

### A2. 不改范围

1. **不改变内容事实**：章节、组件、技术、落地、风险、口诀仍以 `analyze.md` 与 `mindmapData` 当前内容为准，不新增未经内容源支撑的新概念。
2. **不改成非 SVG 实现**：不得将主图替换为 HTML 卡片流、canvas 或图片；可在 SVG 内增加 `g`、`rect`、`path`、`text`、`foreignObject`，但主结构必须仍是 SVG。
3. **不移除右侧详情**：右侧 `renderInspector` 仍保留，但从“主要信息承载区”降级为“解释 / 校验 / 复盘区”。
4. **不牺牲缩放与滚动**：`applyZoom`、`adjustZoom`、`resetZoom`、`centerViewport` 这组能力继续存在，只调整画布尺寸、默认缩放、聚焦行为。
5. **不做实现级重构冒进**：优先在现有单文件架构内改造，避免引入构建工具、外部依赖、框架或异步数据加载。

### A3. 当前问题定位

1. **颜色问题**：当前 Part 主色为紫、蓝、橙、绿，但章节节点统一白底，叶节点组件统一浅紫、技术统一浅蓝；视觉上 Part 只体现在边框和路径，章节之间的分块边界弱。
2. **信息问题**：`leafItems` 只取 `components.slice(0, 2)` 和 `techniques.slice(0, 2)`，导致图上只出现部分组件 / 技术；右侧 `renderInspector` 又完整展示，造成“图不完整、侧栏重复”的体验。
3. **记忆问题**：章节节点目前主要显示“第几章 + 标题 + 口诀”，组件 / 技术只是散落短叶；没有形成可背诵的固定模板。
4. **交互问题**：点击章节后只是右侧显示详情，图上路径和同组节点缺少强引导；用户需要依赖右侧才能看全。
5. **布局问题**：当前 `canvas = { width: 2860, height: 1480 }`、章节集中在左右两列，叶节点高度只有 30，适合少量标签，不适合更多完整信息。

## B. 信息架构重构

### B1. 总体原则：图上记忆，侧栏解释

图上与侧栏不再承载同一份内容：

| 层级 | 图上显示 | 侧栏显示 | 目的 |
| --- | --- | --- | --- |
| 中心 | 主题、总口诀、四段主线 | 全书总记忆、附录工具箱、学习路径说明 | 建立全书主线 |
| Part | Part 名、章节范围、Part 口诀、阶段动词 | Part 内章节列表、阶段说明、复习顺序 | 建立分块 |
| 章节 | 章号、章名、定义压缩句、记忆口诀、组件/技术计数 | 完整定义、完整组件、完整技术、落地、风险 | 图上可背，侧栏可查 |
| 组件 | 全量组件分组显示，最多每章 4-5 个 | 组件解释可后续补充 | 背系统组成 |
| 技术 | 全量技术分组显示，最多每章 5-6 个 | 技术解释可后续补充 | 背实现抓手 |
| 风险 | 图上显示 1 条风险短句或风险图标 | 完整风险边界 | 记住边界 |

### B2. 图上每章固定为“五行记忆卡”

章节节点从当前 72px 高度的小节点，升级为 SVG 内的“记忆卡节点”，每章固定呈现：

1. **标题行**：`第 N 章 · 章名`
2. **定义行**：压缩到 18-24 个中文字符，例如“复杂任务拆成顺序步骤”
3. **组件行**：`组件：任务拆分器 / 提示契约 / ...`
4. **技术行**：`技术：顺序执行 / 结构化输出 / ...`
5. **口诀 / 风险行**：优先显示口诀；记忆模式下切换为风险短句

实施时不建议把所有内容都塞进现有 `subtitle`。应扩展 `addRoundedNode` 为支持多区域文本，例如：

- `titleLines`
- `definitionLine`
- `componentItems`
- `techniqueItems`
- `mottoLine`
- `riskLine`
- `mode`

如果希望保持 `addRoundedNode` 简洁，可新增 `addChapterMemoryCard(chapter)` 专门绘制章节记忆卡，保留 `addRoundedNode` 给中心、Part、叶节点使用。

### B3. 组件 / 技术叶节点从“抽样”改为“全量可见 + 可折叠”

当前 `leafItems(chapter)` 每章最多显示 4 个叶节点，这是图不完整的根因。建议改为：

1. **默认总览**：每章显示全量组件与全量技术，但以“组节点”方式节省空间：
   - 组件组：一个宽节点，内部多枚小标签，显示全部 `chapter.components`
   - 技术组：一个宽节点，内部多枚小标签，显示全部 `chapter.techniques`
2. **展开章节**：点击章节后，将该章组件 / 技术组展开为独立叶节点，路径逐条高亮。
3. **折叠章节**：再次点击或点击“显示全部”恢复组节点。

这样默认图上能背全量内容，展开时又保留脑图细枝结构。

### B4. 侧栏改为“检查清单 + 深层解释”

`renderInspector` 当前完整重复组件和技术。改造后：

1. **Part 侧栏**：
   - 保留章节跳转按钮。
   - 新增“本 Part 背诵顺序”：按章节列出口诀，不重复组件 / 技术。
   - 新增“分块辨识”：说明这个 Part 的阶段动词，例如 Part 1 是“拆、判、并、审、用、规、协”。
2. **Chapter 侧栏**：
   - 顶部显示“你应该能从图上背出”的检查清单：标题、定义、组件、技术、口诀、风险。
   - 完整组件 / 技术仍显示，但改成“解释区”，不再作为唯一信息来源。
   - `典型落地` 和 `风险边界` 保留在侧栏，因为它们是上下文解释，图上只显示短锚点。
3. **Center 侧栏**：
   - 展示全书总口诀、4 个 Part 口诀、附录工具箱。
   - 新增“读图方法”：先看颜色分区，再背章卡，再看组件 / 技术组。

## C. 视觉系统

### C1. Part 色板重建

当前相邻章节颜色接近的核心原因不是主色太近，而是章节和叶子没有继承 Part 色系。建议为每个 Part 定义完整色阶：

```js
palette: {
  trunk: '#6d28d9',
  branch: '#7c3aed',
  soft: '#f3e8ff',
  softer: '#faf5ff',
  border: '#8b5cf6',
  text: '#3b0764',
  accent: '#c084fc'
}
```

推荐色板：

| Part | 语义 | trunk | soft | text | 说明 |
| --- | --- | --- | --- | --- | --- |
| Part 1 | 工作流 / 协作 | `#7c3aed` 紫 | `#f3e8ff` | `#3b0764` | 表示编排与结构 |
| Part 2 | 状态 / 学习 / 控制 | `#0284c7` 蓝 | `#e0f2fe` | `#075985` | 表示状态流和反馈 |
| Part 3 | 可靠 / HITL / RAG | `#d97706` 橙 | `#fffbeb` | `#78350f` | 表示风险、证据和稳定性 |
| Part 4 | 协议 / 优化 / 高级智能 | `#059669` 绿 | `#ecfdf5` | `#064e3b` | 表示扩展、优化和探索 |

实现位点：

- `mindmapData.parts[*].color` 附近增加 `palette` 字段。
- `renderLegend` 使用 `part.palette.trunk` 与 `part.palette.soft`。
- `renderSvg` 中所有 `part.color` 替换为语义色，如 `part.palette.branch`、`part.palette.trunk`。

### C2. 分区背景：4 个 Part 独立色块

在 `linksLayer` 后、`nodesLayer` 前新增 `zonesLayer` 或在 `nodesLayer` 最底部绘制四个半透明区域：

1. 左上：Part 1
2. 左下：Part 2
3. 右上：Part 3
4. 右下：Part 4

每个区域使用大圆角矩形或轻微弧形背景：

- 填充：`part.palette.softer`
- 边框：`part.palette.border`，透明度 0.16
- 标题水印：`PART 1 · 基础工作流与协作`

分区背景不能替代路径，只用于增强分块。路径仍从中心到 Part，再到章节，再到叶节点。

### C3. 形状系统

1. **中心节点**：保持椭圆光晕，增加四段主线短标签，继续作为全图原点。
2. **Part 节点**：保持大圆角矩形，改为纯色标题条 + 浅色底部区域；显示 Part 名、范围、Part 口诀。
3. **章节节点**：改为“记忆卡”：
   - 圆角矩形，宽 330-380，高 120-160。
   - 顶部 28px 使用 Part 主色标题条。
   - 中部白底 / Part 浅底显示定义。
   - 底部显示组件 / 技术计数和口诀。
4. **组件组节点**：胶囊组，实线边框，使用 Part 浅色底。
5. **技术组节点**：胶囊组，虚线边框或轻微蓝绿色叠加；不要所有 Part 都统一蓝色，否则会破坏分块。
6. **风险节点**：小三角或带 `!` 的小胶囊，放在章节卡底部或外侧，橙红色，但边框仍与 Part 色协调。

### C4. 线条层级与路径高亮

当前 `.trunk`、`.branch`、`.twig` 只有粗细差异。建议：

1. `.trunk`：13px，Part 主色，默认 opacity 0.38。
2. `.branch`：6px，Part 主色，默认 opacity 0.50。
3. `.twig.component`：3px，实线，Part 主色，opacity 0.45。
4. `.twig.tech`：3px，虚线，Part 主色，opacity 0.45。
5. `.twig.risk`：2.5px，点线，风险色，opacity 0.50。
6. active 状态：选中章节路径 opacity 1，宽度增加 1.5-2px，并给路径加 `filter: url(#activeShadow)` 或 `stroke-linecap` 强化。

实现位点：

- `addPath(from, to, className, color, direction)` 增加 `dataKey` 或 `pathType` 参数。
- `renderSvg` 绘制路径时传入 `part-${id}`、`chapter-${number}`、`leaf-${kind}` 类名。
- `nodeClasses` 类似增加 `pathClasses`，根据 `state.selected` 判断 `is-active-path`、`is-context-path`、`is-dim-path`。

### C5. 记忆模式视觉

新增“记忆模式”不是另一个页面，而是同一 SVG 的展示状态：

1. **普通总览**：显示章名、定义压缩句、组件 / 技术组、口诀。
2. **背诵模式**：隐藏部分答案，显示提示槽：
   - 章名保留。
   - 组件 / 技术只显示首字或数量，例如 `组件 4：任 / 提 / 中 / 校`。
   - 点击或悬停显示完整。
3. **风险模式**：章节卡底部从口诀切换为风险短句，帮助记边界。

建议 P0 先做普通总览；P1 再做背诵模式按钮和风险模式。

## D. 交互方案

### D1. 默认总览

默认打开页面时：

1. 画布居中到中心主题，缩放保持 0.80-0.88。
2. 四个 Part 分区背景可见。
3. 每章记忆卡可见，至少能看到章名、定义短句、口诀。
4. 组件 / 技术以组节点形式全量可见，默认不要求展开成每个独立叶。
5. 右侧默认显示“读图方法”和全书总口诀，不再提示“点击任一章节才查看完整内容”。

### D2. 点击章节

点击任一章节时：

1. 该章节卡高亮。
2. 中心 → Part → 章节 → 组件 / 技术 / 风险的路径全部高亮。
3. 同 Part 其他章节保持 0.35-0.50 opacity，其他 Part 降到 0.10-0.18 opacity。
4. 该章组件 / 技术组展开为独立叶节点；如果叶节点过多，显示为两列或双轨道。
5. 右侧 `renderInspector` 切换到该章节的“背诵检查 + 解释详情”。
6. 视口自动轻微聚焦到该章节，但不要强制把用户滚动位置跳得过远；建议新增 `focusChapter(chapter, { animate: true })`。

### D3. 点击 Part

点击 Part 时：

1. 高亮 Part 节点、Part 分区背景、中心到 Part 的主干。
2. Part 内所有章节卡高亮，其他 Part 降低透明度。
3. 右侧显示 Part 速记、章节口诀列表、推荐背诵顺序。
4. 不展开所有章节叶节点，避免画面拥挤。

### D4. 折叠 / 展开

新增两个轻量状态：

```js
state.expandedChapter = null;
state.viewMode = 'overview'; // overview | memory | risk
```

行为：

1. 点击章节：`expandedChapter = chapter.number`。
2. 再次点击同一章节：切回折叠，`expandedChapter = null`，但仍可保持 selected。
3. 点击“显示全部”：清空搜索、清空展开、回到 center。
4. 搜索命中章节：自动 selected，但不一定展开；建议只高亮命中，用户点击才展开。

### D5. 路径高亮

路径高亮应成为核心记忆交互：

1. active chapter：高亮中心、所属 Part、章节卡、该章组件 / 技术 / 风险节点。
2. active part：高亮中心、Part、Part 内章节分支。
3. search query：命中节点描边加深，但不覆盖 active path 的颜色。
4. hover chapter：临时高亮路径，不改变 `state.selected`；移出恢复。

实现建议：

- `addPath` 返回 path 元素或支持 class 计算。
- 绘制每条 path 时增加可识别类：`path-part-1`、`path-chapter-14`、`path-kind-tech`。
- CSS 增加 `.is-active-path`、`.is-context-path`、`.is-dim-path`。

### D6. 聚焦 / 还原

现有 `centerViewport` 只居中中心。建议新增：

1. `focusPoint(x, y, zoom = state.zoom)`：复用 `centerViewport` 的滚动计算。
2. `focusChapter(chapter)`：按 `chapter.x/chapter.y` 聚焦。
3. `focusPart(part)`：按 `part.x/part.y` 聚焦。
4. `restoreOverview()`：清空 selected 之外的展开状态，回到中心。

按钮建议：

- “显示全部”：还原总览。
- “总记忆”：进入 center + overview。
- 新增“背诵模式”：切换 `state.viewMode = 'memory'`。
- 新增“风险模式”：切换 `state.viewMode = 'risk'`。

## E. SVG 布局策略

### E1. 画布扩容但保留横向滚动

当前 `viewBox="0 0 2860 1480"`、`canvas = { width: 2860, height: 1480 }` 对全量信息偏紧。建议 P0 调整为：

```js
const canvas = { width: 3600, height: 1900 };
const center = { x: 1800, y: 950, rx: 200, ry: 86 };
```

对应修改：

- SVG `viewBox="0 0 3600 1900"`。
- 背景 `rect width="3600" height="1900"`。
- CSS `svg { width: 3600px; height: 1900px; min-width: 2400px; min-height: 1260px; }`。
- `applyZoom` 继续用 `canvas.width * state.zoom`，不破坏缩放。

### E2. 四象限布局继续保留

保持现有空间逻辑：

- Part 1：左上
- Part 2：左下
- Part 3：右上
- Part 4：右下

但每个 Part 的 `chapterX` 与 `part.x` 需要更外扩：

| Part | part.x / y 建议 | chapterX 建议 | 叶节点方向 |
| --- | --- | --- | --- |
| 1 | 1250 / 430 | 620 | 向外为左，向内为右 |
| 2 | 1250 / 1470 | 620 | 向外为左，向内为右 |
| 3 | 2350 / 430 | 2980 | 向外为右，向内为左 |
| 4 | 2350 / 1470 | 2980 | 向外为右，向内为左 |

中心与 Part 间保留大弧线；Part 与章节卡之间保留枝干曲线。

### E3. 章节纵向间距按 Part 动态计算

当前 `chapterYRanges` 中 Part 4 有 7 章但高度 725，Part 1 有 7 章高度 600；如果章节卡增高会重叠。建议：

```js
const chapterYRanges = {
  1: [140, 800],
  2: [1120, 1720],
  3: [180, 760],
  4: [1040, 1780]
};
```

并在 `normalizeParts` 中引入最小间距校验：

```js
const minChapterGap = 118; // P0 如果卡片高 112
const gap = Math.max(minChapterGap, (endY - startY) / (part.chapters.length - 1));
```

若 `gap * (chapters.length - 1)` 超过范围，则扩大画布高度或调整起止点，而不是压缩卡片。

### E4. 叶节点布局：从 4 个点改为 2 条轨道

当前 `positionedLeaves` 使用 `yOffsets` 四个固定偏移，只能承载 4 个短节点。建议改成“组件轨道 + 技术轨道 + 风险锚点”：

1. `componentTrack`：靠章节卡外侧上方，显示组件组。
2. `techTrack`：靠章节卡外侧下方，显示技术组。
3. `riskAnchor`：靠章节卡底部或内侧，显示风险短锚点。

折叠状态：

- 每章最多 3 个叶组：组件组、技术组、风险锚点。
- 每个叶组内部显示全量短标签，使用换行或 `textLines` 控制。

展开状态：

- 组件按 `index` 分布在组件轨道上，技术按 `index` 分布在技术轨道上。
- 每个叶节点仍通过 twig path 连接到章节卡，保留脑图结构。

### E5. 减少重叠策略

1. **章节卡高度固定**：P0 先设 128px，不按内容无限增长。
2. **文本截断规则固定**：定义短句最多 24 字；组件 / 技术标签单项最多 10-12 字，超出省略，但侧栏有完整内容。
3. **组节点换行**：组件 / 技术组最多两行，超出显示 `+N`，但 P0 目标是尽量全量显示；只有特别长词才折叠。
4. **双侧错位**：`positionedLeaves` 对相邻章节使用 `chapter.index % 2` 做轻微上下波动，避免叶组重叠。
5. **Part 4 特殊处理**：Part 4 有 7 章且技术词较多，优先扩大右下区域，不要压缩字体到不可读。

## F. 数据与代码改造点

### F1. `mindmapData` 数据结构

位置：`mindmap.html` 中 `const mindmapData = { ... }`。

P0 建议扩展字段：

```js
{
  id: 1,
  title: '基础工作流与协作',
  range: '1-7',
  color: '#7c3aed',
  palette: {
    trunk: '#7c3aed',
    branch: '#8b5cf6',
    soft: '#f3e8ff',
    softer: '#faf5ff',
    border: '#a78bfa',
    text: '#3b0764',
    accent: '#c084fc'
  },
  mnemonic: '拆、判、并、审、用、规、协',
  ...
}
```

章节数组在 `normalizeParts` 后补充派生字段：

```js
memoryTitle: `第${chapter[0]}章 ${chapter[1]}`,
definitionShort: shortDefinition(chapter[2]),
riskShort: shortRisk(chapter[6]),
componentCount: chapter[3].length,
techniqueCount: chapter[4].length
```

避免直接在源数组里重复写短句；先用函数从现有内容派生，后续如果质量不够，再手工加 `definitionShort` / `riskShort`。

### F2. `normalizeParts`

当前位置：`normalizeParts`。

改造点：

1. 保持把数组章节转对象。
2. 增加短句派生字段。
3. 增加布局字段：
   - `cardWidth`
   - `cardHeight`
   - `componentGroup`
   - `techniqueGroup`
   - `riskNode`
4. 增加 Part palette 默认值兜底，避免旧数据缺字段时报错。

### F3. `leafItems`

当前位置：`function leafItems(chapter)`。

当前逻辑：

```js
chapter.components.slice(0, 2)
chapter.techniques.slice(0, 2)
```

P0 必须替换。建议改成：

```js
function leafGroups(chapter) {
  return [
    { kind: 'components', label: '核心组件', items: chapter.components, className: 'component-group' },
    { kind: 'techniques', label: '关键技术', items: chapter.techniques, className: 'tech-group' },
    { kind: 'risk', label: '风险', items: [chapter.riskShort], className: 'risk-group' }
  ];
}
```

P1 再补：

```js
function expandedLeafItems(chapter) {
  return [
    ...chapter.components.map(...),
    ...chapter.techniques.map(...),
    { kind: '风险', label: chapter.riskShort, className: 'risk' }
  ];
}
```

### F4. `positionedLeaves`

当前位置：`function positionedLeaves(chapter)`。

当前以 `outsideDistances`、`insideDistances`、`yOffsets` 布局 4 个叶节点。改造为支持两套布局：

1. `positionedLeafGroups(chapter)`：默认总览用。
2. `positionedExpandedLeaves(chapter)`：选中章节展开用。

建议返回统一结构：

```js
{
  x,
  y,
  width,
  height,
  direction,
  kind,
  label,
  items,
  className,
  pathClass
}
```

这样 `renderSvg` 不需要关心叶节点是组还是单项，只调用对应绘制函数。

### F5. `addRoundedNode`

当前位置：`function addRoundedNode(...)`。

保留此函数作为基础节点绘制器，但建议扩展参数：

```js
function addRoundedNode({
  x, y, width, height,
  fill, stroke, className,
  lines, subtitle,
  titleFill,
  badge,
  dataAttrs,
  onClick,
  onMouseEnter,
  onMouseLeave
})
```

但不要把章节卡全部塞进 `addRoundedNode`，否则函数会变复杂。建议新增：

- `addPartNode(part)`
- `addChapterCard(chapter)`
- `addLeafGroupNode(group)`
- `addLeafItemNode(leaf)`
- `addRiskNode(risk)`

`addRoundedNode` 只负责通用矩形和基础交互。

### F6. `renderSvg`

当前位置：`function renderSvg()`。

重排绘制顺序：

1. 清空 `linksLayer`、`nodesLayer`，新增时也清空 `zonesLayer`。
2. 绘制 Part 背景分区。
3. 绘制所有 trunk / branch / twig path。
4. 绘制中心节点。
5. 绘制 Part 节点。
6. 绘制章节记忆卡。
7. 绘制叶组或展开叶节点。

关键判断：

```js
const isExpanded = state.expandedChapter === chapter.number;
const leaves = isExpanded ? positionedExpandedLeaves(chapter) : positionedLeafGroups(chapter);
```

### F7. `nodeClasses` / 新增 `pathClasses`

当前位置：`nodeClasses(item, base = '')`。

扩展状态：

1. `is-active`：当前选中节点。
2. `is-context`：选中章节的父级 Part、同章叶节点。
3. `is-dim`：非相关节点。
4. `is-hit`：搜索命中。
5. `is-expanded`：展开章节。
6. `is-memory-hidden`：背诵模式中被隐藏的答案。

新增：

```js
function pathClasses({ type, part, chapter, kind }, base = '') {}
```

路径不应只靠颜色判断 active，否则搜索与点击叠加时难以维护。

### F8. `renderInspector`

当前位置：`function renderInspector()`。

改造重点：

1. Center 分支：增加“读图方法”和 4 个 Part 口诀，不只展示附录。
2. Part 分支：把章节按钮改为“章节口诀列表”，例如 `第 1 章 提示词链：先拆后传，步步验算`。
3. Chapter 分支：顶部新增背诵检查清单：

```html
<div class="recall-check">
  <h3>看图应能背出</h3>
  <ol>
    <li>这一章解决什么问题</li>
    <li>核心组件有哪些</li>
    <li>关键技术节点有哪些</li>
    <li>一句口诀是什么</li>
    <li>主要风险是什么</li>
  </ol>
</div>
```

4. 组件 / 技术区标题改为“完整解释”，避免用户认为图上不完整。
5. 增加“在图中聚焦本章”按钮，调用 `focusChapter(chapter)`。

### F9. CSS 区块

位置：`<style>` 中现有 SVG 节点样式、inspector 样式、legend 样式。

新增 / 调整：

1. Part 分区：
   - `.part-zone`
   - `.part-zone-label`
2. 节点：
   - `.chapter-card`
   - `.chapter-card-header`
   - `.chapter-definition`
   - `.memory-line`
   - `.leaf-group`
   - `.component-group`
   - `.tech-group`
   - `.risk-group`
3. 路径：
   - `.is-active-path`
   - `.is-context-path`
   - `.is-dim-path`
4. 记忆模式：
   - `.memory-mode .answer`
   - `.memory-mode .answer.is-hidden`
5. 侧栏：
   - `.recall-check`
   - `.inspector-kv`
   - `.motto-list`

### F10. 控件与状态

位置：HTML header actions 与 `init()`。

建议新增：

```html
<button id="toggleMemory" class="secondary">背诵模式</button>
<button id="toggleRisk" class="secondary">风险模式</button>
```

`state` 扩展：

```js
const state = {
  query: '',
  selected: { type: 'center' },
  zoom: 0.82,
  expandedChapter: null,
  viewMode: 'overview'
};
```

`init()` 绑定按钮：

- `toggleMemory`：overview ↔ memory。
- `toggleRisk`：overview ↔ risk。
- `showAll`：重置 `expandedChapter` 与 `viewMode`。

## G. 验收标准

### G1. 结构验收

1. SVG 主图仍存在，`mindmapSvg` 仍是核心可视区域。
2. 图上仍能明确看到：中心主题、4 个 Part、21 个章节。
3. 每个章节仍通过 SVG 曲线路径挂接到对应 Part。
4. 组件 / 技术 / 风险仍通过 SVG 叶节点或叶组挂接到章节，不能脱离为纯 HTML 卡片。
5. 横向滚动条可用，放大 / 缩小 / 重置可用。

### G2. 信息验收

1. 默认总览不点击右侧，也能看到 21 章的章名和记忆口诀。
2. 默认总览不点击右侧，也能看到每章全部组件和全部技术，或在空间不足时看到明确 `+N` 且展开后全量可见。
3. 右侧不再是图上缺失信息的唯一来源；图上应能完成主要背诵。
4. 点击章节后，右侧完整内容与图上内容一致，不出现组件 / 技术数量不一致。
5. 搜索 `MCP`、`RAG`、`Guardrails`、`路由`、`反思` 能命中对应章节或叶节点。
6. 每一章都必须存在醒目的章节编号徽章，且章号不能只混在标题文字里。

### G3. 视觉验收

1. 4 个 Part 不看文字也能通过颜色和分区背景区分。
2. 同一 Part 内章节与叶节点共享同一色系，但相邻章节卡必须有清晰的 tone 差异，不能看起来同色同明度。
3. 组件与技术通过形状 / 线型区分，而不是完全依赖颜色。
4. 点击章节时，中心 → Part → 章节 → 叶节点路径一眼可见。
5. 非相关 Part 被弱化但仍可辨认，不应完全消失。

### G4. 记忆验收

1. 用户能按图背出 4 个 Part 口诀。
2. 用户能按图背出每章口诀。
3. 用户能按图背出每章至少 80% 的组件 / 技术；展开后能背出 100%。
4. 风险模式下，用户能快速复习 21 章风险边界。
5. 背诵模式下，隐藏答案不破坏布局，不造成节点跳动。

### G5. 兼容验收

1. 1366px 宽屏下可横向滚动查看完整 SVG。
2. 1920px 宽屏下默认缩放能看到中心与 4 个 Part 主体。
3. 移动端不要求一次看全，但不应布局崩坏；侧栏应下沉，SVG 可滚动。
4. 键盘 Enter / Space 仍可触发节点选择。
5. 不引入外部网络依赖，单 HTML 文件可直接打开。

## H. 风险与取舍

### H1. P0：必须先做

1. **Part 色板与分区背景**：优先解决颜色接近、分块感弱的问题。
2. **章节记忆卡 + 编号徽章（P0）**：把章节节点从短标题升级为可背诵的固定模板，并在卡片左上角加入醒目的章号徽章，先认编号再背内容。
3. **`leafItems` 全量化**：取消只展示前 2 个组件 / 技术的逻辑，改为全量组节点。
4. **侧栏去重复**：`renderInspector` 改为检查清单 + 解释详情。
5. **路径高亮基础版**：点击章节高亮中心、Part、章节和叶组路径。
6. **画布扩容与间距调整**：避免全量信息上图后重叠。

P0 完成后，应已经满足“看图就能背，不主要依赖右侧”。

### H2. P1：增强体验

1. **章节展开 / 折叠**：选中章节后从组节点展开为独立叶节点。
2. **背诵模式**：隐藏组件 / 技术答案，显示首字或数量。
3. **风险模式**：从口诀视图切换到风险边界复习视图。
4. **hover 路径预览**：鼠标悬停章节临时高亮路径。
5. **聚焦函数**：`focusChapter`、`focusPart`、`restoreOverview`。

P1 完成后，图从“信息完整”升级为“适合复习训练”。

### H3. P2：后续迭代

1. **手工优化短句**：如果自动 `shortDefinition` / `shortRisk` 不够好，为每章补人工短句。
2. **记忆测验模式**：点击空槽显示答案，统计已掌握章节。
3. **导出视图**：提供打印 / 截图友好的全览尺寸。
4. **局部小地图**：当画布扩展到 3600x1900 后，可增加当前视口位置提示。
5. **更多可访问性增强**：为 SVG 节点补充更完整的 `aria-label`。

### H4. 主要风险

1. **信息太多导致拥挤**：解决方式是“章节卡 + 组节点 + 展开”，不要把所有单项默认散开。
2. **画布太大导致迷路**：解决方式是保留中心定位、Part 分区背景、聚焦按钮和默认缩放。
3. **颜色太丰富导致花**：解决方式是每个 Part 内只用一套色阶，组件 / 技术主要靠形状和线型区分。
4. **右侧被削弱后缺上下文**：解决方式是右侧保留完整定义、落地和风险，但定位为解释，不是背诵唯一入口。
5. **改造 `addRoundedNode` 过度复杂**：解决方式是新增专用绘制函数，不把所有复杂节点逻辑塞进一个通用函数。

## 推荐实施顺序

### P0 实施清单

1. 在 `mindmapData.parts` 增加 `palette` 与 `mnemonic`。
2. 扩展 `state`，先加入 `expandedChapter: null` 与 `viewMode: 'overview'`，即使 P0 不做完整切换也预留。
3. 调整 `canvas`、`center`、SVG `viewBox`、背景 `rect`、CSS `svg` 尺寸。
4. 调整 Part 坐标、`chapterX`、`chapterYRanges`，确保 21 章记忆卡不重叠。
5. 新增 `shortDefinition`、`shortRisk`，在 `normalizeParts` 生成短句。
6. 在 `addChapterCard` 中加入大号章节编号徽章，并把章名、定义、口诀、风险短锚点固定到卡片模板。
7. 用 `leafGroups` 替换 `leafItems` 的前 2 项抽样逻辑。
8. 新增 `positionedLeafGroups`，保留或逐步替换 `positionedLeaves`。
9. 新增 `addChapterCard` 和 `addLeafGroupNode`。
10. 在 `renderSvg` 中按“分区 → 路径 → 中心 → Part → 章节卡 → 叶组”顺序重绘。
11. 改造 `renderInspector` 为“读图方法 / 背诵检查 / 解释详情”。
12. 更新 CSS：Part 分区、章节卡、编号徽章、叶组、路径 active / dim 状态。

### P1 实施清单

1. 新增 `positionedExpandedLeaves`，支持章节展开为独立组件 / 技术叶节点。
2. `selectChapter` 支持同章二次点击折叠。
3. 新增 `pathClasses`，让路径高亮与节点高亮分离。
4. 新增 `focusPoint`、`focusChapter`、`focusPart`。
5. 新增背诵模式按钮和风险模式按钮。
6. 增加 hover path preview。

### P2 实施清单

1. 为 21 章补人工 `definitionShort`、`riskShort`。
2. 增加测验模式与掌握状态。
3. 增加打印 / 截图布局。
4. 增加小地图或视口导航。
5. 完善移动端与可访问性细节。

## 最终效果定义

改造完成后，这张图应从“SVG 节点 + 点击右侧查看详情”升级为“可直接背诵的 SVG 记忆脑图”：

1. 用户先通过 4 个颜色分区记住全书结构。
2. 再沿中心 → Part → 章节的路径背章节顺序。
3. 再看每章记忆卡背定义和口诀。
4. 再看组件 / 技术组背系统组成和实现抓手。
5. 最后通过风险模式背边界。

全程仍保留真正 SVG 脑图结构、4 个 Part、21 章、缩放与横向滚动能力；右侧面板只负责辅助解释和自检，不再承担“图上缺失内容”的补救角色。
