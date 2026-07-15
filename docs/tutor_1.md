# 用 ADB + Root 权限开启 USB 网络共享

可以的！下面是完整方案。

## 环境确认

```
Android 14 手机（已 Root）  <--USB--> Ubuntu PC
```

---

## 方法一：ADB Shell 直接操控（推荐）

### 1. PC 端准备

```bash
# 安装 adb
sudo apt install adb

# 确认手机已连接（需先在手机开启 USB 调试）
adb devices
```

### 2. 进入 ADB Root Shell

```bash
# 切换到 root 权限
adb root

# 或者进入 shell 后再 su
adb shell
su
```

### 3. 启用 USB Tethering

```bash
# 写入系统设置
adb shell settings put global tether_supported 1
```

### 4. 检查是否成功

```bash
# 手机端查看网络接口
adb shell ip link show

# 应该能看到 rndis0 或 usb0 接口出现
```

---

## 方法二：Magisk + 脚本自动化

如果用 Magisk 获取 root，可以写一个开机自动执行的脚本：

```bash
# 在手机 /data/local/tmp/ 创建脚本
adb shell su -c "echo 'svc usb setFunctions rndis' > /data/local/tmp/tether.sh"
adb shell su -c "chmod +x /data/local/tmp/tether.sh"

# 执行
adb shell su -c "/data/local/tmp/tether.sh"
```

---

## Ubuntu PC 端配置

手机开启 tethering 后，Ubuntu 这边操作：

```bash
# 查看新出现的网络接口（通常是 usb0 或 enp0s??）
ip link show

# 手动获取 IP（如果没自动获取）
sudo dhclient usb0

# 或者用 nmcli
nmcli device connect usb0

# 验证网络
ping -I usb0 8.8.8.8
```

---

## Android 14 特别注意事项

Android 14 加强了权限管控，可能遇到的问题：

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `adb root` 失败 | 非 userdebug 版本 | 需要 Magisk，用 `adb shell` + `su` |
| `rndis` 接口不出现 | 内核未加载模块 | 执行 `modprobe rndis_host`（Ubuntu端）|
| 接口出现但无 IP | DHCP 未触发 | 手动 `sudo dhclient usb0` |
| SELinux 阻止 | Android 14 SELinux 严格 | `adb shell su -c "setenforce 0"`（临时）|

---

## 完整一键流程

```bash
# === PC 端终端执行 ===

# 1. 确认设备
adb devices

# 2. 开启 tethering
adb shell su -c "svc usb setFunctions rndis"

# 3. 等待接口出现
sleep 3

# 4. 获取 IP
IFACE=$(ip link show | grep -oP 'usb\d+|enp\d+s\d+u\d+' | head -1)
sudo dhclient $IFACE

# 5. 验证
echo "使用接口: $IFACE"
ip addr show $IFACE
```

---

## 如果完全没有 Root

没有 root 也有备用方案：

```bash
# ADB 直接转发（不需要 root，但只能转发特定端口，不是完整网络共享）
adb forward tcp:8080 tcp:8080

# 或者用 gnirehtet 工具（反向网络共享，让手机用电脑的网）
# https://github.com/Genymobile/gnirehtet
```

---

**总结**：有 Root 的情况下，`adb shell su -c "svc usb setFunctions rndis"` 这一条命令基本就能搞定，Ubuntu 端再 `dhclient usb0` 即可上网。


好的修好了多谢，我的英语水平不太好所以就讲回中文了。那么我是否可以在上述获得了手机root权限的情况下，使用adb shell直接操控clash for android打开、关闭、切换节点等操作？


# 用 ADB Shell + Root 操控 Clash for Android

可以，但方式取决于你想操控的具体功能，下面分情况说明。

---

## 方案一：通过 Clash 的 RESTful API（最推荐）

Clash for Android 内置了一个 HTTP API（继承自 mihomo/clash 核心），这是最优雅的控制方式。

### 1. 先确认 API 已开启

在 Clash for Android 设置里找到：
```
外部控制 (External Controller)
地址：127.0.0.1:9090
密钥：7355608*
```

### 2. 用 ADB 做端口转发

```bash
# 把手机的 9090 端口转发到电脑本地
adb forward tcp:9090 tcp:9090
```

### 3. 在 Ubuntu 上直接用 curl 操控

