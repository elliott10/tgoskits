# StarryOS 原生与 AxVisor 单客户机捡球程序复现

本文说明如何在另一台 Orange Pi 5 Plus 三足机器人上复现同一套
`aka-rk3588` 捡球程序，覆盖两种运行方式：

1. Orange Pi 直接启动 StarryOS；
2. Orange Pi 启动 AxVisor，再运行一个单 vCPU StarryOS 客户机。

两种方式使用相同的用户态程序、RKNN 模型和机器人配置。本文以 TGOSKits 主线
`dev` 为准，不使用双客户机分支。

## 1. 仓库与版本

需要以下仓库：

| 仓库 | 地址 | 用途 |
| --- | --- | --- |
| TGOSKits | <https://github.com/rcore-os/tgoskits> | StarryOS、AxVisor、板级配置和测试入口 |
| aka-rk3588 | <https://github.com/bullhh/aka-rk3588> | 捡球程序、模型、脚本和默认配置 |

本文核对时 TGOSKits `dev` 的提交为：

```text
d1d503acc18b5a1e981e6ae81967f4e3387bb353
```

复现时从 `dev` 克隆，并记录实际提交：

```bash
git clone --branch dev https://github.com/rcore-os/tgoskits.git
cd tgoskits
git rev-parse HEAD
```

`dev` 中的 `apps/starry/aka-rk3588/source.env` 固定了经过校验的程序版本。本文核对
时为 `aka-rk3588` 的 `tripod` 分支提交：

```text
8408f1b99afdcfcaff9908084c4b4a1909fa81bf
```

不要用未记录的 `HEAD` 或双客户机分支替换上述程序。

## 2. 硬件与安全要求

基本硬件为：

- Orange Pi 5 Plus（RK3588）；
- UVC 摄像头，建议配置为 `640x480 MJPEG @ 30 FPS`；
- RK3588 NPU；
- Feetech 三轮底盘和六关节机械臂；
- 与目标机器人匹配的电源、USB 接线和串口控制台。

每台机器人的舵机零位和机械结构都有差异。运行任何运动命令前，必须完成本机标定，
检查 `config/lekiwi_calibration.json` 和 `config/lekiwi_pick_config.txt`。标定方法见
`aka-rk3588` 仓库中的 `docs/lekiwi_arm_debug_guide.md`。

首次测试必须把底盘架起，并保证机械臂周围无人、无线缆和障碍物。程序启动成功或
识别到网球，不等于真实捡球已经通过。

## 3. 准备程序包

TGOSKits 主线已提供固定版本和校验值。在 TGOSKits 仓库执行：

```bash
cd apps/starry/aka-rk3588
./prepare-package.sh
cd ../../..
```

脚本会下载固定的 `aka-rk3588` 源码，校验源码和 AArch64 `tennis` 二进制，并生成
部署包。

如果需要自行重新编译用户态程序：

```bash
git clone --branch tripod https://github.com/bullhh/aka-rk3588.git
cd aka-rk3588
git checkout 8408f1b99afdcfcaff9908084c4b4a1909fa81bf
sudo apt install cmake gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
./build_rk3588.sh -b Release -l WARN
file build/tennis
```

性能复现使用 `Release + WARN`。高频 `INFO` 日志或把串口输出持续写入板端文件，都会
明显干扰 StarryOS 调度和 FPS。

部署时应保留以下内容：

```text
build/tennis
models/tennis.rknn
lib/
config/
run_robot_ci_once.sh
run_lekiwi_full.sh
```

已有机器人重新部署前，应先备份它自己的标定和抓取配置。

## 4. 简单配置 ostool

ostool 只负责分配板卡、上传启动文件、电源控制和串口连接。先配置服务器并确认能够
看到目标板卡：

```bash
cargo board config
cargo board ls
```

以下命令中的板卡类型使用 `OrangePi-5-Plus-robot`。如果自建 ostool-server 使用了
其他名称，应替换为实际名称；也可以显式增加 `--server` 和 `--port`。

第一次部署程序资产时，从板卡默认 Linux 启动环境执行：

```bash
cargo xtask starry app board -t aka-rk3588 \
  -b OrangePi-5-Plus-robot --linux-stage
```

该步骤把程序、模型和运行库同步到 StarryOS 与 Linux 共用的根文件系统。更换程序
版本后需要重新执行；仅重新启动 StarryOS 或 AxVisor 时不需要重复部署。

## 5. 直接运行 StarryOS

### 编译

