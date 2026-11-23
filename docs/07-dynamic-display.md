# 第07章 数码管动态显示

[总目录](../README.md#章节目录) · [知识点速查](#知识点速查) · [题型练习](#题型练习) · [课件详解](#课件详解)

## 知识点速查

| 知识点 | 记忆方法 |
| --- | --- |
| 共阴 / 共阳 | 共阴的各段阳极接高时亮。共阳的各段阴极接低时亮。本板使用六位共阳数码管 |
| 段选 / 位选 | 段选决定亮哪些段，位选决定亮哪一位。课件的段线 PG0～PG7 对应 a～g、dp，位选 PE15～PE10 对应第 1～6 位 |
| 段码例子 | 数字 1 亮 b、c。共阳且从高到低按 `dp g f e d c b a` 排列时，只把 b、c 对应位清 0，得到 `0xF9` |
| 静态 / 动态 | 静态显示各位持续保持显示。动态显示多位共用段线，轮流点亮。先关闭旧位再换段码，有助于减少重影 |
| 一帧的刷新时间 | 六位每位 2 ms，忽略切换开销，整帧为 **12 ms**，刷新频率约 83.3 Hz |

## 题型练习

1. **单选**　本板想选中第 1 位数码管，应执行哪个动作？

   A. 拉高 PE15　B. 拉低 PE15　C. 拉低 PG0　D. 拉高 PG7

2. **多选**　关于课件板的动态显示，哪些说法正确？

   A. 多位共享段线　B. 位选决定显示位置　C. 换段前先关旧位有助于避免重影　D. 每位 2 ms 就等于整帧 2 ms

3. **判断**　同一个段码配合不同的位选，可以把相同数字显示在不同的位置。

<details>

<summary>展开答案与理由</summary>

1. **B。** PE15 是第 1 位的低电平位选。
2. **ABC。** 一帧要轮完全部六位。
3. **对。** 段码管内容，位选管位置。

</details>

### 读一句代码

```c
GPIO_ResetBits(GPIOE, GPIO_Pin_15);
```

**参考注释**　拉低 PE15，接通第 1 位数码管的位选。执行前先准备好段码。

## 课件详解

点开对应小节，可以看课件中的解释、寄存器细节和原笔记里的完整示例。

### 目录

<details>

<summary>展开本章原目录</summary>

- [7.1 数码管显示接口](#71-数码管显示接口)
- [7.2 项目分析](#72-项目分析)
- [7.3 项目实施](#73-项目实施)
- [7.4 项目拓展](#74-项目拓展)
- [本章复习自检](#本章复习自检)

</details>

### 7.1 数码管显示接口

<details>

<summary>展开本节详解</summary>

#### 7.1.1 数码管工作原理

数码管由a、b、c、d、e、f、g七个发光段和小数点dp组成。通过不同的段组合形成数字。

| 类型 | 公共端 | 段控制电平 |
| --- | --- | --- |
| 共阴 | 各段阴极接公共端 | 正常直接驱动关系下，高电平点亮段 |
| 共阳 | 各段阳极接公共端 | 正常直接驱动关系下，低电平点亮段 |

本课程使用共阳。段码不仅取决于数字，还取决于共阳/共阴及a～g、dp的位序，不能只背一个十六进制数就套用所有电路。

**【基础补充】** 每段本质上是LED，驱动电路需要限流；六位共用段线时，应通过位选驱动控制哪一位被接通。

#### 7.1.2 数码管编码方式

课件位序为：bit7～bit0对应`dp g f e d c b a`。共阳段码中0表示该段亮，1表示该段灭。

**【重点·0～9段码】**

| 数字 | 点亮的段（不含dp） | 二进制dp g f e d c b a | 共阳段码 |
| --- | --- | --- | --- |
| 0 | a b c d e f | 11000000 | 0xC0 |
| 1 | b c | 11111001 | 0xF9 |
| 2 | a b d e g | 10100100 | 0xA4 |
| 3 | a b c d g | 10110000 | 0xB0 |
| 4 | b c f g | 10011001 | 0x99 |
| 5 | a c d f g | 10010010 | 0x92 |
| 6 | a c d e f g | 10000010 | 0x82 |
| 7 | a b c | 11111000 | 0xF8 |
| 8 | a b c d e f g | 10000000 | 0x80 |
| 9 | a b c d f g | 10010000 | 0x90 |

**推导数字1：** 只亮b、c，所以bit1、bit2为0，其余位为1，得到`11111001=0xF9`。

**【基础补充·小数点与反码】**

```c
uint8_t code = 0xF9U; /* 显示数字1，dp灭 */
code &= 0x7FU;       /* bit7清0，显示1并点亮dp，结果0x79 */
code |= 0x80U;       /* bit7置1，dp熄灭 */
```

在位序完全相同、驱动极性相反的前提下，共阴段码与共阳段码按8位取反转换。例如数字0：共阳0xC0，共阴0x3F。取反后要限定8位，不能把高位也送到其他GPIO。

#### 7.1.3 数码管显示方式

| 项目 | 静态显示 | 动态显示 |
| --- | --- | --- |
| 段线使用 | 每位可分别维持段码，或由专用驱动保持 | 多位共享段线 |
| 位的工作方式 | 多位可以持续显示 | 轮流选中各位 |
| I/O与硬件 | 直接驱动时通常占更多线路 | 节省段线，需要扫描控制 |
| 软件要求 | 不依赖主程序持续逐位扫描 | 必须按足够快且均匀的节奏持续刷新 |

**【重点】** 动态显示在一个瞬间只点亮一位，通过快速循环使人眼看到稳定的多位显示。停止刷新时可能只剩一位亮或全部熄灭；扫描太慢会闪烁。

</details>

### 7.2 项目分析

<details>

<summary>展开本节详解</summary>

#### 段选与位选

PG0～PG7分别连接a～g、dp；PE15、PE14、PE13、PE12、PE11、PE10经PNP三极管分别控制六位数码管。位选为0时选中。

| 数码管位置 | 选中引脚 | 位选掩码 | 课件整端口位码 |
| --- | --- | --- | --- |
| 第1位 | PE15 | 0x8000 | 0x7FFF |
| 第2位 | PE14 | 0x4000 | 0xBFFF |
| 第3位 | PE13 | 0x2000 | 0xDFFF |
| 第4位 | PE12 | 0x1000 | 0xEFFF |
| 第5位 | PE11 | 0x0800 | 0xF7FF |
| 第6位 | PE10 | 0x0400 | 0xFBFF |

课件第一位显示数字1的例子：PE写0x7FFF，使PE15=0；PG低8位写0xF9，使b、c亮。 **位码决定位置，段码决定内容。**

#### 动态扫描的完整顺序

**【基础补充·消除串位】** 推荐按“关闭所有位选→更新段码→打开当前位选→保持短时间→关闭当前位选→下一位”执行。先关位选再改段码，可以减小上一位段码串到下一位的重影。

设有 $n$ 位，每位保持 $t_d$ ，忽略程序开销：

$$
T_{frame}\approx n t_d,\qquad f_{frame}\approx\frac{1}{n t_d}
$$

本示例6位，每位2ms，完整一帧约12ms，刷新频率约83.3Hz。每位的导通占空比约为 $1/6$ ；实际还受关闭间隔和代码开销影响。

**【易错点】** 2ms是单个位的保持时间，不是整屏刷新周期；在扫描过程中插入1秒延时会明显破坏显示。提高亮度也不能不受限制地增大瞬时电流。

</details>

### 7.3 项目实施

<details>

<summary>展开本节详解</summary>

课件建立dsgshow.c/dsgshow.h，`DsgShowInit()`初始化GPIOE/G，`DsgShowNum()`实现循环扫描。以下把显示功能改写为“每次调用扫描一整帧并返回”，方便第8、9章主循环同时处理其他任务；属于补充讲解代码。

#### 头文件与初始化

<details>

<summary>显示驱动接口 · dsgshow.h</summary>

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

</details>

<details>

<summary>数码管初始化与扫描驱动 · dsgshow.c</summary>

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
```

</details>

**代码重点：** `segment[digits[i]]`先用数字查段码；`digit_pin[i]`选择显示位置；`dot_mask`的bit0～bit5对应六位的小数点；段码或位码不要混作数组下标。值大于9时本例显示空白，避免越界读取段码表。

本例只修改PE10～PE15，不会把PE0～PE3按键误改为输出；也只写PG0～PG7，保留端口其他位的状态。

**【讲解示例·固定六位数字】** 将下面main与上述显示驱动及第5章延时驱动组合；数组是演示数字，不代表用户学号。

<details>

<summary>学号扫描示例 · main.c</summary>

```c
#include "dsgshow.h"

int main(void)
{
    const uint8_t number[6] = {1, 2, 3, 4, 5, 6};
    DsgShowInit();
    while (1)
        DsgScanOnce(number, 0U);
}
```

</details>

</details>

### 7.4 项目拓展

<details>

<summary>展开本节详解</summary>

目标是显示hour、minute、second三个变量，在小时个位和分钟个位点亮小数点。六位顺序为：小时十位、小时个位、分钟十位、分钟个位、秒十位、秒个位。

#### 十位与个位的计算

对0～99的整数 $x$ ：

$$
十位=x/10\text{（整数除法）},\qquad 个位=x\bmod10
$$

例如23：23/10=2，23%10=3。若时间为08:05:09，结果数组为`{0,8,0,5,0,9}`，不能省略开头的0。

<details>

<summary>时、分、秒的拆位与显示 · DsgShowTime</summary>

```c
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

</details>

dot_mask=0x0A，即bit1、bit3为1，选中小时个位和分钟个位的小数点。函数通过参数接收一次显示的时间值，避免显示函数直接依赖多个外部变量；这与课件功能相同，但接口作了明确的教学整理。

#### extern：跨文件使用同一个变量

课件在main.c中定义hour、minute、second，在dsgshow.c中用`extern u8 hour, minute, second;`声明后使用。 **定义只写一次，extern声明可以让其他源文件知道它们存在。**

```c
/* 课件式共享变量写法示意 */
/* main.c中： */
uint8_t hour = 8, minute = 5, second = 9;

/* 其他源文件所包含的头文件中： */
extern uint8_t hour, minute, second;
```

第8、9章一旦中断会修改这些变量，定义和声明都应增加`volatile`，并在需要一组一致时间值时取快照；`extern`本身不解决中断并发问题。

**【讲解示例·显示时间main.c】** 这个main替换固定数字示例的main，二者不能同时编入同一工程。

<details>

<summary>时间显示示例 · main.c</summary>

```c
#include "dsgshow.h"

int main(void)
{
    uint8_t hour = 8U, minute = 5U, second = 9U;
    DsgShowInit();
    while (1)
        DsgShowTime(hour, minute, second);
}
```

</details>

**【重点】本章实现的是显示，不是自动计时。** 变量没有递增，显示就保持不变；第8章通过按键修改时间，第9章通过定时器每秒更新时间。

</details>

### 本章复习自检

<details>

<summary>展开本节详解</summary>

1. 不看表推导数字1、8及带小数点数字1的共阳段码。
2. `0x7FFF`选中哪一位？为什么段码0xF9与位码0x7FFF不能互换？
3. 六位、每位2ms时，整帧周期和刷新率分别是多少？
4. 先关位选再换段码有什么作用？显示过程中延时1秒会怎样？
5. 怎样把23:59:08拆成6个数字？哪些位置加小数点？
6. `extern`、变量定义、`volatile`分别解决什么问题？

</details>

---

[上一章](06-keys-and-buzzer.md) · [总目录](../README.md#章节目录) · [下一章](08-interrupts.md)
