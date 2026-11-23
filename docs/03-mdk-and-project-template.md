# 第03章 MDK 软件与工程模板创建

[总目录](../README.md#章节目录) · [知识点速查](#知识点速查) · [题型练习](#题型练习) · [课件详解](#课件详解)

## 知识点速查

| 容易混的名称 | 记住它的作用 |
| --- | --- |
| MDK Core / Software Packs | MDK Core 提供 µVision 编辑、编译、调试跟踪和包安装等工具。Software Packs 提供芯片支持（Device）、CMSIS、所需中间件等组件 |
| CMSIS / STM32 标准外设库 | CMSIS 为 Cortex 微控制器提供统一的软件接口。STM32 标准外设库封装 GPIO、TIM 等外设操作，工程里同时有函数、类型和宏定义 |
| 启动文件和芯片宏 | 本板 F103ZET6 属于大容量产品，选 `startup_stm32f10x_hd.s` 和 `STM32F10X_HD`。启动文件含复位入口和中断向量表 |
| 工程扩展名 | MDK5 的 µVision 工程文件通常是 `.uvprojx`。开启 HEX 输出后，`.hex` 可用于程序烧写 |
| 仿真与硬件调试 | 软件仿真可在 Logic Analyzer 观察 `PORTC.0`。ST-LINK 可向芯片烧写程序，也可连接硬件调试 |

**两个作业问法。** 课程把 CMSIS 相关接口概括为核内外设访问、外设访问和中间件访问。工程中常用 CORE 一组放内核支持和启动文件。

**一眼判断构建错误。** 找不到 `.h` 时先查 Include Path。能找到头文件却报 `Undefined symbol`，先查实现函数的源文件或库是否参与链接。

工具分工可对照 [Keil MDK 工具与软件包说明](https://www.keil.com/download/files/softwarecomponents.pdf)。

## 题型练习

1. **单选**　给课件的 STM32F103ZET6 建工程时，用哪个启动文件？

   A. `startup_stm32f10x_ld.s`　B. `startup_stm32f10x_hd.s`　C. `stm32f10x_gpio.c`

2. **多选**　关于 MDK5，以下哪些说法正确？

   A. MDK Core 中有编辑和调试工具　B. Software Packs 可提供器件支持　C. `.uvprojx` 是常见工程文件扩展名　D. ST-LINK 只能看波形

3. **判断**　头文件找到后，对应的驱动源文件就自动参加链接。

<details>

<summary>展开答案与理由</summary>

1. **B。** `hd` 对应这里的大容量型号。
2. **ABC。** ST-LINK 常用来烧写和硬件调试。
3. **错。** 工程还需加入实现所在的源文件或库。

</details>

### 读一句代码

```c
#include "stm32f10x_gpio.h"
```

**参考注释**　引入 GPIO 驱动的接口声明。

## 课件详解

点开对应小节，可以看课件中的解释、寄存器细节和原笔记里的完整示例。

### 目录

<details>

<summary>展开本章原目录</summary>

- [3.1 STM32固件库认知](#31-stm32固件库认知)
- [3.2 工程模板创建](#32-工程模板创建)
- [3.3 软件模拟仿真](#33-软件模拟仿真)
- [本章复习自检](#本章复习自检)

</details>

### 3.1 STM32固件库认知

<details>

<summary>展开本节详解</summary>

#### 3.1.1 STM32固件库概述

**【重点】** STM32F10x标准外设库在用户程序与寄存器之间提供操作接口，包括函数、结构体和宏定义。调用库函数最终仍是配置或访问硬件寄存器。

CMSIS全称Cortex Microcontroller Software Interface Standard，即ARM Cortex微控制器软件接口标准。课件强调通过统一的内核软件接口，提高基于Cortex-M3微控制器的软件兼容性。

| 层次 | 作用 | 例子 |
| --- | --- | --- |
| 用户应用 | 组织任务和业务逻辑 | 流水灯、显示时间、按键调时 |
| 板级/自编驱动 | 把具体接线封装成函数 | `LEDInit()`、`DsgShowTime()` |
| 标准外设库 | 操作MCU外设 | `GPIO_Init()`、`TIM_Cmd()` |
| CMSIS/器件支持 | 内核定义、器件寄存器定义、系统支持 | `core_cm3.h`、`stm32f10x.h` |
| 硬件 | 实际执行寄存器控制的电路 | GPIO、NVIC、TIM |

**【易错点】** “标准外设库”和HAL不是同一套接口，后续代码不要混入`HAL_GPIO_WritePin`等不同库的调用。

#### 3.1.2 STM32固件库下载

课件通过ST官网获取STM32F1标准外设库。对应官方软件为[STSW-STM32054](https://www.st.com/en/embedded-software/stsw-stm32054.html)。复习重点是确认芯片系列、库类型和文件是否齐全，不需要死记旧版网页按钮位置。

#### 3.1.3 STM32固件库目录结构

| 目录/文件 | 用途 | 常见错误 |
| --- | --- | --- |
| `Libraries/CMSIS/CM3/CoreSupport` | Cortex-M3内核支持，如`core_cm3.h` | 把内核支持与某一外设驱动混淆 |
| `Libraries/CMSIS/CM3/DeviceSupport` | STM32器件定义与系统初始化 | 没加入正确的系统源文件 |
| `stm32f10x.h` | 器件寄存器、类型和宏定义 | 头文件路径错误 |
| `system_stm32f10x.c` | 系统时钟初始化等 | 误认为晶振接上后主频自动为72MHz |
| `startup_stm32f10x_hd.s` | 大容量器件启动文件、向量表等 | 选错容量系列或重复加入启动文件 |
| `STM32F10x_StdPeriph_Driver/src` | 外设库函数的实现 | 只包含头文件，未加入对应`.c` |
| `STM32F10x_StdPeriph_Driver/inc` | 外设接口声明、结构体和宏 | 未添加Include Path |
| `Project/STM32F10x_StdPeriph_Template` | 官方工程模板 | 把模板和最终应用当作同一内容 |
| `Project/STM32F10x_StdPeriph_Examples` | 外设示例 | 未按本板引脚和型号修改就直接使用 |
| `Utilities` | 评估板BSP及第三方固件 | 不区分芯片通用代码和具体板级代码 |

STM32F103ZET6属于大容量产品，对应`hd`启动文件和`STM32F10X_HD`宏。课件个别目录文字有拼写误差，实际应使用CoreSupport、DeviceSupport等文件夹名称。

</details>

### 3.2 工程模板创建

<details>

<summary>展开本节详解</summary>

#### 3.2.1 Keil MDK软件简介

MDK-ARM包括编译工具链、μVision集成开发环境及调试环境。课件还介绍RTX、网络、USB、GUI等支持；前九章重点使用建工程、添加源码、设置编译选项、生成目标文件和调试功能。

**【基础补充·构建过程】** 预处理处理`#include`和宏，编译把C源文件转换成目标代码，链接把目标文件和库组合起来并确定地址，最终可生成调试用目标文件和HEX等下载文件。

#### 3.2.2 工程模板的创建

**【重点·按顺序掌握】**

1. 建立Libraries、Output、Startup、User、APP文件夹。
2. Libraries放库文件，Startup放`startup_stm32f10x_hd.s`，User放主程序、配置与中断文件，APP放自编驱动，Output保存构建输出。
3. Project → New μVision Project，新建工程，器件选择STM32F103ZE。
4. 课件采用手动添加启动文件，自动添加提示选“否”，避免重复。
5. 创建工程分组并添加源文件。
6. Options for Target → Output勾选Create HEX File，设置输出位置。
7. C/C++ → Define填写`USE_STDPERIPH_DRIVER,STM32F10X_HD`。
8. 配置Include Path，覆盖User、APP、CMSIS内核/器件目录和外设库inc目录。
9. 创建公共头文件和主程序，编译后逐项解决错误。

| 工程组 | 课件初始加入的文件 |
| --- | --- |
| User | `main.c`、`stm32f10x_it.c` |
| Cmsis | `core_cm3.c`、`system_stm32f10x.c` |
| Startup | `startup_stm32f10x_hd.s` |
| ST_driver | `stm32f10x_gpio.c`、`stm32f10x_rcc.c` |
| APP | 用户编写的LED、按键、显示等驱动 |

**宏的作用。** `STM32F10X_HD`选择大容量器件相关定义；`USE_STDPERIPH_DRIVER`启用标准外设库配置关系。宏名称必须准确，不能把它们当作任意的工程注释。

**【基础补充·程序从哪里开始】** 复位后先按向量表装入初始栈指针并进入Reset_Handler，再执行启动文件和系统初始化、C运行环境初始化，最后进入main。课件模板通过系统支持文件设置时钟；main并不是芯片上电后执行的第一条指令。初始化全局变量、清零未显式初始化的静态存储变量等，也依赖正确的启动和运行环境。

**【基础补充·头文件和源文件】** `.h`主要提供声明，告诉编译器函数和类型怎么使用；`.c`提供定义，实现函数内容。Include Path让编译器找到头文件；把`.c`加入工程才会编译其函数实现。这两步缺一不可。

下面是供后续实验复用的讲解版文件结构，表示写法，不是对课件截图逐字转录：

```c
/* led.h：函数声明，头文件保护防止重复包含 */
#ifndef LED_H
#define LED_H

#include "stm32f10x.h"
void LEDInit(void);
void LED_On(uint16_t pin);

#endif
```

```c
/* led.c：函数定义 */
#include "led.h"

void LED_On(uint16_t pin)
{
    GPIO_ResetBits(GPIOC, pin); /* 本板低电平点亮 */
}
```

**【基础补充·读代码必备】**

| 写法 | 含义 | 本课程例子 |
| --- | --- | --- |
| `uint8_t / uint16_t / uint32_t` | 8/16/32位无符号整数 | 范围分别为0～255、0～65535、0～4294967295 |
| `u8 / u16 / u32` | 库中常见的对应类型别名 | 旧版课件常用这些名字 |
| `GPIO_InitTypeDef gpio;` | 定义结构体变量，集中保存配置 | 引脚、速度、模式 |
| `gpio.GPIO_Pin` | 用`.`访问结构体成员 | 设置要配置的引脚 |
| `&gpio` | 取结构体变量地址 | 传给`GPIO_Init` |
| `GPIOC->ODR` | 用指针访问寄存器结构体成员 | 访问GPIOC输出数据寄存器 |
| `const` | 表示不应经该对象修改数据 | 固定段码表 |
| `static`局部变量 | 调用结束后保留值 | 按键是否已处理的状态 |
| `extern` | 声明变量或函数在其他位置定义 | 多文件共享hour、minute、second |
| `volatile` | 提醒编译器每次按程序要求访问对象 | 主程序与中断共享的变量 |

**定义与声明示例：**

```c
/* 在一个.c文件中定义一次，分配存储空间 */
volatile uint8_t hour = 12;

/* 在共享头文件中声明，其他.c包含这个头文件 */
extern volatile uint8_t hour;
```

不要在多个`.c`中各定义一次同名全局变量；普通变量的定义也不要直接放在被多个源文件包含的头文件里。`volatile`不负责实现多步操作的互斥，第8章继续说明。

**【基础补充·常见编译/链接问题】**

| 现象 | 优先检查 |
| --- | --- |
| 找不到`.h`文件 | 文件确实存在、Include Path、文件名 |
| 未知类型/宏 | 是否包含相应头文件、器件宏是否正确 |
| 函数未声明 | 是否在使用前提供正确声明 |
| Undefined symbol | 对应`.c`是否加入工程、函数名是否一致 |
| 多重定义 | 是否重复添加源文件、是否重复定义全局变量/中断函数 |
| 能编译但引脚无变化 | 时钟使能、引脚模式、有效电平、是否实际运行 |

</details>

### 3.3 软件模拟仿真

<details>

<summary>展开本节详解</summary>

**【重点·课件操作流程】**

1. 在模板加入PC0输出方波的程序并编译。
2. Options for Target → Debug选择Use Simulator。
3. Debug → Start/Stop Debug Session进入调试。
4. View → Analysis Windows → Logic Analyzer。
5. Setup → New添加`PORTC.0`。
6. Debug → Run，观察PC0电平变化；结束时退出调试。

**【讲解示例·PC0交替高低】** 依赖正确的STM32F10x工程模板及GPIO/RCC库源文件。这里的循环延时只用于观察变化，不能声称精确1秒。

<details>

<summary>PC0 方波完整示例 · main.c</summary>

```c
#include "stm32f10x.h"

static void BusyDelay(void)
{
    volatile uint32_t i;
    for (i = 0; i < 600000U; ++i) { }
}

int main(void)
{
    GPIO_InitTypeDef gpio;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    GPIO_SetBits(GPIOC, GPIO_Pin_0); /* 预置高电平，LED先灭 */
    gpio.GPIO_Pin = GPIO_Pin_0;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    gpio.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOC, &gpio);

    while (1)
    {
        GPIO_ResetBits(GPIOC, GPIO_Pin_0); /* 输出0，LED亮 */
        BusyDelay();
        GPIO_SetBits(GPIOC, GPIO_Pin_0);   /* 输出1，LED灭 */
        BusyDelay();
    }
}
```

</details>

代码顺序是“开端口时钟→设置输出初值→填结构体→初始化GPIO→循环改变电平”。两次延时分别对应低电平和高电平持续时间；完整波形周期是两段时间之和。

**【易错点】** 编译通过证明构建成功；软件仿真证明模拟环境下的行为；下载成功与实物运行正确还需要相应证据。这三者不能互相替代。

</details>

### 本章复习自检

<details>

<summary>展开本节详解</summary>

1. 为什么STM32F103ZET6使用`hd`启动文件？两项预编译宏是什么？
2. 加入源文件与设置Include Path分别解决什么问题？
3. `.h`中的声明与`.c`中的定义有什么区别？`&`和`->`怎样读？
4. 怎样让一个变量在多个文件中共享，且只定义一次？
5. 怎样在Logic Analyzer观察PC0？为什么空循环不能保证精确延时？

</details>

---

[上一章](02-board-hardware.md) · [总目录](../README.md#章节目录) · [下一章](04-gpio.md)
