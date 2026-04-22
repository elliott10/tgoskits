# StarryOS 适配 SG2002 的组件修改说明

为支持 StarryOS 在 SG2002 板卡上运行，相关修改分布在多个依赖组件中，分别覆盖平台 BSP、平台特性传递、SD卡存储驱动、页表属性扩展以及 CPU 特性支持等。整体上，这些改动共同完成了从板级启动到根文件系统挂载，再到用户态shell的运行。

StaryOS所依赖的组件，所作的修改概述：

- `starryos`：宏内核主仓库，负责引入 SG2002 平台支持包及相关依赖，并在任务执行路径中调用硬件 flush cache 接口，同时增加用户态访问的 `/dev` 设备文件接口。
- `axplat-riscv64-sg2002`：SG2002 板子的 BSP 支持包，负责引导启动、串口输出、虚拟内存映射、板级设备地址空间定义等平台功能。
- `axfeat`：负责硬件平台相关 feature 的配置透传，使 SG2002 相关物理特性能够在arceos框架内传递到所依赖的下层组件。
- `axdriver`：因 SG2002 板子无法引导 GPT 分区，而只能使用MBR分区表，增加对 MBR 分区表的识别与加载；同时基于 SD 卡驱动做 OS 接口适配，以提升 rootfs 文件系统总体可用容量。
- `axdriver_block`：增加基于 `sg200x_bsp::sdmmc` 的 SD 卡驱动支持，并提供 OS 对 CVSD 的调用接口。
- `page_table_entry`：增加 C9xx CPU 页表项扩展 flags 支持，包括 `so`、`cache`、`buf`、`share` 等。
- `axcpu`：增加 C9xx CPU 的刷高速缓存 flush cache 的实现，以及对寄存器 `sstatus` 读写能力，用于 CPU 特性设置，例如使能 `sstatus.VS` 以支持向量指令。

## 1. starryos

`starryos` 是 SG2002 适配的集成入口，负责把各个依赖组件接入到内核构建和运行流程中。其修改重点包括：

- 新增 `sg2002` feature，并接入 `axplat-riscv64-sg2002` 与对应平台配置。
- 在构建系统中增加 SG2002 平台目标，固化构建参数。
- 将 icache 刷新从直接执行 `fence.i` 调整为统一走 `axhal::asm::flush_icache()`，把具体实现放置到 axcpu 层。
- 在 ELF 映射、任务切换和系统调用路径中补充 cache 高速缓存刷新。
- 增加 SG2002 相关设备节点，为用户态访问板级设备提供接口。

## 2. axplat-riscv64-sg2002

作为 SG2002 的 BSP，负责把 `axplat` 抽象接口实现成具体硬件。重点包括：

- 在启动阶段建立早期页表，并为内存和 MMIO 设置符合 C9xx 页表项的属性位。
- 初始化 UART0，保证最早期串口输出。
- 接入 PLIC、定时器中断、软件中断和 SBI IPI。
- 提供时钟、物理内存范围、MMIO 范围以及地址转换支持。
- 实现关机和多核启动接口。
- 在 `axconfig.toml` 中配置 UART、PLIC、CLINT、RTC、CVSD 等板级参数。

它可以作为 SG2002 板子能否正确启动和完成基础初始化提供必要的基础支持。

sg2002硬件平台支持包发布于crates.io： https://crates.io/crates/axplat-riscv64-sg2002
串口驱动：https://crates.io/crates/dw_uart_rs

## 3. axfeat

`axfeat` 在这次适配中的作用主要是做平台 feature 的配置透传。 其修改重点包括：
- 将 `sg2002` 平台相关 feature 传递到 `axcpu`、`axdriver`、`axdriver_block` 等下游组件。
- 让平台 BSP、驱动和 CPU 扩展能力能够进入arceos框架.

## 4. axdriver

`axdriver` 的核心修改是添加MBR分区表的支持和加载，以及将 SG2002 的SD卡存储接入系统。其重点包括：

- 增加 MBR 分区识别能力，不再只依赖 GPT。
- 增加 `cvsd` 块设备 feature。
- 新增 `MbrPartitionDev`，从 SD 卡 MBR 中选取可启动 Linux 分区作为 rootfs。
- 在驱动探测路径中注册 CVSD 设备，并包装为系统可用块设备。

这部分修改解决了 GPT 无法在 SG2002 上直接用于引导的问题，使系统可以识别 SD 卡驱动，并从 MBR 分区的 SD 卡上挂载 rootfs。

代码的修改及PR的提交链接：https://github.com/arceos-org/arceos/pull/364

## 5. axdriver_block

`axdriver_block` 负责补齐 CVSD 的底层块设备实现。其重点包括：

- 增加 `cvsd` feature。
- 新增基于 `sg200x_bsp::sdmmc::Sdmmc` 的 SD 卡驱动实现。
- 提供标准块读写接口，并将底层 BSP 错误参数转换为 OS 统一驱动错误参数。

它与 `axdriver` 的 MBR 分区支持组合后，形成了“SD 卡控制器 -> MBR 分区 -> rootfs”的完整存储路径。

代码的修改及PR的提交链接：https://github.com/arceos-org/axdriver_crates/pull/27

## 6. page_table_entry

`page_table_entry` 主要负责补齐 C9xx CPU 页表项扩展属性。其修改重点包括：

- 新增 `SO`、`CACHE`、`BUF`、`SHARE` 等扩展标志位。
- 增加适用于 SG2002 内核页和设备页的组合属性。
- 调整 `MappingFlags` 到硬件页表位的映射逻辑，使普通内存与设备内存拥有不同的缓存属性。

这部分修改解决的是页表能有效映射，设置的属性符合 C9xx CPU的硬件拓展属性。

代码的修改及PR的提交链接：https://github.com/arceos-org/page_table_multiarch/pull/46

## 7. axcpu

`axcpu` 的修改主要是补齐 SG2002/C9xx 相关 CPU 功能。其重点包括：

- 新增 `sg2002` feature，并与页表属性扩展联动。
- 增加 `sstatus` 读写封装。
- 在用户态上下文初始化时设置 `sstatus.VS`，为向量指令支持提供基础。
- 扩展 `flush_icache()`，在通用 `fence.i` 之外增加 T-Head C9xx 专用 cache 同步指令。实现在进入用户空间前刷新所有指令cache

这部分修改保证了 CPU 缓存和状态控制能够按 SG2002 的实际硬件平台工作。

代码的修改及PR的提交链接：https://github.com/arceos-org/axcpu/pull/36

## 8. tpu

sg2002 tpu加速驱动库
发布于crates.io： https://crates.io/crates/tpu-sg2002

## 9. 总结

SG2002 适配设计到系统的多个组件的修改，需要协同地修改包括 内核集成层、跨平台框架层、驱动层、页表层、CPU 层等：

- `starryos` 负责对应硬件平台的系统集成和用户态接口。
- `axplat-riscv64-sg2002` 负责板级启动和硬件抽象。
- `axfeat` 负责平台 feature 透传。
- `axdriver` 与 `axdriver_block` 负责存储的分区识别、驱动调用、rootfs 加载。
- `page_table_entry` 与 `axcpu` 负责定义 C9xx CPU硬件的页表属性和缓存语义。

这些改动共同支撑起 StarryOS 在 SG2002 板卡上的引导启动、存储访问和用户态shell的交互运行。
