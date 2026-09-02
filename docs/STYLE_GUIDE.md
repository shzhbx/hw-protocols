# hw-protocols 协议页编写规范（子 agent 必读）

本站是"硬件协议图解 · HW Protocol Lab"：面向嵌入式工程师的**单文件交互式 HTML 协议图解页**。
你负责新建一个 `docs/<proto>.html` 页面。写作前必须通读 `docs/sdio.html`（章节深度与代码风格的首要基准）与 `docs/uart.html`（波形动画的次要参考）。

## 硬性约束

- **单文件 HTML**：所有 CSS/JS/SVG 全部内联，除 Google Fonts 外**禁止任何外部资源**（无 CDN 脚本、无图片文件），双击直接打开 `file://` 也能完整运行。
- `<html lang="zh-CN">`，UTF-8，正文中文（专业术语保留英文）。
- 复用全站设计令牌（`:root` 变量与 `docs/index.html` 完全一致，直接复制）：

```css
:root{
  --bg:#070b09; --panel:#0d1510; --panel2:#101a13; --line:#1d2c22;
  --ink:#dcebe1; --dim:#8aa294; --faint:#5c7264;
  --green:#34e08a; --amber:#ffb02e; --cyan:#57d8f2;
  --mono:'IBM Plex Mono',ui-monospace,monospace;
  --disp:'Chakra Petch','Noto Sans SC',sans-serif;
}
body{font-family:'Noto Sans SC',-apple-system,sans-serif;}
```

字体引入（与现有页一致）：
`https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@500;600;700&family=IBM+Plex+Mono:wght@400;500;600&family=Noto+Sans+SC:wght@300;400;500;700&display=swap`

- **只允许创建你负责的那一个文件**。禁止修改 `index.html`、`README.md`、`STYLE_GUIDE.md` 及任何已有页面——跨页导航、首页卡片由主 agent 统一维护。
- 原生 vanilla JS，无框架；JS 放 `</body>` 前；不产生 console 报错；所有 `id` 唯一。

## 页面骨架（从上到下，与 sdio.html 对齐）

1. **顶栏 nav**：`<nav><div class="nav-in">` 内三段：
   - `<div class="brand">XXX<b>·</b>LAB</div>`（XXX = 协议名，如 `SPI`、`CAN`、`JTAG`）
   - `<div class="nav-links">` 页内锚点（每章一个，精简到 6–9 个）
   - `<div class="nav-x">` 跨页链接（见下方清单，**去掉自己**，自己所在协议不出现）
2. **hero**：eyebrow（大写英文短语点题）→ `<h1>中文名<span class="en">ENGLISH FULL NAME</span></h1>` → `lead` 一段话（把协议讲成一个有悬念的故事）→ `hero-chips` 特性标签 → CTA 按钮 → **hero SVG 示意图动画**（两芯片/拓扑连线 + 流动的数据包，JS 驱动）。
3. **章节**：用 sdio.html 的编号节头模式：
   `<div class="sec-head reveal"><span class="no">01</span><h2>中文标题</h2><span class="en">ENGLISH</span></div>`
   建议 8–10 章：它是什么&为什么 → 物理层/信号 → 核心时序动画（交互实验室）→ 协议机制深挖 → 配置/枚举流程 → 计算器或对比 → Linux 实战（终端风格）→ 调试指南 → 踩坑指南（症状→病因表）→ 速查表。
4. **footer**：与全站一致的 `HW-PROTOCOLS · MAINTAINED WITH ♥ FOR EMBEDDED ENGINEERS` 风格。
5. `reveal` 滚动进入动画：IntersectionObserver（照抄 sdio.html 的实现思路）。

## 交互底线（这是本站的灵魂，缺一不可）

1. **至少一个"逐位/逐包"核心时序动画**：示波器/逻辑分析仪视角的波形（SVG 绘制），带 ▶播放 / ⏸ / 单步 / 速度调节，采样沿、位窗口、状态跳变要随动画高亮并配文字解说。
2. **至少一个参数交互**：模式切换器（对比两种模式的同一过程）或可调参数计算器（输入 → 实时算出带宽/波特率/时隙等）。
3. **至少一个状态机或拓扑动画**（可点击步进）。
4. 表格用暗色面板 + 细边框风格；代码/终端块用 `--mono` 深色底、仿真终端外观（参考 sdio.html 的 Linux 章节）。

## 全站页面清单（nav-x 顺序，去掉自己后原样写入）

```
<a href="index.html">« HOME</a>
<a href="i2c.html">I²C</a><a href="spi.html">SPI</a><a href="uart.html">UART</a><a href="ir.html">IR</a>
<a href="rs485.html">RS-485</a><a href="onewire.html">1-WIRE</a><a href="can.html">CAN</a>
<a href="i2s.html">I²S</a><a href="dmic.html">DMIC</a><a href="tdm.html">TDM</a>
<a href="sdio.html">SDIO</a><a href="emmc.html">eMMC</a><a href="ddr.html">DDR</a>
<a href="rgb.html">RGB</a><a href="lvds.html">LVDS</a><a href="mipi-dsi.html">DSI</a><a href="csi2.html">CSI-2</a>
<a href="hdmi.html">HDMI</a><a href="usb.html">USB</a><a href="pcie.html">PCIE</a><a href="jtag.html">JTAG</a>
```

链接其他协议页讲过的知识点时，优先用 `<a href="xxx.html">` 跨页引用而不是重复展开。

## 完成后自验（必做）

1. `python3` 用 `html.parser` 把整页走一遍，无异常、标签配平。
2. 写个小脚本检查：页内所有 `href="*.html"` 的目标都在上表清单中；所有 `href="#xxx"` 都有对应 `id="xxx"`。
3. 若本机有 node：提取 `<script>` 内容存临时文件跑 `node --check`。
4. `git status`：你的产出必须只有一个新增 untracked 文件，不得出现对已跟踪文件的改动。

## 交付报告格式

```
FILE: docs/<proto>.html
TITLE: <h1> 页面标题
CARD: <40–60 字首页卡片描述，仿 index.html 现有卡片文案：一句话卖点 + 涵盖的关键交互点>
DEMO: <3–5 条核心交互点>
ISSUES: <遗留问题，没有写"无">
```
