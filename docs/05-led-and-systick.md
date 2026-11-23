# 第05章 LED 流水灯与 SysTick 定时器

[总目录](../README.md#章节目录) · [知识点速查](#知识点速查) · [题型练习](#题型练习) · [课件详解](#课件详解)

## 知识点速查

- **三个常用输出函数。** `GPIO_SetBits` 将选中位设为 1，`GPIO_ResetBits` 将选中位清为 0，`GPIO_Write` 写整个端口的输出值。`GPIO_WriteBit` 按 `Bit_SET` 或 `Bit_RESET` 写一个引脚。
- **流水灯看哪一位是 0。** GPIOC 低八位为 `0xFE` 时 PC0 为 0，L1 亮。写 `0xFD` 时 PC1 为 0，L2 亮。
- **SysTick 是 Cortex-M3 内核里的 24 位递减计数器。** `LOAD` 是重装值，`VAL` 是当前计数，`CTRL` 是控制和状态，`CALIB` 提供校准信息。
- **控制位各管一件事。** `ENABLE` 开关计数，`TICKINT` 控制节拍异常，`CLKSOURCE` 选择 HCLK 或 HCLK/8。计到 0 会置 `COUNTFLAG`，读 `CTRL` 会清这个标志。
- **只算一个连续周期。** 72 MHz 的 HCLK 选八分频后是 9 MHz。1 ms 需要 9000 个节拍，所以按连续计时公式 `LOAD + 1`，写 `LOAD = 8999`。课件的一次性延时写法见 5.4。

## 题型练习

1. **单选**　流水灯写 GPIOC 的低八位为 `0xFD` 时，哪盏灯亮？

   A. L1　B. L2　C. L7　D. L8

2. **多选**　下列哪些是 SysTick 的寄存器？

   A. `LOAD`　B. `VAL`　C. `CTRL`　D. `GPIOx_CRL`

3. **判断**　SysTick 是一个 24 位递增计数器。

<details>

<summary>展开答案与理由</summary>

1. **B。** PC1 是唯一的低电平，L2 和它相连。
2. **ABC。** SysTick 还有 `CALIB`。
3. **错。** 它从重装值向下数到 0。

</details>

### 读一句代码

```c
GPIO_WriteBit(GPIOC, GPIO_Pin_0, Bit_RESET);
```

**参考注释**　将 PC0 输出位清 0。本板 PC0 为推挽输出时，L1 点亮。

## 课件详解

点开对应小节，可以看课件中的解释、寄存器细节和原笔记里的完整示例。

### 目录

<details>

<summary>展开本章原目录</summary>

- [5.1 库函数开发方法](#51-库函数开发方法)
- [5.2 GPIO输出库函数](#52-gpio输出库函数)
- [5.3 LED流水灯控制](#53-led流水灯控制)
- [5.4 SysTick定时器](#54-systick定时器)
- [本章公式与易错点自检](#本章公式与易错点自检)

</details>

### 5.1 库函数开发方法

<details>

<summary>展开本节详解</summary>

库函数为用户提供API（Application Program Interface，应用程序接口）。一次典型外设使用顺序为：开启时钟→填写配置结构体→调用初始化函数→按需要使能外设→读写或处理中断。

**【基础补充·结构体调用过程】**

```c
GPIO_InitTypeDef gpio;                 /* 定义一份配置 */
gpio.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1;/* 用按位或选择两个引脚 */
gpio.GPIO_Speed = GPIO_Speed_2MHz;      /* 输出速度档位 */
gpio.GPIO_Mode = GPIO_Mode_Out_PP;      /* 普通推挽输出 */
GPIO_Init(GPIOC, &gpio);               /* 把配置地址交给库函数 */
```

`GPIOC`决定操作哪个端口，`GPIO_Pin`决定该端口哪些位。端口和引脚必须一起确定。

</details>

### 5.2 GPIO输出库函数

<details>

<summary>展开本节详解</summary>

#### 5.2.1 函数RCC_APB2PeriphClockCmd

```c
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph,
                            FunctionalState NewState);
```

使能/关闭APB2外设时钟。第二个参数用`ENABLE`或`DISABLE`；多个外设可用`|`组合。

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC |
                      RCC_APB2Periph_GPIOE, ENABLE);
```

课件列举GPIOA～GPIOE、AFIO、ADC1/2、TIM1、SPI1、USART1等时钟标识；本型号还需按对应头文件选择GPIOF/G等。APB1外设要调用相应的APB1时钟函数，例如第9章TIM3、TIM6，不能用APB2函数替代。

#### 5.2.2 函数GPIO_Init

```c
void GPIO_Init(GPIO_TypeDef *GPIOx, GPIO_InitTypeDef *GPIO_InitStruct);
```

| 成员 | 填写内容 | 重点 |
| --- | --- | --- |
| `GPIO_Pin` | `GPIO_Pin_0`至`GPIO_Pin_15`的组合 | 是掩码，不是数字编号 |
| `GPIO_Speed` | 2MHz、10MHz、50MHz | 对输出速度配置有意义，不决定实际波形频率 |
| `GPIO_Mode` | 8种工作模式之一 | 按输入/输出、普通/复用、电气方式选择 |

`GPIO_Pin_All`选择全16位。只想改变PC0～PC7时可使用`0x00FF`；不要不加判断地把整个端口改成输出，影响PC8或其他功能。

#### 5.2.3 函数GPIO_Write

```c
void GPIO_Write(GPIO_TypeDef *GPIOx, uint16_t PortVal);
```

向整个端口的输出数据寄存器写入16位值。`GPIO_Write(GPIOC, 0x00FE)`会把低8位写成`11111110`，同时把高8位也写成0。

**【易错点】** 写整端口会影响共享端口的其他输出，PC8若控制蜂鸣器，就可能被意外拉低。需要只改变LED时，优先使用按位函数或明确的掩码操作。

#### 5.2.4 函数GPIO_SetBits

```c
void GPIO_SetBits(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

把选中的输出位设为1，其他位保持。对于本板LED，`GPIO_SetBits(GPIOC, GPIO_Pin_0)`使L1熄灭。

#### 5.2.5 函数GPIO_ResetBits

```c
void GPIO_ResetBits(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

把选中的输出位设为0。对于本板LED，`GPIO_ResetBits(GPIOC, GPIO_Pin_0)`使L1点亮。函数名中的Reset指清输出位，不是让MCU复位。

#### 5.2.6 函数GPIO_WriteBit

```c
void GPIO_WriteBit(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin,
                   BitAction BitVal);
```

第三个参数取`Bit_SET`或`Bit_RESET`。

```c
GPIO_WriteBit(GPIOC, GPIO_Pin_0, Bit_RESET); /* L1亮 */
GPIO_WriteBit(GPIOC, GPIO_Pin_0, Bit_SET);   /* L1灭 */
```

**三组名称不要混用：** 使能控制用ENABLE/DISABLE；位写入用Bit_SET/Bit_RESET；状态查询常返回SET/RESET。看函数参数类型选择符号。

</details>

### 5.3 LED流水灯控制

<details>

<summary>展开本节详解</summary>

**【重点·低8位状态】**

| 亮灯位置 | 二进制PC7…PC0 | 十六进制 |
| --- | --- | --- |
| L1/PC0 | 11111110 | 0xFE |
| L2/PC1 | 11111101 | 0xFD |
| L3/PC2 | 11111011 | 0xFB |
| L4/PC3 | 11110111 | 0xF7 |
| L5/PC4 | 11101111 | 0xEF |
| L6/PC5 | 11011111 | 0xDF |
| L7/PC6 | 10111111 | 0xBF |
| L8/PC7 | 01111111 | 0x7F |

每次只有一个0，因此只有一个LED亮。低8位编码可由`(~(1U << i)) & 0xFFU`计算，其中 $i$ 为0～7。

课件把代码分成LED.c/LED.h，提供`LEDInit()`、`LEDdisplay()`及延时函数，main先初始化再显示。自编`.c`加入APP组，`.h`提供声明并包含到公共头文件中。

**【讲解示例·流水灯main.c】** 与本章后面的`delay.c`一起使用；依赖GPIO/RCC库。只操作低8位，避免影响PC8。

<details>

<summary>流水灯完整示例 · main.c</summary>

```c
#include "stm32f10x.h"
#include "delay.h"

static void LEDInit(void)
{
    GPIO_InitTypeDef gpio;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    GPIO_SetBits(GPIOC, 0x00FFU); /* 所有LED先灭 */
    gpio.GPIO_Pin = 0x00FFU;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    gpio.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOC, &gpio);
}

int main(void)
{
    uint8_t i;
    LEDInit();
    while (1)
    {
        for (i = 0; i < 8U; ++i)
        {
            GPIO_SetBits(GPIOC, 0x00FFU); /* 先全灭 */
            GPIO_ResetBits(GPIOC, (uint16_t)(1U << i));
            Delay_Ms(1000U);             /* 当前一盏保持约1秒 */
        }
    }
}
```

</details>

**代码阅读：** `1U<<i`每次把唯一的1移动到目标位置，ResetBits清该位；`for`完成8步，外层`while`不断重复；一次完整流水循环名义约8秒，实际略含函数执行开销。

</details>

### 5.4 SysTick定时器

<details>

<summary>展开本节详解</summary>

#### 5.4.1 SysTick定时器概述

SysTick是Cortex-M3内核中的24位向下计数定时器。它从重装值倒数到0，然后按使能状态继续重装计数；可轮询计数完成标志，也可产生SysTick异常。

**【重点·本芯片的时钟选择】** 在STM32F103中，CLKSOURCE=1选择HCLK，CLKSOURCE=0选择HCLK/8。若HCLK=72MHz，则后者为9MHz，每微秒有9个计数时钟。

**【基础补充】** “内核定时器”与“定时器TIM6”不是同一个外设；SysTick不需要像GPIO或TIM6一样先开启APB外设时钟。

#### 5.4.2 SysTick定时器寄存器

| 课件名称 / C成员 | 关键内容 | 使用规则 |
| --- | --- | --- |
| STK_CSR / `CTRL` | ENABLE bit0，TICKINT bit1，CLKSOURCE bit2，COUNTFLAG bit16 | 读CTRL会清COUNTFLAG |
| STK_LOAD / `LOAD` | 低24位RELOAD | 指定重装值，最大0xFFFFFF |
| STK_VAL / `VAL` | 当前计数值 | 写入任意值会清当前值，同时清COUNTFLAG |
| STK_CALRB / `CALIB` | NOREF、SKEW、TENMS | 校准信息；代码成员为CALIB，课件标题CALRB有拼写问题 |

- ENABLE=1：计数器工作；=0：停止。
- TICKINT=1：倒数到0产生SysTick异常请求；=0：可以只轮询标志而不进入ISR。
- COUNTFLAG=1：自上次读取CTRL以来曾数到0；它只说明“至少发生过”，不能累计多次溢出次数。
- NOREF说明参考时钟可用性；SKEW说明10ms校准值是否精确；TENMS与10ms校准计数有关。不能把CALIB当作普通延时重装寄存器。

**【重点·周期定时公式】** 设计数时钟为 $f_s$ ，一个重复计数周期需要 $N$ 个时钟：

$$
T=\frac{LOAD+1}{f_s},\qquad LOAD=f_sT-1=N-1
$$

| 变量 | 意义与单位 |
| --- | --- |
|  $f_s$  | SysTick计数时钟，Hz |
|  $T$  | 定时周期，s |
|  $N$  | 一个周期所需时钟数，无单位 |
| LOAD | 写入的重装寄存器值， $N-1$  |

**周期计算示例：HCLK=72MHz，选择HCLK/8。**

- 1μs： $N=9\times10^6\times10^{-6}=9$ ，LOAD=8。
- 1ms： $N=9000$ ，LOAD=8999。
- 1000ms： $N=9000000$ ，LOAD=8999999。

24位LOAD的最大重装值为 $2^{24}-1$ ，最大周期计数为 $2^{24}$ ：

$$
T_{max}=\frac{2^{24}}{f_s}
$$

9MHz时约1.864s；72MHz时约0.233s。超出单次范围应分段延时。计数为0的请求应直接返回，不能做无符号`0-1`；LOAD=0也不能正常产生一次1→0的完成标志。

**【基础核对·单次延时与重复周期】** 编程手册区分重复周期的`LOAD=N-1`与一次性倒计时的`LOAD=N`。课件每次启动后只等待一次归零，写`LOAD=9*i`、毫秒时写`LOAD=9000*i`，不能仅因没有减1就判错；它依赖72MHz及HCLK/8条件。下面的阻塞延时保留单次写法。示例中的`CTRL=0x01`表示ENABLE=1、TICKINT=0、CLKSOURCE=0。计算时要先说明采用哪种工作过程，并把软件执行开销另行考虑。[ST Cortex-M3编程手册](https://www.st.com/resource/en/programming_manual/cd00228163-stm32f10xxx20xxx21xxxl1xxxx-cortexm3-programming-manual-stmicroelectronics.pdf)

**CALIB补充：** 通用TENMS名称与10ms校准有关，但本器件RM0008列出的校准值9000在9MHz下对应1ms。实际延时应按选定时钟计算，不应看到CALIB就直接当作准确10ms使用。

#### 5.4.3 SysTick定时器应用

课件新增systick.c/systick.h，将流水灯原来的循环延时改为SysTick毫秒延时。以下讲解版统一使用Delay_Us、Delay_Ms，后续章节可复用。

**使用条件：** 工程已配置HCLK=72MHz；SysTick专供这些阻塞延时函数使用，没有同时运行RTOS节拍或另一份SysTick中断代码。若主频变化，需重新计算系数；函数调用、GPIO操作及中断会使实际间隔长于理论计数时间。

<details>

<summary>SysTick 延时声明 · delay.h</summary>

```c
/* delay.h */
#ifndef DELAY_H
#define DELAY_H
#include "stm32f10x.h"
void Delay_Us(uint32_t us);
void Delay_Ms(uint32_t ms);
#endif
```

</details>

<details>

<summary>SysTick 延时实现 · delay.c</summary>

```c
/* delay.c：HCLK=72MHz，SysTick时钟=HCLK/8=9MHz */
#include "delay.h"

static void SysTick_Wait(uint32_t ticks)
{
    SysTick->CTRL = 0U;          /* 先停止 */
    SysTick->LOAD = ticks;       /* 单次倒计时；本函数只接受正数 */
    SysTick->VAL = 0U;           /* 清当前值和COUNTFLAG */
    SysTick->CTRL = 0x01U;       /* HCLK/8，无中断，开始计数 */
    while ((SysTick->CTRL & (1U << 16)) == 0U)
    {
        /* 阻塞等待本次倒数结束 */
    }
    SysTick->CTRL = 0U;
    SysTick->VAL = 0U;
}

void Delay_Us(uint32_t us)
{
    while (us != 0U)
    {
        uint32_t part = (us > 1000U) ? 1000U : us;
        SysTick_Wait(part * 9U); /* 每段最多9000个时钟 */
        us -= part;
    }
}

void Delay_Ms(uint32_t ms)
{
    while (ms != 0U)
    {
        Delay_Us(1000U);
        --ms;
    }
}
```

</details>

**逐段解释：** 先关计数器防止配置期间计数；LOAD设置时长；VAL清旧状态；CTRL=1选择本例时钟并启动；轮询bit16判断完成；结束后关计数器。分段让乘法结果不超过24位范围；零延时直接不进入循环。

**【基础补充·阻塞与非阻塞】** 上面的等待期间CPU一直检查计数器，不能同时在主循环做其他任务；中断仍可能打断它。第9章的定时器中断由硬件计数，到时才执行服务程序，更适合一边显示一边计时。不要在主程序和ISR中同时调用这份延时函数，它们会相互改写同一个SysTick。

</details>

### 本章公式与易错点自检

<details>

<summary>展开本节详解</summary>

1. 能解释六个GPIO输出相关函数，并判断哪些改整端口、哪些只改指定引脚。
2. 能推导流水灯8个值，说明为何低电平点亮。
3. 能解释GPIO_Pin是掩码、GPIO_Speed不是闪烁频率。
4. 能从CTRL=0x01读出三项控制位；知道读CTRL会清COUNTFLAG。
5. 能根据HCLK及分频计算LOAD，并检查24位范围。
6. 若LED亮1秒、灭1秒，完整闪烁周期为2秒，频率为0.5Hz；不能把1秒保持时间当作完整周期。

</details>

---

[上一章](04-gpio.md) · [总目录](../README.md#章节目录) · [下一章](06-keys-and-buzzer.md)
