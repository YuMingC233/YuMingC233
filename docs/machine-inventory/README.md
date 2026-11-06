# Ubuntu 工作机迁移清单

最后核验：**2026-11-06（Asia/Taipei）**。用途：换机、重装系统前清点依赖和恢复顺序。

这是一份本机只读清点快照，不是一键重装脚本。系统版本、驱动、软件包、扩展和服务来自现场命令；历史修复说明来自本机保留的安装记录，并与当前安装状态交叉核对。软件存在不代表仍在日常使用，也不代表每项都是手动安装。

公开仓库仅保存软件名称、版本、方案、配置路径和恢复方法。密码、令牌、SSH 密钥、网络地址、代理端口、浏览器资料、业务数据、实际连接参数和私人词库内容通过私有备份迁移。

## 1. 系统概述

| 项目 | 本次核验结果 |
| --- | --- |
| 系统 | Ubuntu **24.04.4 LTS**，Noble Numbat |
| 架构 | x86_64 / amd64 |
| 当前内核 | `7.0.0-31-generic`，不能根据发行版名称推断为默认内核 |
| 桌面 / 会话 | GNOME Shell **46.0**，Ubuntu，**X11** |
| CPU | Intel Core i5-6500，4 核 4 线程 |
| 内存 / Swap | 约 16 GB 内存，8 GiB Swap |
| 磁盘 | SATA 1 TB，ST1000DM010；根文件系统 ext4 |
| 图形驱动 | Intel HD Graphics 530，内核驱动 `i915` |
| 有线网卡 | Realtek RTL8111/8168 系列，内核驱动 `r8169` |
| 当前 USB 无线网卡 | AICSemi AIC 8800D80，USB ID `368b:8d81` |
| Secure Boot | disabled；固件处于 Setup Mode |
| Shell / 语言 | zsh；`LANG=zh_CN.UTF-8`，当前终端 `LC_CTYPE=C.UTF-8` |
| 时区 | Asia/Taipei |
| APT hold | `linux-headers-generic`、`linux-image-generic`；见 [完整列表](inventory/apt-held.tsv) |

迁移时重新检测新机器硬件。内核 hold 只记录现状，不应在新机器上未经核对直接照搬。

启动配置目前为 `GRUB_DEFAULT=0`、`GRUB_TIMEOUT_STYLE=hidden`、`GRUB_TIMEOUT=0`、内核参数 `quiet splash`、`GRUB_DISABLE_OS_PROBER=FALSE`。来源是 `/etc/default/grub`，本次未判断每个值是否由用户修改。`/etc/sysctl.d/10-bufferbloat.conf`、`10-map-count.conf` 和 `/etc/modprobe.d/disable-algif_aead.conf` 分别归属发行版的 procps/kmod 包，不列为个人手动配置。

## 2. 手动安装的网卡驱动（优先保全）

### AIC8800D80：当前正在使用

| 项目 | 状态 / 路径 |
| --- | --- |
| DKMS 包 | `aic8800/1.0.0` |
| 已构建内核 | `7.0.0-29-generic`、`7.0.0-31-generic` |
| 模块 | `aic8800_fdrv`、`aic_load_fw`；本次两者均已加载 |
| 主模块版本 | `6.4.3.0`；与 DKMS 包版本不同 |
| 当前模块文件 | `/lib/modules/7.0.0-31-generic/updates/dkms/aic8800_fdrv.ko.zst` |
| 源码工作目录 | `~/Coding/OpenSource_Project/aic8800d80/` |
| 工作目录远端 | <https://github.com/shenmintao/aic8800d80> |
| 工作目录提交 | `a5db07c`，`fix: install all firmware variants and usb_modeswitch config`；不能据此保证工作目录无未提交改动 |
| DKMS 源码 | `/usr/src/aic8800-1.0.0/` |
| 固件 | `/lib/firmware/aic8800D80/`（与 `/usr/lib/firmware/` 对应） |
| 相关配套 | 源码中的 `fw/`、`usb_modeswitch/`、`aic.rules`、`install.sh`、`INSTALL_SCRIPT.md` |

