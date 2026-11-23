# 第 05 章 LED 流水灯与 SysTick 定时器

[总目录](../README.md#章节目录)

## 5.1～5.3 GPIO 输出与流水灯

固件库把常用的寄存器操作封装成函数。配置 GPIO 时，用 `ENABLE` 打开外设时钟，选择端口后再填 `GPIO_Pin`、`GPIO_Mode` 和 `GPIO_Speed`。同一端口可用 `GPIO_Pin_0 | GPIO_Pin_1` 一次选中两个引脚。

| 标准库函数 | 基本作用 |
| --- | --- |
| `RCC_APB2PeriphClockCmd` | 开启或关闭指定的 APB2 外设时钟 |
| `GPIO_Init` | 按结构体指定的引脚、模式和速度配置端口 |
| `GPIO_SetBits` / `GPIO_ResetBits` | 把选中的输出位置 1 / 清 0 |
| `GPIO_WriteBit` | 按 `Bit_SET` 或 `Bit_RESET` 写一个输出位 |
| `GPIO_Write` | 写整个端口的输出值 |

本板 L1～L8 接 PC0～PC7，低电平点亮。GPIOC 的低八位依次写入 `0xFE`、`0xFD`、`0xFB`，唯一的 0 依次移到第 0、1、2 位，亮点就从 L1 移到 L2、L3。每次写入后延时一段时间，让亮灭变化看得清楚。

接下来送 `0xF7`、`0xEF`、`0xDF`、`0xBF`、`0x7F`，就能依次点亮 L4～L8。读流水灯程序时，`LEDInit` 负责配置引脚，`LEDdisplay` 负责按顺序更新输出。

## 5.4 SysTick

**SysTick 是 Cortex-M3 内核里的 24 位向下计数器。** 在 STM32F103 上，计数时钟可选 HCLK 或 HCLK/8。

| 寄存器 | 作用 |
| --- | --- |
| `LOAD` | 保存重装值，最大是 `0xFFFFFF` |
| `VAL` | 保存当前计数。写入会清当前计数，也会清 `COUNTFLAG` |
| `CTRL` | 控制计数和中断，也能读出计数状态 |
| `CALIB` | 提供校准信息 |

`ENABLE` 控制计数开关，`TICKINT` 控制节拍中断。`CLKSOURCE` 为 1 选 HCLK，为 0 选 HCLK/8。计数到 0 时 `COUNTFLAG` 置位，读 `CTRL` 会清掉该标志。

连续计时时，**一个周期有 `LOAD + 1` 个节拍**。若 HCLK=72 MHz，选 HCLK/8 得 9 MHz，1 ms 有 9000 个节拍，周期重装值是 8999。课件的单次延时也按 9 MHz 换算，每微秒有 9 个节拍。

课件的 `delay_us` 按微秒设定，`delay_ms` 按毫秒设定。单次延时可轮询 `COUNTFLAG` 判断时间到，启用 `TICKINT` 后也可在数到 0 时产生节拍中断。

[上一章](04-gpio.md) · [下一章](06-keys-and-buzzer.md)
