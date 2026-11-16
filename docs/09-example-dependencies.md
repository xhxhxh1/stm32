# 第09章示例配套驱动

[返回第九章笔记](09-timers-and-pwm.md) · [返回目录](../README.md)

本附录收录第九章示例直接依赖的延时、数码管显示和按键驱动，沿用已整理并检查过的教学代码。它们原先用于前面章节的项目，这里只作为第九章计时与PWM示例的配套文件。

## 使用条件与文件组合

- 芯片及库：STM32F103ZET6、STM32F10x标准外设库；这些示例使用标准库接口。
- 延时：HCLK = 72 MHz，SysTick = HCLK / 8 = 9 MHz。延时函数占用SysTick，不能与其他SysTick用途同时使用。
- 数码管：段线PG0～PG7，位选PE10～PE15；按键PE0、PE1、PE2低电平按下。
- 计时项目：加入本附录的6个文件，以及第九章的`timer_review.h`、`timer_review.c`和计时项目`main.c`。主程序统一设置`NVIC_PriorityGroup_2`，并定义`hour`、`minute`、`second`。
- PWM项目：只需本附录的`delay.h`、`delay.c`，配合第九章独立PWM项目的`main.c`。PC6、PC7使用TIM3完整重映射，LED低电平亮。
- 每个项目只保留一个`main`；`TIM6_IRQHandler`及各EXTI中断处理函数也各保留一份，避免重复定义。
- 工程仍需标准库、CMSIS、启动文件、时钟初始化和相应头文件路径；本资料不是完整的Keil工程。

下面每个代码块按标题分别保存为对应的`.h`或`.c`文件。主要示例此前通过Cortex-M3编译器及实际标准库头文件的语法和库接口检查，未进行完整工程链接、烧录或实机验证。

## delay.h

```c
/* delay.h */
#ifndef DELAY_H
#define DELAY_H
#include "stm32f10x.h"
void Delay_Us(uint32_t us);
void Delay_Ms(uint32_t ms);
#endif
```

## delay.c

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

## dsgshow.h

```c
/* dsgshow.h */
#ifndef DSGSHOW_H
#define DSGSHOW_H
#include "stm32f10x.h"
void DsgShowInit(void);
void DsgScanOnce(const uint8_t digits[6], uint8_t dot_mask);
void DsgShowTime(uint8_t hour, uint8_t minute, uint8_t second);
#endif
```

## dsgshow.c

```c
/* dsgshow.c：与本章后面的DsgShowTime函数放在同一源文件 */
#include "dsgshow.h"
#include "delay.h" /* 第5章，HCLK=72MHz */

#define DIGIT_ALL 0xFC00U /* PE10～PE15 */
#define SEG_ALL   0x00FFU /* PG0～PG7 */

static const uint8_t segment[10] = {
    0xC0, 0xF9, 0xA4, 0xB0, 0x99,
    0x92, 0x82, 0xF8, 0x80, 0x90
};
static const uint16_t digit_pin[6] = {
    0x8000, 0x4000, 0x2000, 0x1000, 0x0800, 0x0400
};

void DsgShowInit(void)
{
    GPIO_InitTypeDef gpio;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOE |
                          RCC_APB2Periph_GPIOG, ENABLE);
    GPIO_SetBits(GPIOE, DIGIT_ALL); /* 全部位选关闭 */
    GPIO_SetBits(GPIOG, SEG_ALL);   /* 全部段熄灭 */

    gpio.GPIO_Pin = DIGIT_ALL;
    gpio.GPIO_Mode = GPIO_Mode_Out_PP;
    gpio.GPIO_Speed = GPIO_Speed_10MHz;
    GPIO_Init(GPIOE, &gpio);

    gpio.GPIO_Pin = SEG_ALL;
    GPIO_Init(GPIOG, &gpio);
}

void DsgScanOnce(const uint8_t digits[6], uint8_t dot_mask)
{
    uint8_t i;
    uint8_t code;
    for (i = 0; i < 6U; ++i)
    {
        GPIO_SetBits(GPIOE, DIGIT_ALL); /* 1.关所有位 */
        code = (digits[i] < 10U) ? segment[digits[i]] : 0xFFU;
        if ((dot_mask & (1U << i)) != 0U)
            code &= 0x7FU;             /* 第i位小数点亮 */

        GPIO_SetBits(GPIOG, SEG_ALL);  /* 2.先让全部段为1 */
        GPIO_ResetBits(GPIOG, (uint16_t)((~code) & 0xFFU));
        GPIO_ResetBits(GPIOE, digit_pin[i]); /* 3.选当前位 */
        Delay_Ms(2U);                        /* 4.保持 */
        GPIO_SetBits(GPIOE, digit_pin[i]);  /* 5.关当前位 */
    }
}

/* 添加到dsgshow.c */
void DsgShowTime(uint8_t hour, uint8_t minute, uint8_t second)
{
    uint8_t digits[6];
    digits[0] = hour / 10U;
    digits[1] = hour % 10U;
    digits[2] = minute / 10U;
    digits[3] = minute % 10U;
    digits[4] = second / 10U;
    digits[5] = second % 10U;
    DsgScanOnce(digits, (1U << 1) | (1U << 3));
}
```

