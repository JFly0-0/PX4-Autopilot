# PX4 中文上手指南：在 Windows + WSL2 中运行仿真

这份指南面向第一次使用 PX4 的读者。目标是先在电脑上编译并运行一架模拟四旋翼，完成起飞和降落，再认识代码目录。无需飞控硬件或遥控器；本文的飞行操作仅用于仿真。

本文依据当前仓库的脚本和文档编写，默认使用 x86_64 电脑、WSL2 和 Ubuntu 22.04。命令已与仓库入口核对，但未在你的 Windows 机器上实际安装或验证。

## 1. 先认识四个组件

| 组件 | 用途 |
| --- | --- |
| PX4 | 飞行控制软件，负责姿态估计、控制、飞行模式等功能；这个仓库就是它的源码 |
| SITL | Software In The Loop，让 PX4 在电脑上运行，代替真实飞控板 |
| Gazebo | 模拟无人机、传感器和物理环境，并显示三维画面 |
| QGroundControl（QGC） | 地面站，用于查看飞行状态、发送起飞和降落等指令 |

运行时，Gazebo 给 PX4 提供模拟传感器数据，PX4 计算控制输出，Gazebo 据此更新飞机运动；QGC 通过 MAVLink 与 PX4 通信。

## 2. 准备 Windows 和 WSL2

图形仿真需要 WSLg。微软要求 Windows 10 build 19044 或更高版本，或 Windows 11，并建议安装适配的显卡驱动。详见 [WSL 图形应用说明](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps)。

**Windows PowerShell：**先检查已有环境。

```powershell
wsl --list --verbose
```

如果已经有 Ubuntu 22.04 且 `VERSION` 为 `2`，直接使用它。当前仓库也支持 Ubuntu 24.04；已有该版本可以继续使用，无需重装，后续命令中的发行版名称相应替换。

尚未安装时，在**管理员 PowerShell** 执行：

```powershell
wsl --install -d Ubuntu-22.04
```

按照提示重启 Windows，并创建 Ubuntu 用户名和密码。安装失败且提示虚拟化不可用时，检查 BIOS/UEFI 中的硬件虚拟化设置。

已有发行版但显示 WSL1 时，在 **PowerShell** 执行转换：

```powershell
wsl --set-version Ubuntu-22.04 2
```

在 **PowerShell** 进入 Ubuntu：

```powershell
wsl -d Ubuntu-22.04
```

进入后执行的是 Linux 命令。下文标注“WSL Ubuntu”的命令都在这里运行，不要在 PowerShell 中直接执行 `make` 或 `sudo apt`。

## 3. 使用你已经拉取的仓库

把仓库放在 WSL 的 Linux 文件系统中，例如 `~/PX4-Autopilot`，避免在 `/mnt/c/` 或 `/mnt/d/` 内编译，以减少性能和文件权限问题。

如果仓库已在 WSL 中，直接进入实际路径。**WSL Ubuntu：**

```bash
cd ~/PX4-Autopilot
pwd
git status --short
```

这里的 `~` 是你自己的 Ubuntu 用户目录，路径不一定与其他人的机器相同。

如果仓库在 Windows 的 `C:\Users\你的用户名\PX4-Autopilot`，可以先复制完整目录，包括隐藏的 `.git`。下面仅为路径示例，替换用户名后在 **WSL Ubuntu** 执行；目标 `~/PX4-Autopilot` 应尚不存在：

```bash
cp -a "/mnt/c/Users/你的用户名/PX4-Autopilot" "$HOME/PX4-Autopilot"
cd ~/PX4-Autopilot
```

复制后保留 Windows 原目录，在 WSL 副本中继续操作。如果 `.git` 是文件而非目录，这可能是 Git worktree，不要直接复制，应先确认它依赖的主仓库位置。若只有 ZIP 解压的源码而没有 Git 元数据，则需使用 Git 获取完整仓库，参见 [源码构建说明](docs/en/dev_setup/building_px4.md)。

PX4 依赖多个 Git 子模块。确认没有需要保留的子模块修改后，在**仓库根目录的 WSL Ubuntu** 中执行：

```bash
git submodule update --init --recursive
git submodule status --recursive
```

第一条会联网下载缺少的依赖。第二条中，行首 `-` 表示尚未初始化，`+` 表示与主仓库记录的提交不同。下载失败时先解决网络问题，再重试。

