# 第 05 章 LED 流水灯与 SysTick 定时器

## 5.1～5.3　流水灯为什么看起来会移动

课件的八盏灯接 PC0～PC7，**低电平点亮**。GPIOC 的低八位依次变为 `0xFE`、`0xFD`、`0xFB` 时，唯一的 0 依次移到第 0、1、2 位，所以亮点从 L1 移到 L2、L3。每个状态保持一小段时间，人眼就能看出移动顺序。

| 库函数 | 一句话注释 |
| --- | --- |
| `RCC_APB2PeriphClockCmd`、`GPIO_Init` | 先打开相应端口时钟，再按结构体参数配置引脚 |
| `GPIO_SetBits` / `GPIO_ResetBits` | 把所选引脚置高 / 置低 |
| `GPIO_WriteBit` / `GPIO_Write` | 前者按参数写一个位，后者写整个端口的输出值 |

## 5.4　SysTick 怎样计时

SysTick 是 Cortex-M3 内核中的 **24 位递减计数器**。`LOAD` 存重装值，`VAL` 是当前值，`CTRL` 控制计数开关和时钟源。计到 0 时，`COUNTFLAG` 置位。程序可以查询这个标志，也可以启用节拍中断。

**周期计算**　连续计时时，一个周期经历 `LOAD + 1` 个时钟节拍。若 HCLK 为 72 MHz，SysTick 选 HCLK/8，则计数时钟为 9 MHz，1 ms 有 9000 个节拍。[ST 重装值说明](https://www.st.com/resource/en/programming_manual/cd00228163.pdf)

```c
SysTick->LOAD = 8999U;  // 在 9 MHz 时钟下，连续计数的节拍周期为 1 ms
```

---

[上一章](04-gpio.md) · [章节目录](../README.md#章节目录) · [下一章](06-keys-and-buzzer.md)
