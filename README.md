# OpenClaw LaTeX 渲染器

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

一个 Tampermonkey / ScriptCat 用户脚本，为 [OpenClaw](https://github.com/openclaw/openclaw) Web UI 自动渲染 LaTeX 数学公式。

> 市面上没有现成方案。从零自研，历经 18 个已知问题迭代修复，现已在个人环境中稳定运行。

## 功能

- **四种分隔符支持**：`$...$`、`$$...$$`、`\(...\)`、`\[...\]`
- **Shadow DOM 完全隔离**：渲染结果迁移到 Shadow DOM，不污染主页面样式，不受 OpenClaw CSP 影响
- **内联 KaTeX 0.16.9**：无 CDN 依赖，国内网络友好
- **Markdown 破坏恢复**：修复 OpenClaw Markdown 渲染器吃掉的反斜杠、换行符、括号等
- **智能分隔符检测**：区分 LaTeX 分隔符与 `\big[`、`\left(` 等语法，避免误匹配
- **配置面板**：支持匹配 URL 管理、渲染选项切换（ScriptCat 扩展菜单入口）
- **环境兼容**：可配置 `\require{color}` / `\definecolor` 自动清理、`multline` → `gather*` 自动转换

## 安装

### 前置要求
- [Tampermonkey](https://www.tampermonkey.net/) 或 [ScriptCat](https://scriptcat.org/) 浏览器扩展

### 步骤
1. 安装上述任一扩展
2. 打开 [openclaw-latex.user.js](openclaw-latex.user.js) 原始文件，扩展会自动识别并提示安装
3. 访问 OpenClaw Web UI（默认 `http://127.0.0.1:18789`）

> 也可手动复制脚本内容，在扩展管理器中新建脚本并粘贴。

### 初始配置
安装后可通过浏览器扩展菜单 → "OpenClaw LaTeX Settings..." 打开配置面板，管理匹配 URL 列表和渲染选项。

## 效果预览

| 输入 | 渲染 |
|------|------|
| `$E = mc^2$` | 行内公式 |
| `$$\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}$$` | 独立块级公式 |
| `\begin{pmatrix} a & b \\ c & d \end{pmatrix}` | 矩阵 |
| `\begin{cases} x^2 & x<0 \\ e^{-x} & x\ge0 \end{cases}` | 分段函数 |

## 已知限制

| 功能 | 原因 |
|------|------|
| `\ce{C6H6}` 化学式 | KaTeX 0.16.9 不原生支持 mhchem |
| `\require{...}` / `\definecolor` | KaTeX 不支持运行时宏包加载，脚本自动移除 |
| `\sideset` / `\enclose` | KaTeX 不支持的 AMS / unicode-math 扩展 |
| `\begin{multline}` | KaTeX 不支持，脚本自动转为 `gather*` |
| `\\` 连分数嵌套深层 `<br>` 合并 | 某些极端嵌套场景需关注 |

## 工作原理

```
OpenClaw AI 回复（Markdown 含 LaTeX）
  ↓ Markdown 渲染器（吃掉反斜杠、插入 <br>）
DOM 中的聊天消息
  ↓ restoreDelimiters() — 恢复被 Markdown 破坏的 \[ 和 \( 分隔符
  ↓ restoreInlineMath() — 平衡括号算法恢复 (...) 为 \(...\)
  ↓ renderMathInElement() — KaTeX 自动渲染
  ↓ isolateKatex() — Shadow DOM 迁移（样式完全隔离）
最终渲染结果
```

详细技术文档见 [HANDOFF.md](HANDOFF.md)。

## 开发历史

- **v2.15.3** — 最新稳定版
- **v2.15.0** — 修复 Shadow DOM CSS 选择器转换遗漏导致的分数布局崩溃
- **v2.14.0** — 修复 `_{}` 被 Markdown 转为 `<em>` 导致化学式不渲染
- **v2.13.0** — 新增配置面板（URL 管理、渲染选项）
- **v2.12.0** — 修复 `multline` 不支持、`\ce` 识别
- **v2.11.0** — 修复 `$$...$$` 分隔符未被恢复
- 更多版本见 Git 历史

## License

[MIT](LICENSE)

---

*原为个人自用工具，开源以供有相同需求的用户参考和使用。*

**Author**: [筱天](https://github.com/tered12114-bot)