```bash
# 设置你的 API 密钥（没设置就留空）
SECRET="your_secret"

# ===== 查看所有代理节点 =====
curl http://127.0.0.1:9090/proxies \
  -H "Authorization: Bearer $SECRET" | python3 -m json.tool

# ===== 查看当前选中的节点 =====
curl http://127.0.0.1:9090/proxies/GLOBAL \
  -H "Authorization: Bearer $SECRET"

# ===== 切换节点（以代理组名为例）=====
# 先查出你的代理组名，比如叫 "Proxy"
curl -X PUT http://127.0.0.1:9090/proxies/Proxy \
  -H "Authorization: Bearer $SECRET" \
  -H "Content-Type: application/json" \
  -d '{"name": "节点名称"}'

# ===== 测试节点延迟 =====
curl "http://127.0.0.1:9090/proxies/节点名称/delay?timeout=2000&url=http://www.gstatic.com/generate_204" \
  -H "Authorization: Bearer $SECRET"

# ===== 查看实时流量日志 =====
curl http://127.0.0.1:9090/logs \
  -H "Authorization: Bearer $SECRET"
```

---

## 方案二：ADB Shell 直接控制 App 启停

```bash
# ===== 启动 Clash for Android =====
adb shell am start -n com.github.kr328.clash/.MainActivity

# ===== 开启 VPN（模拟点击"启动"）=====
# 需要先知道包名，常见有两种：
# 免费版：com.github.kr328.clash
# Premium：com.github.kr328.clash.premium
# meta: com.github.metacubex.clash.meta

# ===== 停止 Clash 服务 =====
adb shell am force-stop com.github.kr328.clash

# ===== 用 root 直接杀进程 =====
adb shell su -c "pkill -f clash"

# ===== 重启 Clash =====
adb shell su -c "pkill -f clash" && \
adb shell am start -n com.github.kr328.clash/.MainActivity
```

---

## 方案三：用 ADB 模拟 UI 点击（无 API 时的备用）

```bash
# 查看当前界面的控件树（找到按钮坐标）
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml

# 模拟点击屏幕坐标（x y）
adb shell input tap 540 1200

# 模拟下拉快捷磁贴（如果有 Clash 磁贴）
adb shell cmd statusbar expand-notifications
```

---

## 写个一键切换节点脚本

```bash
#!/bin/bash
# switch_node.sh - 一键切换 Clash 节点

SECRET="7355608*"          # 你的 API 密钥
PROXY_GROUP="Proxy" # 你的代理组名称
NODE_NAME="$1"     # 从命令行参数传入节点名

# 确保端口转发存在
adb forward tcp:9090 tcp:9090

if [ -z "$NODE_NAME" ]; then
  echo "=== 当前可用节点 ==="
  curl -s http://127.0.0.1:9090/proxies/$PROXY_GROUP \
    -H "Authorization: Bearer $SECRET" \
    | python3 -c "
import json,sys
data=json.load(sys.stdin)
print('当前节点:', data['now'])
print('所有节点:')
[print(' -', n) for n in data['all']]
"
else
  echo "正在切换到节点: $NODE_NAME"
  curl -s -X PUT http://127.0.0.1:9090/proxies/$PROXY_GROUP \
    -H "Authorization: Bearer $SECRET" \
    -H "Content-Type: application/json" \
    -d "{\"name\": \"$NODE_NAME\"}"
  echo "切换完成"
fi
```

```bash
# 使用方式
chmod +x switch_node.sh
./switch_node.sh              # 列出所有节点
./switch_node.sh "香港-01"   # 切换到指定节点
```

---

## 各方案对比

| 方案 | 稳定性 | 是否需要 Root | 功能完整度 |
|------|--------|--------------|-----------|
| RESTful API | ⭐⭐⭐⭐⭐ | 不需要 | 最完整 |
| AM 命令启停 | ⭐⭐⭐⭐ | 不需要 | 只能启停 |
| UI 模拟点击 | ⭐⭐ | 不需要 | 有限 |
| pkill 杀进程 | ⭐⭐⭐ | 需要 | 只能关闭 |

---

**结论：最推荐 RESTful API 方案**，不需要 Root、功能最全、最稳定，只需要 `adb forward` 做端口转发即可在 Ubuntu 上完全操控 Clash。

i was trying upgrade pinyin input method was faild, now im cannot typing chinese, how to reinstall ibus-libpinyin input method? i got those two pkg:
ibus-libpinyin-1.16.5.tar.gz
libpinyin-2.10.3.tar.gz

