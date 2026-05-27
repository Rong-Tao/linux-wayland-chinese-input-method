---
name: linux-wayland-chinese-input-method
description: >-
  Install and configure a Chinese pinyin input method on Linux running GNOME
  Wayland (Ubuntu/Debian), using fcitx5 + Rime with the rime-ice (雾凇拼音)
  config. Use this whenever a user on Linux/GNOME/Wayland wants Chinese input,
  or mentions 中文输入法 / 拼音 / fcitx5 / rime / 雾凇 / 搜狗(Sogou), or reports
  symptoms like: the candidate window flickers, Sogou (搜狗) doesn't work on
  Wayland, Qt apps can't type Chinese, no candidates appear in the terminal, or
  they want to theme/style the candidate popup (kimpanel / mellow). Covers the
  whole fix: why Sogou (fcitx4) fails on Wayland and the fcitx5+Rime
  replacement, the critical IM_MODULE=fcitx env-var gotcha, autostart on
  Wayland, candidate count, theming, and a clean rollback.
---

# Chinese input method on Linux + GNOME Wayland (fcitx5 + Rime/雾凇)

> Battle-tested on **Ubuntu 24.04, GNOME, Wayland**, installed from scratch and
> verified working. Adapt version numbers to your distro.
>
> **In one line:** fcitx5 (framework) + Rime/雾凇拼音 rime-ice (the open-source
> setup closest to Sogou's feel) + mellow theme (modern rounded corners). Fully
> local, no flicker, no privacy upload.

## When and why to use this

GNOME deliberately integrates **only ibus** natively and does not speak the
Wayland input-method protocol for third-party IMEs. Two consequences drive this
whole guide:

- **Sogou (搜狗) on Linux is still fcitx4.** Under GNOME Wayland its candidate
  window **flickers endlessly and you can't type**. It's closed-source and can't
  be ported to fcitx5. Don't install fcitx4 Sogou on Wayland.
- **fcitx5 + Rime is the most practical, stable replacement.** It works for the
  vast majority of GTK/Qt apps via the IM modules (no flicker, far better than
  fcitx4), and Rime + 雾凇 (rime-ice) is the closest open-source feel to Sogou.

If the user insists on Sogou specifically, the only working path is to log into
an **Xorg** session instead of Wayland (see the rollback/alternatives notes at
the end and `references/install-walkthrough.md`).

## Target environment

| Item | Value |
|------|-------|
| OS | Ubuntu/Debian (tested on Ubuntu 24.04 LTS) |
| Desktop | GNOME / **Wayland** |
| Framework | switch from ibus to **fcitx5** |
| sudo | required (installing packages) |

## Steps

Follow these in order. The full, annotated walkthrough with every gotcha is in
`references/install-walkthrough.md` — read it if a step misbehaves. For styling
the candidate popup, see `references/kimpanel-theming.md`.

### 1. Install the fcitx5 stack

```bash
sudo apt update
sudo apt install -y \
  fcitx5 fcitx5-chinese-addons fcitx5-config-qt \
  fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 fcitx5-frontend-qt5 \
  fcitx5-material-color
```

Switch the default IM framework to fcitx5:

```bash
im-config -n fcitx5      # writes ~/.xinputrc as: run_im fcitx5
```

### 2. Environment variables (⚠️ the #1 gotcha: the value is `fcitx`, NOT `fcitx5`)

fcitx5's Qt/GTK plugins still register under the name **`fcitx`** (the Qt class
is `fcitx::QFcitxPlatformInputContext`). Writing `fcitx5` makes Qt apps unable to
type. `fcitx5-diagnose` will warn about this explicitly.

```bash
# Clear any stale values first, then write
sudo sed -i '/^GTK_IM_MODULE=/d;/^QT_IM_MODULE=/d;/^XMODIFIERS=/d' /etc/environment
sudo tee -a /etc/environment >/dev/null <<'EOF'
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
EOF
```

> On GNOME Wayland, Qt5/XWayland apps need these variables. GTK apps could use
> native text-input-v3, but keeping a global `GTK_IM_MODULE=fcitx` is the most
> universal, hassle-free choice.

### 3. Autostart

GNOME Wayland does not run `~/.xinputrc`; use an XDG autostart entry instead:

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

### 4. (Optional) Default pinyin engine settings

Skip this entire step if you only use Rime/雾凇 (recommended). It's only for
keeping fcitx5's built-in pinyin around for comparison.

Cloud pinyin via Baidu (Google's endpoint often times out in mainland China):

```bash
mkdir -p ~/.config/fcitx5/conf
cat > ~/.config/fcitx5/conf/cloudpinyin.conf <<'EOF'
Backend=Baidu
MinimumPinyinLength=4
EOF
```

fcitx5's candidate count maxes out at 10 — set `PageSize=10` in
`~/.config/fcitx5/conf/pinyin.conf` (that file is generated after the first run).

### 5. Rime + 雾凇拼音 (the core)

**5.1 Install the engine**

```bash
sudo apt install -y fcitx5-rime
```

> Pulls in the key plugins: `librime-plugin-lua` (雾凇 Lua features) and
> `librime-plugin-octagram` (whole-sentence grammar model). librime 1.10 is
> compatible with 雾凇 with no errors.

**5.2 Fetch the 雾凇 (rime-ice) config**

```bash
git clone --depth 1 https://github.com/iDvel/rime-ice.git /tmp/rime-ice
mkdir -p ~/.local/share/fcitx5/rime
cp -r /tmp/rime-ice/* ~/.local/share/fcitx5/rime/
```

**5.3 Set Rime's candidate count to 10** (Rime's count is independent of
fcitx5; 雾凇 defaults to 5):