## 4. 安装依赖并首次编译

**WSL Ubuntu，仓库根目录：**

```bash
bash Tools/setup/ubuntu.sh --no-nuttx
```

脚本会安装编译工具、Python 依赖和 Gazebo，并使用 `sudo` 请求 Ubuntu 密码。输入密码时终端不显示字符是正常现象。`--no-nuttx` 跳过真实飞控所需的 NuttX 工具链，保留仿真工具；首次安装需要下载依赖，等待脚本成功结束。

安装后保存所有 WSL 工作，在 **Windows PowerShell** 重启 WSL。注意 `--shutdown` 会停止所有正在运行的 WSL 发行版：

```powershell
wsl --shutdown
wsl -d Ubuntu-22.04
```

**WSL Ubuntu：**

```bash
cd ~/PX4-Autopilot
make px4_sitl
```

这一步只编译，不会打开模拟飞机。成功时命令正常返回，生成的程序位于 `build/px4_sitl_default/bin/px4`。不要使用 `sudo make`。

## 5. 启动 Gazebo 仿真

**WSL Ubuntu，仓库根目录：**

```bash
make px4_sitl gz_x500
```

该命令会按需编译，然后启动 PX4 和 Gazebo 的 X500 四旋翼模型。首次加载资源可能较慢。成功标志是：

- Gazebo 窗口出现地面和四旋翼。
- 启动终端出现 PX4 日志和 `pxh>` 控制台提示符。

保留该终端运行。`pxh>` 是 PX4 自己的控制台，不能在其中执行 Ubuntu 的 `cd`、`apt` 等命令；需要其他 Linux 操作时，另开一个 Ubuntu 终端。

## 6. 连接 QGC 并完成模拟飞行

默认把 **Linux 版 QGC 运行在同一个 WSL 发行版中**，这样通常可以自动连接仿真。

