# 第 03 章 MDK 软件与工程模板创建

[总目录](../README.md#章节目录)

## 3.1 MDK 和固件库

| 名称 | 作用 |
| --- | --- |
| MDK Core | 提供 µVision 编辑、编译、调试与跟踪、包安装等工具 |
| Software Packs | 提供器件支持（Device）、CMSIS 和中间件等组件 |
| CMSIS | Cortex 微控制器的软件接口标准。课程把相关接口概括为核内外设访问、外设访问和中间件访问 |
| STM32F10x 标准外设库 | 用库函数封装外设操作。`inc` 是头文件目录，`src` 是源文件目录 |

课件中的固件库主要看三个目录。

- `Libraries` 放 CMSIS、启动文件和标准外设驱动。
- `Project` 放 ST 提供的工程模板和外设驱动示例。
- `Utilities` 放官方评估板的 BSP（板级支持包）和附加固件。

## 3.2 工程文件

| 文件 | 需要记住的作用 |
| --- | --- |
| `.uvprojx` | Keil MDK5 的 µVision 工程文件 |
| `startup_stm32f10x_hd.s` | F103ZET6 是大容量型号，选 `hd` 启动文件。它提供复位入口和中断向量表 |
| `core_cm3.h`、`system_stm32f10x.c` | 前者是内核支持头文件，后者负责系统初始化和时钟配置 |
| `main.c`、`stm32f10x_it.c` | 分别放主程序和中断服务函数 |
| `stm32f10x_conf.h` | 标准外设库的配置头文件，课件放在 User 目录 |
| `.h`、`.c` | 头文件通常声明函数和类型，源文件提供实现 |
| `.hex` | 构建后可生成的程序烧写文件 |

建标准库工程时选对芯片和启动文件。课件模板中，**Startup** 放启动文件，**ST_Driver** 放用到的标准库源文件，**APP** 放自己写的外设驱动。

在 C/C++ 选项里，**Define** 填宏，常用的有 `STM32F10X_HD` 和 `USE_STDPERIPH_DRIVER`。**Include Path** 指定头文件搜索目录。在 Output 选项勾选 **Create HEX File** 后，构建可生成 `.hex` 文件。

## 3.3 编译、下载与调试

- **编译、链接**　编译把源文件变成目标文件，链接把所需目标文件与库组合起来。
- **软件仿真**　选择 Use Simulator 后在电脑上模拟。课件用 Logic Analyzer 观察 `PORTC.0` 的波形。
- **硬件调试**　通过调试器连接开发板。**ST-LINK 可烧写程序、调试硬件**。课件板的外部主晶振为 8 MHz。

[上一章](02-board-hardware.md) · [下一章](04-gpio.md)