迁移前私下归档源码工作目录（包含本地修改）、DKMS 源码、实际固件目录，以及 `/etc/usb_modeswitch.d/`、`/etc/udev/rules.d/40-usb_modeswitch.rules` 中适用配置。只复制 `.ko` 不能替代针对新内核重新构建。

恢复时先准备有线网络或离线包，安装 `build-essential`、`dkms`、对应目标内核的 headers，阅读归档源码的安装说明后安装驱动和固件。若新机器开启 Secure Boot，还需要处理模块签名；本机关闭状态不代表新机器也关闭。

### RTL8851BU：已安装，当前未加载

| 项目 | 状态 / 路径 |
| --- | --- |
| DKMS 包 | `rtl8851bu/1.19.10` |
| 模块 / 版本 | `8851bu`，`v1.19.10-78-gfbe3fba11.20240422` |
| 已构建内核 | `7.0.0-29-generic`、`7.0.0-31-generic` |
| 实际 DKMS 源码 | `/usr/src/rtl8851bu-1.19.10/` |
| 另有旧源码目录 | `/usr/src/rtl8851bu-0.1/`；不能当作当前启用版本 |
| 来源核验限制 | 本地 README 提及 `fofajardo/rtl8851bu`，未确认原安装来源或精确源码提交 |

保留源码和 `dkms.conf`。迁移后仅在需要对应硬件时恢复；当前 USB 设备是 AIC8800D80，并非 RTL8851BU。

驱动核验命令：

```bash
uname -r
lsusb
lspci -nnk
dkms status
modinfo aic8800_fdrv
modinfo 8851bu
lsmod | rg 'aic|8851|r8169'
```

## 3. 桌面修补、扩展与输入法

### 本地修补版 Mutter

当前安装的四个运行时包均为 **`46.2-1ubuntu0.24.04.18+local1`**：`libmutter-14-0`、`mutter-common`、`mutter-common-bin`、`gir1.2-mutter-14`。

本机目录 `~/mutter-repair-20261021/` 保存完整修复资料：源码、补丁、构建脚本、安装包、校验文件和官方原包回退目录。历史修复针对 X11 SelectionNotify 的 BadWindow 异常捕获，以及 `meta_compositor_sync_stack` 崩溃。来源记录包含 GNOME 提交 `df48348452407c58e4681c23c91fc174efbf4fed` 和 GNOME 46 MR `!3785`。

迁移前保全该目录，尤其是 `README.md`、`source-release/`、`downloads/`、四个运行时 `.deb`、`rollback/` 和 `rollback.sh`。本次确认的是安装版本，没有重新执行睡眠唤醒验收；历史记录也不能代替当前验收。新系统若已包含官方修复，应重新评估是否还需要本地包，避免跨 GNOME ABI 安装。

### GNOME 扩展和外观

当前启用：

- `kimpanel@kde.org`：Input Method Panel **83**，GNOME 46 兼容版本；当前 Fcitx5 的界面为 `kimpanel`。
- `wired-only-topbar@local`：本地自定义 Wired-only Top Bar **1**，只在顶栏展示有线网络状态；保留 Wi-Fi 控制及路由。
- Ubuntu Dock、Desktop Icons NG、Tiling Assistant、AppIndicator。

完整扩展列表见 [gnome-extensions.tsv](inventory/gnome-extensions.tsv)。本地扩展在 `~/.local/share/gnome-shell/extensions/`，迁移时先检查目标 GNOME 版本兼容性。自定义字体包括 MesloLGS NF 和 JetBrains Mono，见 [字体文件列表](inventory/user-fonts.tsv)。

### Fcitx5 + Rime + 微软双拼 + Kimpanel

