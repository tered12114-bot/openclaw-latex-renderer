# OpenClaw LaTeX 渲染脚本规格

## 目标
为 OpenClaw web 端 (http://127.0.0.1:18789/) 增加 LaTeX 渲染能力，使 AI 输出中的 `$...$` 行内公式自动渲染为可视公式。

## 技术方案

### 渲染库
MathJax 3.x，通过 CDN 动态加载 (`tex-mml-chtml.js`)

### 核心逻辑
1. **加载 MathJax**：CDN 异步加载，配置行内分隔符 `$...$`
2. **监听 DOM**：MutationObserver 监听消息容器的 childList 变化
3. **去抖渲染**：2000ms 无新变化后才调用 MathJax.typeset()
4. **防重复**：已渲染节点打 `data-mathjax-rendered` 标记
5. **安全边界**：跳过代码块（\`\`\` 内）和用户输入区的 `$`

### 文件
单文件 `openclaw-latex.user.js`

### @match
`http://127.0.0.1:18789/*`