# 从源码重装 ibus-libpinyin 输入法

你有两个包，安装顺序是：**先装 libpinyin（底层库）→ 再装 ibus-libpinyin（上层输入法）**

---

## 第一步：安装编译依赖

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  automake \
  autoconf \
  libtool \
  pkg-config \
  libibus-1.0-dev \
  libsqlite3-dev \
  libboost-dev \
  gnome-common \
  intltool \
  gettext \
  python3-dev
```

---

## 第二步：先编译安装 libpinyin

```bash
# 解压
tar -xzf libpinyin-2.10.3.tar.gz
cd libpinyin-2.10.3

# 配置
./configure --prefix=/usr

# 编译（-j$(nproc) 使用全部CPU核心加速）
make -j$(nproc)

# 安装
sudo make install

# 更新动态链接库缓存（重要！）
sudo ldconfig

cd ..
```

---

## 第三步：再编译安装 ibus-libpinyin

```bash
# 解压
tar -xzf ibus-libpinyin-1.16.5.tar.gz
cd ibus-libpinyin-1.16.5

# 配置
./configure --prefix=/usr

# 编译
make -j$(nproc)

# 安装
sudo make install

cd ..
```

---

## 第四步：重启 IBus 并重新注册

```bash
# 杀掉旧的 ibus 进程
ibus exit 2>/dev/null; pkill ibus-daemon

# 重新加载组件列表
ibus write-cache

# 重启 ibus
ibus-daemon -drx
```

---

## 第五步：添加输入法

```bash
# 命令行方式添加（Ubuntu/GNOME）
gsettings set org.gnome.desktop.input-sources sources \
  "[('xkb', 'us'), ('ibus', 'libpinyin')]"
```

或者图形界面操作：

```
IBus 首选项 → 输入法 → 添加 → 中文 → Intelligent Pinyin
```

---

## 验证是否成功

```bash
# 检查 ibus 能否识别到 libpinyin
ibus list-engine | grep pinyin

# 应该看到类似输出：
#  libpinyin - Intelligent Pinyin
#  libbopomofo - Intelligent Bopomofo
```

---

## 常见报错处理

| 报错信息 | 原因 | 解决 |
|----------|------|------|
| `libpinyin.so not found` | ldconfig 没跑 | `sudo ldconfig` |
| `configure: error: ibus not found` | 缺少 libibus-dev | `sudo apt install libibus-1.0-dev` |
| `make: *** No rule to make target` | autoconf 没生成 | 先跑 `./autogen.sh` 再 `./configure` |
| 安装后看不到输入法 | 缓存未刷新 | `ibus write-cache && ibus restart` |

---

## 如果 `./configure` 不存在

部分源码包需要先生成构建脚本：

```bash
# 在 configure 不存在时执行
./autogen.sh --prefix=/usr
# 然后再正常 make && sudo make install
```

---

装完重新登录一次，`Super + Space` 或 `Ctrl + Space` 切换输入法就能用了。



```
我是否可以在上述获得了手机root权限的情况下，使用adb shell直接操控clash for android打开、关闭、切换节点等操作？


# 用 ADB Shell + Root 操控 Clash for Android

可以，但方式取决于你想操控的具体功能，下面分情况说明。

---

## 方案一：通过 Clash 的 RESTful API（最推荐）

Clash for Android 内置了一个 HTTP API（继承自 mihomo/clash 核心），这是最优雅的控制方式。

### 1. 先确认 API 已开启

在 Clash for Android 设置里找到：
```
外部控制 (External Controller)
地址：127.0.0.1:9090
密钥：7355608*
```

### 2. 用 ADB 做端口转发

```bash
# 把手机的 9090 端口转发到电脑本地
adb forward tcp:9090 tcp:9090
```

### 3. 在 Ubuntu 上直接用 curl 操控

