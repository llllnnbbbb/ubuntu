# Ubuntu 26 上 fcitx5 配置与优化

本文基于 Ubuntu 26（GNOME / Wayland）环境，说明在 **系统已预装 fcitx5** 的前提下，如何安装并启用 Rime + **雾凇拼音（rime-ice）**、安装候选栏主题，以及卸载多余输入法相关包。

---

## 1. 配置 fcitx5 为默认输入法，并安装 / 启用 Rime（雾凇）

### 1.1 确认 fcitx5 已安装

```bash
fcitx5 --version
dpkg -l fcitx5 | awk '/^ii/{print}'
```

若未安装，可执行：

```bash
sudo apt update
sudo apt install fcitx5 fcitx5-config-qt fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 \
  fcitx5-frontend-qt5 fcitx5-frontend-qt6
```

### 1.2 将 fcitx5 设为系统默认输入法

#### （1）用 im-config 选择 fcitx5

```bash
im-config -n fcitx5
```

成功后，用户配置文件应为：

`~/.xinputrc`

```text
run_im fcitx5
```

#### （2）写入环境变量（建议统一放在 `/etc/environment`）

编辑 `/etc/environment`，保留 PATH，并加入：

```bash
# fcitx5 — 输入法环境变量
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
QT_IM_MODULES=wayland;fcitx
XMODIFIERS=@im=fcitx
```

说明：

| 变量 | 作用 |
|------|------|
| `GTK_IM_MODULE` | GTK 程序加载 fcitx 模块 |
| `QT_IM_MODULE` | Qt5 使用 fcitx |
| `QT_IM_MODULES` | Qt6 / Wayland 下优先尝试的模块列表；若不设置，`gnome-session` 可能自动注入 `wayland;ibus` |
| `XMODIFIERS` | XIM / 部分老程序使用 |

#### （3）登录自启

将 fcitx5 加入用户自启（若尚无）：

```bash
mkdir -p ~/.config/autostart
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

#### （4）使配置生效

**注销并重新登录**（或重启）。登录后检查：

```bash
echo "$GTK_IM_MODULE $QT_IM_MODULE $QT_IM_MODULES $XMODIFIERS"
# 期望类似：fcitx fcitx wayland;fcitx @im=fcitx

pgrep -a fcitx5
```

### 1.3 安装 fcitx5-rime 与雾凇拼音

#### （1）安装 Rime 前端

```bash
sudo apt install fcitx5-rime
```

建议同时确认已有 Lua 插件（雾凇部分功能依赖）：

```bash
dpkg -l 'librime-plugin-lua*' | awk '/^ii/{print}'
```

#### （2）安装雾凇拼音（rime-ice）

仓库：https://github.com/iDvel/rime-ice  

面向内地简体用户，开箱即用，词库长期维护。用户目录为：

```text
~/.local/share/fcitx5/rime/
```

示例（本机已有 zip 时）：

```bash
cd ~/downloads
unzip -o rime-ice-main.zip
RIME_DIR=~/.local/share/fcitx5/rime

# 备份 installation.yaml（若已有）
cp -a "$RIME_DIR/installation.yaml" /tmp/rime-installation.yaml.bak 2>/dev/null || true

# 拷贝雾凇文件（勿覆盖 installation.yaml；无需拷 weasel/squirrel）
rsync -a \
  --exclude 'build/' \
  --exclude 'others/' \
  --exclude 'LICENSE' \
  --exclude 'README.md' \
  --exclude 'AGENTS.md' \
  --exclude 'recipe.yaml' \
  --exclude 'weasel.yaml' \
  --exclude 'squirrel.yaml' \
  ~/downloads/rime-ice-main/ "$RIME_DIR/"