| 项目 | 当前配置 |
| --- | --- |
| 框架 / 引擎 | Fcitx5 `5.1.7-1build3`；fcitx5-rime `5.1.4-2build2` |
| 默认框架 | `~/.xinputrc` 中 `run_im fcitx5` |
| 会话变量 | `GTK_IM_MODULE=fcitx`、`QT_IM_MODULE=fcitx`、`XMODIFIERS=@im=fcitx`、`SDL_IM_MODULE=fcitx` |
| 变量入口 | `~/.config/environment.d/fcitx5.conf`；原有 `/etc/environment`、`~/.profile` 也有部分输入法变量 |
| 登录启动 | `~/.config/autostart/org.fcitx.Fcitx5.desktop` |
| 输入法组 | 保留 `keyboard-us`、`shuangpin`、`wetype-im`，新增并默认使用 `rime` |
| Rime 方案来源 | <https://github.com/iDvel/rime-ice>，本次安装时仓库提交 `99662ba` |
| Rime 目录 | `~/.local/share/fcitx5/rime/` |
| 当前方案 | `double_pinyin_mspy`，微软双拼 |
| 候选页大小 | **7**，已确认部署后的 `build/default.yaml` 和 `build/rime_ice.schema.yaml` |
| 当前界面 | GNOME Kimpanel；经典界面配置仍保存在 `~/.config/fcitx5/conf/classicui.conf` |
| 终端预编辑 | `~/.config/fcitx5/conf/rime.conf`：`PreeditInApplication=False` |
| GNOME 设置 | `gtk-im-module=fcitx`；输入源保留 `xkb/us` |
| IBus | 保留安装包和词库；用户级 GNOME / generic 会话服务已 mask，避免争抢自启动 |

IBus 服务名称为 `org.freedesktop.IBus.session.GNOME.service` 和 `org.freedesktop.IBus.session.generic.service`。历史 IBus 还包含自定义 AI 拼音；相关目录仍保留，不因切换框架而删除。

**本次用户设置的页大小仍直接写在 `default.yaml` 中。** 尚无 `default.custom.yaml`，上游更新可能覆盖此处修改。后续调整时可迁移到 Rime 用户 patch，但清点过程不更改现有设置。

迁移必须私下备份整个 `~/.local/share/fcitx5/`、`~/.config/fcitx5/`，以及旧 `~/.config/fcitx/`、`~/.config/ibus/`、`~/.local/share/ibus/`、`~/.local/share/ibus-table/`、`~/.local/share/ibus-ai-pinyin/`、`~/.local/share/wetype-ime/` 中存在的目录。不能只克隆雾凇仓库而遗漏用户词库、`*.userdb`、同步目录和自定义文件。

本次配置备份在 `~/input-method-backups/`；它不是云端备份，需要另存到私人介质。恢复配置后执行 Rime「重新部署」并重启 Fcitx5，登录变量和自启动需注销重登后核验。

## 4. 安装的软件与开发环境

### 主要软件

| 类别 | 已确认的安装 / 文件存在 |
| --- | --- |
| 编辑与 API 开发 | VS Code、Apifox、CC Switch、Vim / spf13-vim、Git、GitHub CLI、Subversion |
| 办公与阅读 | LibreOffice、Obsidian AppImage、Notion Snap、PDF Arranger、Inkscape、Newsboat |
| 截图、剪贴板、传输 | Capturer、Snipaste（`/opt/Snipaste` 与 AppImage 均有）、tiez、LocalSend、BaiduPCS-Go、Crow Translate |
| 自研 / 本地桌面工具 | `ai-disk-cleaner`、`tiez-app`、Lumo；目录或 AppImage 见附表，文件存在不等于每项当前运行 |
| 浏览器 | Mozilla APT 版 Firefox及中文语言包 |
| 远程、共享与网络 | Remmina（RDP/VNC 插件）、SSHFS、Samba/CIFS、Tailscale、OpenVPN、v2ray/v2rayA；另有手机 Clash Meta 控制方案 |
| 容器、数据库 | Docker CE、Buildx、Compose、containerd、PostgreSQL 16、SQLite；容器内另有 MySQL / Redis |
| 系统和硬件 | DKMS、usb-modeswitch、Piper/ratbagd、ADB/Fastboot、GParted、BleachBit、Tesseract 中文 OCR |
| 开发构建 | GCC / build-essential、CMake、Meson、Autotools、Java/Maven、Perl/Tk、.NET、Rust、Go、Node、Python |
| 其他本地依赖 | Sentinel LDK Runtime：`aksusbd` / `hasplmd`；需单独保留合法安装介质和授权迁移资料 |