```bash
cat > ~/.local/share/fcitx5/rime/default.custom.yaml <<'EOF'
patch:
  menu/page_size: 10
EOF
```

**5.4 Deploy** (compiles hundreds of thousands of entries; first run is slow):

```bash
rime_deployer --build ~/.local/share/fcitx5/rime /usr/share/rime-data
```

Success marker: `~/.local/share/fcitx5/rime/build/rime_ice.table.bin` (~58 MB)
and friends are generated with no errors.

**5.5 Add Rime to the input-method group**

Edit `~/.config/fcitx5/profile`: put `rime` **first**, followed by
`keyboard-us`, and do **not** include fcitx5's built-in `pinyin` (mixing them
makes different apps remember different engines and show two candidate sets).
Set `DefaultIM=rime`. Minimal template:

```ini
[Groups/0]
Name="Default"
Default Layout=us
DefaultIM=rime

[Groups/0/Items/0]
Name=rime
Layout=

[Groups/0/Items/1]
Name=keyboard-us
Layout=

[GroupOrder]
0="Default"
```

> ⚠️ **`pkill -9 fcitx5` before editing `profile`**, otherwise fcitx5 overwrites
> the file from its in-memory config on exit. Then restart: `fcitx5 -d`.

### 6. Theme (modern rounded corners, optional)

```bash
git clone --depth 1 https://github.com/sanweiya/fcitx5-mellow-themes.git /tmp/mellow
mkdir -p ~/.local/share/fcitx5/themes
cp -r /tmp/mellow/mellow-* ~/.local/share/fcitx5/themes/   # opaque variant for GNOME
```

Set theme + font in `~/.config/fcitx5/conf/classicui.conf`:

```ini
Vertical Candidate List=False
WheelForPaging=True
Font="Sans 11"
PerScreenDPI=True
Theme=mellow-wechat
```

Palettes: `mellow-{youlan|sakura|vermilion|wechat|graphite}`, each with a
`-dark` variant. For deeper customization of the candidate popup (especially the
flicker fix via kimpanel), see **`references/kimpanel-theming.md`**.

### 7. Finish & verify

```bash
fcitx5-remote -r            # reload config (or pkill -9 fcitx5 then fcitx5 -d)
fcitx5-diagnose | less      # self-check: env vars, frontends
```

- Toggle IME: **Ctrl+Space**. Switch Rime schema (雾凇/double-pinyin/etc.): **Ctrl+`** (backtick).
- Verify whole-sentence 雾凇: type a long unsegmented string like
  `jintiantianqibucuo` → it should produce「今天天气不错」.
- In the IME list Rime shows as「**中州韵 / Rime**」, default schema「雾凇拼音」.

### ⚠️ Must do: log out and back in

The `/etc/environment` variables only take full global effect after a re-login.
Log out → log back in once after installing.

## Common pitfalls

1. **Env-var value is `fcitx`, not `fcitx5`** (Step 2) — the most common mistake; gets it wrong and Qt apps can't type.
2. **`pkill fcitx5` before editing `profile`** (Step 5.5), or it gets overwritten by the in-memory config.
3. **Don't click "Apply" in an old `fcitx5-configtool` window** — it may overwrite the profile you just edited. Reopen a fresh one.
4. **Rime's candidate count and fcitx5 pinyin's PageSize are two separate settings** — patch Rime via `default.custom.yaml` (Step 5.3).
5. **No real frosted-glass blur on GNOME** (the compositor won't blur third-party IME popups) — use an opaque theme.
6. **Candidate window occasionally mis-positioned / flickering in native Wayland apps** (e.g. the terminal) — install the Kimpanel GNOME extension to fix it root-and-branch. See `references/kimpanel-theming.md`. Not required.
7. **Inconsistent skin/engine across apps** — fcitx5 renders candidates on two paths (X11/XWayland vs Wayland); a hot reload may refresh only one. `pkill -9 fcitx5` + restart unifies them (and a re-login prevents it entirely). Also, with both 雾凇 and pinyin in the group, apps remember different engines — so this guide uses 雾凇 only.

## Rollback (back to ibus)

```bash
im-config -n ibus
sudo sed -i '/^GTK_IM_MODULE=/d;/^QT_IM_MODULE=/d;/^XMODIFIERS=/d' /etc/environment
rm -f ~/.config/autostart/fcitx5.desktop
# Full uninstall if desired: sudo apt purge 'fcitx5*' && sudo apt autoremove
# Log out and back in to apply.
```