# 恢复 installation.yaml
cp -a /tmp/rime-installation.yaml.bak "$RIME_DIR/installation.yaml" 2>/dev/null || true
```

或用 git：

```bash
git clone --depth=1 https://github.com/iDvel/rime-ice.git
# 再按上面方式同步到 ~/.local/share/fcitx5/rime/
```

也可用官方 plum（东风破）安装，参见雾凇仓库 README。

### 1.4 在 fcitx5 中启用 Rime

#### 图形界面

1. 打开 **Fcitx 5 配置**（`fcitx5-configtool` 或 `fcitx5-config-qt`）  
2. 左侧「输入法」→ 点 `+`  
3. 取消「只显示当前语言」  
4. 搜索并添加 **Rime / 中州韵**  
5. 将不需要的输入法移除；把 Rime 设为组内默认  

#### 配置文件方式

编辑 `~/.config/fcitx5/profile`，示例如下（仅保留 Rime + 美式键盘布局）：

```ini
[Groups/0]
Name=默认
Default Layout=us
DefaultIM=rime

[Groups/0/Items/0]
Name=rime
Layout=

[GroupOrder]
0=默认
```

然后重启 fcitx5：

```bash
fcitx5 -d -r
```

### 1.5 只启用雾凇全拼

雾凇自带的 `default.yaml` 会列出全拼、多种双拼、九键等。若只要全拼，写用户补丁：

`~/.local/share/fcitx5/rime/default.custom.yaml`

```yaml
# encoding: utf-8
patch:
  schema_list:
    - schema: rime_ice
```

然后重新部署（Rime 菜单「重新部署」或重启 fcitx5）：

```bash
fcitx5 -d -r
```

主要文件：

```text
~/.local/share/fcitx5/rime/rime_ice.schema.yaml   # 雾凇拼音
~/.local/share/fcitx5/rime/rime_ice.dict.yaml
~/.local/share/fcitx5/rime/cn_dicts/              # 词库
~/.local/share/fcitx5/rime/lua/                   # Lua 扩展
~/.local/share/fcitx5/rime/default.custom.yaml    # 方案列表补丁
~/.local/share/fcitx5/rime/rime_ice.custom.yaml   # 方案显示名等补丁
```

可选双拼（若要用，在 `schema_list` 中加入对应 id）：

| 方案 | schema id |
|------|-----------|
| 雾凇全拼 | `rime_ice` |
| 小鹤双拼 | `double_pinyin_flypy` |
| 自然码双拼 | `double_pinyin` |
| 微软双拼 | `double_pinyin_mspy` |

### 1.6 托盘文字与字体样式

#### （1）启用托盘文字图标

编辑 `~/.config/fcitx5/conf/classicui.conf`：

```ini
PreferTextIcon=True
```

开启后，**Rime 托盘文字规则**（写在 fcitx5-rime 里，不能靠普通配置关掉）：

| 状态 | 托盘显示 |
|------|----------|
| 中文 | 当前方案 `name` 的 **第一个字** |
| 英文 (ascii) | 固定 **A** |

输入法条目仍是 **中州韵**（`Name` / `Icon`），与托盘首字无关。  
**不要**用「方案名改成 `.cn`」这类技巧。

#### （2）改托盘显示的字（方案显示名）

本机示例：托盘显示 **中**，方案菜单显示 **中-雾凇拼音**。

`~/.local/share/fcitx5/rime/rime_ice.custom.yaml`

```yaml
# encoding: utf-8
# 托盘文字图标取方案名首字 → 「中」
patch:
  schema/name: 中-雾凇拼音
```

然后重新部署 / 重启 fcitx5：

```bash
fcitx5 -d -r
```

#### （3）候选栏字体与托盘文字样式

字体**不在** Rime scheme 里配置，而在经典用户界面：

`~/.config/fcitx5/conf/classicui.conf`

```ini
# 候选栏字体
Font="SF Pro 13"
# 菜单字体
MenuFont="Noto Sans CJK SC Medium 11"
# 托盘文字字体（字号、粗细影响「中」的观感）
TrayFont="Helvetica Bold 13"
# 托盘文字颜色 / 描边
TrayTextColor=#ffffff
TrayOutlineColor=#000000
```

也可在「Fcitx 5 配置 → 附加组件 → 经典用户界面」中修改。改完后重启 fcitx5。

---

## 2. 配置候选栏主题（从 GitHub 下载安装）

### 2.1 主题安装目录

用户主题目录（推荐，不需 root）：

```text
~/.local/share/fcitx5/themes/
```

系统主题目录（需 root，升级可能被覆盖）：

```text
/usr/share/fcitx5/themes/
```

每个主题是一个**独立文件夹**，内部至少包含 `theme.conf`，以及面板 / 高亮等图片资源。

### 2.2 从 GitHub 下载并安装

推荐主题仓库（Catppuccin）：

- https://github.com/catppuccin/fcitx5

其它可选：Fluent（https://github.com/Reverier-Xu/Fluent-fcitx5）等。

以 Catppuccin 为例：

```bash
cd ~/downloads