微信输入法具有 `wetype-im` 输入法组项、`wetype-ime.desktop` 启动入口及用户数据目录；不能仅靠 APT 清单恢复。`/opt/kingsoft` 也存在，但本次没有确认 WPS 的有效安装包、版本和可运行状态，列为迁移前复核项。

详细版本以附表为准。示例：Apifox `2.8.46`、CC Switch `3.19.2`、Capturer `1.6.3`、Obsidian AppImage 文件名 `1.12.7`、Snipaste AppImage 文件名 `2.11.3`、Lumo AppImage 文件名 `0.3.1`。AppImage 文件名仅是标称版本，不等于运行时版本校验。

### 语言运行时与包管理器

| 环境 | 当前状态 | 迁移注意 |
| --- | --- | --- |
| Node / nvm | `v22.22.1`、`v24.18.0`；默认 alias `22` | 使用 nvm；项目依赖优先按 `pnpm-lock.yaml` 恢复 |
| pnpm | nvm Node 22 下安装 `10.32.1`；另有独立全局目录 | 同一 CLI 可能在 nvm、pnpm 等位置各有一份 |
| Python / Miniforge | `~/miniforge3`，Python `3.13.12`、conda `26.1.1`、mamba `2.5.0` | 系统 Python 与用户开发环境分开恢复 |
| micromamba `default312` | Python `3.12.13` | 含 Conda 和 pip 安装项 |
| micromamba `default313` | Python `3.13.14` | 当前主要用户工具环境之一 |
| micromamba `job-schedule` | Python `3.13.14` | 保留浏览器调度依赖隔离 |
| Go / gvm | `go1.26.2`，另有 system 项 | 全局工具在 `~/.gvm/pkgsets/go1.26.2/global/bin/` |
| Rust / rustup | stable，rustc/cargo `1.94.0` | APT 中也有 Rust 包，注意 PATH 优先级 |
| Java | OpenJDK 8、17、21；本次 `java -version` 默认是 8 | 不把所有项目直接切换到同一 JDK |
| .NET | SDK `10.0.112` | 用项目配置核对目标框架 |
| Perl | 系统 `5.38.2`；`~/perl5` 有本地 CPAN 模块 | 保留 `local::lib` / Perl/Tk 路径与依赖 |

Python 每环境包版本见 [python-packages.tsv](inventory/python-packages.tsv)，不是可直接安装的 `requirements.txt`。其中 `conda` 和 `python-distribution` 两类记录会重叠，不可将两列包名混在一起执行安装。项目自带 `.venv` / `venv` 未穷举，按各项目锁文件和私有配置恢复。

### 全局 CLI 与开发扩展

- Node / pnpm：Codex、Claude Code、OpenSpec、Ark Helper、MCP VirusTotal、antd-init、Carbonyl、Yarn；[版本和安装位置](inventory/node-global.tsv)。
- Claude Code 另有原生安装：`~/.local/bin/claude` 指向 `~/.local/share/claude/versions/2.1.318`。pnpm 的同名包不一定是实际执行版本。
- uv tools：`boss-agent-cli`、`specify-cli`、`tieba-cli`；[版本](inventory/uv-tools.tsv)。
- 其他入口：`exam`、`english-chat`、`hermes`、`phone`、`rtk`。部分是指向源码、skill 或项目虚拟环境的符号链接，需恢复其目标，不能只复制链接。
- Go：ffuf、gobuster、dnsx、nuclei、subfinder、Fyne、Wails；[模块和版本](inventory/go-tools.tsv)。
- Cargo：ripgrep `15.2.0`；[Rust 工具](inventory/rust-tools.tsv)。
- VS Code：本次检测到 **94** 个扩展；[扩展 ID 和版本](inventory/vscode-extensions.tsv)。扩展清单不包含编辑器设置、数据库密码或 SSH 连接资料。
- 本地 Perl 模块见 [CPAN 安装回执列表](inventory/perl-local-modules.tsv)；该列表只覆盖 `~/perl5` 的 `.packlist`，不替代 APT Perl 模块清单。

