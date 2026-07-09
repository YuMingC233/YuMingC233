# Incam Pro 远程代码执行 & 集成测试指南

## 1. 远程执行原理

```
本地 Linux (你的机器)                    Incam Windows Server
┌──────────────────┐                   ┌──────────────────────┐
│  perl 脚本       │  ── TCP Socket ──>│  Genesis 服务进程     │
│  use Genesis;    │  默认端口: 56773     │  处理 COM/PAUSE/AUX  │
│  $f = new Genesis($host)             │  返回 COMANS         │
│  $f->COM("...")  │  <─────────────── │                      │
└──────────────────┘                   └──────────────────────┘
```

`Genesis.pm` 模块封装了 TCP 通信，`new Genesis($host)` 连接远程 Incam 服务器。
现有 `run_pm.pl` 已使用此模式：`$f = new Genesis($host)`。

## 2. 需要你在 Incam 端做的事情

### 步骤1: 确认 Genesis 服务器端口

在 Incam Windows 服务器上，打开 Genesis/Incam 的脚本编辑器，运行：

```perl
use Genesis;
my $f = new Genesis();
$f->PAUSE("Genesis server is running on this machine");
```

如果能看到 PAUSE 弹窗，说明本地 Genesis 服务正常。

然后检查 Genesis 服务监听的端口：
- 方法1: 在 Windows 命令行运行 `netstat -an | findstr LISTEN` 查看
- 方法2: 查看 Incam 安装目录下的配置文件（通常在 `$INCAM_PRODUCT/etc/` 下）
- 方法3: 查看 `Genesis.pm` 源码，搜索 `IO::Socket::INET` 或 `sock` 关键字

**请找到端口号后告诉我。**

### 步骤2: 确认 Genesis.pm 可用

将 Incam 服务器上的 `Genesis.pm` 复制到本地可访问的路径：

```bash
# Incam 服务器上的路径通常是:
# $INCAM_PRODUCT/app_data/perl/Genesis.pm
# 或
# $GENESIS_DIR/e$GENESIS_VER/all/perl/Genesis.pm

# 复制到本地:
mkdir -p /mnt/incamhyx/lib/
cp <Genesis.pm路径> /mnt/incamhyx/lib/
```

### 步骤3: 确认网络可达

确保你的 Linux 机器能访问 Incam 服务器的 TCP 端口：
- Incam 服务器 IP: 通常是 `192.9.100.xxx` (参考 README.md 中的 Host 字段)
<!-- 192.9.100.87 -->
- 防火墙需放行 Genesis 服务端口

### 步骤4: 准备测试 Job

在 Incam 上准备一个简单的测试 Job：
- Job 名称: 如 `test_unit_order`
- Step 名称: 如 `test_unit_order`
- 包含一个 m 层，上面有规则网格排列的文本（3行6列）
- 已完成 SR 套装（NUM_SR > 0）

## 3. 一旦你提供了以下信息，我就能编写集成测试

| 需要的信息 | 示例 |
|-----------|------|
| Incam 服务器 IP | `192.9.100.87` |
| Genesis 服务端口 | `56773`|
| Genesis.pm 本地路径 | `/mnt/incamhyx/lib/Genesis.pm` |
| Genesis.pl 本地路径 | `/mnt/incamhyx/scripts/local-coding/lib/eall/all/perl/Genesis.pl` |
| 测试 Job 名称 | `ts222757a00.fnl.hyx` |
| 测试 Step 名称 | `set` |
| 测试 m 层名称 | `m1` |

## 4. 集成测试的架构（信息就绪后我会写）

```perl
#!/usr/bin/env perl
# t/unit_order_v2_integration.t
use Test::More;
use lib '/mnt/incamhyx/lib';
use Genesis;

my $host = '192.9.100.222';   # 你提供的 IP
my $f = new Genesis($host);

# 设置环境变量
$ENV{JOB}  = 'test_unit_order';
$ENV{STEP} = 'test_unit_order';

# 测试1: 连接成功
ok(defined $f, 'Genesis connection established');

# 测试2: 获取 NUM_SR
$f->COM("info,out_file=/tmp/test_info.$$,write_mode=replace,args= -t step -e $ENV{JOB}/$ENV{STEP} -d NUM_SR");
# ... 解析并验证

# 测试3: flatten_layer
$f->COM("flatten_layer,source_layer=m1,target_layer=m1-+-txt");
# ... 验证

# 测试4: 完整 auto_numbering 流程
# ... 使用 mock 的 mouse,p 坐标（不需要真实点击）
```

## 5. 当前已有的纯算法单测

不需要 Incam 连接，已通过 21 个测试：

```bash
perl /mnt/incamhyx/scripts/t/unit_order_v2.t
```

覆盖范围：
- `format_number` — 5种格式 + fallback
- `cluster_into_rows` — 正常网格、单行、单列、Y 抖动、Y 间距、不完整行
- `determine_order` — 4种模式完整编号顺序验证 + 中间行起始 + 1行/1列退化
- `parse_features` — 正常解析 + 跳过标题行