```bash
# 设置你的 API 密钥（没设置就留空）
SECRET="your_secret"

# ===== 查看所有代理节点 =====
curl http://127.0.0.1:9090/proxies \
  -H "Authorization: Bearer $SECRET" | python3 -m json.tool

# ===== 查看当前选中的节点 =====
curl http://127.0.0.1:9090/proxies/GLOBAL \
  -H "Authorization: Bearer $SECRET"

# ===== 切换节点（以代理组名为例）=====
# 先查出你的代理组名，比如叫 "Proxy"
curl -X PUT http://127.0.0.1:9090/proxies/Proxy \
  -H "Authorization: Bearer $SECRET" \
  -H "Content-Type: application/json" \
  -d '{"name": "节点名称"}'

# ===== 测试节点延迟 =====
curl "http://127.0.0.1:9090/proxies/节点名称/delay?timeout=2000&url=http://www.gstatic.com/generate_204" \
  -H "Authorization: Bearer $SECRET"

# ===== 查看实时流量日志 =====
curl http://127.0.0.1:9090/logs \
  -H "Authorization: Bearer $SECRET"
```
```
这个restful方案如何做到单纯的开启关闭？


我打算使用公司电脑ssh到家用的windows 11内的linux subsystem进行远程开发，是否可行？如果可行，具体该如何执行？在家里没有公网ip的情况下我需要做哪些准备？

## 完全可行！方案概述

核心挑战有两个：**①让 WSL 开启 SSH 服务** + **②没有公网 IP 时如何从外部连入**。

---

## 第一步：配置 WSL 的 SSH 服务

在家用电脑的 WSL 内执行：

```bash
# 安装 OpenSSH Server
sudo apt update && sudo apt install openssh-server -y

# 修改配置，允许密码登录（或只用密钥，更安全）
sudo nano /etc/ssh/sshd_config
```

确认以下几行存在且未被注释：

```
Port 2222          # 建议改成非默认端口，避免和 Windows 自带 SSH 冲突
PasswordAuthentication yes   # 初期调试用，稳定后建议改为 no
```

```bash
# 启动 SSH 服务
sudo service ssh start

# 设置 WSL 启动时自动开启 SSH（加入 ~/.bashrc 或用任务计划）
sudo service ssh start 2>/dev/null
```

---

## 第二步：解决「没有公网 IP」的问题

这是关键难点，推荐以下方案，**难度从低到高**：

### ✅ 方案 A：Tailscale（最推荐，免费且最简单）

Tailscale 是一个基于 WireGuard 的 P2P VPN，家用机和公司机加入同一虚拟网络，**无需公网 IP、无需路由器设置**。

**家用 Windows 11：**
1. 下载安装 [Tailscale for Windows](https://tailscale.com/download)
2. 用 Google/GitHub 账号登录
3. 记下分配的虚拟 IP（如 `100.x.x.x`）

**WSL 内也需要加入（可选，直接用 Windows 的 Tailscale IP 即可）：**
```bash
# WSL 内可以直接通过 Windows 宿主机的 Tailscale IP 访问
# 因为 WSL2 和 Windows 共享网络（需要配置端口转发，见下）
```

**公司电脑：**
1. 同样安装 Tailscale 并登录同一账号
2. 直接 SSH 连接家用机的 Tailscale IP

```bash
ssh -p 2222 your_username@100.x.x.x
```

---

### 方案 B：Cloudflare Tunnel（免费，适合穿透）

```bash
# 在家用 WSL 内安装 cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# 建立隧道（需要 Cloudflare 账号）
cloudflared tunnel login
cloudflared tunnel create home-dev
cloudflared tunnel route dns home-dev ssh.yourdomain.com
cloudflared tunnel run home-dev
```

公司端连接：
```bash
# 公司电脑也需装 cloudflared
ssh -o ProxyCommand="cloudflared access ssh --hostname ssh.yourdomain.com" user@ssh.yourdomain.com
```

---

### 方案 C：有 VPS 时用 FRP 反向代理

如果你有一台云服务器（有公网 IP），可以用 FRP：

```ini
# 家用 WSL 内 frpc.ini
[common]
server_addr = your_vps_ip
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 2222
remote_port = 6022   # 公司电脑连 VPS 的这个端口
```

```bash
# 公司电脑
ssh -p 6022 user@your_vps_ip
```

---

## 第三步：WSL2 端口转发（重要！）

WSL2 有独立的虚拟 IP，Windows 需要把端口转发给它：

在 **Windows PowerShell（管理员）** 中执行：

```powershell
# 查看 WSL 的 IP
wsl hostname -I

# 设置端口转发（把 Windows 的 2222 转发到 WSL 的 2222）
netsh interface portproxy add v4tov4 `
  listenport=2222 listenaddress=0.0.0.0 `
  connectport=2222 connectaddress=$(wsl hostname -I).Trim()

