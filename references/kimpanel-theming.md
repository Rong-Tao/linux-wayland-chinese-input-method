# 候选框皮肤设计自由度（kimpanel / St CSS）

> 目标：说清楚 GNOME Wayland 下这个候选框**能改什么、不能改什么、用什么交付**。
> 一句话：装了 Kimpanel 后，候选框由 **GNOME Shell 的 kimpanel 扩展**渲染，样式语言是 **GNOME Shell St CSS**（GTK/Clutter 的 CSS 子集，**不是网页 CSS**）。交付物是一份 `stylesheet.css`，或设计稿 + 规格（由工程转写）。
>
> 前置：先按 `install-walkthrough.md` 第 6.5 节装好 Kimpanel 扩展。不装 Kimpanel 时候选框由 fcitx5 的 classicui（mellow 主题）渲染，走的是另一套机制（见文末）。

---

## 1. 渲染架构 / 画布

| 项 | 说明 |
|----|------|
| 渲染器 | GNOME Shell 扩展 **kimpanel**（不是网页，也不是 fcitx 自带的 classicui） |
| 样式文件 | `~/.local/share/gnome-shell/extensions/kimpanel@kde.org/stylesheet.css` |
| 样式语言 | **St CSS**（GNOME Shell 的 CSS 子集） |
| 生效方式 | 改完必须**注销重新登录**（Wayland 无法热重载 gnome-shell） |
| 环境 | GNOME Wayland（本文在 GNOME 46 上验证，版本可不同） |

> ⚠️ 因为是 St CSS 不是 Web CSS，**很多网页 CSS 特性不支持**（见第 4 节），设计前务必看清边界。

---

## 2. 候选框结构 = 可设计的区域

横排模式下，框体竖直堆叠为「上半区（提示 + 你打的拼音）」+「候选行」：

```
┌─ .kimpanel-popup-content ──────────────────────────┐  ← 框体
│  [aux 提示]  [preedit:你打的拼音 ｜]                 │  ← .kimpanel-label
│  ① 候选1   ② 候选2   ③ 候选3   ④ ...               │  ← 每个 = .kimpanel-candidate-item
└────────────────────────────────────────────────────┘
        ↑ 其中“当前选中那一项”额外带 :active
```

可用的 CSS 选择器（钩子）：

| 选择器 | 对应区域 | 常用可设属性 |
|--------|----------|--------------|
| `.kimpanel-popup-content` | **整个框体** | 背景色、圆角、边框、阴影、内边距 |
| `.kimpanel-popup-boxpointer` | 外层指针框 | 箭头相关 `-arrow-*`、与目标的间隙 |
| `.kimpanel-label` | **所有文字**（提示 aux / 拼音 preedit / 候选词都共用） | 文字色、内边距 |
| `.kimpanel-candidate-item` | 单个候选项 | 文字色、圆角、内边距 |
| `.kimpanel-candidate-item:hover` | 鼠标悬停的候选 | 背景、圆角 |
| `.kimpanel-candidate-item:active` | **键盘当前选中（高亮）的候选** | 背景、文字色、圆角 |

> 关键点：**`:active` 不是“鼠标按下”，而是“键盘当前高亮选中的那个候选”** —— 这是唯一能把“选中项”单独做高亮的钩子。

---

## 3. ✅ 支持的属性（St CSS 子集）

- **颜色**：`color`、`background-color`（支持 `rgba()` 透明）
- **背景图**：`background-image: url("...")`（可指向本地 png/svg）、`background-size`、`background-position`
- **线性渐变**（St 专有写法，**不是** `linear-gradient()`）：
  ```css
  -st-background-gradient-direction: vertical;   /* 或 horizontal */
  background-gradient-start: rgba(40,40,48,0.98);
  background-gradient-end:   rgba(28,28,34,0.98);
  ```
- **边框**：`border` / `border-width` / `border-color` / `border-radius`（支持各角，如 `border-radius: 14px`）
- **间距**：`padding`、`margin`、`spacing`（St 专有：box 内子元素间距）
- **阴影**：`box-shadow`、`text-shadow`（效果随 shell 版本略有差异）
- **字体**：`font-family` / `font-size` / `font-weight` / `font-style`（⚠️ 见第 5 节，字体优先用扩展设置）
- **尺寸**：`width` / `min-width` / `max-width` / `height`、`text-align`
- **过渡**：`transition-duration`（有限）
- **箭头框专有**：`-arrow-background-color`、`-arrow-border-color`、`-arrow-rise`、`-arrow-base`、`-boxpointer-gap`

