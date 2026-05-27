# Fcitx5 + Rime/雾凇拼音 安装文档（实战验证版）

> 在一台 **Ubuntu 24.04 GNOME Wayland** 上从零装好并验证通过。换电脑照这份做即可。
> 版本号按你的发行版自行替换。
>
> **一句话**：fcitx5（框架）+ Rime/雾凇拼音（逼近搜狗手感的开源方案）+ mellow 主题（现代圆角）。纯本地、不闪屏、不上传隐私。

---

## 0. 适用环境

| 项 | 值 |
|----|----|
| 系统 | Ubuntu/Debian（本文在 Ubuntu 24.04 LTS 上验证） |
| 桌面 | GNOME / **Wayland** |
| 框架 | 从 ibus 切到 **fcitx5** |
| sudo | 需要（装包） |

> ⚠️ 为什么不用搜狗：搜狗 Linux 版至今是 **fcitx4**，在 GNOME Wayland 下**候选窗口无限闪屏**，且闭源无法移植到 fcitx5。Rime+雾凇是开源里最接近搜狗手感的替代。

---

## 1. 安装 fcitx5 全家桶

```bash
sudo apt update
sudo apt install -y \
  fcitx5 fcitx5-chinese-addons fcitx5-config-qt \
  fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 fcitx5-frontend-qt5 \
  fcitx5-material-color
```

切换默认输入法框架到 fcitx5：

```bash
im-config -n fcitx5      # 写入 ~/.xinputrc 为 run_im fcitx5
```

---

## 2. 环境变量（⚠️ 最关键的坑：值用 `fcitx` 不是 `fcitx5`）

fcitx5 的 Qt/GTK 插件注册名仍是 **`fcitx`**（Qt 类名 `fcitx::QFcitxPlatformInputContext`）。
写成 `fcitx5` 会导致 Qt 应用打不出字。用 `fcitx5-diagnose` 自检会明确报警。

```bash
# 先清理可能存在的旧值，再写入
sudo sed -i '/^GTK_IM_MODULE=/d;/^QT_IM_MODULE=/d;/^XMODIFIERS=/d' /etc/environment
sudo tee -a /etc/environment >/dev/null <<'EOF'
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
EOF
```

> GNOME Wayland 说明：Qt5/XWayland 应用必须靠这几个变量；GTK 应用本可走原生 text-input-v3（理论更不易闪），但保留全局 `GTK_IM_MODULE=fcitx` 覆盖面最全、最省心。

---

## 3. 开机自启