下面的配置与主线原生机器人用例一致：

```bash
cargo xtask starry build \
  -c test-suit/starryos/board-orangepi-5-plus/robot-flow/build-aarch64-unknown-none-softfloat.toml
```

实际板测可直接执行下一条命令，它也会自动完成构建。

### 安全复现

保持底盘架起，然后运行主线板级用例：

```bash
cargo xtask starry test board -c orangepi-5-plus-robot
```

该用例直接启动 StarryOS，然后运行一次有边界的摄像头、NPU、车轮和机械臂流程。
没有网球时仍可验证视觉吞吐和受控动作。

成功时可看到类似输出：

```text
[ROBOT_CI] USB_READY ...
[ROBOT_CI] ATTEMPT_BEGIN index=1/2
[ROBOT_CI] SAFE_POSE=PASS pose=carry ...
[ROBOT_CI] RESULT=PASS attempts=1
```

第一次打开 Feetech 失败时脚本会安全收臂并最多重试一次。出现
`[ROBOT_CI] RESULT=FAIL`、panic、段错误或安全姿态失败时，应停止测试并先排查设备和
标定。

## 6. AxVisor 单 StarryOS 客户机

单客户机使用主线配置：

```text
os/axvisor/configs/vms/orangepi-5-plus/starry-smp1.toml
```

该配置创建一个单 vCPU StarryOS 客户机，从共享根文件系统加载 StarryOS 镜像，并按
主线的平台与设备规则建立客户机运行环境。

### 编译

```bash
cargo xtask axvisor build \
  -c test-suit/axvisor/normal/board-orangepi-5-plus/robot-starry/build-aarch64-unknown-none-softfloat.toml
```

### 安全复现

程序资产完成 Linux 阶段部署后，保持底盘架起并运行：

```bash
cargo xtask axvisor test board \
  -g normal -c orangepi-5-plus-robot-starry
```

正常现象是先看到 AxVisor 初始化和 StarryOS 客户机启动，随后在客户机中出现
`HOSTNAME=starry`，最后由同一个 `run_robot_ci_once.sh` 输出：

```text
[ROBOT_CI] RESULT=PASS attempts=1
```

如果出现 `Failed to initialize guest VM`，说明失败发生在客户机建立阶段；如果客户机
已经进入 shell，但缺少摄像头、NPU 或 Feetech，则应检查设备分配和 USB 枚举，不能
把它记为单客户机复现成功。

## 7. 完整捡球

只有在上述安全用例通过、本机标定完成并清空运动区域后，才进入 StarryOS shell，
切换到部署后的 `aka-rk3588` 目录执行：

```bash
export LD_LIBRARY_PATH="$PWD/lib:${LD_LIBRARY_PATH:-}"
RKNN_CORE_MASK=0 ./run_lekiwi_full.sh
```

原生 StarryOS 和 AxVisor 单客户机使用相同命令。正常状态流为：

```text
LEKIWI_CHASE
-> PICK_BALL
PICK_BALL done
-> FIND_BUCKET
-> PUT_BALL
PUT_BALL done
```

真实复现至少应分别记录：是否识别到球、底盘是否正确追踪、抓球是否成功、是否找到
桶、放球是否完成，以及停止后机械臂是否回到安全姿态。

## 8. 性能和结果判定

既有的原生/单客户机对比测试中，两种 StarryOS 路径均稳定在约 `15~16 FPS`：

| 环境 | 历史稳态 effective FPS |
| --- | ---: |
| 直接运行 StarryOS | 约 15.1 FPS |
| AxVisor 单 vCPU StarryOS 客户机 | 约 16.3 FPS |

样本数量有限，不能据此认为虚拟化比原生更快。可靠结论是单客户机没有破坏完整视觉
路径，两者处于相同区间。这些数值是历史参考，不是当前 `dev` 每次运行都必须精确
复现的结果；应以新机器人记录的原始窗口为准。

复测时应记录 TGOSKits 和 `aka-rk3588` 的完整提交、每个 10 秒窗口的
`effective_fps`、摄像头配置、`RKNN_CORE_MASK`、是否架起底盘及是否完成真实抓放。

结果应分层描述：

- 构建通过；
- StarryOS 或客户机启动通过；
- 摄像头、NPU 和 Feetech 设备通过；
- 架空安全流程通过；
- 真实捡球与放球通过；
- 稳态性能达到参考区间。

只有最后两项实际完成，才能称为在另一台机器人上复现了完整捡球流程。
