# 第09章 定时器与脉冲宽度调制

[总目录](../README.md#章节目录) · [知识点速查](#知识点速查) · [题型练习](#题型练习) · [课件详解](#课件详解)

## 知识点速查

| 知识点 | 一句话记法 |
| --- | --- |
| 基本、通用、高级 | TIM6/7 是基本定时器，可定时和触发 DAC。TIM2～5 是通用定时器，可输入捕获和输出 PWM。TIM1/8 是高级定时器，可做带死区的互补输出等 |
| `PSC`、`ARR`、`CNT`、`CCR` | PSC 定分频，ARR 定计数上限，CNT 是当前计数，CCR 是捕获或比较值 |
| 向上计数周期 | `T = (PSC + 1) × (ARR + 1) ÷ 定时器输入时钟`。PSC 和 ARR 都要加 1 |
| 定时器与 APB 时钟 | APB 不分频时，定时器时钟等于 PCLK。APB 分频大于 1 时，本章这些定时器的输入时钟是 PCLK 的两倍 |
| PWM | 固定时钟、PSC、ARR 时，改变 CCR 可以调脉宽和占空比。PWM1 向上计数时，`CNT < CCR` 是有效区间，实际输出高低由极性决定 |

**定时算例。** TIM6 输入为 72 MHz，PSC=1999、ARR=35999 时，`2000 × 36000 ÷ 72000000 = 1 s`。若每次到时只翻转一次引脚，则高低各保持 1 s，完整方波是 **2 s**。

## 题型练习

1. **单选**　下列哪一组属于基本定时器？

   A. TIM1/TIM8　B. TIM2/TIM3　C. TIM6/TIM7　D. TIM4/TIM5

2. **多选**　定时器更新周期与哪些量有关？按内部时钟、向上计数理解。

   A. 输入时钟　B. PSC　C. ARR　D. PWM 通道的 CCR

3. **判断**　每隔 1 秒翻转一次输出引脚，所得方波的周期也是 1 秒。

<details>

<summary>展开答案与理由</summary>

1. **C。** TIM6、TIM7 是基本定时器。
2. **ABC。** CCR 可改变 PWM 脉宽，时基周期由前三项决定。
3. **错。** 完整周期包括一段高和一段低，共 2 秒。

</details>

### 读一句代码

```c
TIM_SetCompare1(TIM3, 300U);
```

**参考注释**　将 TIM3 通道 1 的比较值设为 300。在 PWM 模式中，这会改变该通道的有效脉宽。

## 课件详解

点开对应小节，可以看课件中的解释、寄存器细节和原笔记里的完整示例。

### 目录

<details>

<summary>展开本章原目录</summary>

- [9.1 STM32F103定时器概述](#91-stm32f103定时器概述)
- [9.2 基本定时器](#92-基本定时器)
- [9.3 通用定时器](#93-通用定时器)
- [9.4 高级定时器](#94-高级定时器)
- [9.5 定时器相关库函数](#95-定时器相关库函数)
- [9.6 项目实例](#96-项目实例)
- [本章公式和代码复习清单](#本章公式和代码复习清单)
- [本章示例配套驱动](09-example-dependencies.md)

</details>

### 9.1 STM32F103定时器概述

<details>

<summary>展开本节详解</summary>

定时器用于定时、外部事件计数、脉宽/频率测量、PWM、编码器等。本课程STM32F103ZET6包含以下TIM资源。

| 特征 | 基本TIM6、TIM7 | 通用TIM2～TIM5 | 高级TIM1、TIM8 |
| --- | --- | --- | --- |
| 所挂总线 | APB1 | APB1 | APB2 |
| 预分频器 | 16位 | 16位 | 16位 |
| 计数器 | 16位 | 16位 | 16位 |
| 计数方向 | 向上 | 向上、向下、中央对齐 | 向上、向下、中央对齐 |
| 更新中断/DMA | 支持 | 支持 | 支持 |
| 捕获/比较通道 | 无 | 4个 | 最多4个 |
| PWM/输入捕获 | 无对应输出/捕获通道 | 支持 | 支持 |
| 编码器、霍尔接口 | 无 | 支持 | 支持 |
| 互补PWM、死区、刹车 | 无 | 无 | 支持 |
| 特别用途 | 可触发DAC | 通用测量与波形输出 | 电机等控制用途 |

**【易错点】** 基本定时器不能直接从某引脚输出PWM，因为它没有相应输出比较通道；想做PWM应选择具有该功能的通用或高级定时器。

</details>

### 9.2 基本定时器

<details>

<summary>展开本节详解</summary>

#### 9.2.1 基本定时器简介

TIM6、TIM7各有独立的16位自动重装载向上计数器，由各自的可编程预分频器驱动。可以独立配置与运行。

#### 9.2.2 基本定时器的主要特性

16位累加计数、16位可编程预分频、DAC触发同步电路、更新事件产生中断或DMA请求。计数器到达周期终点并重新计数，称为一次更新相关的过程；中断是对更新事件配置的一种处理方式。

#### 9.2.3 基本定时器的功能

**【重点·时基的三个寄存器】**

| 寄存器 | 含义 | 直观理解 |
| --- | --- | --- |
| PSC | 预分频寄存器 | 决定计数走多快 |
| CNT | 当前计数值 | 现在数到哪里 |
| ARR | 自动重装载寄存器 | 决定一轮数到哪里结束 |

基本定时器只使用内部时钟CK_INT，并向上计数：0、1、2、…、ARR，再回0。因为包含0，所以一轮有ARR+1次计数。

**【基础补充·先确定定时器输入时钟】**

$$
f_{TIM}=\begin{cases}
f_{PCLK}, & \text{APB分频系数}=1\\
2f_{PCLK}, & \text{APB分频系数}\ne1
\end{cases}
$$

常见配置：HCLK=72MHz，APB1分频2，则PCLK1=36MHz，而TIM2～TIM7的输入时钟为72MHz。 **PCLK1与TIMxCLK不是同一个量。** 课件将APB1时钟直接写作72MHz应纠正；代公式使用实际 $f_{TIM}$ 。[ST RM0008时钟规则](https://www.st.com/resource/en/reference_manual/cd00171190-stm32f101xx-stm32f102xx-and-stm32f103xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)

**【重点·公式推导】**

$$
f_{CNT}=\frac{f_{TIM}}{PSC+1}
$$

$$
T_{tick}=\frac{PSC+1}{f_{TIM}},\qquad
T_{update}=(ARR+1)T_{tick}=\frac{(PSC+1)(ARR+1)}{f_{TIM}}
$$

$$
f_{update}=\frac{f_{TIM}}{(PSC+1)(ARR+1)}
$$

 $f$ 用Hz， $T$ 用s；PSC和ARR是寄存器值。要分频7200倍，应写PSC=7199；要计数10000次，应写ARR=9999。

**【重点·16位范围】** PSC、ARR寄存器的数值范围为0～65535，PSC对应分频系数1～65536。计数器值从0开始；不要把“最多65536次计数”写成“寄存器可以写65536”。

**计算例1：课件1秒配置。**  $f_{TIM}=72$ MHz，PSC=2000−1=1999，ARR=36000−1=35999。

$$
f_{CNT}=\frac{72000000}{2000}=36000\ \mathrm{Hz}
$$

$$
T=\frac{36000\times2000}{72000000}=1\ \mathrm{s}
$$

**计算例2：先设定计数粒度。** 72MHz下，取PSC=7200−1，让 $f_{CNT}=10000$ Hz，即每次计数0.1ms。

| 目标时间 | 需要计数次数 | ARR写入值 |
| --- | --- | --- |
| 1ms | 10 | 9 |
| 10ms | 100 | 99 |
| 100ms | 1000 | 999 |
| 1s | 10000 | 9999 |

**反推步骤：** 先确定 $f_{TIM}$ →选择合适的分频系数→用 $N=T f_{CNT}$ 算计数次数→检查 $N$ 为整数且在范围内→写ARR=N−1。如果不能整除，要计算实际误差，不能只写理想时间。

**【基础补充·溢出时间和方波周期】** 中断每1秒进入一次，在ISR中每次翻转一个GPIO，那么GPIO高1秒、低1秒，完整方波周期是2秒，频率0.5Hz。

#### 9.2.4 基本定时器寄存器

| 寄存器 | 复习重点 |
| --- | --- |
| CR1 | 基本控制，如计数使能CEN、ARR预装载ARPE |
| CR2 | 主模式/触发输出等控制 |
| DIER | 更新中断、DMA等请求使能 |
| SR | 状态标志，例如更新标志UIF |
| EGR | 软件产生更新事件等，UG可使配置更新 |
| CNT | 当前计数值 |
| PSC | 预分频值 |
| ARR | 自动重装载值 |

**【基础补充·更新事件与更新中断】** 更新事件可使预装载值转入实际工作寄存器，即使没有打开更新中断也会发生。要进入ISR，还需对应中断源使能、NVIC使能且未被屏蔽。初始化函数可能产生UG，因此开启中断前清一次UIF可避免一启动就先进入一次中断。

</details>

### 9.3 通用定时器

<details>

<summary>展开本节详解</summary>

#### 9.3.1 通用定时器简介

TIM2～TIM5仍以16位计数器、PSC和ARR构成时基，同时增加捕获/比较通道，可测输入信号或产生输出波形。各定时器独立，也可通过内部触发等方式配合。

#### 9.3.2 通用定时器主要功能

向上、向下、中央对齐计数；4个独立捕获/比较通道；输入捕获、输出比较、PWM、单脉冲；外部时钟、同步、编码器、霍尔传感器接口；更新、触发、捕获/比较等中断或DMA请求。

**【重点】** 同一个定时器的多个通道共享PSC、ARR和CNT，因此普通PWM配置下通常共享频率；各通道有自己的CCR，可以分别设置占空比。

#### 9.3.3 通用定时器功能描述

**计数模式。** 向上：0→ARR→0；向下：ARR→0→ARR；中央对齐：向上和向下往返计数。

| 模式 | 一个完整计数往复/周期 | 适用说明 |
| --- | --- | --- |
| 向上/向下边沿对齐 | ARR+1个计数时钟 | 本章一秒定时与PWM示例使用 |
| 中央对齐 | 完整往返通常为2×ARR个计数时钟，ARR>0 | 不能直接套用边沿对齐PWM公式 |

**【基础补充】** 中央对齐PWM的完整波形周期按往返过程计算， $T=2ARR/f_{CNT}$ ；更新事件可能在上、下边界发生，更新中断频率还要结合更新控制配置判断。

**时钟选择。** 内部时钟CK_INT；外部时钟模式1使用TIx；外部时钟模式2使用ETR；内部触发ITRx可用于定时器之间连接。内部时钟定时与外部脉冲计数使用了相似的计数器，但计数脉冲的来源不同。

**捕获/比较通道。** CCR1～CCR4分别对应四个通道。输入捕获在指定输入边沿把CNT值记录到CCR；输出比较把CNT与CCR比较，在匹配或指定区间改变输出状态。输入部分还涉及滤波、极性与预分频。

#### 9.3.4 通用定时器工作模式

课件列出输入捕获、PWM输入、强置输出、输出比较、PWM五种模式。

| 模式 | 应理解的核心动作 |
| --- | --- |
| 输入捕获 | 输入边沿到来时记录CNT，用来求周期、脉宽等 |
| PWM输入 | 配合捕获通道测输入PWM的周期和有效脉宽 |
| 强置输出 | 强制内部输出参考为有效或无效，最终电平还受极性影响 |
| 输出比较 | 比较CNT与CCR，在比较条件下设置、清除或翻转输出 |
| PWM输出 | 周期性比较CNT与CCR，形成固定周期、可变脉宽波形 |

**【基础补充·输入捕获计算】** 计数时钟为1MHz时，一次计数为1μs；若相邻同类边沿的捕获计数相差1500，且已正确处理计数器回绕，则周期1.5ms、频率约666.7Hz。上升沿到下降沿的间隔可用于计算高电平宽度。跨回绕时需按ARR+1处理，不能直接用无符号大减法；若发生多次溢出，还需记录溢出次数。

#### PWM的原理、公式和电平

PWM是Pulse Width Modulation，脉冲宽度调制。可用数字电路、普通I/O软件模拟或定时器外设产生；本课程采用TIM外设输出。

**【重点·向上计数，PWM1，输出高极性】** CNT从0数到ARR；CNT<CCR时输出高，CNT≥CCR时输出低。令 $N=ARR+1$ ：

$$
f_{PWM}=\frac{f_{TIM}}{(PSC+1)(ARR+1)},\qquad
D_H=\frac{CCR}{ARR+1}
$$

$$
t_H=\frac{CCR(PSC+1)}{f_{TIM}},\qquad
t_L=\frac{(ARR+1-CCR)(PSC+1)}{f_{TIM}}
$$

以上占空比公式适用于 $0\le CCR\le ARR+1$ 及所述模式/极性； $D_H$ 为物理引脚高电平占整个周期的比例，乘100%才写成百分数。

| CNT与CCR关系 | PWM1、高极性 | PWM1、低极性 | PWM2、高极性 | PWM2、低极性 |
| --- | --- | --- | --- | --- |
| CNT<CCR | 高 | 低 | 低 | 高 |
| CNT≥CCR | 低 | 高 | 高 | 低 |

**【重点·本板LED是低电平亮】** PWM1+低极性时，LED导通比例为 $CCR/(ARR+1)$ ；PWM1+高极性时，LED导通比例为 $1-CCR/(ARR+1)$ 。先判断实际高低电平，再判断负载亮灭，不能只凭“CCR增大”就认定更亮。

**计算例：** 72MHz，PSC=71，ARR=999，CCR=250。

1.  $f_{CNT}=72MHz/72=1MHz$ 。
2. 周期 $1000/1MHz=1ms$ ，PWM频率1kHz。
3. PWM1高极性下，高电平250μs，低电平750μs，高电平占空比25%。
4. 若改为低极性，低电平250μs；对于本板LED，导通比例25%。

**【基础补充·边界】** CCR=0时PWM1有效时间为0；CCR=ARR+1时整周期有效。CCR=ARR仍留有1个计数时钟的另一状态，不能直接当作100%。当ARR=65535时，ARR+1=65536超出16位CCR，不能直接写65536，可改选周期或使用合适的强制输出控制。

**【基础补充·平均值】** 对于0与 $V_H$ 之间的理想PWM，平均电压 $V_{avg}=D_HV_H$ 。这表示时间平均值，原始引脚仍是脉冲；LED感知亮度也不保证与占空比严格线性。占空比步进约为 $1/(ARR+1)$ ，ARR越大可分配的脉宽等级越多，但频率也会受影响。

</details>

### 9.4 高级定时器

<details>

<summary>展开本节详解</summary>

#### 9.4.1 高级定时器简介

TIM1、TIM8包含16位计数器和可编程预分频器，具备捕获、比较、PWM等功能，并增加用于更复杂控制的结构。它们与通用定时器独立，可进行同步配置。

#### 9.4.2 高级定时器特性

- 互补PWM输出：提供相关联的主输出和互补输出。
- 死区时间：切换时插入一段间隔，避免一对功率开关同时导通。
- 刹车输入：故障条件下使输出进入预定状态。
- 重复计数器：可在一定数量的计数周期后产生更新相关操作。
- 还支持多通道、同步、编码器/霍尔接口、中断及DMA等。

#### 9.4.3 高级定时器结构

核心仍为PSC、CNT、ARR，主要增加BRK、DTG等结构。课件未把死区编码等细节作为展开重点，本章掌握这些附加模块的作用。

**【基础补充·使用差异】** TIM1/TIM8输出还涉及主输出使能MOE，标准库中常用`TIM_CtrlPWMOutputs(TIMx, ENABLE)`；通用定时器TIM3示例不需要这一步。不能把TIM3代码只改成TIM1就默认所有设置完全相同。

</details>

### 9.5 定时器相关库函数

<details>

<summary>展开本节详解</summary>

#### 9.5.1 函数TIM_DeInit

`void TIM_DeInit(TIM_TypeDef *TIMx);`将指定定时器恢复默认配置。它会重置该定时器已有功能，不能在共享定时器其他通道正在工作时不加判断调用；恢复默认也不能代替开启外设时钟。

#### 9.5.2 函数TIM_TimeBaseInit

`TIM_TimeBaseInit(TIMx, &base)`初始化时基。

| TIM_TimeBaseInitTypeDef成员 | 对应含义 | 典型值 |
| --- | --- | --- |
| TIM_Prescaler | PSC寄存器值 | 想分频2000，写1999 |
| TIM_Period | ARR寄存器值 | 想计数36000次，写35999 |
| TIM_CounterMode | 向上、向下、中央对齐 | TIM_CounterMode_Up |
| TIM_ClockDivision | 与数字滤波/死区采样时钟TDTS有关 | TIM_CKD_DIV1 |
| TIM_RepetitionCounter | 高级定时器重复计数配置 | 基本/通用示例置0 |

**【易错点】** `TIM_ClockDivision`不是代替PSC的计数分频器，不能把DIV2又直接乘进普通定时周期公式。用`TIM_TimeBaseStructInit(&base)`给所有成员默认值，再覆盖需要的字段，避免结构体含未初始化数据。

#### 9.5.3 函数TIM_OC1Init

`TIM_OC1Init(TIMx, &oc)`配置通道1输出比较。常用字段：

| 成员 | 用途 |
| --- | --- |
| TIM_OCMode | Timing、Active、Inactive、Toggle、PWM1、PWM2等模式 |
| TIM_OutputState | 通道输出是否使能 |
| TIM_Pulse | 初始CCR值，计数值而非百分数 |
| TIM_OCPolarity | High/Low，输出有效极性 |

先用`TIM_OCStructInit(&oc)`初始化结构体。`TIM_OCMode_Timing`等宏的大小写应以库头文件为准，不能照抄课件排版造成的拼写错误。

#### 9.5.4 函数TIM_OC2Init

`TIM_OC2Init(TIMx, &oc)`配置通道2，字段含义与通道1相同。课件该项功能描述仍写“通道1”为复制文字错误，实际OC2操作通道2。

#### 9.5.5 函数TIM_Cmd

`TIM_Cmd(TIMx, ENABLE/DISABLE)`控制计数器运行。使能定时器不等于自动配置了GPIO、输出通道或NVIC。

#### 9.5.6 函数TIM_ITConfig

实际函数名为`TIM_ITConfig(TIMx, source, ENABLE/DISABLE)`。常见source有Update、CC1～CC4、Trigger；要根据定时器实际支持的功能选择。TIM6一秒计时使用`TIM_IT_Update`。

#### 9.5.7 函数TIM_OC1PreloadConfig

`TIM_OC1PreloadConfig(TIMx, TIM_OCPreload_Enable)`开启CCR1预装载；通道2对应`TIM_OC2PreloadConfig`。开启后写入的比较值通常在更新时进入实际比较，减少一个周期中途改值造成的异常脉宽。

**【基础补充】** ARR预装载通过`TIM_ARRPreloadConfig`控制；PSC也有更新加载机制。写寄存器值与新值立即在当前周期生效，不应视为同一件事。

#### 9.5.8 函数TIM_ClearFlag

`TIM_ClearFlag(TIMx, TIM_FLAG_Update)`清更新标志。ISR中常配合`TIM_GetITStatus`与`TIM_ClearITPendingBit`使用。

**【易错点】** TIM状态标志的清除语义与EXTI的PR不同，不要把“EXTI写1清除”直接套到TIM。使用对应库函数，避免读改写误处理其他标志。

#### 9.5.9 函数TIM_SetCompare1

`TIM_SetCompare1(TIMx, value)`修改CCR1；通道2用`TIM_SetCompare2`。只改变CCR通常改变脉宽/占空比；改变PSC或ARR还会改变周期，不能混为一谈。

</details>

### 9.6 项目实例

<details>

<summary>展开本节详解</summary>

#### 9.6.1 定时器项目

**目标：TIM6每秒产生一次更新中断，使时、分、秒自动递增，主程序继续扫描显示，按键仍可调时。** 条件为TIM6输入时钟72MHz。

新增timer_review.c/timer_review.h，工程加入`stm32f10x_tim.c`。配套的延时、数码管显示和按键驱动已放入[本章示例配套驱动](09-example-dependencies.md)，按该附录加入工程。以下为整理后的讲解代码，ISR不要再在别处重复定义。

<details>

<summary>计时驱动接口 · timer_review.h</summary>

```c
/* timer_review.h */
#ifndef TIMER_REVIEW_H
#define TIMER_REVIEW_H
#include "stm32f10x.h"
void TIM6Init(void);
#endif
```

</details>

<details>

<summary>TIM6 初始化与秒进位 · timer_review.c</summary>

```c
/* timer_review.c */
#include "timer_review.h"
extern volatile uint8_t hour, minute, second;

void TIM6Init(void)
{
    TIM_TimeBaseInitTypeDef base;
    NVIC_InitTypeDef nvic;

    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM6, ENABLE);
    TIM_TimeBaseStructInit(&base);
    base.TIM_Prescaler = 2000U - 1U;
    base.TIM_Period = 36000U - 1U;
    base.TIM_CounterMode = TIM_CounterMode_Up;
    base.TIM_ClockDivision = TIM_CKD_DIV1;
    base.TIM_RepetitionCounter = 0U;
    TIM_TimeBaseInit(TIM6, &base);

    TIM_SetCounter(TIM6, 0U);
    TIM_ClearITPendingBit(TIM6, TIM_IT_Update); /* 清初始化产生的UIF */
    TIM_ITConfig(TIM6, TIM_IT_Update, ENABLE);

    nvic.NVIC_IRQChannel = TIM6_IRQn;
    nvic.NVIC_IRQChannelPreemptionPriority = 1U;
    nvic.NVIC_IRQChannelSubPriority = 0U;
    nvic.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&nvic); /* 分组2由main统一设置 */
    TIM_Cmd(TIM6, ENABLE); /* 各项配置完成后再开始计数 */
}

void TIM6_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM6, TIM_IT_Update) != RESET)
    {
        TIM_ClearITPendingBit(TIM6, TIM_IT_Update);
        ++second;
        if (second >= 60U)
        {
            second = 0U;
            ++minute;
            if (minute >= 60U)
            {
                minute = 0U;
                ++hour;
                if (hour >= 24U) hour = 0U;
            }
        }
    }
}
```

</details>

**逐段解释：** APB1函数开TIM6时钟；PSC将72MHz分为36kHz；ARR使36000次计数形成1秒；清UIF避免初始假触发；TIM_ITConfig打开外设请求，NVIC_Init打开CPU接收通道；TIM_Cmd真正启动计数。ISR先确认来源并清标志，再按60秒、60分、24小时进位。

<details>

<summary>计时与动态显示示例 · main.c</summary>

```c
/* 计时项目的main.c，替换第8章的main.c */
#include "dsgshow.h"
#include "exti_keys.h"
#include "timer_review.h"

volatile uint8_t hour = 23U, minute = 59U, second = 55U;

int main(void)
{
    uint8_t h, m, s;
    uint32_t saved;
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    DsgShowInit();
    EXTIInit();
    TIM6Init();
    while (1)
    {
        saved = __get_PRIMASK();
        __disable_irq();
        h = hour; m = minute; s = second;
        __set_PRIMASK(saved);
        DsgShowTime(h, m, s);
        Key_AdjustTask();
    }
}
```

</details>

**【重点·显示与计时分离】** 主循环不断显示，中断每秒修改变量；不能用`Delay_Ms(1000)`阻塞动态扫描来代替这个结构。配套按键驱动更新时间使用短临界区，避免与TIM6的进位过程交错；主循环取快照避免把23:59:59与00:00:00的部分字段混在一帧中。

**【基础补充】** 更新标志不是无限深的计数队列；若长时间屏蔽中断，多个溢出可能只保留一个挂起状态，导致丢秒。所谓“一秒定时”还依赖实际时钟频率和晶振误差，代码并不自动保证钟表级长期精度。

#### 9.6.2 PWM项目

**课件硬件与参数：** PC6、PC7连接L7、L8；TIM3完整重映射后对应CH1、CH2；ARR=900−1=899，PSC=1−1=0。假设TIM3输入时钟72MHz：

$$
f_{PWM}=\frac{72000000}{1\times900}=80000\ \mathrm{Hz}=80\ \mathrm{kHz}
$$

比较值300对应有效时间比例 $300/900=1/3$ ，即约33.3%， **不是300%** 。课件主循环把比较值逐渐增减形成呼吸效果；文字写到300反向，截图条件写`Duty>300`，理解时注意边界差别。

**【讲解示例】** 保留课件端口、TIM3及ARR/PSC参数，明确使用PWM1+低极性，适配低电平亮LED。本例把比较值在0～900之间往返，演示从全灭到全亮的完整范围；这是说明边界后的教学改写，不把0～900冒充课件原来的0～300。

<details>

<summary>双通道呼吸灯完整示例 · main.c</summary>

```c
/* 独立PWM项目main.c；与前面的计时项目分别编译 */
#include "stm32f10x.h"
#include "delay.h"

static void TIM3_PWMInit(void)
{
    GPIO_InitTypeDef gpio;
    TIM_TimeBaseInitTypeDef base;
    TIM_OCInitTypeDef oc;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC |
                          RCC_APB2Periph_AFIO, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);
    GPIO_SetBits(GPIOC, GPIO_Pin_6 | GPIO_Pin_7);
    GPIO_PinRemapConfig(GPIO_FullRemap_TIM3, ENABLE);
    gpio.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    gpio.GPIO_Speed = GPIO_Speed_50MHz;
    gpio.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_Init(GPIOC, &gpio);

    TIM_TimeBaseStructInit(&base);
    base.TIM_Prescaler = 0U;
    base.TIM_Period = 899U;
    base.TIM_CounterMode = TIM_CounterMode_Up;
    base.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseInit(TIM3, &base);

    TIM_OCStructInit(&oc);
    oc.TIM_OCMode = TIM_OCMode_PWM1;
    oc.TIM_OutputState = TIM_OutputState_Enable;
    oc.TIM_OCPolarity = TIM_OCPolarity_Low; /* 有效区间输出低，LED亮 */
    oc.TIM_Pulse = 0U;                    /* 初始有效时间为0 */
    TIM_OC1Init(TIM3, &oc);
    TIM_OC2Init(TIM3, &oc);
    TIM_OC1PreloadConfig(TIM3, TIM_OCPreload_Enable);
    TIM_OC2PreloadConfig(TIM3, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM3, ENABLE);
    TIM_GenerateEvent(TIM3, TIM_EventSource_Update);
    TIM_SetCounter(TIM3, 0U);
    TIM_Cmd(TIM3, ENABLE);
}

int main(void)
{
    int32_t duty = 0;
    int32_t step = 1;
    TIM3_PWMInit();
    while (1)
    {
        TIM_SetCompare1(TIM3, (uint16_t)duty);
        TIM_SetCompare2(TIM3, (uint16_t)duty);
        Delay_Ms(5U); /* 调整亮度的节奏，不是PWM单个周期 */

        duty += step;
        if (duty >= 900)
        {
            duty = 900;
            step = -1;
        }
        else if (duty <= 0)
        {
            duty = 0;
            step = 1;
        }
    }
}
```

</details>

**代码重点：** GPIO使用复用推挽；AFIO负责重映射；TIM_TimeBaseInit确定周期；OC1/OC2配置两通道的模式、极性、初值；预装载让新CCR在更新时统一生效；主循环仅改变CCR。duty是计数值，step决定增减方向；有符号变量和边界检查避免减到0后无符号下溢。

5ms每次改变1个比较计数，从0到900约需4.5秒；一轮变亮再变暗约9秒，另有代码开销。PWM本身仍为80kHz，所以“PWM频率”与“呼吸变化速度”相差多个数量级。

**排错顺序：** 先查GPIO/AFIO/TIM3时钟→PC6/PC7模式→FullRemap→PSC/ARR→输出通道使能→PWM模式和极性→CCR范围→TIM_Cmd。只会调用TIM_Cmd而前面配置缺项，不能得到正确PWM。

</details>

### 本章公式和代码复习清单

<details>

<summary>展开本节详解</summary>

| 必会内容 | 检查方法 |
| --- | --- |
| TIM1～TIM8分类 | 写出总线、计数方向、是否有输出通道 |
| 定时器时钟 | 先查APB分频，再决定是否乘2 |
| 定时周期 |  $T=(PSC+1)(ARR+1)/f_{TIM}$ ，注明边沿对齐条件 |
| 一秒计算 | 72MHz、PSC1999、ARR35999→1秒 |
| PWM频率 | PSC与ARR决定；本例0、899→80kHz |
| PWM占空比 | CCR/(ARR+1)先表示有效比例，再按模式/极性转换成高低电平比例 |
| 中断配置 | 外设中断使能与NVIC使能都需要；启动前清残留标志 |
| 预装载 | 写入值与当前周期实际工作值可能不同 |
| 动态显示+计时 | 扫描在主循环，秒进位在TIM6 ISR |
| 低电平LED | 分清高电平占空比、低电平占空比、LED导通比例 |

**自测答案示例：** 72MHz下PSC=71、ARR=9999时周期10ms；同配置每次中断翻转GPIO，方波周期20ms、频率50Hz。PWM1高极性、ARR=999、CCR=250时高电平占25%，本板低电平LED导通75%；改为低极性后LED导通25%。

</details>

---

[上一章](08-interrupts.md) · [总目录](../README.md#章节目录) · [下一章](10-usart.md)
