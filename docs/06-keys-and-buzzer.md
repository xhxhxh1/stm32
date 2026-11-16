# 第06章 按键输入与蜂鸣器

本章复习目标：区分GPIO输入/输出读取函数，掌握接地按键的有效电平、消抖和单次按下处理，理解无源蜂鸣器的频率控制。课件引脚为PE0～PE3接KEY1～KEY4，PC8控制PNP蜂鸣器。

## 目录

- [6.1 GPIO输入库函数](#61-gpio输入库函数)
- [6.2 项目分析](#62-项目分析)
- [6.3 项目实施](#63-项目实施)

## 6.1 GPIO输入库函数

### 6.1.1 函数GPIO_ReadInputDataBit

```c
uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

读取一个指定引脚的输入状态，结果为0或1，来自输入数据寄存器IDR。

```c
if (GPIO_ReadInputDataBit(GPIOE, GPIO_Pin_0) == Bit_RESET)
{
    /* 在上拉输入、按键另一端接地的条件下，KEY1当前为按下电平 */
}
```

 **【易错点】** 单个位读取函数使用一个引脚掩码。不要给它传多个引脚后，再假设会返回多个按键的位图；需要位图时用整个端口读取函数。

### 6.1.2 函数GPIO_ReadInputData

```c
uint16_t GPIO_ReadInputData(GPIO_TypeDef *GPIOx);
```

读取整个端口的输入数据。检测PE0～PE3可先取低4位，再把低有效转换成“1表示按下”的位图：

```c
uint16_t input = GPIO_ReadInputData(GPIOE);
uint8_t pressed = (uint8_t)((~input) & 0x000FU);
/* pressed bit0～bit3分别对应KEY1～KEY4；该位为1表示按下 */
```

若PE3…PE0为`1101`，表示PE1为0，即KEY2按下；取反并掩码后为`0010`。`&0x000F`避免把其他引脚的反相结果也当作按键。

### 6.1.3 函数GPIO_ReadOutputDataBit

```c
uint8_t GPIO_ReadOutputDataBit(GPIO_TypeDef *GPIOx, uint16_t GPIO_Pin);
```

读取一个输出数据位，来自ODR，可用于了解程序最后写入的输出状态。它不直接代表外部输入，也不能替代按键输入读取。

### 6.1.4 函数GPIO_ReadOutputData

```c
uint16_t GPIO_ReadOutputData(GPIO_TypeDef *GPIOx);
```

读取整个端口的输出数据寄存器。

| 函数 | 数据来源 | 范围 | 典型用途 |
| --- | --- | --- | --- |
| ReadInputDataBit | IDR | 一个输入位 | 判断某按键电平 |
| ReadInputData | IDR | 整个16位端口 | 按掩码提取多个输入 |
| ReadOutputDataBit | ODR | 一个输出位 | 查询某个输出锁存值 |
| ReadOutputData | ODR | 整个16位端口 | 读取输出配置的数据状态 |

 **【重点】** “输入/输出”决定读IDR还是ODR；“Bit/Data”决定读一个引脚还是整个端口。

## 6.2 项目分析

### 按键连接与输入控制方式

KEY1～KEY4分别接PE0～PE3，按键另一端接地。将引脚设为`GPIO_Mode_IPU`：松开时为1，按下时为0。

 **【基础补充·轮询】** 主循环不断读取按键状态，称为轮询。它容易理解，但如果主循环正在长时间延时或发声，就可能错过短促按键；第8章会使用外部中断捕获边沿。

 **【基础补充·消抖】** 机械触点在按下和松开瞬间可能在0、1之间反复跳变，直接把每次变化都算一次按键，会造成一次按下触发多次。

常见软件处理步骤：检测有效电平→等待一段消抖时间→再次读取确认→产生一次按键事件→等稳定松开后允许下一次事件。本章讲解示例选20ms作演示参数，实际需根据按键和系统响应要求调整。

 **电平检测与事件检测的区别：**

- 电平检测：只要按住就持续执行，适合“按住时工作”。
- 单次事件：从松开到按下只执行一次，适合“按一下切换一次”。
- 长按连发：还需要明确首次等待和重复间隔，本示例不做连发。

### 蜂鸣器原理与频率

课件板使用无源蜂鸣器BUZ1，PC8通过PNP三极管驱动。PC8=0时导通，PC8=1时关闭。通过高低电平交替形成周期性驱动。

 **【基础补充】** 有源蜂鸣器内部有振荡电路，合适的直流供电即可发声；无源蜂鸣器需要外部周期波形。这里“有源/无源”主要区分是否自带振荡电路，不是有无供电。

 **【重点·声音频率的计算】**

$$
T=t_H+t_L,\qquad f=\frac{1}{T}
$$

若高低电平各持续 $t$ 微秒：

$$
f=\frac{10^6}{2t}\ \mathrm{Hz}
$$

| 每半周期延时 | 完整周期 | 名义频率 |
| --- | --- | --- |
| 500μs | 1000μs=1ms | 1kHz |
| 250μs | 500μs=0.5ms | 2kHz |

频率较高通常音调较高；响多久由重复周期数决定。发声持续时间为 $t_{sound}=N/f$ 。例如1kHz波形重复200次，名义持续200ms。软件延时、函数和GPIO操作存在开销，实际频率应以波形测量为准。

## 6.3 项目实施

课件建立beepkey.c/beepkey.h，提供`BeepInit()`、`KeyInit()`、延时、`sound1()`、`sound2()`和`BeepKey()`。主程序先初始化，再调用按键控制程序。课件截图没有完整展示全部发声和按键处理源码，以下明确作为 **补充讲解示例** ，不冒充原课件完整代码。

### 初始化：先明确安全的空闲电平

```c
/* beepkey.c的一部分 */
#include "stm32f10x.h"
#include "delay.h"       /* 第5章的Delay_Us、Delay_Ms */

static void BeepInit(void)
{
    GPIO_InitTypeDef gpio;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    GPIO_SetBits(GPIOC, GPIO_Pin_8); /* PC8高，蜂鸣器先关闭 */
    gpio.GPIO_Pin = GPIO_Pin_8;
    gpio.GPIO_Mode = GPIO_Mode_Out_PP;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    GPIO_Init(GPIOC, &gpio);
}

static void KeyInit(void)
{
    GPIO_InitTypeDef gpio;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOE, ENABLE);
    gpio.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 |
                    GPIO_Pin_2 | GPIO_Pin_3;
    gpio.GPIO_Mode = GPIO_Mode_IPU;
    gpio.GPIO_Speed = GPIO_Speed_2MHz; /* 输入模式下不决定扫描频率 */
    GPIO_Init(GPIOE, &gpio);
}
```

课件蜂鸣器截图采用50MHz，本例低速声音控制选择2MHz； **GPIO速度档位改变并不等于声音频率改变** 。

### 原始读取与单次按下识别

本例同时按下多个键时按KEY1、KEY2、KEY3、KEY4顺序取优先者；必须所有按键稳定松开后才允许下一次按下事件。

```c
static uint8_t Key_ReadRaw(void)
{
    uint16_t value = GPIO_ReadInputData(GPIOE);
    if ((value & GPIO_Pin_0) == 0U) return 1U;
    if ((value & GPIO_Pin_1) == 0U) return 2U;
    if ((value & GPIO_Pin_2) == 0U) return 3U;
    if ((value & GPIO_Pin_3) == 0U) return 4U;
    return 0U;
}

static uint8_t Key_GetPress(void)
{
    static uint8_t ready = 1U; /* static使这个状态跨调用保存 */
    uint8_t key = Key_ReadRaw();

    if ((key != 0U) && (ready != 0U))
    {
        Delay_Ms(20U);           /* 按下消抖 */
        if (Key_ReadRaw() == key)
        {
            ready = 0U;         /* 已经报告过这次按下 */
            return key;
        }
    }
    else if ((key == 0U) && (ready == 0U))
    {
        Delay_Ms(20U);           /* 松开也需要确认 */
        if (Key_ReadRaw() == 0U)
            ready = 1U;
    }
    return 0U;
}
```

 **逐行重点：** `ready`为1时允许报告新事件；第一次稳定按下后改0，按住不会重复触发；稳定松开后改回1。`return 0`表示“本次没有新的按下事件”，不一定表示“当前所有按键都松开”。

### 两种发声与主循环

下面与前两个代码块合在同一个main.c即可构成讲解程序，另加入第5章delay.c/头文件及GPIO、RCC库源文件。

```c
static void Beep_Pulse(uint32_t half_us, uint32_t cycles)
{
    while (cycles != 0U)
    {
        GPIO_ResetBits(GPIOC, GPIO_Pin_8); /* 导通 */
        Delay_Us(half_us);
        GPIO_SetBits(GPIOC, GPIO_Pin_8);   /* 关闭 */
        Delay_Us(half_us);
        --cycles;
    }
    GPIO_SetBits(GPIOC, GPIO_Pin_8);       /* 返回时保持关闭 */
}

static void sound1(void)
{
    Beep_Pulse(500U, 200U); /* 名义1kHz，200ms */
}

static void sound2(void)
{
    Beep_Pulse(250U, 400U); /* 名义2kHz，200ms */
}

int main(void)
{
    uint8_t key;
    BeepInit();
    KeyInit();
    while (1)
    {
        key = Key_GetPress();
        if (key == 1U)
            sound1();
        else if (key == 2U)
            sound2();
        /* KEY3、KEY4已可读取，本示例不另外分配功能 */
    }
}
```

 **示例边界：** 这是便于理解的阻塞式程序，发声期间主循环不处理新按键，所以不能保证捕捉每次短按；若需要发声、显示、按键同时及时响应，可在后续使用定时器和非阻塞状态处理。不要把课件未给出的特定报警旋律或四键功能当作课程原要求。

## 本章复习自检

1. 松开时读1、按下时读0的电路应如何配置？为什么不用浮空输入？
2. IDR与ODR、位读取与端口读取分别有什么不同？
3. 一次按下出现多次动作通常是什么原因？如何实现“按一下只响一次”？
4. 为什么只把无源蜂鸣器引脚拉低还不足以控制稳定音调？
5. 半周期250μs对应2kHz；400个周期对应200ms。能够自己列出公式并检查单位。
6. 将循环延时次数减半，不一定精确把音调翻倍；实际还受指令执行和中断开销影响。
