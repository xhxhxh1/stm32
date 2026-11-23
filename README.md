# STM32 / 嵌入式期末复习（第 1～10 章）

依据课程 PPT 和配套的 STM32F103ZET6 开发板整理。芯片与开发工具、GPIO、中断的复习点还参考了已完成的理论作业。先看章首速查，再做同类型练习，做完可展开答案看理由。原笔记按课件编号放在各章的「课件详解」里。

## 章节目录

| 章节 | 知识点 |
| --- | --- |
| [第 01 章　ARM Cortex-M3 嵌入式系统](docs/01-embedded-and-cortex-m3.md) | [嵌入式系统、Cortex-M3 与流水线](docs/01-embedded-and-cortex-m3.md#知识点速查) |
| [第 02 章　STM32 开发板硬件系统](docs/02-board-hardware.md) | [型号字母、芯片参数、启动及存储](docs/02-board-hardware.md#知识点速查) |
| [第 03 章　MDK 软件与工程模板创建](docs/03-mdk-and-project-template.md) | [MDK 组成、启动文件、编译与调试](docs/03-mdk-and-project-template.md#知识点速查) |
| [第 04 章　通用目的输入输出口](docs/04-gpio.md) | [GPIO 模式、速度、寄存器和调试脚](docs/04-gpio.md#知识点速查) |
| [第 05 章　LED 流水灯与 SysTick 定时器](docs/05-led-and-systick.md) | [库函数、流水灯和 SysTick](docs/05-led-and-systick.md#知识点速查) |
| [第 06 章　按键输入与蜂鸣器](docs/06-keys-and-buzzer.md) | [输入读取、有效电平、消抖](docs/06-keys-and-buzzer.md#知识点速查) |
| [第 07 章　数码管动态显示](docs/07-dynamic-display.md) | [共阳共阴、段码、位码、动态扫描](docs/07-dynamic-display.md#知识点速查) |
| [第 08 章　中断系统与基本应用](docs/08-interrupts.md) | [优先级、EXTI 数量、中断与事件](docs/08-interrupts.md#知识点速查) |
| [第 09 章　定时器与脉冲宽度调制](docs/09-timers-and-pwm.md) | [定时器分类、PSC、ARR、CCR、PWM](docs/09-timers-and-pwm.md#知识点速查) |
| [第 10 章　串行通信接口 USART](docs/10-usart.md) | [串行帧、校验、波特率和收发标志](docs/10-usart.md#知识点速查) |

## 做题时先分清的几个说法

| 题目里的关键词 | 对应内容 |
| --- | --- |
| 144 个、112 个 | 分别是 STM32F103ZET6 的总引脚数和 GPIO 数。[第 02 章](docs/02-board-hardware.md#知识点速查) |
| 占先、响应 | 占先也叫抢占，决定能否打断。抢占相同时，响应优先级决定等待顺序。[第 08 章](docs/08-interrupts.md#知识点速查) |
| 计数周期、方波周期 | 每次计时只翻转一次引脚时，经过两次翻转才完成一个方波周期。[第 09 章](docs/09-timers-and-pwm.md#知识点速查) |

## 阅读标记

- **【重点】**　课件的关键概念、参数和配置步骤。
- **【基础补充】**　用于理解电路、公式和代码的相关知识。
- **【讲解示例】**　按课件任务整理的程序，使用条件见各章。
- **【易错点】**　易混的电平、寄存器、计数边界或状态标志。

## 代码使用与验证

[第 09 章示例配套驱动](docs/09-example-dependencies.md)列出计时用的延时、数码管和按键文件。第 10 章还复用 TIM6 计时模块。各示例的 main.c 按项目分别使用，同名中断服务函数只保留一个定义。

示例基于 STM32F10x 标准外设库。延时示例以 HCLK=72 MHz 为条件，TIM 和 USART 的时钟按各章条件核对。原笔记中的主要完整示例曾用 Cortex-M3 编译器与标准库头文件检查语法及接口，完整链接、烧录和实机效果需用对应工程和开发板验证。
