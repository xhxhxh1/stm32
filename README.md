# STM32 期末理论复习（第 1～10 章）

按原课件顺序复习，示例采用 **STM32F103ZET6** 配套开发板。读代码时，把“操作了什么”和“硬件会怎样”连成一句注释。例如，将 PC0 拉低，会点亮板上的 L1。

## 章节目录

| 章节 | 本章重点 |
| --- | --- |
| [01　ARM Cortex-M3 嵌入式系统](docs/01-embedded-and-cortex-m3.md) | 嵌入式系统、处理器内核与 MCU 的关系 |
| [02　STM32 开发板硬件系统](docs/02-board-hardware.md) | 电源、时钟、复位与有效电平 |
| [03　MDK 软件与工程模板创建](docs/03-mdk-and-project-template.md) | 工程文件分工，编译、链接与仿真 |
| [04　通用目的输入输出口](docs/04-gpio.md) | 八种模式，输入与输出寄存器 |
| [05　LED 流水灯与 SysTick 定时器](docs/05-led-and-systick.md) | 低电平点灯，SysTick 的 1 ms 周期 |
| [06　按键输入与蜂鸣器](docs/06-keys-and-buzzer.md) | 上拉输入、按键消抖与发声频率 |
| [07　数码管动态显示](docs/07-dynamic-display.md) | 共阳段码、位选与动态扫描 |
| [08　中断系统与基本应用](docs/08-interrupts.md) | 边沿触发、优先级与挂起标志 |
| [09　定时器与脉冲宽度调制](docs/09-timers-and-pwm.md) | 一秒定时、PWM 频率与占空比 |
| [10　串行通信接口 USART](docs/10-usart.md) | 8N1 数据帧、收发标志与串口校时 |

以后学习完整程序时，可查阅[改版前的详细笔记](https://github.com/xhxhxh1/stm32/tree/0ef339795b6e95e93a273a5485d1e5758c5444cb/docs)和[第 09 章示例配套驱动](docs/09-example-dependencies.md)。