1. 打开 [QGC 官方安装页](https://docs.qgroundcontrol.com/master/en/qgc-user-guide/getting_started/download_and_install.html#ubuntu)，选择适配 Ubuntu 和机器架构的稳定版，按该页安装 Linux 运行依赖。
2. 将下载的 AppImage 放到 WSL 用户目录。下面约定文件名为 `QGroundControl.AppImage`；实际名称不同则替换命令中的文件名。
3. 在**另一个 WSL Ubuntu 终端**执行：

```bash
cd ~
chmod +x QGroundControl.AppImage
./QGroundControl.AppImage
```

若从 Windows 浏览器下载，先将文件从 `/mnt/c/Users/你的用户名/Downloads/` 复制到 `~`。AppImage 权限或依赖报错时，按官方安装页对应的 Ubuntu 版本处理，不要用 `sudo` 启动 QGC。

保持仿真运行，等待 QGC 显示已连接车辆、姿态和高度。然后在 QGC 飞行页面操作：

1. 等待车辆完成初始化并可起飞；若有检查失败，先查看车辆消息。
2. 使用 **Takeoff（起飞）** 操作，确认界面中的目标高度并按提示确认。
3. 在 Gazebo 中观察飞机上升、悬停，同时查看 QGC 高度变化。
4. 使用 **Land（降落）**，等待飞机落地并解除武装。

菜单位置可能随 QGC 版本变化。完成后回到启动仿真的终端按 `Ctrl+C` 停止仿真，关闭残留的 Gazebo 和 QGC 窗口。

如果你已安装 Windows 版 QGC，可参考仓库的 [Windows QGC 连接步骤](docs/en/dev_setup/dev_env_windows_wsl.md#qgroundcontrol-on-windows) 配置到 WSL 的 UDP 链路；不要假定它与 WSL 内的版本一样能直接自动连接。

## 7. 从哪些目录开始读代码

| 路径 | 内容与阅读用途 |
| --- | --- |
| `src/modules/` | 核心功能模块，如 `commander` 飞行状态管理、`ekf2` 状态估计、`mc_att_control` 多旋翼姿态控制 |
| `src/drivers/` | 传感器和外设驱动 |
| `src/lib/` | 控制、数学等可复用库 |
| `src/examples/px4_simple_app/` | 简单应用示例，适合学习模块结构和消息订阅 |
| `msg/` | uORB 消息定义 |
| `ROMFS/` | 启动脚本、机型配置等运行时文件 |
| `boards/` | 飞控板和仿真目标的构建配置 |
| `platforms/` | NuttX、POSIX 等平台适配 |
| `Tools/` | 环境安装、仿真、格式检查等工具 |
| `test/`、`test_data/` | 测试程序及数据；部分单元测试也位于源码旁 |
| `docs/` | 使用和开发文档，图片等资源位于 `docs/assets/` |
| `build/` | 构建后生成的产物，不是主要源码编辑位置 |

**uORB 是模块间的发布/订阅机制。** 例如，估计模块发布 `vehicle_attitude`，姿态控制模块订阅它，以取得当前姿态。可以从 `msg/versioned/VehicleAttitude.msg` 开始，再搜索源码中对应的话题名。详见 [uORB 文档](docs/en/middleware/uorb.md)。

初次阅读建议按“简单应用 → 消息定义 → 感兴趣的功能模块”推进，不必先读完整仓库。

## 8. 日常开发命令

以下命令均在 **WSL Ubuntu 的仓库根目录** 执行：

| 命令 | 用途 |
| --- | --- |
| `make px4_sitl gz_x500` | 再次启动仿真；修改源码后也使用它增量编译并运行 |
| `make tests` | 构建并运行主机端测试目标，不等同于完整飞行验证 |
| `make check_format` | 检查代码格式和差异中的空白问题 |
| `make format` | 使用仓库工具修正代码格式，会修改文件；提交 C/C++ 改动前按仓库要求执行并检查差异 |
| `git diff` | 查看自己的修改，确认没有意外变更 |

运行新一次仿真前先停止旧实例。日常启动无需反复安装依赖或清空 `build/`。若使用 VS Code，可安装其 WSL 扩展，然后在 WSL 仓库目录执行 `code .`。

开发与提交时遵循 [AGENTS.md](AGENTS.md) 和匹配修改路径的局部指导；提交消息采用 `type(scope): description`，例如 `fix(commander): correct state transition`。

## 9. 常见问题

| 现象 | 先检查什么 |
| --- | --- |
| 提示缺少 Git 仓库或子模块文件 | 确认使用 Git 仓库，在根目录执行 `git submodule update --init --recursive` |
| 安装依赖失败 | 找到安装日志中首个失败的下载或软件包，检查 WSL 网络及软件源，解决后重跑安装脚本 |
| 编译很慢或权限异常 | 用 `pwd` 确认不在 `/mnt/c/` 下，并确认没有使用 `sudo make` |
| 编译进程被 `Killed` | 检查内存是否不足；可先尝试 `make px4_sitl -j2` 降低并行度 |
| Gazebo 无窗口或显示错误 | 确认 WSL2、显卡驱动及 WSLg；按微软文档更新 WSL 后重启 |
| QGC 提示未连接 | 确认 PX4 仍运行，QGC 与仿真在同一发行版，且没有多余仿真实例；在 `pxh>` 中运行 `mavlink status` 查看链路状态 |
| 飞机无法起飞 | 查看 QGC 车辆消息和 PX4 日志，等待传感器及位置估计就绪，按具体失败信息排查 |
| 不知道错误在哪里 | 保存所用命令、Ubuntu 版本和日志中最早的错误；末尾 `make` 失败通常只是结果 |

为区分图形问题和仿真本身的问题，可先停止旧仿真，再在 **WSL Ubuntu** 使用无图形模式：

```bash
HEADLESS=1 make px4_sitl gz_x500
```

这仍运行仿真，只是不启动 Gazebo 图形界面；仍可用 QGC 观察车辆。

## 10. 后续阅读

- [仓库内的 WSL 开发环境说明](docs/zh/dev_setup/dev_env_windows_wsl.md)
- [Ubuntu 工具链安装说明](docs/zh/dev_setup/dev_env_linux_ubuntu.md)
- [Gazebo 模型与运行选项](docs/zh/sim_gazebo_gz/index.md)
- [固件构建说明](docs/zh/dev_setup/building_px4.md)：准备接触真实飞控时再阅读，先确认具体板型。
- [ROS 2 集成](docs/zh/ros2/index.md)：需要外部程序控制或获取数据时再学习。

本仓库可能与网上其他版本教程不同。涉及构建目标和安装脚本时，优先对照当前分支的文件及内置文档。