## 4. ❌ 不支持（请勿在设计里依赖）

- **`backdrop-filter` / `filter`** → **没有毛玻璃/模糊**（GNOME 不给输入法弹窗做模糊；只能用半透明深色“假装”质感）
- **`transform`**（旋转/缩放/位移）
- **`::before` / `::after`** 伪元素
- **`var()` CSS 变量、`calc()`**
- **flex / grid 等布局**（布局是固定的：框体竖排、上半区横排、候选横/竖排）
- **任意 `linear-gradient()/radial-gradient()` 语法**（只支持第 3 节那种 St 两端色线性渐变）
- **`@keyframes` 关键帧动画**

---

## 5. 非 CSS 的可调项（走扩展设置，不在 stylesheet.css 里）

| 想改 | 怎么改 | 备注 |
|------|--------|------|
| 字体（全局） | 扩展设置 `font` | JS 用 inline style 设字体，**会覆盖 css 里的 `font-*`**，所以字号/字体优先用这个 |
| 横排 / 竖排候选 | 扩展设置 `vertical` | `true`=竖排 |
| 候选项结构（序号位置、各区顺序、序号与词的排布） | 改扩展 JS（`panel.js`） | **纯 CSS 改不了**；要动结构需工程改代码 |

---

## 6. 设计边界提示（避免返工）

- **“你打的拼音（preedit）”无法单独配色**：提示 aux、拼音 preedit、候选词共用 `.kimpanel-label`，CSS 层面区分不开。要单独样式需改 JS 给它加独立 class（可做，但属工程改代码）。
- **选中高亮**只有 `:active` 这一个钩子，务必用它表达“当前选中项”。
- 颜色尽量给 **rgba**（支持透明，做半透明卡片很合适）。
- **能做**：圆角、半透明、边框、阴影、内边距、渐变、背景图 → 足够做现代卡片/拟物风。
- **做不到**：真模糊、异形（非矩形圆角）、伪元素装饰、复杂动效。

---

## 7. 交付方式（二选一）

- **A. 直接给 `stylesheet.css`** —— 用第 2 节选择器 + 第 3 节属性写。
- **B. 给设计稿 + 规格（色值/圆角半径/内边距/阴影/字号）** —— 工程转写成 St CSS。
- **推荐 B**：St CSS 有坑（不支持的属性多），给规格由工程落地最稳。

---

## 8. 当前样式（基线，可在此基础上改）

这是一版深色圆角，可作为起点：

```css
/* 外层 boxpointer：去箭头，让深色圆角的 content 承载视觉 */
.popup-menu-boxpointer.kimpanel-popup-boxpointer {
  -arrow-background-color: transparent; -arrow-border-color: transparent;
  -arrow-border-width: 0; -arrow-base: 0; -arrow-rise: 0; -boxpointer-gap: 4px;
}
/* 框体：深色半透明 + 圆角 + 阴影 */
.popup-menu-content.kimpanel-popup-content {
  background-color: rgba(32,33,38,0.97);
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 14px;
  padding: 6px 8px;
  box-shadow: 0 6px 20px 0 rgba(0,0,0,0.45);
}
.kimpanel-label { color: #e6e6ea; padding: 0.35em; }
.kimpanel-candidate-item { color: #e6e6ea; border-radius: 10px; padding: 0.25em 0.55em; }
.kimpanel-candidate-item:hover  { border-radius: 10px; background-color: rgba(255,255,255,0.10); }
.kimpanel-candidate-item:active { border-radius: 10px; background-color: rgba(96,165,250,0.95); color: #fff; }
```

> css 改完同样要**注销重登**才生效（Wayland 无法热重载 gnome-shell）；扩展更新可能覆盖此文件。高亮色改 `rgba(96,165,250,..)`，框体底色改 `rgba(32,33,38,..)`。

---

## 附：另一条自由度更高的路线（有代价）

如果想要**完全自由的图形皮肤**（任意背景图、9-patch 拉伸边框、任意纹理/质感），可以改走 **fcitx5 的 classicui 主题**机制（`theme.conf` 纯文本 + PNG/SVG 图片做 9-patch，就是 mellow 主题那套），自由度远高于 St CSS。

**代价**：classicui 在 GNOME Wayland 下，原生 Wayland 应用（如终端）的候选框**会闪烁**——这正是改用 kimpanel 的原因。

即：**「图形自由度高」与「不闪烁」二选一**。只追求配色/圆角/卡片风，kimpanel 完全够用；若一定要复杂图形质感，再评估是否切回 classicui 并接受闪烁。
