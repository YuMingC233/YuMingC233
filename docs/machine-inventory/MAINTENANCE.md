# 清单维护方法

本目录记录 Ubuntu 工作机的可公开迁移清单。安装或调整本机环境时按需更新，不创建定时任务，不自动运行全量安装或卸载。

## 更新步骤

1. 确认操作对象确实是本清单描述的工作机，不把远程主机、容器或新电脑的状态混入同一快照。
2. 核对旧记录与现场事实，更新受影响章节和相应 TSV。整批附表没有重新采集时，注明受影响文件的单独核验日期，不能假称全部已刷新。
3. 新增手动软件时记录名称、来源、已核实版本、安装位置、配置/数据路径、恢复方式。手动驱动额外记录硬件 ID、内核、DKMS、固件及源码。
4. 只采集包元数据，不导出私人配置、shell 环境、容器环境、浏览器数据、业务文件或凭据。网络和私有源只记用途及配置位置。
5. 变更属于文档清点时，检查表结构、链接、事实对应关系、脱敏结果和 Git diff。这里没有应用功能代码，不为文档新增单元测试。
6. 提交时只包含这次清单及相关文档入口，不混入已有工作区修改。提交信息不含 AI 署名。

## 常用只读采集命令

```bash
cat /etc/os-release
uname -r
gnome-shell --version
loginctl list-sessions
lsusb
lspci -nnk
dkms status
apt-mark showmanual
apt-mark showhold
dpkg-query -W -f='${binary:Package}\t${Version}\t${db:Status-Status}\n'
snap list
gnome-extensions list
systemctl list-unit-files --state=enabled,masked --no-pager
systemctl --user list-unit-files --state=enabled,masked --no-pager
systemctl --user list-timers --all --no-pager
code --list-extensions --show-versions
uv tool list
rustup show active-toolchain
cargo install --list
```

`dpkg-query` 需要过滤状态恰好为 `installed` 的行；通配查询可能返回已删除包或空版本，不能全部算作已安装。APT manual 标记也不能当作手动安装行为的证明。

Python 各环境分别运行对应解释器的 `-m pip list --format=json --disable-pip-version-check`，并读取该环境 `conda-meta/*.json` 的 name/version 字段。不要把 Conda URL、pip index 配置、direct URL、环境变量或完整环境文件直接上传。

Node 全局包读取 nvm 各版本及 pnpm global 的 package.json，只导出包名/版本；pnpm 的声明范围与实际版本分别记录。CLI 可能有多种安装方式，应再检查实际入口。

容器如需公开摘要，仅选择 name/image/state；私有镜像仓库地址、完整镜像参数及业务标识需另行审查。`docker inspect`、`systemctl cat`、`crontab -l` 的原始输出默认保留本地，不能整份写入公开文档。

## 发布前检查

- 表格每行列数和表头一致，版本来自本次核验，链接目标存在。
- 搜索意外出现的 IP 地址、URL 用户名密码、私有主机名、访问令牌、API key、密码、私钥和用户数据；字段名出现在说明中不代表包含秘密。
- 再看一次 `git diff` / `git diff --cached`，确保没有包含已有的其他文档、备份、日志或私人文件。
- 确认本次是完整独立的文档更新，再提交并推送；不改写远端历史。

## 待补全与复核

- RTL8851BU 原安装来源、源码提交和是否有本地补丁。
- `/opt/kingsoft` 对应软件是否仍有效，微信输入法实际安装来源和版本。
- Obsidian vault、exam 数据库、业务归档及容器卷的实际私人备份位置。
- 各项目虚拟环境、浏览器和其他 IDE 插件依赖；本次未穷举。
- Mutter 修复的长期实际验收，目标系统是否已由官方包含修复。
- 新电脑迁移后实际验收，以及私人备份的恢复演练。
