# OpenClaw LaTeX 渲染器 — 最终总结

## 基本信息

| 项目 | 内容 |
|------|------|
| 桌面版脚本 | `openclaw-latex.user.js`（v2.16.8） |
| 移动版脚本 | `openclaw-latex-mobile.user.js`（v2.16.8-m） |
| 历史版本 | `openclaw-latex-v2.12.0.user.js`（最后已知可用的移动版） |
| 运行环境 | ScriptCat / Tampermonkey（桌面）；ScriptCat + Edge Android（移动） |
| 目标页面 | `http://127.0.0.1:18789/*`、`http://localhost:18789/*`、`https://*.ts.net/*`（可通过配置面板扩展） |
| KaTeX 版本 | 0.16.9（内联，无外部依赖） |

## 核心功能

1. 自动检测 OpenClaw 聊天消息中的 LaTeX 公式，支持四种分隔符：
   - `$...$`（行内）、`$$...$$`（块级）
   - `\(...\)`（行内）、`\[...\]`（块级）
2. 使用内联 KaTeX 0.16.9 渲染为 HTML/CSS
3. 将渲染结果迁移到 Shadow DOM 完全隔离，规避 OpenClaw CSP 样式冲突

## 关键技术问题与修复

### 问题 1：矩阵换行符丢失（v2.1.0 → v2.1.1）
- **现象**：`\begin{pmatrix} a & b \\ c & d \end{pmatrix}` 渲染为单行
- **根因**：Markdown 渲染器将 `\\` 处理为 `\`
- **修复**：`preProcess` 回调在矩阵/数组环境内恢复 `\\` 换行符

### 问题 2：幽灵文本（v2.1.2 → v2.1.4）
- **现象**：矩阵下方出现纯文本幽灵，如 `(acbd)`、`(147258369)`
- **根因**：Shadow DOM 内 CSS 选择器 `.katex .katex-mathml` 匹配失败
  - `.katex` 是 shadow host，位于 shadow root **外部**
  - `.katex-mathml` 在 shadow root **内部**
  - CSS 选择器找不到 `.katex` 祖先，规则失效
- **修复**（v2.1.4）：注入 Shadow DOM 的 CSS 进行选择器转换
  - `.katex{` → `:host{`、`.katex>` → `:host>`、`.katex .xxx` → `.xxx`

### 问题 3：`\[...\]` 和 `\(...\)` 无法渲染（v2.1.5 → v2.3.0）⭐ 核心修复
- **现象**：AI 回复中的 `\[...\]` display math 和 `\(...\)` inline math 显示为纯文本
- **根因**：OpenClaw Markdown 渲染器将 `\[` 处理为 `[`、`\(` 处理为 `(`
  - 反斜杠被当作 Markdown 转义字符吃掉
  - DOM 中 `\[...\]` 变成 `[<br>\n...<br>\n]`
  - DOM 中 `\(...\)` 变成 `(\cmd...)`
- **修复**（v2.2.0）：新增 `restoreDelimiters()` 函数
  - 在 `renderMathInElement` 之前预处理 DOM
  - 对每个 `<p>` 的 innerHTML 做正则替换
  - `[...含LaTeX命令...]` → `\[...\]`（display math 恢复）
  - `(...含\cmd...)` → `\(...\)`（inline math 恢复）
  - 排除 `\left(...\right)` 中的括号（LaTeX 语法，非分隔符）

### 问题 4：`\big[...\big]` 和嵌套括号导致渲染失败（v2.2.0 → v2.3.0）
- **现象**：含 `\big[...\big]`、`\left[...\right]` 的 display math 和含嵌套括号的 inline math 渲染失败
- **根因**：v2.2.0 的正则有两个缺陷
  - Display math 正则 `/\[([\s\S]*?)\]/g` 非贪婪匹配，遇到 `\big]` 或 `\right]` 中的 `]` 就提前结束，把 `\big]` 变成 `\big\]` 破坏 LaTeX 语法
  - Inline math 正则 `/\(([^)]*?\cmd[^)]*?)\)/g` 中 `[^)]*?` 在第一个 `)` 就停止，无法处理 `(z = \frac12(x^2 + y^2))` 等嵌套括号
- **修复**（v2.3.0）：重写 `restoreDelimiters()` + 新增 `restoreInlineMath()`
  - Display math：改用 `startsWith('[') && endsWith(']')` 锚点判断，只在 `<p>` 首尾的 `[...]` 恢复，中间的 `\big]`、`\right]` 不受影响
  - Inline math：用平衡括号算法替代正则，正确匹配嵌套括号对
    - 扫描 innerHTML 中的 `(` 和 `)` 位置（跳过 HTML 标签）
    - 用栈匹配平衡括号对
    - 对每对检查内容是否含 `\cmd`，排除 `\left(...\right)`
    - 从后向前替换，保持位置有效

### 问题 5：`<td>` 和 `<li>` 中的 LaTeX 不渲染 + 截断 display math（v2.3.0 → v2.4.0）
- **现象**：表格单元格中的 `\cos\gamma` 和被 Markdown 截断的 `\begin{aligned}` 块无法渲染
- **根因**：
  - `restoreDelimiters()` 只处理 `<p>` 元素，忽略了 `<td>` 和 `<li>`
  - Markdown 渲染器把 `- \cmd` 开头的行当成无序列表项（`- ` 是列表语法），导致 `\begin{aligned}` 块被拆成 `<p>` + `<ul><li>`，`<p>` 以 `[` 开头但不以 `]` 结尾
- **修复**（v2.4.0）：
  - 选择器从 `'p'` 扩展为 `'p,td,li'`，表格和列表中的 inline math 也被处理
  - 新增截断 display math 合合逻辑：检测 `<p>` 以 `[` 开头但不以 `]` 结尾时，检查下一个兄弟 `<ul>` 的最后一个 `<li>` 是否以 `]` 结尾，如果是则合并 `<p>` + 所有 `<li>` 内容为完整 `\[...\]`，并恢复 `- ` 减号前缀（Markdown 吃掉的减法运算符），删除 `<ul>`

### 问题 6：inline math 无 `\cmd` 时无法识别（v2.4.0 → v2.5.0）
- **现象**：表格中的 `(x^2+y^2 = 4)` 不渲染
- **根因**：`restoreInlineMath` 仅检查括号内是否含 `\cmd`（如 `\frac`），但 `(x^2+y^2 = 4)` 只有 `^` 上标，无 `\cmd`
- **修复**（v2.5.0）：扩展匹配模式 `MATH_RE`
  - 旧：`/\\[a-zA-Z]/`（仅匹配 `\cmd`）
  - 新：`/\\[a-zA-Z]|\^[\d{]|_\d|_\{/`（额外匹配 `^2` 上标、`_1` 下标、`_{xy}` 下标）

### 问题 7：display math 中 `<br>` 导致 auto-render 无法匹配分隔符（v2.5.0 → v2.6.0）⭐ 关键修复
- **现象**：`\boxed{-8\pi}` 等 display math 不渲染，显示为原始 `\[...\]` 文本
- **根因**：Markdown 渲染器在 `\[...\]` 内容中插入 `<br>` 标签，导致 `\[` 和 `\]` 被分割到不同的 text node
  - auto-render 的 `splitAtDelimiters` 函数在单个 text node 内查找分隔符
  - `\[` 在第一个 text node，`\]` 在第三个 text node，无法匹配
  - 结果：auto-render 跳过整个 `\[...\]` 块
- **修复**（v2.6.0）：在 `restoreDelimiters` 转换 display math 后，移除 innerHTML 中的 `<br>` 标签
  - `nh='\\['+inner.replace(/<br>/g,'')+'\\]'`
  - 移除 `<br>` 后，整个 `\[...\]` 在单个 text node 中，auto-render 可正确匹配
  - LaTeX 中 `<br>` 对应的换行符在 math mode 内无意义（KaTeX 忽略空白）

### 问题 8：移除 `<br>` 后 preProcess 双倍 `\` 导致 KaTeX 报错（v2.6.0 → v2.7.0）
- **现象**：`\begin{aligned}...\end{aligned}` 环境渲染为红色错误文本
- **根因**：v2.6.0 移除 `<br>` 后，已有的 `\\`（双反斜杠换行符）后面跟着 `\n`
  - preProcess 正则 `/\\([ \t\n\r])/g` 匹配了第二个 `\` + 空白，将 `\\` 变成 `\\\`（三反斜杠）
  - KaTeX 遇到 `\\\` 报错，显示红色错误文本
- **修复**（v2.7.0）：preProcess 正则添加负向后瞻（negative lookbehind）
  - 旧：`/\\([ \t\n\r])/g`（匹配任何 `\` + 空白）
  - 新：`/(?<!\\\\)\\([ \t\n\r])/g`（仅匹配不在 `\\` 后面的 `\` + 空白）
  - 效果：`\\` + 空白不会被重复加倍，单个 `\` + 空白仍正确恢复为 `\\`

### 问题 9：`restoreInlineMath` 在 display math 内误识别括号为 inline math（v2.7.0 → v2.8.0）
- **现象**：多个 display math 公式显示为红色源码，KaTeX 报错 `Can't use function '\(' in math mode`
- **根因**：`restoreDelimiters` 先将 `[...(z^2+x)...]` 转为 `\ [...(z^2+x)...\]`，然后 `restoreInlineMath` 把 `(z^2+x)` 误转为 `\(z^2+x\)`
  - KaTeX 在 display math `\ [...\]` 内遇到 `\(...\)` → 语法错误
  - 影响范围：所有含括号对的 display math 公式，如 `\iint_\Sigma (z^2 + x)dydz`、`\frac12(x^2+y^2)`、`\big[(z^2+x)(-x)\big]`
- **修复**（v2.8.0）：对已识别为 display math 的元素（`nh` 以 `\[` 开头），跳过 `restoreInlineMath`
  - 判断条件：`var isDisplayMath = nh.indexOf('\\[') === 0;`
  - 仅对非 display math 元素调用 `restoreInlineMath`

### 问题 10：嵌套括号导致 `restoreInlineMath` 位置偏移 + `\;` 丢失 + heading 未处理（v2.8.0 → v2.9.0）
- **现象**：
  - inline math 内嵌套括号 `(z = \frac12(x^2+y^2))` 渲染为乱码
  - `\;`（thin space）被 Markdown 吃掉，显示为裸分号 `;`
  - `<h2>` 标题中的 `(dydz \to dxdy)` 未渲染为 inline math
- **根因**：
  - 嵌套括号：`restoreInlineMath` 从后向前替换时，外层括号先被转为 `\(...\)`，内层括号的位置偏移导致错误替换，产生 `\(\frac1\((x^2+y^\))\)` 乱码
  - `\;` 丢失：Markdown 将 `\;` 处理为 `;`（吃掉反斜杠），在 `\pmb{n} = \left(...,; ...\right)` 和 `;\Longrightarrow;` 中可见
  - heading 未处理：选择器 `'p,td,li'` 不包含 `<h2>` 等标题元素
- **修复**（v2.9.0）：
  - 嵌套括号：添加内层括号过滤——若某对括号完全包含于另一对已匹配的括号内，则跳过内层
  - `\;` 恢复：三个正则依次恢复 `,;` → `,\\;`、`;\\cmd` → `\\;\\cmd`、`\\cmd;` → `\\cmd\\;`
  - heading 处理：选择器扩展为 `'p,td,li,h1,h2,h3,h4,h5,h6'`

### 问题 11：截断 display math 合并时 `- ` 前缀 + `&amp;` 未解码（v2.9.0 → v2.10.0）
- **现象**：
  - 第五步的 aligned 环境渲染异常，`- \int_0^{2\pi}...` 出现多余减号
  - aligned 环境对齐符 `&` 失效，公式不按 `&` 对齐
- **根因**：
  - `<li>` 内容合并时被加上 `- ` 前缀（模拟 Markdown 列表标记），但 `<li>` innerHTML 本身就是 LaTeX 续行
  - innerHTML 中 `&amp;` 未解码为 `&`，KaTeX 无法识别 aligned 环境的对齐符
- **修复**（v2.10.0）：
  - 移除 `- ` 前缀：`allInner+='<br>\n'+liContent`
  - HTML 实体解码：`inner.replace(/&amp;/g,'&').replace(/&lt;/g,'<').replace(/&gt;/g,'>')`
  - display math 转换时也加入实体解码

### 问题 12：Shadow DOM CSS 不完整导致分子缺失 + 分数横线延伸 + 根号不可见（v2.10.0）
- **现象**：
  - `\frac{x}{\sqrt{...}}` 的分子 `x` 完全不显示（不是颜色问题）
  - 分数横线 `width:100%` 延伸到屏幕外（7744px）
  - 根号 `√` 与深色背景同色，几乎不可见
- **根因**：
  - `_CSS` 变量比 KaTeX 0.16.9 官方 CSS 少 4680 字符，缺少关键布局规则
  - `isolateKatex()` 对 `.katex` 创建 Shadow DOM，但 `.katex-display` 是 `.katex` 的父元素，留在 Shadow DOM 外面
  - `.katex-display > .katex > .katex-html { display: block }` 等 CSS 选择器无法跨 Shadow DOM 边界匹配
  - `.vlist` 的 `display: table-cell` 失效变成 `inline`，分子被挤压到不可见
  - `.frac-line` 的 `display: inline-block` + `width: 100%` 在无约束父元素下延伸到屏幕外
- **修复**（v2.10.0）：
  - 用 KaTeX 0.16.9 完整官方 CSS 替换不完整的 `_CSS`
  - `isolateKatex()` 改为对 `.katex-display` 创建 Shadow DOM（display math），对独立 `.katex` 创建 Shadow DOM（inline math）
  - 新增 `_SHADOW_CSS`（display 版本）和 `_SHADOW_CSS_INLINE`（inline 版本）两套 CSS 转换
  - display 版本：`.katex-display{` → `:host{`，`.katex-display>` → `:host>`，`.katex .` → `.`，`.katex{` 和 `.katex>` 保持不变
  - inline 版本：`.katex{` → `:host{`，`.katex>` → `:host>`，`.katex .` → `.`

### 问题 13：`$$...$$` 分隔符未被恢复导致多行公式渲染失败（v2.10.0 → v2.11.0）
- **现象**：
  - 曲面积分 `\iint_\Sigma (z^2+x)dydz - z,dxdy = -8\pi` 显示为红色错误文本
  - 行列式 `$${\det\begin{pmatrix}...}...$$` 完全未渲染，显示为纯 `$$...$$` 源码
- **根因**：
  - `restoreDelimiters()` 只处理 `[...]` 和 `(...)` 分隔符，完全不处理 `$$...$$`
  - 当 `$$...$$` 公式跨多行时，Markdown 插入 `<br>` 分割内容到不同 text node
  - auto-render 的 `splitAtDelimiters` 在单个 text node 内匹配分隔符，`<br>` 把 `$$` 分隔符分到不同 text node，无法匹配完整 `$$...$$`
  - 单行 `$$` 公式理论上应被 auto-render 匹配，但如果 `$$` 前后元素被 `<p>` 分割或 `$$` 内部有 `<br>` 则同样失败
- **修复**（v2.11.0）：
  - 早期跳过检查加入 `$$`：`h.indexOf('$$') === -1` 才跳过
  - 新增单行 `$$` 处理：`trimmed.startsWith('$$') && trimmed.endsWith('$$')` 锚点判断 + `<br>` 移除 + 实体解码
  - 新增截断 `$$` 处理：`trimmed.startsWith('$$') && !trimmed.endsWith('$$')` 合并 `<ul><li>` 续行
  - 显示数学检测加入 `$$`：`nh.startsWith('$$')` 也标记为显示数学，跳过 `restoreInlineMath`

### 问题 14：KaTeX 不支持的环境导致渲染失败（v2.11.0 → v2.12.0）
- **现象**：
  - 长公式 `\begin{multline}...\end{multline}` 显示为红色错误文本
  - 化学式 `\ce{CO2 + H2O -> H2CO3 -> H+ + HCO3-}` 显示为纯 `$$...$$` 源码
- **根因**：
  - KaTeX 0.16.9 **不支持 `multline` 环境**（GitHub issue #3560 仍开放，开发者表示不再计划实现）
  - KaTeX auto-render 匹配到 `$$...$$` 后将 `\begin{multline}...\end{multline}` 传给 KaTeX 渲染
  - KaTeX 解析器遇到不支持的环境直接抛出 `ParseError: No such environment: multline`，以红色错误文本展示
  - 化学式：`\ce` 不在 `LATEX_CMD` 正则列表中，`restoreDelimiters` 跳过 `$$...$$` 恢复，`\ce` 公式未被 auto-render 识别
- **修复**（v2.12.0）：
  - `preProcess` 回调中加入环境转换：`\\begin{multline}` → `\\begin{gather*}`，`\\end{multline}` → `\\end{gather*}`
  - `gather*` 与 `multline` 视觉相似（都居中显示多行公式），是多行公式的合理替代
  - `LATEX_CMD` 正则加入 `ce|mhchem`，确保 `\ce{...}` 公式被 `$$...$$` 恢复逻辑识别
- **已知限制**：
  - KaTeX 不支持 mhchem 的 `\ce` 化学命令，化学式会以 `\ce` 红色文字 + 化学内容作为数学模式渲染，非完美化学公式效果

### 问题 15：用户需求 —— 动态配置网址和渲染选项（v2.12.0 → v2.13.0）
- **需求**：用户需要通过设置面板管理匹配网址和 KaTeX 渲染选项，而非硬编码
- **实现**：
  - 新增 `ConfigManager` 模块：`load()` / `save()` / `defaults()` / `matchUrlPattern()` / `matchCurrentUrl()`
  - 新增 `SettingsPanel` 模块：Shadow DOM 隔离的模态面板，支持 URL 增删/验证/去重、渲染选项切换、配置持久化
  - 使用 `GM_setValue` / `GM_getValue` / `GM_registerMenuCommand`（`@grant` 从 `none` 改为三权限）
  - `start()` 加载配置并检查当前 URL，不匹配则跳过渲染
  - `render(cfg)` 使用 `cfg.throwOnError` 和 `cfg.shadowDOM` 控制行为
- **配置结构**：`{version, urls:[], throwOnError, shadowDOM, displayMode}`
- **存储**：`GM_setValue('openclaw-latex-config', JSON)`
- **UI 触发**：ScriptCat 扩展菜单 → "OpenClaw LaTeX Settings..."

### 问题 16：Markdown 把 LaTeX 下标 `_{...}` 转成 `<em>` 标签导致化学式不渲染（v2.13.0 → v2.14.0）
- **现象**：核反应方程式 `$\ce{^{235}_{92}U + n -> ^{141}_{56}Ba + ^{92}_{36}Kr + 3n}$` 完全未渲染，显示为纯文本
- **根因**：
  - Markdown 的 `_` 斜体语法把 `_{92}..._{56}` 中的两个 `_` 当成斜体标记的开始和结束
  - 结果：`_{92}U + n -> ^{141}_{56}` → `<em>{92}U + n -> ^{141}</em>{56}`
  - `<em>` 标签把 `$...$` 分成了多个 text node，auto-render 无法匹配完整分隔符
  - 另外：`\ce{...}` 被 `preProcess` 转成 `\text{...}`，但 `\text` 不支持 `^{}` 和 `_{}` 上下标
- **修复**（v2.14.0）：
  - 新增 `fixEmInDollarMath()` 函数：在 `$...$` 范围内将 `<em>` → `_`，`</em>` → `_`
  - 在 `restoreDelimiters()` 开始时调用：检测 `$` 和 `<em>` 同时存在时执行修复
  - `preProcess` 修改：`\ce{...}` → `\mathrm{...}`（支持上下标，而非 `\text`）
- **教训**：
  - Markdown 的 `_` 斜体语法与 LaTeX 的 `_` 下标语法冲突
  - `<em>` 标签会破坏 `$...$` 分隔符的完整性
  - `\text` 是纯文本模式，不支持数学上下标；`\mathrm` 是数学模式的正体文本

### 问题 17：Shadow DOM CSS 选择器转换遗漏导致分数布局崩溃（v2.14.0 → v2.15.0）
- **现象**：二次求根公式 `$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$` 在 Shadow DOM 隔离后渲染错误——分数线延伸到屏幕外，分母 `2a` 缺失
- **根因**：
  - `_SHADOW_CSS_INLINE` 的 CSS 选择器转换规则不完整
  - 只处理了 `.katex{`、`.katex>`、`.katex.xxx`、`.katex .xxx` 四种模式
  - 遗漏了 `.katex *`、`.katex svg`、`.katex img` 等 `.katex` 后跟空格+非 `.` 字符的模式
  - 缺失的 `.katex *` 规则包含 `-ms-high-contrast-adjust:none!important;border-color:currentColor`
  - 缺失的 `.katex svg` 规则包含 SVG 填充/描边样式
  - 这些规则缺失导致 `.base` 的 `width:min-content` 无法正确计算，`.vlist-t` 宽度从 `83px` 膨胀到 `5449px`
  - `.frac-line` 的 `width:100%` 继承了膨胀的父元素宽度，从 `81px` 变成 `5447px`
- **修复**（v2.15.0）：
  - `_SHADOW_CSS_INLINE` 新增 `.replace(/\.katex\s+/g, '')` 替换，处理 `.katex *` → `*`、`.katex svg` → `svg`、`.katex img` → `img`
  - `_SHADOW_CSS` 同步添加相同替换
- **教训**：
  - Shadow DOM CSS 选择器转换必须覆盖所有 `.katex` 后跟空格的模式，不仅是 `.katex .xxx`
  - `.katex *`（通配符选择器）和 `.katex svg`/`.katex img`（元素选择器）也是合法的 CSS 模式
  - 每次修改 `_SHADOW_CSS` 后，应检查剩余 `.katex` 选择器数量（排除 `.katex-html`、`.katex-mathml` 等类名部分）

### 问题 18：`\require{color}` 和 `\definecolor` 命令导致公式渲染失败（v2.15.0 → v2.15.1）
- **现象**：拉普拉斯变换公式 `\require{color}\definecolor{blue}{RGB}{0, 114, 189}\mathcal{L}f(t)=...` 显示红色错误文本，整条公式废弃
- **根因**：
  - LLM 输出的 LaTeX 使用 `\require{color}` 格式（`\require` 命令带参数 `{color}`），而非 `\requirecolor`
  - KaTeX 不支持 `\require` 命令（这是 LaTeX 宏包加载机制，浏览器端无法实现）
  - KaTeX 不支持 `\definecolor` 命令（颜色定义需要前置处理）
  - v2.15.0 的 preProcess 正则 `/\\requirecolor(?:\{[^}]*\})?/g` 只匹配 `\requirecolor`，不匹配 `\require{color}`
  - `\requirecolor{xcolor}` 带参数格式也留下 `{xcolor}` 残留
- **修复**（v2.15.1）：
  - 正则改为 `/\\require(?:color)?(?:\{[^}]*\})?/g`，拆分 `color` 和 `{...}` 为两个独立可选部分
  - 现支持：`\requirecolor`（无参数）、`\requirecolor{package}`（带参数）、`\require{color}`、`\require{amsmath}` 等所有格式
  - `\definecolor` 正则保持不变，已正确匹配三参数格式
- **验证**：
  - 原始公式84字符被移除，处理后公式 `\color{blue}\mathcal{L}...` 正常渲染
  - auto-render + preProcess 组合测试成功
- **教训**：
  - LaTeX 的 `\require` 命令有多种变体格式：`\require{pkg}` 和 `\requirecolor{pkg}`
  - 正则设计时需考虑 alternation 和可选参数的独立性
  - 每次遇到新格式，应扩展测试用例覆盖所有变体

### 问题 19：`\qquad` 和 `\xrightarrow` 命令不在LATEX_CMD正则中导致公式未渲染（v2.15.1 → v2.15.2）
- **现象**：二阶微分方程 `$$y'' + p(x)y' + q(x)y = 0,\qquad y(x_0) = y_0,; y'(x_0) = y_0'$$` 显示为原始 `$$...$$` 文本，未渲染
- **根因**：
  - `LATEX_CMD` 正则包含 `quad`，但**不包含 `qquad`**
  - `\qquad`（大空格命令，两个quad）不在匹配列表中
  - `LATEX_CMD.test(dollarInner)` 返回 false，公式跳过 `$$...$$` 恢复逻辑
  - `\xrightarrow` 同样缺失（用户测试公式中有）
- **修复**（v2.15.2）：
  - `LATEX_CMD` 正则添加 `qquad` 和 `xrightarrow`
  - 注意：`qquad` 必须放在 `quad` 前面，否则正则的 alternation 会先匹配 `quad`，`\qquad` 变成 `\q` + `quad`
- **验证**：
  - `\qquad` 测试：true ✓
  - `\xrightarrow` 测试：true ✓
  - 完整公式识别：true ✓
- **教训**：
  - LaTeX 空格命令有三个级别：`\,`（thin space）、`\quad`（1em）、`\qquad`（2em）
  - 正则的 alternation 顺序影响匹配结果，长命令名应放在短命令名前面
  - 每次遇到新命令不识别，应扩展 `LATEX_CMD` 列表

### 问题 20：`\cfrac` 命令不在 LATEX_CMD 正则中导致连续分数未渲染（v2.15.2 → v2.15.3）
- **现象**：5层嵌套连续分数 `$$x = \cfrac{1}{1 + \cfrac{1}{...}}$$` 显示为原始文本，完全未渲染
- **根因**：
  - `\cfrac` 是 KaTeX 支持的连续分数（continued fraction）专用命令
  - LATEX_CMD 正则中有 `frac|tfrac|dfrac`，但缺少 `cfrac`
  - restoreDelimiters 检测 `LATEX_CMD.test(dollarInner)` 返回 false
  - 公式跳过恢复，auto-render 无法处理
- **修复**（v2.15.3）：
  - 在 LATEX_CMD 的 alternation 中添加 `cfrac`（放在 `frac` 后面）
  - 同时清理末尾重复的 `xrightarrow|xrightarrow`
- **验证**：
  - `\cfrac` 单命令检测：true ✓
  - 5层嵌套公式检测：true ✓
  - KaTeX 直接渲染：成功 ✓
- **教训**：
  - KaTeX 支持的分数命令有四个：`\frac`、`\tfrac`、`\dfrac`、`\cfrac`
  - `\cfrac` 用于连续分数（如黄金分割连分数），需要特定的分子分母间距
  - 每次发现新命令，应检查是否属于某个"命令族"，族内成员是否全部包含

### 问题 21：移动端 ScriptCat 脚本完全不运行（v2.16.0-m）
- **现象**：v2.12.0 在手机版 Edge + ScriptCat 可用，但后续版本安装后完全失效
- **根因**（3 个，叠加效应）：
  1. `ConfigManager.matchCurrentUrl()` 启动门控 — v2.12.0 无此检查，v2.13.0+ 新增但移动端 URL 可能不匹配 → 脚本静默跳过
  2. `GM_registerMenuCommand` 在移动端 ScriptCat 的沙箱模式下不可用 → 未捕获异常导致脚本崩溃
  3. `render(cfg)` 参数化 — v2.12.0 的 `render()` 无参数，v2.13.0+ 要求传入 `cfg` 对象
- **修复**（v2.16.0-m）：创建移动端特供版本
  - `@grant none` — 完全避免沙箱激活（沙箱本身破坏 DOM 访问，不是 API 可用性问题）
  - `localStorage` 替代 `GM_setValue/GM_getValue` — `@grant none` 下无法使用 GM_* API
  - DOM 浮动齿轮按钮替代 `GM_registerMenuCommand` — 移动端无扩展菜单
  - 移除 `ConfigManager.matchCurrentUrl()` 启动门控 — 直接渲染
  - `render()` 去参数化 — 内部调用 `ConfigManager.load()`
- **教训**：
  - ScriptCat Android 的 MV3 模式要求"Allow User Scripts"或 Developer Mode
  - `@grant none` 是移动端唯一可靠方案 — `@grant` 任何权限都会激活沙箱，破坏 DOM 访问
  - 移动端 `@match` 使用 `https://*.ts.net/*` 通配符保护隐私（不暴露 Tailscale 域名）

### 问题 22：`LATEX_CMD` 缺少希腊字母命令导致 `$$...$$` 公式未渲染（v2.16.0）
- **现象**：`$$e^{i\pi} + 1 = 0$$` 显示为原始文本，完全未渲染
- **根因**：
  - `LATEX_CMD` 正则中缺少 `\pi`、`\phi`、`\psi`、`\chi`、`\rho`、`\tau`、`\xi`、`\eta`、`\zeta`、`\kappa`、`\nu` 等 17 个希腊字母命令
  - `restoreDelimiters` 中 `LATEX_CMD.test(dollarInner)` 返回 false
  - 公式跳过 `$$...$$` 恢复逻辑，auto-render 无法处理
- **修复**（v2.16.0）：`LATEX_CMD` 正则添加所有缺失的希腊字母命令及 `var` 变体
  - `pi|phi|psi|chi|rho|tau|xi|eta|zeta|kappa|nu|varphi|vartheta|varpi|varrho|varsigma|varepsilon`
  - 同时修复桌面版 `openclaw-latex.user.js` 中的 `LATEX_CMD`
- **教训**：
  - `LATEX_CMD` 是 `restoreDelimiters` 的守卫条件 — 缺少任何 LaTeX 命令都会导致含该命令的 `$$` 公式被跳过
  - 应定期对照 KaTeX 支持的命令列表检查 `LATEX_CMD` 覆盖率

### 问题 23：前缀文本+display math 不渲染（v2.16.1）⭐ 显示问题19
- **现象**：`\[...\]` display math 前有文本（如"计算：\[...\]"、"用...解方程组：\[...\]"）时，公式显示为纯文本，完全未渲染
- **根因**：
  - Markdown 把 `"计算：\[...\]"` 渲染为同一个 `<p>` 标签：`"计算：<br>\n[<br>\n\begin{vmatrix}...<br>\n]"`
  - `restoreDelimiters` 的 display math 检测使用 `trimmed.indexOf('[')===0` 锚点
  - `[` 不在位置 0（前面有 `"计算：<br>\n"` 前缀文本），整个 `\[...\]` 恢复被跳过
  - 这是 Markdown 破坏 LaTeX 的新模式：display math 与前缀文本在同一个 `<p>` 中
- **修复**（v2.16.1）：`restoreDelimiters` 新增前缀文本+display math 检测分支
  - 条件：`!(trimmed.indexOf('[')===0) && trimmed.endsWith(']')` — 不以 `[` 开头但以 `]` 结尾
  - 定位：`trimmed.indexOf('[<br>')` 或 `trimmed.indexOf('[\\begin')` 找到 display math 起点
  - 恢复：`nh = trimmed.substring(0,lb) + '\\[' + preInner.replace(/<br>/g,'') + '\\]'`
  - 前缀文本保留在 `<p>` 中，display math 内容恢复为 `\[...\]` 格式
- **影响范围**：所有"中文文本 + display math"组合（计算、设、推导、用...解方程组等）

### 问题 24：前缀文本+display math 中 `restoreInlineMath` 误伤括号（v2.16.2）⭐ 问题 9 回归
- **现象**：含括号的 display math 公式（如 `\frac{1}{2}(x^2+y^2)`、`\left(...\right)`）渲染为红色错误文本：`Can't use function '\(' in math mode`
- **根因**：
  - 问题 23 的修复（v2.16.1）产生 `nh = "计算：<br>\n\[\frac{1}{2}(x^2+y^2)\]"`
  - `isDisplayMath` 检测使用 `nh.indexOf('\\[')===0` — `\[` 不在位置 0（前面有前缀文本），返回 false
  - `restoreInlineMath` 在 display math 内部把 `(x^2+y^2)` 错误转为 `\(x^2+y^2\)`
  - KaTeX 在 display math `\[...\]` 内遇到 `\(...\)` → 语法错误
  - 这是**问题 9**（v2.8.0）的历史回归 — 当时的修复 `isDisplayMath = nh.indexOf('\\[')===0` 仅在 `nh` 以 `\[` 开头时生效
- **修复**（v2.16.2）：`isDisplayMath` 检测从位置 0 匹配改为全文匹配
  - 旧：`var isDisplayMath = nh.indexOf('\\[') === 0 || nh.indexOf('$$') === 0;`
  - 新：`var isDisplayMath = nh.indexOf('\\[') !== -1 || nh.indexOf('$$') !== -1;`
  - `\[` 或 `$$` 出现在 `nh` 中任何位置都标记为 display math，跳过 `restoreInlineMath`
- **验证**：
  - vmatrix + prefix text → 2 行渲染 ✓
  - 3x3 pmatrix + prefix text → 3 行渲染 ✓
  - `\frac{1}{2}(x^2+y^2)` + prefix → 括号未误伤 ✓
  - `\left(...\right)` + prefix → 括号未误伤 ✓
  - `(z^2+x)=(x^2+y^2)^{1/2}` + prefix → 括号未误伤 ✓
  - 正常 `\[...\]` display math → 无回归 ✓
  - `$$...$$` display math → 无回归 ✓
- **教训**：
  - 任何修改 `restoreDelimiters` 输出 `nh` 格式的改动都必须重新检查 `isDisplayMath` 检测
  - `===0` 锚点假设 `nh` 以 display math 分隔符开头 — 前缀文本分支打破了这个假设
  - 问题 9（v2.8.0）的 `isDisplayMath` 修复是防御性的 — 它必须覆盖所有 `nh` 包含 display math 的场景，不仅是 `nh` 以 display math 开头的场景

### 问题 25：`<br>` 和 `<em>` 在 `\(...\)` inline math 内导致渲染失败（v2.16.4）⭐ 显示问题22/23
- **现象**：表格 `<td>` 中的 `\(\vec{a}\cdot\vec{b}\)` 显示为纯文本；含 `\dfrac{...}{...}` 的公式在 `\dfrac{` 处截断
- **根因**：
  - Markdown 在 `\dfrac` 的花括号内插入 `<br>` 标签
  - Markdown 将 `_{...}` 下标中的 `_` 转为 `<em>` 标签
  - `restoreInlineMath` 恢复 `\(...\)` 后，`<br>` 和 `<em>` 仍然留在 inline math 内容中
  - KaTeX 无法渲染含 `<br>` 的 `\dfrac`，也无法识别 `<em>` 标签
  - display math 分支有 `inner.replace(/<br>/g,'')` 移除 `<br>`，但 inline math 分支没有
  - `fixEmInDollarMath` 只处理 `$...$` 范围内的 `<em>`，不处理 `\(...\)` 中的 `<em>`
- **修复**（v2.16.4）：`restoreDelimiters` 在 `restoreInlineMath` 之后新增 `\(...\)` 内容清理步骤
  - 正则 `/\\\(([\s\S]*?)\\\)/g` 匹配所有 inline math 内容
  - 对每个匹配的内部内容移除 `<br>` 和修复 `<em>` → `_`
  - 仅在内容有变化时替换，避免无意义的DOM更新
- **验证**：
  - 含 `<br>` 的 `\(\cos\theta = \dfrac{...}{...}\)` → 渲染成功 ✓
  - 含 `<em>` 的 `\(\vec{a}_{b}\)` → 渲染成功 ✓
  - 简单 `\(\vec{a}\cdot\vec{b}\)` → 无回归 ✓
  - `$...$` 分隔符 → 无回归 ✓
  - `\[...\]` display math → 无回归 ✓
- **教训**：
  - `restoreInlineMath` 恢复分隔符后，分隔符内的HTML残留物（`<br>`、`<em>`）同样需要清理
  - display math 和 inline math 的处理必须对称——display math 有 `<br>` 移除，inline math 也需要
  - `fixEmInDollarMath` 的 `$` 分隔符限制不覆盖 `\(...\)` 场景，需要在恢复后统一处理

### 问题 26：Markdown 表格 `|` 管道符截断 LaTeX 绝对值符号导致公式丢失（v2.16.7）⭐ 显示问题24
- **现象**：表格中含 `|\vec{a}|` 绝对值的公式（如 `\(\cos\theta = \dfrac{\vec{a}\cdot\vec{b}}{|\vec{a}||\vec{b}|}\)`）截断显示——`\dfrac` 花括号未闭合，公式后半部分完全丢失
- **根因**：
  - Markdown 表格语法用 `|` 作为列分隔符
  - `|\vec{a}||\vec{b}|` 中的 `|` 被 Markdown 解析为列边界，公式在 `|` 处被截断
  - 截断后 cell 内容只保留第一个 `|` 前的部分（如 `\dfrac{\vec{a}\cdot\vec{b}}{`），后面全部丢失
  - DOM 中 `<td>` 的内容是截断后的 HTML，纯 DOM 操作无法恢复丢失的内容
- **修复**（v2.16.5→v2.16.6）：新增 `fixTablePipeTruncation()` 函数
  - 从 `openclaw-app.chatMessages` 获取原始 Markdown 源码
  - `parseMdCells()`：LaTeX-aware 的 Markdown 表格行解析器，在 `\(...\)`、`\[...\]` 内和花括号深度 > 0 时保留 `|`
  - 通过 header 列数 + body 行首 cell 文本指纹匹配 DOM `<table>` 到 Markdown 表格
  - `mdCellToHtml()`：将 Markdown cell 内容转为 HTML（保留 `\(...\)` 分隔符）
  - **v2.16.5 初版**：用 `hasTruncatedMath()` 5种模式检测截断 → 遗漏2种截断场景
  - **v2.16.6 修复**：移除 `hasTruncatedMath`，改为 textContent 比较——将 MD 源码解析的 cell 文本（去掉 `\(\)` 反斜杠和 Markdown 格式标记）与 DOM cell 的 textContent 比较，不同则替换
- **v2.16.6 修复原因**：
  - `hasTruncatedMath` 的5种模式遗漏2种截断：`"平行四边形面积 (="`（`(=`结尾）和 `"("`（单独一个`(`）
  - 有限的截断模式列表无法覆盖所有 Markdown 破坏 LaTeX 的方式
  - textContent 比较更可靠：MD 源码是 ground truth，DOM 内容不同即说明被截断
- **验证**：
  - 6 个截断 cell 全部修复 ✓（表格0: 求夹角/求投影行，表格1: 求面积行，表格2: 公式行数量积列/向量积列+垂直行向量积列）
  - 0 个正常 cell 被误改 ✓
  - `\dfrac` 花括号平衡 ✓
  - `|\vec{a}||\vec{b}|` 完整保留 ✓
- **教训**：
  - Markdown 表格 `|` 与 LaTeX `|` 绝对值符号冲突是系统性的——任何含 `|` 的公式在表格中都会被截断
  - DOM 中截断后的内容无法凭空恢复，必须利用原始 Markdown 源码重新解析
  - `parseMdCells` 必须同时跟踪 `\(...\)`、`\[...\]` 分隔符状态和 `{}` 花括号深度，三者缺一都会导致 `|` 误判
  - 行首 cell 文本指纹匹配比 mdMap key 遍历更可靠——mdMap 的 key 可能与 DOM `<table>` 的顺序不一致
  - 不要用有限的截断检测模式列表——改用 ground truth（MD 源码）与 DOM 内容的文本比较
- **v2.16.7 修复**（round3）：`restoreInlineMath` 双重包装 `\(...\)` 导致 KaTeX 不渲染
  - **根因**：`fixTablePipeTruncation` 将 `<td>` innerHTML 设为含 `\(...\)` 的 MD 源码，随后 `restoreInlineMath` 把 `\(` 中的 `(` 当作普通括号再包装一层，产生 `\\(\\)` 双重分隔符，KaTeX 无法识别
  - **修复**：在 `restoreInlineMath` 的括号扫描循环中，使用奇偶反斜杠计数跳过 `\(` 和 `\)` — 奇数个连续 `\` 前的 `(` 或 `)` 是 LaTeX 分隔符，不应被二次包装
   - **验证**：4个浏览器测试全部通过（`\(...\)` 不被双重包装、纯 `(...)` 被正确转为 `\(...\)`、混合内容不受影响、嵌套括号正常处理）
- **v2.16.8 修复**（round4）：`fixTablePipeTruncation` 无法检测 `\(` 反斜杠被 Markdown 吃掉的情况
  - **问题**：表格中 `(\vec{a}\cdot\vec{b})` 和 `(\vec{a}\times\vec{b})` 未渲染
  - **根因**：Markdown 吃掉 `\(` 和 `\)` 的 `\`，但 `fixTablePipeTruncation` 的 textContent 比较 — MD 解析的文本（去掉 `\(` 的 `\` 后）与 DOM textContent（本身就没有 `\`）相同，无法检测到截断
  - **修复**：新增 `backslashParenBroken` 检测，修改条件为 `if(mdText!==domText||backslashParenBroken)`
  - **验证**：`(\vec{a}\cdot\vec{b})` 渲染成功 ✓、`(\vec{a}\times\vec{b})` 渲染成功 ✓、`|\vec{a}|` 绝对值正常 ✓、表格中正常 `\(...\)` 无回归 ✓

### 失败的尝试（已排除）
- ❌ `mathml.remove()` → 矩阵退化为单行
- ❌ `mathml.innerHTML = ''` → 矩阵渲染失效
- ❌ 多重 CSS 属性隐藏 → 无效（选择器根本未匹配）
- ❌ 仅添加 `\[...\]` 分隔符配置 → 无效（DOM 中已无 `\[`）

## Shadow DOM 隔离架构

```
<span class="katex" data-ks-iso="1">    ← Light DOM（空）
  #shadow-root (open)
    <style>                             ← _SHADOW_CSS（已转换选择器）
    <span class="katex-mathml">         ← 被 display:none!important 隐藏
    <span class="katex-html">           ← 正常渲染的数学公式
      <span class="base">...</span>
    </span>
</span>
```

## 渲染流水线

```
OpenClaw AI 回复（Markdown 含 LaTeX）
  ↓ Markdown 渲染器处理
  ↓ ⚠️ \[ → [、\( → （反斜杠被吃掉）
  ↓ ⚠️ \\ → \（换行符丢失）
  ↓ ⚠️ 前缀文本 + \[...\] 在同一个 <p> 中（显示问题19）
  ↓ ⚠️ 表格 | 截断 |\vec{a}| 绝对值（显示问题24）
DOM 中的聊天消息
  ↓ fixTablePipeTruncation() — 从原始 Markdown 修复表格中被 | 截断的 cell（v2.16.5 新增，v2.16.6 改进）
  ↓ restoreDelimiters() — 恢复分隔符
  ↓   [含LaTeX...] → \[...\]（首尾锚点）
  ↓   前缀文本+[含LaTeX...] → 前缀文本+\[...\]（v2.16.1 新增）
  ↓   ⚠️ isDisplayMath 检测用 indexOf!==-1，不依赖位置0锚点（v2.16.2 修复）
  ↓   display math 元素跳过 restoreInlineMath（v2.8.0 修复，v2.16.2 加强）
  ↓ restoreInlineMath() — 平衡括号恢复 inline math（仅非 display math）
  ↓   (含\cmd...) → \(...\)（支持嵌套括号）
  ↓ renderMathInElement() — KaTeX 渲染
  ↓   preProcess: 矩阵环境内恢复 \\ 换行符
  ↓ isolateKatex() — Shadow DOM 迁移
  ↓   _SHADOW_CSS: 选择器转换（.katex → :host）
最终渲染结果
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `openclaw-latex.user.js` | 桌面版用户脚本（v2.16.8） |
| `openclaw-latex-mobile.user.js` | 移动版用户脚本（v2.16.8-m，@grant none + localStorage） |
| `openclaw-latex-v2.12.0.user.js` | 历史版本（最后已知可用的移动版，参考用） |
| `HANDOFF.md` | 本文档 |
| `AGENTS.md` | 开发经验手册 |
| `skills/` | 调试技能（openclaw-latex-debugger） |
