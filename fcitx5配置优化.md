# Ubuntu 26 上 fcitx5 配置与优化

本文基于 Ubuntu 26（GNOME / Wayland）环境，说明在 **系统已预装 fcitx5** 的前提下，如何安装并启用 Rime、安装候选栏主题，以及卸载多余输入法相关包。

---

## 1. 配置 fcitx5 为默认输入法，并安装 / 启用 Rime

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

### 1.3 安装 Rime

```bash
sudo apt install fcitx5-rime rime-data-luna-pinyin
```

- `fcitx5-rime`：fcitx5 的 Rime 前端  
- `rime-data-luna-pinyin`：朙月拼音等方案（含简化字方案）  
- 依赖会自动带上 `rime-essay`、`rime-prelude`；`rime-data-stroke` 往往被 luna 依赖，勿随意删  

可选其它方案（按需，非必须）：

```bash
# 示例：地球拼音、注音、仓颉（不装也不影响基础拼音）
# sudo apt install rime-data-terra-pinyin rime-data-bopomofo rime-data-cangjie5
```

### 1.4 启用 Rime

#### 图形界面

1. 打开 **Fcitx 5 配置**（`fcitx5-configtool` 或 `fcitx5-config-qt`）  
2. 左侧「输入法」→ 点 `+`  
3. 取消「只显示当前语言」  
4. 搜索并添加 **Rime / 中州韵**  
5. 将不需要的输入法（如系统自带拼音）移除；把 Rime 设为组内默认  

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
fcitx5 -r
# 或
fcitx5 -d -r
```

### 1.5 托盘显示文字（可选）

若不想用主题图标，可在「附加组件 → 经典用户界面」中勾选 **优先使用文字图标**，或编辑：

`~/.config/fcitx5/conf/classicui.conf`

```ini
PreferTextIcon=True
TrayTextColor=#ffffff
TrayOutlineColor=#000000
```

Rime 默认标签多为 `ㄓ`。若要改成自定义文字（如 `CN`），可写用户覆盖：

`~/.local/share/fcitx5/inputmethod/rime.conf`

```ini
[InputMethod]
Name=Rime
Name[zh_CN]=中州韵
Icon=fcitx-rime
Label=CN
LangCode=zh
Addon=rime
Configurable=True
```

注意：Rime 在中文态会用「方案名首字」覆盖 Label。若要完整显示多字符 Label，需给方案名加前导 `.`，例如：

`~/.local/share/fcitx5/rime/luna_pinyin_simp.custom.yaml`

```yaml
# encoding: utf-8
patch:
  schema/name: .cn
```

然后重新部署 / 重启 fcitx5。英文态仍显示 `A`。

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

目标：只保留 **fcitx5 + Rime 拼音 + 必要前端**，去掉多余引擎与方案。

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
rime-data-luna-pinyin
rime-data-stroke              # luna 依赖，勿单独卸
rime-essay
rime-prelude
librime1t64 / librime-data 等自动依赖
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

#### 不用的 Rime 方案与工具

在只用朙月拼音的前提下：

```bash
sudo apt remove --purge -y \
  rime-data-bopomofo \
  rime-data-cangjie5 \
  rime-data-terra-pinyin \
  librime-bin \
  fcitx5-frontend-gtk2
```

**不要**卸载：`rime-data-luna-pinyin`、`rime-data-stroke`、`rime-essay`、`rime-prelude`。

#### 清理孤立依赖

```bash
sudo apt autoremove -y
```

### 3.3 不要卸载

| 包 | 原因 |
|----|------|
| `ibus` / `ibus-gtk*` / `ibus-data` | 被 `gnome-shell`、`ubuntu-desktop-minimal` 依赖 |
| `fcitx5` / `fcitx5-rime` | 当前输入法核心 |
| `rime-data-luna-pinyin` | 正在使用的拼音方案 |

### 3.4 瘦身后自检

```bash
# 应能看到 fcitx5-rime 与 luna
dpkg -l 'fcitx5*' 'rime-*' | awk '/^ii/{print $2}'

# 进程与当前输入法
pgrep -a fcitx5
fcitx5-remote -n   # 期望：rime
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
| 经典 UI / 主题 | `~/.config/fcitx5/conf/classicui.conf` |
| 用户主题 | `~/.local/share/fcitx5/themes/` |
| Rime 用户数据 | `~/.local/share/fcitx5/rime/` |
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
```

改环境变量或 im-config 后，务必 **注销重新登录** 再验证应用程序内中文输入。
