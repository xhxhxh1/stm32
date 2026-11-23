# 第08章 中断系统与基本应用

[总目录](../README.md#章节目录) · [知识点速查](#知识点速查) · [题型练习](#题型练习) · [课件详解](#课件详解)

## 知识点速查

中断使 CPU 暂停正在执行的程序，运行服务函数，处理完再返回。作业里的“先高后低”和“高打断低”分别对应同时等待和中断嵌套。

### 先区分谁先执行、谁能打断谁

| 给出的情况 | 判断方法 |
| --- | --- |
| 两个请求同时到达，主程序正在运行 | 优先级数值越小越高。抢占值为 **0 和 2** 时，0 先响应，2 等待 |
| 一个中断已经在运行 | 只有抢占优先级更高的中断可以打断它。高优先级正在运行时，低优先级请求等待处理 |
| 两者抢占优先级相同 | 都在等时，先看响应优先级（副优先级、子优先级）。正在运行时，同抢占级的另一个请求需要等待 |
| 抢占和响应优先级都相同 | 同时等待时，异常编号较小的先处理 |

STM32F103 课程中把 **4 位**优先级分成抢占与响应两部分，共 **16 种**编码。标准库组 0～4 中，组号就是抢占位数。比如**组 1**是 1 位抢占、3 位响应，取值分别为 **0～1**、**0～7**。分组对全系统生效，普通工程在初始化时统一设置。

### 再区分 NVIC、EXTI 以及中断源

| 作业问法 | 要记住的区别 |
| --- | --- |
| 谁管中断优先级 | **NVIC** 管理中断使能和优先级。GPIO 输入经 AFIO 映射后，EXTI 对边沿作检测 |
| 有多少条 EXTI 线 | **19 条**对应 19 个边沿检测器。其中 **16 条**可接 GPIO（EXTI0～EXTI15）。同编号的 PA0、PE0 只能选一个接 EXTI0 |
| 哪些线共用一个入口 | EXTI5～EXTI9 共用 `EXTI9_5_IRQHandler`，EXTI10～EXTI15 共用 `EXTI15_10_IRQHandler` |
| 屏蔽与挂起 | EXTI 的 `IMR` 是中断屏蔽寄存器，`PR` 记录挂起。服务函数处理后应清掉对应挂起标志 |
| 中断与事件 | 两者都可由外部边沿或软件触发。中断路径通过 NVIC 请求 CPU 执行服务函数，事件路径可以直接给出硬件事件信号，减少 CPU 进入服务函数的开销 |
| 中断源的范围 | 从来源看，可有内核异常和外设请求。软件也能触发异常或 EXTI。做多选时还要考虑**系统异常、软件触发**，例如 SysTick、故障异常和软件触发的 EXTI |

**题目口径。** 课件关于“84 个中断源”的题目按 16 个内核位置加 68 个外部位置来记。器件实际实现多少外部中断通道要查具体型号，本课配套 STM32F103ZET6 的数据手册列的是最多 60 个可屏蔽外部通道。[ST 器件数据手册的 NVIC 与 EXTI 说明](https://www.st.com/resource/en/datasheet/stm32f103ze.pdf)

## 题型练习

1. **单选**　主程序运行时，抢占值为 1 和 3 的两个中断请求同时到达，且都可响应。哪个先进入服务函数？

   A. 值为 1 的　B. 值为 3 的　C. 等主循环退出　D. 看哪个服务函数更短

2. **多选**　标准库优先级分组 1 中，哪些设置有效？

   A. 抢占值 0　B. 副优先级 6　C. 抢占值 2　D. 副优先级 7

3. **多选**　哪些 EXTI 线使用 EXTI9_5 的中断入口？

   A. EXTI4　B. EXTI5　C. EXTI9　D. EXTI10

4. **判断**　F103ZET6 的 EXTI 一共有 16 个边沿检测器。

5. **判断**　抢占优先级相同，响应优先级高的一方可以打断对方。

<details>

<summary>展开答案与理由</summary>

1. **A。** 1 的抢占优先级较高。若值为 3 的中断已经开始执行，值为 1 的才会抢占它。
2. **ABD。** 1 位抢占能写 0～1，3 位副优先级能写 0～7。
3. **BC。** 这个入口覆盖 5 到 9。
4. **错。** 总数为 19，可接 GPIO 的有 16 条。
5. **错。** 响应优先级用于两者都在等待时的排序。

</details>

### 读一句代码

```c
EXTI_ClearITPendingBit(EXTI_Line0);
```

**参考注释**　清除 EXTI0 的中断挂起标志，表示本次请求已处理。

## 课件详解

点开对应小节，可以看课件中的解释、寄存器细节和原笔记里的完整示例。

### 目录

<details>

<summary>展开本章原目录</summary>

- [8.1 中断的基本概念](#81-中断的基本概念)
- [8.2 STM32F103中断系统](#82-stm32f103中断系统)
- [8.3 STM32F103外部中断/事件控制器EXTI](#83-stm32f103外部中断事件控制器exti)
- [8.4 STM32中断相关库函数](#84-stm32中断相关库函数)
- [8.5 EXTI项目实例](#85-exti项目实例)
- [本章复习自检](#本章复习自检)

</details>

### 8.1 中断的基本概念

<details>

<summary>展开本节详解</summary>

#### 8.1.1 什么叫中断

CPU暂时中止当前程序，转去执行某外设或事件的中断服务程序，处理完毕后返回原来的程序继续执行，称为中断。中断源产生请求，不表示服务函数必然立即执行，还要满足使能、屏蔽、优先级等条件。

#### 8.1.2 为什么要设置中断

课件归纳四个作用：提高CPU效率、实现实时处理、实现故障处理、实现分时操作。与主程序不断检查外设相比，中断允许CPU先做其他任务，等事件出现再处理。

**【基础补充】** 中断也有进入、保存现场、返回等开销；中断频率过高、服务程序过长，仍会占用大量CPU时间。

#### 8.1.3 中断源与中断屏蔽

中断源是引发请求的事件，例如按键边沿、定时器更新、串口收到数据。挂起/状态标志保存“发生了某事件”的信息；中断使能或屏蔽位决定是否允许把请求继续送往CPU。

**【重点·三件事分别判断】** 事件是否发生、标志是否置位、中断是否允许响应。屏蔽中断不一定清除已经产生的标志，重新使能时可能立即响应已有请求。

可屏蔽中断可由软件控制是否响应；NMI是不可屏蔽中断。 **【基础核对】** Cortex-M3还区分HardFault、MemManage、BusFault、UsageFault等异常，不应把所有电源、存储器或总线故障笼统写成NMI。

#### 8.1.4 中断处理过程

**请求产生→判断是否响应→保存现场→按向量转入服务函数→执行处理并处理标志→恢复现场→返回。**

- 硬件自动完成基本现场保存与恢复，软件负责服务函数里的处理逻辑。
- C语言中断函数通常形如`void EXTI0_IRQHandler(void)`，无参数、无返回值。
- 中断向量负责把事件与正确函数连接起来，不能随意改服务函数名。
- 服务函数应短，避免长延时、长循环、等待按键释放。
- 主程序和中断共享且可能异步改变的变量应使用`volatile`。

**【基础补充·volatile的边界】** `volatile`要求按程序语义访问变量，但不能保证`count++`这种“读→加→写”整体不被打断，也不能保证同时读取hour、minute、second时得到同一时刻的一组值。需要时用短临界区取快照或保护更新。

#### 8.1.5 中断优先级与中断嵌套

优先级决定多个请求的处理次序；高抢占优先级中断可以打断低抢占优先级中断，形成中断嵌套。同一抢占优先级的中断之间不能仅凭子优先级不同就互相抢占。

**【基础补充】** 主循环不是一个普通外设IRQ，不要把主程序当成“子优先级更高”的任务参与NVIC分组比较。

</details>

### 8.2 STM32F103中断系统

<details>

<summary>展开本节详解</summary>

#### 8.2.1 嵌套向量中断控制器NVIC

NVIC集成在Cortex-M3内核，负责中断使能、优先级与挂起管理，支持嵌套、向量处理、自动现场保存/恢复和尾链等。课件以16个内部异常位置、68个非内核中断位置介绍系统；其中包含保留或具体型号未实现的项，不应理解为每一项都可由本板触发。

本课程STM32F103使用4位可编程优先级，共16种编码。 **数值越小，优先级越高。**

**【基础补充·尾链】** 处理完一个中断后，如果还有合适的挂起请求，处理器可减少不必要的恢复/再保存过程，直接进入下一个服务，提高效率。

#### 8.2.2 STM32F103中断优先级

**【重点·两类优先级】** 抢占优先级决定是否能打断正在执行的中断；响应优先级（子优先级）在抢占优先级相同、多个请求等待处理时决定先后。

| 分组 | 抢占位数 | 子优先级位数 | 抢占值范围 | 子优先级值范围 |
| --- | --- | --- | --- | --- |
| `NVIC_PriorityGroup_0` | 0 | 4 | 只能0 | 0～15 |
| `NVIC_PriorityGroup_1` | 1 | 3 | 0～1 | 0～7 |
| `NVIC_PriorityGroup_2` | 2 | 2 | 0～3 | 0～3 |
| `NVIC_PriorityGroup_3` | 3 | 1 | 0～7 | 只能0～1 |
| `NVIC_PriorityGroup_4` | 4 | 0 | 0～15 | 只能0 |

分组数不表示“总共只有几个优先级”，而是分配4位中的抢占位数。若抢占位数为 $p$ ，抢占取值数为 $2^p$ ，子优先级取值数为 $2^{4-p}$ 。

**判断例子，采用分组2：**

| 中断 | 抢占 | 子优先级 |
| --- | --- | --- |
| A | 1 | 3 |
| B | 0 | 3 |
| C | 1 | 0 |

- B可抢占A或C，因为0比1高。
- C不能抢占正在运行的A，因为二者抢占值都是1。
- A、C同时挂起且都可被服务时，C先执行，因为子优先级0比3高。
- 两类优先级都相同，再按异常号等硬件规则仲裁；不要把“同时到来”理解成同时执行。

#### 8.2.3 STM32F103中断向量表

中断向量表保存入口信息。 **【基础核对】** 第一个字是初始栈指针值，其后才是Reset、NMI、HardFault等处理函数地址；不是所有表项都是中断函数。复位时从地址0处的映射启动，随后可通过VTOR设置向量表位置，Flash程序通常把表放在用户Flash中。[ST Cortex-M3编程手册](https://www.st.com/resource/en/programming_manual/cd00228163-stm32f10xxx20xxx21xxxl1xxxx-cortexm3-programming-manual-stmicroelectronics.pdf)

| 相对表起始地址的偏移 | 内容 |
| --- | --- |
| 0x00 | 初始栈指针 |
| 0x04 | Reset入口 |
| 0x08 | NMI入口 |
| 0x0C | HardFault入口 |
| 0x3C | SysTick入口 |
| 0x40起 | 外设IRQ入口 |

**【基础补充·地址计算】** 对非负外设IRQ编号 $q$ ，其表项偏移为 $4(q+16)$ 字节，因为每个入口4字节，外设中断在前16个位置之后。IRQ编号、向量表偏移、优先级编码是三种不同概念。

#### 8.2.4 STM32F103中断服务函数

启动文件提供对应入口，用户在stm32f10x_it.c或另一个源文件中定义准确同名的函数即可连接到该入口。正确拼写为`IRQHandler`，不是课件个别文字中的`IRQHanlder`。

```c
void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line0);
        /* 处理对应事件，或记录标志交给主程序 */
    }
}
```

同一函数只能定义一次。如果把这个函数写进exti_keys.c，就不能同时在stm32f10x_it.c再放一个同名定义。

</details>

### 8.3 STM32F103外部中断/事件控制器EXTI

<details>

<summary>展开本节详解</summary>

#### 8.3.1 EXTI内部结构

课件所述EXTI有19条中断/事件线路，其中EXTI0～15用于GPIO映射，其他线路连接片内特定信号。内部包括输入选择、边沿检测、屏蔽控制、软件触发、挂起标志及事件/中断路径。

**【重点·同编号映射】** PA0、PB0、PC0、PE0等都只能作为EXTI0的候选来源；PA1、PE1等对应EXTI1。 **同一条EXTI线同一时刻只能选择一个端口来源** ，不能让PA0和PE0独立占用同一个EXTI0。

GPIO作外部中断输入时配置为输入模式；同时开启GPIO端口和AFIO时钟。

#### 8.3.2 EXTI工作原理

GPIO电平发生变化→选择的上升/下降沿检测到变化→产生挂起请求→通过中断屏蔽控制送NVIC→由CPU按优先级执行ISR。

事件路径产生硬件事件信号，不等同于CPU进入服务函数；它能否唤醒或触发其他模块取决于芯片支持的连接和配置。不能认为任意EXTI事件都自动连接到任意定时器或DMA。

**【基础补充·相关寄存器】** IMR控制中断屏蔽，EMR控制事件屏蔽，RTSR/FTSR选择上升/下降沿，SWIER用于软件触发，PR保存挂起标志。

#### 8.3.3 EXTI主要特性

- 各线可独立选择上升沿、下降沿或双边沿。
- 各线可独立屏蔽并保存挂起状态。
- GPIO可映射到16条外部输入线；引脚数量多不等于可同时拥有112条独立外部中断线。
- 课件说明其可检测较窄外部信号；实际仍应满足器件的输入电气及时序要求。

| GPIO中断线 | 对应NVIC通道与服务函数 |
| --- | --- |
| EXTI0～EXTI4 | 各有独立通道，例如`EXTI0_IRQn`、`EXTI0_IRQHandler` |
| EXTI5～EXTI9 | 共用`EXTI9_5_IRQn`、`EXTI9_5_IRQHandler` |
| EXTI10～EXTI15 | 共用`EXTI15_10_IRQn`、`EXTI15_10_IRQHandler` |

共用服务函数中必须逐一检查可能的挂起位，不能只凭进入函数就断定是哪一根线触发。

</details>

### 8.4 STM32中断相关库函数

<details>

<summary>展开本节详解</summary>

#### 8.4.1 STM32F10x的NVIC相关库函数

- `NVIC_PriorityGroupConfig(group)`：划分抢占/子优先级位数。通常在系统初始化时统一设置一次，避免各模块互相改组；这不是硬件永久只能写一次的限制。
- `NVIC_Init(&nvic)`：设置IRQ通道、抢占值、子优先级及使能。
- 课件列出`NVIC_DeInit()`复位NVIC配置；当前配套标准库头文件不提供这一函数，不在讲解示例中照抄调用。应以工程实际版本接口为准。

| NVIC_InitTypeDef成员 | 示例 | 含义 |
| --- | --- | --- |
| NVIC_IRQChannel | EXTI0_IRQn | 中断通道编号 |
| NVIC_IRQChannelPreemptionPriority | 2 | 抢占优先级 |
| NVIC_IRQChannelSubPriority | 0 | 子优先级 |
| NVIC_IRQChannelCmd | ENABLE | 使能该NVIC通道 |

#### 8.4.2 STM32F10x的EXTI相关库函数

| 函数 | 功能与参数要点 |
| --- | --- |
| `EXTI_DeInit()` | 将EXTI配置恢复默认值，使用前考虑是否影响其他线 |
| `EXTI_Init(&exti)` | 配置Line、Mode、Trigger、LineCmd |
| `EXTI_GetFlagStatus(line)` | 检查PR中的标志状态 |
| `EXTI_GetITStatus(line)` | 检查挂起且中断未被屏蔽的状态 |
| `EXTI_ClearFlag(line)` | 清指定挂起标志 |
| `EXTI_ClearITPendingBit(line)` | 清指定中断挂起位 |

EXTI_InitTypeDef四个字段：`EXTI_Line`选线路；`EXTI_Mode`选Interrupt/Event；`EXTI_Trigger`选Rising/Falling/Rising_Falling；`EXTI_LineCmd`选ENABLE/DISABLE。

**【重点·清标志】** EXTI的PR对应位写1清除，所以库函数内部按这个规则处理。不能用`EXTI->PR &= ~EXTI_Line0`这种普通清位写法；也不要用可能误清其他已挂起位的读改写。

#### 8.4.3 EXTI中断线GPIO引脚映射库函数

```c
GPIO_EXTILineConfig(GPIO_PortSourceGPIOE, GPIO_PinSource0);
```

含义是“把PE0选作EXTI0的输入”。第二个参数是PinSource编号， **不是** `GPIO_Pin_0`掩码；二者数字碰巧有时相近也不能混用。

</details>

### 8.5 EXTI项目实例

<details>

<summary>展开本节详解</summary>

#### 8.5.1 项目分析

**【重点·六步配置链】** 开GPIOE/AFIO时钟→PE0/1/2上拉输入→映射EXTI0/1/2→设置下降沿中断→设置NVIC分组、优先级与使能→实现三个ISR。

按键按下由1变0，所以选下降沿；按键松开由0变1，对应上升沿。主程序继续扫描数码管，按键用于独立调整时、分、秒。

#### 8.5.2 项目实施

加入`stm32f10x_exti.c`、`misc.c`及已有GPIO/RCC库源文件。下面提供讲解版：ISR只记请求，主循环消抖后修改时间；三键可分别识别，长按不连续递增。它保留课件引脚与任务，补全课件未完整展开的消抖细节。

<details>

<summary>按键中断驱动接口 · exti_keys.h</summary>

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

</details>

<details>

<summary>外部中断初始化与处理 · exti_keys.c 上半段</summary>

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
```

</details>

**初始化顺序解释：** 先准备输入与映射，再打开EXTI路径，清初始化期间残留的请求，最后设置NVIC。三个ISR抢占值相同，不会互相嵌套修改key_requests；主程序读取并清该变量时仍用临界区。

**主循环消抖。** 以下函数每扫描完一帧调用一次，按第7章六位×2ms计算，调用间隔约12ms。发现请求后，连续两次读取为按下才执行；稳定松开两次才重新允许触发。这是教学采样参数，实际确认延迟受按下时刻和扫描周期影响，不能写成固定精确24ms。

<details>

<summary>消抖与时间修改 · exti_keys.c 下半段</summary>

```c
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

</details>

`__get_PRIMASK()`保存原屏蔽状态，`__disable_irq()`建立很短的临界区，`__set_PRIMASK(saved)`恢复原状态。临界区只做共享变量操作，不把整个数码管扫描包进去。按键调分、调秒在本例中独立回绕，不向上一单位进位；自动走时的进位放在第9章。

<details>

<summary>EXTI 调时示例 · main.c</summary>

```c
/* main.c：与第7章显示驱动、第5章延时驱动组合 */
#include "exti_keys.h"
#include "dsgshow.h"

volatile uint8_t hour = 12U, minute = 0U, second = 0U;

int main(void)
{
    uint8_t h, m, s;
    uint32_t saved;
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    DsgShowInit();
    EXTIInit();
    while (1)
    {
        saved = __get_PRIMASK();
        __disable_irq();
        h = hour; m = minute; s = second; /* 取一致快照 */
        __set_PRIMASK(saved);
        DsgShowTime(h, m, s);
        Key_AdjustTask();
    }
}
```

</details>

**【课件写法与改进】** 课件ISR示例中直接延时确认按键后修改小时；这种写法容易理解，但会延长ISR。本讲解版把消抖放在主循环，同时保持“ISR应简短”的原则。

</details>

### 本章复习自检

<details>

<summary>展开本节详解</summary>

1. 能按顺序说明请求、响应、保存现场、服务和返回。
2. 分组2下，抢占1/子3与抢占1/子0能否互相嵌套？答案：不能；同时等待时子0先。
3. PA0与PE0能否同时独立占用EXTI0？答案：不能。
4. 能写出GPIO→AFIO映射→EXTI→NVIC→ISR完整配置链。
5. 能解释GPIO_PinSource0、GPIO_Pin_0、EXTI_Line0、EXTI0_IRQn四者各用于何处。
6. 能解释为什么必须清标志、为什么volatile不能代替临界区、为什么ISR中不宜长延时。

</details>

---

[上一章](07-dynamic-display.md) · [总目录](../README.md#章节目录) · [下一章](09-timers-and-pwm.md)