## exti_keys.h

```c
/* exti_keys.h */
#ifndef EXTI_KEYS_H
#define EXTI_KEYS_H
#include "stm32f10x.h"
extern volatile uint8_t hour, minute, second;
void EXTIInit(void);
void Key_AdjustTask(void);
#endif
```

## exti_keys.c

```c
/* exti_keys.c：第一部分 */
#include "exti_keys.h"
static volatile uint8_t key_requests = 0U;

void EXTIInit(void)
{
    GPIO_InitTypeDef gpio;
    EXTI_InitTypeDef exti;
    NVIC_InitTypeDef nvic;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOE |
                          RCC_APB2Periph_AFIO, ENABLE);
    gpio.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2;
    gpio.GPIO_Mode = GPIO_Mode_IPU;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    GPIO_Init(GPIOE, &gpio);

    GPIO_EXTILineConfig(GPIO_PortSourceGPIOE, GPIO_PinSource0);
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOE, GPIO_PinSource1);
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOE, GPIO_PinSource2);

    exti.EXTI_Line = EXTI_Line0 | EXTI_Line1 | EXTI_Line2;
    exti.EXTI_Mode = EXTI_Mode_Interrupt;
    exti.EXTI_Trigger = EXTI_Trigger_Falling;
    exti.EXTI_LineCmd = ENABLE;
    EXTI_Init(&exti);
    EXTI_ClearITPendingBit(EXTI_Line0 | EXTI_Line1 | EXTI_Line2);

    /* main统一设置分组2，本函数只设置各通道 */
    nvic.NVIC_IRQChannelPreemptionPriority = 2U;
    nvic.NVIC_IRQChannelCmd = ENABLE;
    nvic.NVIC_IRQChannel = EXTI0_IRQn;
    nvic.NVIC_IRQChannelSubPriority = 0U;
    NVIC_Init(&nvic);
    nvic.NVIC_IRQChannel = EXTI1_IRQn;
    nvic.NVIC_IRQChannelSubPriority = 1U;
    NVIC_Init(&nvic);
    nvic.NVIC_IRQChannel = EXTI2_IRQn;
    nvic.NVIC_IRQChannelSubPriority = 2U;
    NVIC_Init(&nvic);
}

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line0);
        key_requests |= 0x01U;
    }
}
void EXTI1_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line1) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line1);
        key_requests |= 0x02U;
    }
}
void EXTI2_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line2) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line2);
        key_requests |= 0x04U;
    }
}

/* 继续放入exti_keys.c */
static void AdjustOne(uint8_t index)
{
    uint32_t saved = __get_PRIMASK();
    __disable_irq();
    if (index == 0U)
        hour = (uint8_t)((hour + 1U) % 24U);
    else if (index == 1U)
        minute = (uint8_t)((minute + 1U) % 60U);
    else
        second = (uint8_t)((second + 1U) % 60U);
    __set_PRIMASK(saved);
}

void Key_AdjustTask(void)
{
    static uint8_t pending = 0U, armed = 0x07U;
    static uint8_t low_count[3] = {0U, 0U, 0U};
    static uint8_t high_count[3] = {0U, 0U, 0U};
    uint8_t i, bit, events;
    uint16_t input;
    uint32_t saved = __get_PRIMASK();

    __disable_irq();
    events = key_requests;
    key_requests = 0U;
    __set_PRIMASK(saved);
    pending |= events;
    input = GPIO_ReadInputData(GPIOE);

    for (i = 0U; i < 3U; ++i)
    {
        bit = (uint8_t)(1U << i);
        if ((input & bit) == 0U) /* 当前读到按下 */
        {
            high_count[i] = 0U;
            if (((pending & bit) != 0U) && ((armed & bit) != 0U))
            {
                if (++low_count[i] >= 2U)
                {
                    AdjustOne(i);
                    armed &= (uint8_t)~bit;
                    pending &= (uint8_t)~bit;
                    low_count[i] = 0U;
                }
            }
            else if ((armed & bit) == 0U)
                pending &= (uint8_t)~bit; /* 按住/抖动不重复计数 */
        }
        else /* 当前读到松开 */
        {
            low_count[i] = 0U;
            if (high_count[i] < 2U) ++high_count[i];
            if (high_count[i] >= 2U)
            {
                armed |= bit;
                pending &= (uint8_t)~bit;
            }
        }
    }
}
```
