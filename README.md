# linux-wayland-chinese-input-method

一份实战验证的指南（打包成 [Claude skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)），用来在 **Linux + GNOME Wayland 上把中文拼音输入法装好**——方案是 **fcitx5 + Rime**，配合 **rime-ice（雾凇拼音）** 配置。

> 搜狗在 Wayland 下闪屏、打不出字；这里给出 fcitx5 + Rime/雾凇 的完整替代方案，纯本地、不上传隐私。

![候选框示意 — 深色圆角，选中项蓝色高亮](assets/candidate-mockup.svg)

## 它解决什么问题

如果你在 **GNOME Wayland** 上折腾过中文输入，多半撞过下面某一条：

- **搜狗 (Sogou) 候选框无限闪屏、打不出字**——因为它的 Linux 版至今是 fcitx4，GNOME Wayland 不接它。
- **Qt 应用打不出中文**，即使装了 fcitx5——几乎都是 `IM_MODULE` 环境变量的坑（值必须是 `fcitx`，**不是** `fcitx5`）。
- **终端里候选框闪烁**，但 Chrome / Electron 应用却正常。
- 好不容易装好了，但**候选框丑**，想换皮肤。

这份指南把整套修法走一遍，每个踩过的坑都标了出来。

## 仓库内容

| 文件 | 说明 |
|------|------|
| [`SKILL.md`](SKILL.md) | skill 本体——带 frontmatter 的精简安装/配置流程。 |
| [`references/install-walkthrough.md`](references/install-walkthrough.md) | 完整中文实战 walkthrough，每一步 + 每个坑。 |
| [`references/kimpanel-theming.md`](references/kimpanel-theming.md) | 候选框皮肤能改到什么程度（kimpanel / St CSS），附一版深色圆角基线。 |

## 快速上手

精简版（完整带解释的版本见 `SKILL.md`）：

```bash
# 1. 装 fcitx5 + Rime
sudo apt update
sudo apt install -y fcitx5 fcitx5-chinese-addons fcitx5-config-qt \
  fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 fcitx5-frontend-qt5 fcitx5-rime
im-config -n fcitx5

# 2. 环境变量 —— 值是 `fcitx`,不是 `fcitx5`
sudo sed -i '/^GTK_IM_MODULE=/d;/^QT_IM_MODULE=/d;/^XMODIFIERS=/d' /etc/environment
printf 'GTK_IM_MODULE=fcitx\nQT_IM_MODULE=fcitx\nXMODIFIERS=@im=fcitx\n' | sudo tee -a /etc/environment

# 3. 装雾凇 (rime-ice) 配置
git clone --depth 1 https://github.com/iDvel/rime-ice.git /tmp/rime-ice
mkdir -p ~/.local/share/fcitx5/rime && cp -r /tmp/rime-ice/* ~/.local/share/fcitx5/rime/
rime_deployer --build ~/.local/share/fcitx5/rime /usr/share/rime-data

# 4. 注销,重新登录。
```

切换输入法 **Ctrl+Space**；切换 Rime 方案 **Ctrl+`**（反引号）。

## 当作 Claude skill 使用

[Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) 能让 Claude 在你问到「Linux 上中文输入」相关问题时自动调用这套流程。在 [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) 里安装：

```bash
git clone https://github.com/Rong-Tao/linux-wayland-chinese-input-method.git \
  ~/.claude/skills/linux-wayland-chinese-input-method
```

然后对 Claude 说一句类似 *「我在 Ubuntu Wayland 上中文输入法打不出字，帮我装一下」*，它就会照这份指南来做。

## 致谢

基于这些开源项目：

- [Rime / 中州韵](https://rime.im/) —— 输入法引擎
- [rime-ice / 雾凇拼音](https://github.com/iDvel/rime-ice) —— 让它逼近搜狗手感的配置
- [fcitx5](https://github.com/fcitx/fcitx5) —— 输入法框架
- [fcitx5-mellow-themes](https://github.com/sanweiya/fcitx5-mellow-themes) —— 圆角主题

## 许可证

[MIT](LICENSE)