# Windows 防火墙放行
New-NetFirewallRule -DisplayName "WSL SSH" -Direction Inbound -Protocol TCP -LocalPort 2222 -Action Allow
```

由于 WSL2 每次重启 IP 会变，建议写成开机脚本：

```powershell
# 保存为 wsl-port-forward.ps1，加入任务计划程序开机运行
$wslIp = (wsl hostname -I).Trim()
netsh interface portproxy delete v4tov4 listenport=2222 listenaddress=0.0.0.0
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=2222 connectaddress=$wslIp
```

---

## 第四步：让 WSL 开机自启 SSH

在 Windows 任务计划程序中新建任务，触发器选「用户登录时」，操作：

```
程序：wsl.exe
参数：-u root service ssh start
```

或者更优雅地用 `/etc/wsl.conf`：

```ini
# /etc/wsl.conf
[boot]
command = service ssh start
```

---

## 推荐方案对比

| 方案 | 难度 | 费用 | 稳定性 | 适合场景 |
|------|------|------|--------|----------|
| **Tailscale** | ⭐ 最低 | 免费 | ⭐⭐⭐⭐⭐ | 强烈推荐首选 |
| Cloudflare Tunnel | ⭐⭐ | 免费 | ⭐⭐⭐⭐ | 需要自定义域名 |
| FRP + VPS | ⭐⭐⭐ | VPS费用 | ⭐⭐⭐⭐⭐ | 已有VPS时 |

**最简路径：Tailscale + WSL SSH + VS Code Remote SSH**，配置完成后在 VS Code 里装 Remote - SSH 插件，体验和本地开发几乎无差异。


好了不用算了，那台笔记本开不了机了，在我清灰换成相变片后回光返照跑了一段时间，结果在我打开steam还是打算更新内部的软件什么的时候，听到啪的一声，电脑就黑屏了，然后就是尝试各种按开关都没反应....这玩意儿收废品都只能卖个100多，值钱点的也就是里头的32g内存，看这情况是不是直接大电流击穿了？

哦豁，听到"啪"一声然后彻底没反应，这个症状很典型……

---

## 基本可以判断：电容或MOS管爆了

"啪"的一声 + 立即黑屏 + 完全无法开机，这个组合八九不离十是：

**电源管理模块（VRM）上的电容或MOSFET击穿**

清灰换相变片之后"回光返照"这个细节很关键——说明主板散热改善后满血运行，但某个已经老化的元器件撑不住瞬间的电流冲击（Steam更新触发磁盘+GPU+CPU同时高负载）直接挂了。

---

## 能抢救的东西

| 零件 | 状态 | 建议 |
|------|------|------|
| **32GB内存** | 大概率完好，内存不怕主板爆 | **优先拆出来卖，¥900** |
| **硬盘（SSD/HDD）** | 完好，数据应该还在 | 拆出来接USB盒子读数据 |
| 屏幕 | 看品牌，小众品牌值钱不大 | 可以挂闲鱼，¥100~300 |
| CPU/GPU | 焊死在主板上，主板坏了基本废 | 放弃 |
| 电源适配器 | 完好 | 闲鱼挂¥30~80 |

---

## 建议操作顺序

1. **先拆32G内存** → 接其他机器测试能不能认到 → 正常就挂闲鱼
2. **拆硬盘** → 买个M.2/SATA硬盘盒（¥20~50）→ 接出来备份数据
3. 剩下的整机当零件机挂闲鱼，**标注"主板故障/零件拆用"**，还能卖¥150~250

---

## 一句话总结

> 这台机器作为整机已经没有抢救价值了，但内存+硬盘拆出来还能回血¥400~600，别直接当废品卖。

顺便说，你现在的直播方案也要重新考虑了——推流机没了，是打算直接上Mac Mini，还是另有打算？


有没有多平台推流的obs推流方案？具体可以怎么做？对网络要求高吗？比如我想推流到抖音、B站、Youtube和kick，具体可以怎么配置？

可以。你这个组合最现实的方案有两种：

| 方案 | OBS 实际上传 | 适合情况 |
|---|---:|---|
| 本机多路直推 | 四个平台码率相加 | 你有稳定高上传、想省订阅费 |
| 云端中转分发 | 只上传一路 | 想稳定、省本机带宽，接受订阅费 |

我更推荐你先按“云端中转”理解，再看网络决定是否改本机直推。

## 方案一：OBS → 云端中转 → 抖音/B站/YouTube/Kick

例如 Restream 这类服务。OBS 只推给 Restream 一路，Restream 再分发给各平台。它支持自定义 RTMP 目标，所以只要抖音、B 站后台给了 RTMP 地址和推流码，也能加进去；但 3 个以上平台及自定义 RTMP 需要付费套餐。 [Restream 自定义 RTMP 说明](https://support.restream.io/en/articles/369436-stream-to-a-custom-rtmp-channel) [套餐限制](https://support.restream.io/en/articles/9127747-can-i-use-restream-for-free-yes)

配置流程：

1. 分别在四个平台开通直播权限，拿到各自的推流地址和推流码。  
   - YouTube：YouTube Studio → 创建 → 进行直播 → Stream，复制 Server URL 和 Stream Key。 [官方步骤](https://support.google.com/youtube/answer/2907883?hl=en)  
   - Kick：Creator Dashboard → Channel → Stream URL and Key。 [官方步骤](https://help.kick.com/en/articles/7066931-how-to-stream-on-kick-com)  
   - B站、抖音：先在各自直播后台或直播伴侣中开播；如果账户开放“OBS/第三方推流”，复制对应 RTMP 地址和推流码。不同账号权限和界面会有差异。  
2. 在 Restream 添加 YouTube、Kick，并以“Custom RTMP”添加 B站、抖音。
3. Restream 会给你一组输入地址和推流码。
4. OBS → 设置 → 直播：填入 Restream 的地址和码，一次开播即可。

优点：你的电脑只编码、上传一次。  
缺点：多一个服务商和月费；从国内推向境外中转节点的线路必须提前测试稳定性。

## 方案二：OBS 本机四路直推

装 **Aitum Multistream** 或 **Multiple RTMP Outputs** 插件，在 OBS 里为四个平台各加一个 RTMP 输出。前者支持 Windows、macOS、Linux；后者可以复用主输出的编码器，减少 CPU/GPU 编码压力。 [Aitum Multistream](https://obsproject.com/forum/resources/aitum-multistream.1991/) [Multiple RTMP Outputs](https://obsproject.com/forum/resources/multiple-rtmp-outputs-plugin.964/)

每一路填：

```text
Server：平台给你的 RTMP/RTMPS 地址
Stream Key：平台给你的推流码
```

四路都用同一套画面，建议先统一为：

```text
分辨率：1920 × 1080
帧率：30 FPS
编码：H.264（NVENC / AMD AMF / Intel QSV 优先）
码率控制：CBR
视频码率：6000 kbps
关键帧间隔：2 秒
音频：AAC，48kHz，160 kbps
```

这套参数对四个平台相对保守、兼容性高。Kick 目前要求 H.264/x264、CBR、1080p 以内、60fps 以内，最高 8000kbps；YouTube 也建议 CBR、2 秒关键帧。 [Kick 官方要求](https://help.kick.com/en/articles/14994318-obs-or-streamlabs-not-connecting-to-kick) [YouTube 编码建议](https://support.google.com/youtube/answer/2853702?hl=en-EN)

## 网络到底要多高？

本机四路直推时，`6000kbps × 4` 就是约 **24Mbps 视频上传**；再算音频、协议开销和波动，实际至少要有 **40Mbps 稳定上行**，我会建议 **50Mbps 以上的真实可持续上行** 才舒服。

如果改成 1080p60、每路 8000kbps，那就是约 32Mbps 视频上传，建议实际稳定上行 60Mbps 以上。

云端中转则只需推一路 6000kbps；准备 **12–15Mbps 稳定上行**即可，但网络质量比测速峰值更重要。YouTube 也明确建议总上传码率保留至少约 20% 余量。 [YouTube 网络建议](https://support.google.com/youtube/answer/2853856?hl=en)

最后几个坑：

- 抖音更偏竖屏，B站/YouTube/Kick 的游戏或桌面内容通常更适合 16:9 横屏；第一版建议统一横屏，别一开始就搞横竖双画面。
- 四个平台的标题、分区、封面、违禁内容规则是独立的；自定义 RTMP 通常不能由中转平台自动改标题。
- 推流码等同直播间密码，截图、录屏、发日志时不要露出；泄露就立刻重置。
- 先分别开“仅自己可见/测试直播”，再一次性四推；OBS 右下角重点看 `Dropped Frames`，只要持续涨就先降码率或改云端中转。