### 软件来源

APT 源文件包括 Docker、GitHub CLI、Mozilla、VS Code、Tailscale、Remmina PPA、LibreOffice PPA 和 Inkscape PPA；见 [源文件名](inventory/apt-source-files.tsv)。只记录名称，不复制可能带代理、私有源或认证信息的完整配置。迁移时重新安装相应官方签名密钥并核对目标发行版支持。

Snap 见 [snap.tsv](inventory/snap.tsv)，其中大量项目是运行时。Flatpak 本次没有得到可用的应用记录；空表不保证机器从未安装过 Flatpak，也不代表 `~/.local/share/flatpak` 不存在。

## 5. 自定义启动、网络与维护方案

### 登录启动

`~/.config/autostart/` 包含 CC Switch、Capturer、Fcitx5、Remmina 面板、Obsidian、Snipaste、tiez；见 [登录启动项](inventory/login-autostart.tsv)。

自定义用户服务包括：

- `clash-ctl-server.service`：手机 Clash Meta 控制界面。
- `lumo-daemon.service`：Lumo 后台服务。
- `aurabot-cloud-tunnel.service`：SSH 隧道；连接参数和密钥只通过私有备份恢复。
- `login-container@.service` / `login-container@.timer`：图形登录后等待网络和 Docker 就绪，再启动选定容器。

三套容器定时器依赖链接位于 `~/.config/systemd/user/graphical-session.target.wants/`：`gitea-runner`、`default_runner`、`hyx-daily-report-server`。延迟 10 秒；当前登录会话已触发。注意 `list-unit-files --state=enabled` 可能只显示模板状态，迁移时也要核对实例链接和 `list-timers --all`。

就绪脚本为 `~/.local/libexec/login-container-start`。旧安装记录将容器策略设置为 `on-failure`；迁移时需用 Docker 实际配置再次核对，公开清单没有导出容器环境变量、卷、端口、标签或内部镜像仓库信息。

### 系统级方案

| 方案 | 当前证据 / 应保存的入口 |
| --- | --- |
| InCAM 远程文件系统 | `mnt-incamhyx.automount` 已启用；`mnt-incamhyx.mount`、SSHFS、私有 SSH 配置与凭据 |
| PostgreSQL 按需启动 | `postgresql.service` 和 `postgresql@16-main.service` 均 disabled；`/etc/postgresql/16/main/start.conf` 为 `manual` |
| 日志轮转错峰 | `/etc/systemd/system/logrotate.timer.d/boot-stagger.conf`，激活后 5 分钟、此后每 1 天；`Persistent=false` |
| dpkg 数据库备份错峰 | `/etc/systemd/system/dpkg-db-backup.timer.d/boot-stagger.conf`，激活后 7 分钟、此后每 1 天；`Persistent=false` |
| ext4 旧快照清理错峰 | 原 `e2scrub_reap.service` disabled；`e2scrub-reap-delayed.timer` enabled |
| Docker 网络 | `/etc/systemd/system/docker.service.d/http-proxy.conf` 存在；地址不公开 |
| Tailscale 网络 | `/etc/systemd/system/tailscaled.service.d/noproxy.conf` 存在；重新登录授权 |
| 额外日志维护 | `aurabot-seven-day-maintenance.timer` 及配套 service、检查服务；保留当前规则后复核 |
| 授权运行时 | `aksusbd.service`、`hasplmd.service`；授权资料单独迁移 |

按需启动 PostgreSQL：`sudo systemctl start postgresql@16-main.service`。不要把数据目录直接覆盖到不同大版本数据库；业务库迁移需要另做一致性备份和恢复验证。