# 1. 下载并解压
git clone --depth=1 https://github.com/catppuccin/fcitx5.git
# 或浏览器下载 ZIP 后：unzip fcitx5-main.zip

# 2. 仓库内 themes/ 下是各配色变体（每子目录含 theme.conf）
#    例如：catppuccin-macchiato-blue、catppuccin-mocha-mauve 等
ls fcitx5/themes

# 3. 拷贝到用户主题目录（可全部拷贝，或只拷需要的）
mkdir -p ~/.local/share/fcitx5/themes
cp -r fcitx5/themes/* ~/.local/share/fcitx5/themes/
# 只装一个示例：
# cp -r fcitx5/themes/catppuccin-macchiato-blue ~/.local/share/fcitx5/themes/
```

要求：拷贝后的路径类似：

```text
~/.local/share/fcitx5/themes/catppuccin-macchiato-blue/theme.conf
```

### 2.3 启用主题

#### 图形界面

打开 **Fcitx 5 配置 → 附加组件 → 经典用户界面 → 主题**，选择刚安装的主题。

#### 配置文件

编辑 `~/.config/fcitx5/conf/classicui.conf`：

```ini
Theme=主题目录名
DarkTheme=主题目录名
UseDarkTheme=False
```

例如当前机器上使用的：

```ini
Theme=catppuccin-macchiato-blue
DarkTheme=catppuccin-macchiato-blue
```

对应目录：

```text
~/.local/share/fcitx5/themes/catppuccin-macchiato-blue/
```

改完后重启：

```bash
fcitx5 -r
```

### 2.4 注意

- 主题只影响 **候选面板 / 经典 UI**，不等于托盘图标主题（托盘图标走系统图标主题，如 WhiteSur）。  
- 若列表里看不到新主题，检查目录名、`theme.conf` 是否存在，以及是否重启了 fcitx5。  

---

## 3. fcitx5 瘦身（卸载不必要的包）

目标：只保留 **fcitx5 + Rime（雾凇）+ 必要前端**，去掉多余引擎与系统自带拼音方案包。

### 3.1 建议保留

```text
fcitx5
fcitx5-data
fcitx5-modules
fcitx5-rime
fcitx5-config-qt              # 图形配置，不用可卸
fcitx5-frontend-gtk3
fcitx5-frontend-gtk4
fcitx5-frontend-qt5           # 仍有 Qt5 程序时建议留
fcitx5-frontend-qt6
fcitx5-frontend-all           # 元包，依赖上面若干前端
librime-plugin-lua            # 雾凇部分功能需要
rime-essay / rime-prelude     # librime 基础数据
librime1t64 / librime-data 等自动依赖
# 词库与方案在用户目录雾凇文件中，不依赖 apt 的 luna 包
```

### 3.2 可安全卸载（示例）

#### fcitx5 自带拼音 / 码表（与 Rime 重复）

```bash
sudo apt remove --purge -y \
  fcitx5-chinese-addons \
  fcitx5-chinese-addons-bin \
  fcitx5-chinese-addons-data \
  fcitx5-pinyin \
  fcitx5-pinyin-gui \
  fcitx5-table \
  fcitx5-module-chttrans \
  fcitx5-module-cloudpinyin \
  fcitx5-module-fullwidth \
  fcitx5-module-pinyinhelper \
  fcitx5-module-punctuation \
  fcitx5-module-lua \
  fcitx5-module-lua-common
```

#### IBus 具体引擎（勿卸 `ibus` 本体，GNOME 依赖它）

```bash
sudo apt remove --purge -y \
  ibus-chewing \
  ibus-libpinyin \
  ibus-m17n \
  ibus-table \
  ibus-table-cangjie \
  ibus-table-cangjie-big \
  ibus-table-cangjie3 \
  ibus-table-cangjie5 \
  ibus-table-quick-classic \
  ibus-table-wubi
```

#### 系统自带 Rime 方案包（已用雾凇后可卸）

```bash
sudo apt remove --purge -y \
  rime-data-luna-pinyin \
  rime-data-bopomofo \
  rime-data-cangjie5 \
  rime-data-terra-pinyin \
  rime-data-stroke \
  librime-bin \
  fcitx5-frontend-gtk2
```

**不要**卸载：`fcitx5-rime`、`rime-essay`、`rime-prelude`、`librime-plugin-lua`。

务必配合 **§1.5** 的 `default.custom.yaml`（只保留 `rime_ice`），否则方案菜单可能仍列出无效项。

#### 清理孤立依赖

```bash
sudo apt autoremove -y
```

### 3.3 不要卸载

| 包 | 原因 |
|----|------|
| `ibus` / `ibus-gtk*` / `ibus-data` | 被 `gnome-shell`、`ubuntu-desktop-minimal` 依赖 |
| `fcitx5` / `fcitx5-rime` | 当前输入法核心 |
| `librime-plugin-lua` | 雾凇扩展功能依赖 |

### 3.4 瘦身后自检

```bash
# 应能看到 fcitx5-rime；不应再有 rime-data-luna-pinyin
dpkg -l 'fcitx5*' 'rime-*' | awk '/^ii/{print $2}'

# 进程与当前输入法
pgrep -a fcitx5
fcitx5-remote -n   # 期望：rime

# 雾凇方案文件
ls ~/.local/share/fcitx5/rime/rime_ice.schema.yaml
```

磁盘上的 fcitx5 输入法定义宜只剩 Rime：

```bash
ls /usr/share/fcitx5/inputmethod/
# 期望主要有：rime.conf
```

---

## 附录：常用路径速查

| 用途 | 路径 |
|------|------|
| 用户配置 | `~/.config/fcitx5/` |
| 输入法列表 | `~/.config/fcitx5/profile` |
| 经典 UI / 字体 / 托盘样式 | `~/.config/fcitx5/conf/classicui.conf` |
| 用户主题 | `~/.local/share/fcitx5/themes/` |
| Rime / 雾凇用户数据 | `~/.local/share/fcitx5/rime/` |
| 雾凇方案 | `~/.local/share/fcitx5/rime/rime_ice.schema.yaml` |
| 方案列表补丁 | `~/.local/share/fcitx5/rime/default.custom.yaml` |
| 方案显示名补丁（托盘首字） | `~/.local/share/fcitx5/rime/rime_ice.custom.yaml` |
| 雾凇上游 | https://github.com/iDvel/rime-ice |
| 默认 IM（im-config） | `~/.xinputrc` |
| 全局环境变量 | `/etc/environment` |
| 自启 | `~/.config/autostart/org.fcitx.Fcitx5.desktop` |
| 可执行文件 | `/usr/bin/fcitx5` |

---

## 附录：重启与排错

```bash
# 重启 fcitx5
fcitx5 -d -r

# 查看环境变量
printenv | grep -E 'IM_MODULE|XMODIFIERS'

# 若 Qt 仍像走 ibus：确认 /etc/environment 含
# QT_IM_MODULES=wayland;fcitx
# 然后重新登录

# 确认当前 Rime 方案列表（应含 rime_ice）
# 可在托盘 Rime 菜单查看，或重新部署后再试输入
```

改环境变量或 im-config 后，务必 **注销重新登录** 再验证应用程序内中文输入。