GNOME Wayland 不走 `~/.xinputrc`，需要 xdg autostart：

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/fcitx5.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Fcitx5
Exec=fcitx5 -d
Icon=fcitx
Terminal=false
X-GNOME-Autostart-enabled=true
EOF
```

---

## 4. 基础配置：默认拼音引擎 + 输入法组

（本配置**只用雾凇**，这步可整段跳过；仅当你想额外保留 fcitx5 自带拼音作对比时才需要。）

**云拼音用百度**（谷歌接口国内常超时）：

```bash
mkdir -p ~/.config/fcitx5/conf
cat > ~/.config/fcitx5/conf/cloudpinyin.conf <<'EOF'
Backend=Baidu
MinimumPinyinLength=4
EOF
```

**默认拼音候选数设 10**（fcitx5 上限就是 10），编辑 `~/.config/fcitx5/conf/pinyin.conf`，把 `PageSize=7` 改成 `PageSize=10`（首次运行 fcitx5 后该文件才生成）。

---

## 5. Rime + 雾凇拼音（核心）

### 5.1 装引擎

```bash
sudo apt install -y fcitx5-rime
```

> 会自动带上关键插件：`librime-plugin-lua`（雾凇 Lua 功能）、`librime-plugin-octagram`（整句语法模型）。librime 1.10 与雾凇**兼容，无报错**。

### 5.2 拉雾凇配置

```bash
git clone --depth 1 https://github.com/iDvel/rime-ice.git /tmp/rime-ice
mkdir -p ~/.local/share/fcitx5/rime
cp -r /tmp/rime-ice/* ~/.local/share/fcitx5/rime/
```

### 5.3 候选数也设 10（Rime 候选数独立于 fcitx5，雾凇默认 5）

```bash
cat > ~/.local/share/fcitx5/rime/default.custom.yaml <<'EOF'
patch:
  menu/page_size: 10
EOF
```

### 5.4 部署（编译几十万词条，首次较慢）

```bash
rime_deployer --build ~/.local/share/fcitx5/rime /usr/share/rime-data
```

成功标志：`~/.local/share/fcitx5/rime/build/rime_ice.table.bin`（约 58M）等文件生成，且无 error。

### 5.5 把 Rime 加入输入法组

编辑 `~/.config/fcitx5/profile`，**把 `rime` 放第一位（置顶/首选），后面跟 `keyboard-us`，不放 fcitx5 自带的 `pinyin`**（放 pinyin 会导致不同程序各记各的引擎、出现两套候选，皮肤也可能不一致）。设 `DefaultIM=rime`（默认即中文/雾凇；若想默认英文态、按需 Ctrl+Space 切雾凇，把它设成 `DefaultIM=keyboard-us`）。fcitx5 重启后会保留这个顺序。最简模板：

```ini
[Groups/0]
Name="分组 1"
Default Layout=us
DefaultIM=rime

[Groups/0/Items/0]
Name=rime
Layout=

[Groups/0/Items/1]
Name=keyboard-us
Layout=

[GroupOrder]
0="分组 1"
```

> ⚠️ **改 profile 前先 `pkill -9 fcitx5`**，否则 fcitx5 退出时会用内存里的旧配置把文件覆盖回去。改完再启动：`fcitx5 -d`。

---

## 6. 主题（现代圆角，可选）

```bash
git clone --depth 1 https://github.com/sanweiya/fcitx5-mellow-themes.git /tmp/mellow
mkdir -p ~/.local/share/fcitx5/themes
cp -r /tmp/mellow/mellow-* ~/.local/share/fcitx5/themes/   # opaque 版，GNOME 用这个（kwinblur 版只在 KDE 有模糊）
```

设主题 + 字体，编辑 `~/.config/fcitx5/conf/classicui.conf`：

```ini
Vertical Candidate List=False
WheelForPaging=True
Font="Sans 11"
PerScreenDPI=True
Theme=mellow-wechat
```

可选配色：`mellow-{youlan|sakura|vermilion|wechat|graphite}`，每个都有 `-dark` 深色版。
也可在 GUI「全局选项 → 主题」里实时切换预览。

> 主题可自定义：每个主题就是 `theme.conf`（纯文本，改颜色）+ `panel.svg`/`highlight.svg`（改形状）。候选框皮肤的完整设计自由度见 `kimpanel-theming.md`。

---

## 6.5 （可选）Kimpanel：根治 Wayland 候选框闪烁

GNOME Wayland 下，**原生 Wayland 应用（如 gnome-terminal）打中文时候选框可能闪烁**（XWayland 应用如微信/Chrome 不受影响）。原因是 fcitx 用 `xdg_popup` 在客户端内渲染候选框，挪位置只能 show/hide → 闪。根治办法是装 Kimpanel 扩展，让候选框改由 gnome-shell 统一渲染。

> ⚠️ **取舍**：装 Kimpanel 后候选框由 GNOME 渲染，**fcitx5 主题（mellow 等）对候选框不再生效**，样式变朴素、跟随系统主题。不想要随时在「扩展」里关掉即可恢复 mellow。

```bash
UUID=kimpanel@kde.org
# 注意 shell_version 换成你机器的 GNOME 大版本（如 46，用 `gnome-shell --version` 查）
INFO=$(curl -s "https://extensions.gnome.org/extension-info/?uuid=$UUID&shell_version=46")
DL=$(echo "$INFO" | grep -oP '"download_url"\s*:\s*"\K[^"]+')
curl -sL "https://extensions.gnome.org$DL" -o /tmp/kimpanel.zip
gnome-extensions install --force /tmp/kimpanel.zip
gnome-extensions enable $UUID 2>/dev/null || true   # Wayland 下不能即时 enable，重登才生效
```

Wayland 下新装扩展无法即时启用，需**注销重登**；若 `enable` 没写进列表，重登后在「扩展」应用里手动打开 Kimpanel 开关即可。

### 装 Kimpanel 后建议的两个调整

**① 让拼音（你打的字母）显示在候选框里** —— gnome-terminal 等不支持嵌入式预编辑的应用，默认看不到你打的拼音。改成统一由候选框显示：编辑 `~/.config/fcitx5/config` 把 `[Behavior]` 段的 `PreeditEnabledByDefault` 设为 `False`，重启 fcitx5。

```bash
pkill -9 fcitx5; sleep 1
sed -i 's/^PreeditEnabledByDefault=.*/PreeditEnabledByDefault=False/' ~/.config/fcitx5/config
fcitx5 -d
```

**② 候选框配色（可选，深色圆角）** —— kimpanel 默认跟随系统主题，想定制就改它的样式表 `~/.local/share/gnome-shell/extensions/kimpanel@kde.org/stylesheet.css`。关键类：`.kimpanel-popup-content`（框体）、`.kimpanel-label`（文字色）、`.kimpanel-candidate-item:active`（键盘高亮项；`:hover` 是鼠标悬停）。完整可改项与一版深色圆角样式见 `kimpanel-theming.md`。

---

## 7. 收尾与验证

```bash
# 让运行中的 fcitx5 重载配置（或 pkill -9 fcitx5 后重新 fcitx5 -d）
fcitx5-remote -r