历史启动优化包位于 `~/boot-tuning-20261026/`，包含原配置备份、安装/回退脚本和验证记录；应私下保全，不将其中内部网络参数上传到公开仓库。

### 调度与 shell

- crontab 当前有 **6** 个活动条目，包含 OA 备份、工作日 Git 拉取、日报等；公开只保留 [时间和脚本文件名](inventory/cron-summary.tsv)。完整命令、业务目标、环境变量和凭据在本机私下清点。
- `~/.zshrc` 集成 Oh My Zsh、Powerlevel10k、autosuggestions、syntax-highlighting、nvm、gvm、mamba/conda。
- 自定义函数有 `getproxy`、`setproxy`、`unsetproxy`、`proxystat`、`clearVSCache`；迁移时保留功能，重新获取新网络的代理地址。
- 保留 `~/.p10k.zsh`、`~/.oh-my-zsh/` 的自定义部分、`~/.vimrc*`、`~/.spf13-vim-3/`，以及 `~/clash_ctl.sh`、`~/claude_update.sh` 等本地脚本；完整 shell 文件先做秘密审查再决定是否公开。
- Engram 使用 `~/engram`；相关 agent 配置在私人目录中。记忆仓库与工具配置按其同步流程迁移，不将整份用户配置复制到本公开仓库。

## 6. 迁移前私有备份清单

以下只记录位置，不在公开仓库保存内容。

| 数据 | 位置 / 注意 |
| --- | --- |
| 驱动、固件、桌面修补 | 前述 `/usr/src/`、AIC 固件/源码、Mutter 修复目录、启动优化目录 |
| 输入法数据 | Fcitx5/Rime、IBus、AI 拼音、微信输入法的配置及词库；`~/input-method-backups/` |
| SSH / Git / 网络 | `~/.ssh/`、`~/.git-credentials`、网络连接、VPN、代理、远程桌面；凭据加密备份 |
| AI 工具 | `~/.codex/`、`~/.claude/`、`~/.cc-switch/`、`~/.config/com.ccswitch.desktop/`、`~/.hermes/` 中所需配置；令牌恢复后核对权限 |
| 工作文件与记忆 | `~/Coding/` 内各项目未提交内容、`~/engram/`、业务文件及私有项目配置 |
| 远程桌面 | `~/.config/remmina/`、`~/.local/share/remmina/`；凭据可能在系统 keyring |
| 剪贴板、笔记和云盘 | `~/.local/share/com.tiez.app/`、Obsidian vault 实际目录、`~/.config/obsidian/`、BaiduPCS-Go 配置；剪贴板历史是私人数据 |
| CLI 业务数据 | exam-cli、tieba-cli、Hermes 等的实际数据库/输出位置逐项核实；保留项目 `output/` 归档 |
| 容器和数据库 | Compose/创建参数、私有镜像、Docker volumes、数据库导出；只保存镜像 ID 无法重建服务 |
| 浏览器、认证、授权 | Firefox 配置、证书、keyring、Sentinel 授权等；重新认证优先于盲目复制令牌 |
| Python 和项目环境 | 各环境的 Conda 历史/显式包清单、pip 版本清单、项目锁文件；本清单不携带私有源认证 |

本次没有归档这些私人数据，也没有验证它们已具备异机恢复能力。表中有些位置仅用于提示必须进一步清点，例如 Obsidian vault、exam 数据库和业务输出的实际位置。

## 7. 推荐恢复顺序与验收

1. 安装目标系统，确认硬件、分区、网络和 Secure Boot 状态；先准备无线驱动的离线恢复资料。
2. 安装匹配内核 headers、DKMS、所需无线驱动与固件，验证连接及重启后的模块加载。
3. 恢复基础桌面、字体和兼容扩展；检查官方 Mutter 是否仍需要补丁。
4. 恢复 Fcitx5/Rime 及私人词库，重新部署；注销重登验证自动启动、微软双拼、7 候选和 Kimpanel。
5. 恢复 nvm/pnpm、mamba 环境、gvm、rustup、Java/.NET/Perl，再按各项目锁文件恢复依赖。
6. 安装常用桌面软件和 CLI，核对 PATH、符号链接目标、VS Code 扩展及文件关联。
7. 恢复私有网络配置、挂载、容器卷和数据库。最后启用定时任务、隧道和后台服务，避免恢复过程中提前运行业务任务。
8. 实测冷启动、登录启动、睡眠唤醒、网络切换、输入、截图/剪贴板、远程挂载，以及关键项目启动和业务备份。

## 8. 附表索引和完整性边界

所有附表均为本次快照，只含允许公开的字段。

| 附表 | 内容 |
| --- | --- |
| [apt-installed.tsv](inventory/apt-installed.tsv) | 2559 个已安装 Debian 包及版本，带 manual/auto 标记 |
| [apt-manual.tsv](inventory/apt-manual.tsv) | 496 个已安装且被 APT 标记为手动的包；包含系统预装和构建依赖，不等于用户亲自安装清单 |
| [apt-held.tsv](inventory/apt-held.tsv) / [apt-source-files.tsv](inventory/apt-source-files.tsv) | hold 与源文件名 |
| [snap.tsv](inventory/snap.tsv) / [flatpak.tsv](inventory/flatpak.tsv) | Snap 快照、Flatpak 探测结果 |
| [desktop-apps.tsv](inventory/desktop-apps.tsv) | 137 个桌面启动项名称；不含完整 Exec 参数，少数无法解析的入口已标注 |
| [local-apps-and-commands.tsv](inventory/local-apps-and-commands.tsv) | `/opt`、`/usr/local/bin`、`~/.local/bin`、`~/Application` 的直接条目 |
| [drivers.tsv](inventory/drivers.tsv) | 两套手动 DKMS 驱动 |
| [gnome-extensions.tsv](inventory/gnome-extensions.tsv) / [user-fonts.tsv](inventory/user-fonts.tsv) | GNOME 扩展和用户字体 |
| [login-autostart.tsv](inventory/login-autostart.tsv) / [login-container-timers.tsv](inventory/login-container-timers.tsv) | 登录应用、容器启动实例 |
| [systemd-enabled-masked.tsv](inventory/systemd-enabled-masked.tsv) | 系统和用户 enabled/masked 单元；不覆盖模板实例依赖链接的全部情况 |
| [cron-summary.tsv](inventory/cron-summary.tsv) / [containers.tsv](inventory/containers.tsv) | 脱敏调度摘要和容器状态；非可恢复服务定义 |
| [python-environments.tsv](inventory/python-environments.tsv) / [python-packages.tsv](inventory/python-packages.tsv) | 4 个实际 Python 环境及包版本 |
| [node-global.tsv](inventory/node-global.tsv) / [uv-tools.tsv](inventory/uv-tools.tsv) | Node/pnpm、uv 全局工具 |
| [go-tools.tsv](inventory/go-tools.tsv) / [rust-tools.tsv](inventory/rust-tools.tsv) | Go、Rust 全局工具 |
| [perl-local-modules.tsv](inventory/perl-local-modules.tsv) | 本地 CPAN 模块回执 |
| [vscode-extensions.tsv](inventory/vscode-extensions.tsv) | VS Code 扩展 |

不能仅凭 APT 状态还原“谁在何时亲自安装了什么”。本次覆盖软件包、典型用户级安装位置及已识别方案；未全盘扫描所有可执行文件，也未枚举每个项目虚拟环境、浏览器插件、IDE 插件或下载目录的离线安装包。不要将“未列出”解释为“未安装”。

## 9. 更新约定

**不设置定时任务。** 后续在这台机器上安装/卸载软件、更改驱动/输入法/桌面、调整启动服务或运行时环境时，顺手更新相应章节、附表和核验日期；没有新核验时保留旧日期并标注待确认。

更新流程、常用采集命令和发布前检查见 [MAINTENANCE.md](MAINTENANCE.md)。公开文档、私人备份和实际迁移演练分别维护；更新清单不代表已经完成私人数据备份。