# 自检（看环境变量、前端是否齐全）
fcitx5-diagnose | less
```

- 切中英 / 切输入法：默认 **Ctrl+Space**；Rime 内部切方案（雾凇/双拼/九宫格）：**Ctrl+`**（反引号）。
- 验证雾凇整句：打一长串不加空格的拼音如 `jintiantianqibucuo`，能整句出「今天天气不错」。
- 输入法列表里 Rime 显示为「**中州韵**」，当前方案默认是「雾凇拼音」。

### ⚠️ 一定要做：注销重新登录
`/etc/environment` 的环境变量要重登才全局生效。装完务必**注销→重新登录**一次，整套才稳定。

---

## 8. 常见坑（实战踩过的）

1. **环境变量值是 `fcitx` 不是 `fcitx5`**（见第 2 节）。最容易错、错了 Qt 应用打不出字。
2. **改 profile 前要 pkill fcitx5**，否则被内存配置覆盖（见 5.5）。
3. **别在旧的 fcitx5-configtool 窗口点「应用」**——它可能拿旧配置覆盖你刚改的 profile。要改重开一个。
4. **Rime 候选数和 fcitx5 拼音的 PageSize 是两套**，雾凇要单独用 `default.custom.yaml` 补丁（见 5.3）。
5. **GNOME 下没有真·毛玻璃模糊**（compositor 不给第三方输入法弹窗做模糊），用 opaque 主题即可。
6. **GNOME 下候选窗口偶尔位置偏/压不到 shell 之上**：可装 Kimpanel GNOME 扩展（apt 无此包，需 extensions.gnome.org）根治，非必须（见 6.5）。
7. **不同程序皮肤/引擎不一致**：fcitx5 对 X11（XWayland）和 Wayland 应用分两路渲染候选框，热重载（`fcitx5-remote -r`）换主题可能只刷一路 → 临时不一致；`pkill -9 fcitx5` 重启即统一（注销重登后不会有）。另外 fcitx5 默认每个程序各记各的输入法 → 若输入法组里同时有雾凇和拼音，不同程序会出现两套候选，所以本配置只放雾凇。

---

## 9. 回滚（退回 ibus）

```bash
im-config -n ibus
sudo sed -i '/^GTK_IM_MODULE=/d;/^QT_IM_MODULE=/d;/^XMODIFIERS=/d' /etc/environment
rm -f ~/.config/autostart/fcitx5.desktop
# 如需彻底卸载：sudo apt purge 'fcitx5*' && sudo apt autoremove
# 注销重登生效
```

---

## 附：如果坚持要用搜狗本体 → 切 Xorg 会话

重装搜狗 deb，然后**登录界面右下角齿轮选「Ubuntu on Xorg」**。fcitx4 搜狗在 Xorg 下不闪屏、可正常用。代价是放弃 Wayland（对日常桌面几乎无感）。步骤同 fcitx4：`sudo apt install -y <sogoupinyin>.deb` → `im-config -n fcitx` → 注销选 Xorg 登录。
