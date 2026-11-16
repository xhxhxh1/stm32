# 第10章 串行通信接口USART

本章复习目标：理解串行数据帧、波特率与USART寄存器，分清TXE、TC、RXNE，能够配置USART1并解释PC通过三个数值字节给单片机校时的过程。 **【重点】** 根据课件整理； **【基础补充】** 补齐计算、电平、数据格式及程序边界； **【讲解示例】** 为根据课件任务整理的代码。

## 目录

- [10.1 数据通信基本概念](#101-数据通信基本概念)
- [10.2 SM32F103的USART工作原理](#102-sm32f103的usart工作原理)
- [10.3 USART相关库函数](#103-usart相关库函数)
- [10.4 项目实例](#104-项目实例)

## 10.1 数据通信基本概念

### 10.1.1 并行通信与串行通信

| 方式 | 数据怎样传 | 理解要点 |
| --- | --- | --- |
| 并行通信 | 多条数据线同时传各位，可按字或字节传送 | 数据线较多；各位同时到达还涉及线路与时序配合 |
| 串行通信 | 一条数据线上逐位传送 | 连接线少；收发双方按相同规则解释比特流 |

 **【基础补充】** “一条数据线”描述单个串行方向；普通全双工USART使用TX、RX两条方向相反的信号线，通常还要共地。串行并不意味着只能单向或一定比所有并行接口慢。

### 10.1.2 异步通信与同步通信

 **【重点·异步】** 按帧传送，通常由起始位、数据位、可选校验位和停止位构成。双方各自生成时钟，通过起始位对齐接收时序，不单独传送连续的时钟线。

 **【重点·同步】** 课件用同步字符和连续数据块说明：同步信息用于建立接收节奏，连续数据之间按规定时序传送。STM32 USART的同步模式还可输出CK时钟，不能把课件的同步字符描述当成所有同步接口唯一的实现方法。

 **【基础补充·8N1读法】** 8表示8个数据位，N表示无奇偶校验，1表示1个停止位；起始位另计。一个8N1数据帧共10个比特时间。

### 10.1.3 串行通信的制式

| 制式 | 方向与同时性 | 关键词 |
| --- | --- | --- |
| 单工Simplex | 只能固定单向传输 | 收发角色固定 |
| 半双工Half Duplex | 能双向，不能同时收发 | 轮流占用传输方向 |
| 全双工Full Duplex | 可以同时发送和接收 | 发送、接收通道分开 |

普通USART1的TX/RX双线配置可全双工。半双工不是“只能发送半个字节”，双工也不表示数据位数。

### 10.1.4 串行通信的校验

-  **奇偶校验：** 让“数据位中的1 + 校验位中的1”的总数满足偶数或奇数。例：0x35 = 00110101，有4个1；偶校验位为0，奇校验位为1。
-  **累加和校验：** 对约定的数据块求和，把校验和附在末尾，接收方按相同规则重新计算。 **【基础补充】** 若约定8位校验和，0x12 + 0x34 + 0xF0 = 0x136，只保留低8位则为0x36；保留几位、算哪些字节要双方先约定。
-  **CRC循环冗余校验：** 按约定生成多项式求余数附在数据后。 **【基础补充】** 这是模2多项式除法，减法按异或处理；还需约定初值、位序等参数，不能直接用普通整数除法代替。

 **【易错点】** 奇偶校验可检测奇数个比特翻转，但可能漏掉偶数个翻转；校验通过不能证明一定无错。课件校时项目的8N1不带奇偶校验，也未提供带CRC的数据包格式，不应自行说成“已实现CRC校时”。

### 10.1.5 串行通信的波特率

 **【重点】** 本章USART采用二进制NRZ，一个码元承载一个比特，因此常用的9600波特对应9600 bit/s。波特率本义是每秒码元数；课件“1波特=1 bit/s”在这里成立，不能推广至所有调制方式。

 **【基础补充·位时间、帧时间与有效速率】**

$$
T_{bit}=\frac{1}{B},\qquad
N_{frame}=1+n+p+s,\qquad
T_{frame}=\frac{N_{frame}}{B}
$$

B为波特率，单位baud；n为有效数据位数，p为校验位数（0或1），s为停止位数。上述公式用于逐位连续发送的一帧，未计额外帧间空隙。

$$
R_{payload}=B\frac{n}{N_{frame}},\qquad
R_{byte}=\frac{B}{10}\quad\text{（8N1且连续发送）}
$$

 **算例：9600、8N1。** 位时间约104.167μs，一帧10位约1.04167ms，连续发送最多约960个有效数据字节/秒；三个校时字节约3.125ms。不能用9600÷8直接得到该配置下的有效字节率。

双方应匹配波特率、有效数据位、校验、停止位和流控。时钟误差会累计为采样偏差；“数值都填9600”还要求芯片时钟实际配置正确。

## 10.2 SM32F103的USART工作原理

保留课件原标题“SM32F103”；本节实际讨论STM32F103。

### 10.2.1 USART介绍

USART是Universal Synchronous/Asynchronous Receiver/Transmitter，即通用同步异步收发器。课件介绍NRZ全双工异步通信、同步单向、单线半双工、LIN、智能卡、IrDA、CTS/RTS、多处理器通信与DMA多缓冲传输。

 **【基础补充】** UART通常专指异步收发器，USART还支持同步工作方式；工程里说“串口”可能指逻辑外设、接线接口或PC上的COM端口，要结合语境区分。

### 10.2.2 USART主要特性

- 发送与接收共用可编程波特率发生器；课件列出最高4.5 Mbit/s，实际须结合外设总线时钟及器件限制判断。
- 字长可选8位或9位；配置停止位、奇偶校验、独立发送与接收使能。
- LIN可产生13位断开符，检测10/11位断开符；IrDA提供编码解码；智能卡模式支持相应协议和停止位配置。
- 可使用DMA在SRAM与外设间传数据；支持地址匹配/空闲总线等唤醒或静默机制。
- 状态与错误检测包括发送数据寄存器空、发送完成、接收非空，以及ORE、NE、FE、PE。

 **【基础补充·字长与校验的组合】** STM32F1的8/9位字长设置包含可选校验占用的位置：8位字长+校验通常留下7个有效数据位；若需8个有效数据位再加校验，应使用9位字长+校验。无校验的8N1设置为`USART_WordLength_8b`与`USART_Parity_No`。

### 10.2.3 USART功能概述

 **引脚和数据通路：** TX输出串行数据，RX输入串行数据；同步模式可用CK输出时钟；nCTS/nRTS用于硬件流控；IrDA另有相应收发信号。课件所述nCTS高时可阻止下一次发送，nRTS低表示准备接收，注意低有效名称中的n。

发送时CPU写DR，数据经发送移位寄存器逐位输出；接收移位寄存器组装一帧后把数据交给接收数据寄存器，置RXNE供CPU读取。程序看到的是同一个DR地址的读写接口，不是要求软件对每个比特逐一操作GPIO。

 **【基础补充·USART时钟与波特率】** USART1使用PCLK2；USART2/3使用PCLK1。这里不能套用上一章定时器的APB分频后自动乘2规则。若HCLK=72MHz、APB1分频2、APB2不分频，则USART1输入72MHz，USART2/3输入36MHz。

$$
B=\frac{f_{PCLK}}{16\,USARTDIV},\qquad
USARTDIV=M+\frac{F}{16}
$$

$$
BRR=(M\ll4)\;|\;F,\qquad
M=\lfloor USARTDIV\rfloor,\quad
F\approx\operatorname{round}(16(USARTDIV-M))
$$

BRR的整数部分M占12位，小数部分F占4位；若四舍五入得到F=16，应向M进1且F归0。M与F要符合寄存器范围。

 **算例：USART1、72MHz、9600。** USARTDIV = 72000000/(16×9600) = 468.75；M=468=0x1D4，F=12=0xC，因此BRR=0x1D4C。若是36MHz的USART2，同波特率得到234.375，BRR=0x0EA6。标准库`USART_Init`会依据RCC给出的PCLK计算BRR，通常不用手写该寄存器。

实际波特率由最终量化后的分频值决定，误差可算`(实际波特率−目标波特率)/目标波特率×100%`。

基础核对依据：[ST RM0008 USART参考章节](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)，并对照课件配套标准库`stm32f10x_usart.h/.c`的函数原型与标志清除说明。

### 10.2.4 USART通信时序

 **【重点·异步8N1】** 线路空闲为高，起始位为低，数据最低有效位先发，停止位为高。以0x55 = 01010101为例，D0到D7依次为1、0、1、0、1、0、1、0；不要把二进制字符串从左到右当成实际发送顺序。

课件时序图比较8位与9位字长，并标出可选校验、空闲帧和断开帧；同步时钟波形中的CK与异步8N1无独立时钟线的使用方式应分别理解。空闲是保持高电平，不能把“没有发送”理解为引脚浮空。

### 10.2.5 USART中断

| 标志 | 表示什么 | 常见使能条件/用途 |
| --- | --- | --- |
| TXE | 发送数据寄存器空，可写下一份数据 | TXEIE；持续发送 |
| TC | 发送数据寄存器和移位发送过程均完成 | TCIE；最后一帧已发完 |
| RXNE | 接收数据寄存器非空，数据可读 | RXNEIE；及时读取DR |
| IDLE | 检测到空闲线路 | IDLEIE；辅助判断接收空闲 |
| ORE | 接收溢出，旧数据未及时读走 | RXNEIE或相应错误使能条件 |
| PE | 奇偶校验错误 | PEIE |
| NE、FE | 噪声、帧错误 | 结合EIE及接收/DMA配置 |
| CTS、LBD | CTS变化、LIN断开检测 | CTSIE、LBDIE |

 **【重点·TXE与TC】** TXE=1时，前一字节仍可能正在引脚上发送；它只说明能继续写DR。要确认最后一个停止位也已发完，应等待TC。RXNE是接收事件，清TC不能代替读取接收数据。

同一USART的多个事件共用一个中断入口，例如`USART1_IRQHandler`。ISR应辨认来源，进行对应处理；还要同时开启外设中断源与NVIC通道。ISR宜短，避免等待发送完成或执行长延时。

### 10.2.6 USART相关寄存器

| 寄存器 | 复习重点 |
| --- | --- |
| SR | TXE、TC、RXNE、IDLE及错误状态 |
| DR | 发送写入、接收读取；9位无校验时不要提前截成8位 |
| BRR | 12位整数+4位小数分频 |
| CR1 | UE、TE、RE、字长、校验及部分中断使能 |
| CR2 | 停止位、同步时钟、LIN等控制 |
| CR3 | 流控、DMA、半双工及错误控制等 |
| GTPR | 智能卡等用途的保护时间和预分频 |

 **【基础补充·状态清除】** RXNE通常通过读DR清除；TXE通过写DR清除；PE/FE/NE/ORE/IDLE按“先读SR，再读DR”的顺序处理。状态位不是都能写0或写1统一清除。应看具体位语义与库函数允许的参数。

## 10.3 USART相关库函数

### 10.3.1 函数USART_DeInit

`void USART_DeInit(USART_TypeDef *USARTx);`把指定外设复位到默认配置。会影响该外设已有通信设置；不能用它代替开启GPIO或USART工作时钟。

### 10.3.2 函数USART_Init

`void USART_Init(USART_TypeDef *USARTx, USART_InitTypeDef *init);`按结构体配置串口。

| 成员 | 含义 | 本章设置 |
| --- | --- | --- |
| USART_BaudRate | 波特率 | 9600 |
| USART_WordLength | 8/9位字长 | USART_WordLength_8b |
| USART_StopBits | 1、2、0.5、1.5停止位，按模式使用 | USART_StopBits_1 |
| USART_Parity | No/Even/Odd | USART_Parity_No |
| USART_Mode | 独立选择发送、接收 | USART_Mode_Tx \| USART_Mode_Rx |
| USART_HardwareFlowControl | None/RTS/CTS/RTS_CTS | USART_HardwareFlowControl_None |

可先`USART_StructInit(&init)`设默认值再覆盖所需字段。结构体传地址，函数根据指针取成员；按位或连接收发两个选项，不是把两个模式相加猜数值。

### 10.3.3 函数USART_Cmd

`void USART_Cmd(USART_TypeDef *USARTx, FunctionalState state);`用ENABLE/DISABLE控制USART。USART外设使能、发送接收使能、GPIO模式、时钟及NVIC是不同环节。

### 10.3.4 函数USART_SendData

`void USART_SendData(USART_TypeDef *USARTx, uint16_t data);`把一个数据写入DR；本函数本身不等待发送完毕，也不自动发送字符串或十进制文本。连续发多个字节时，应在每次写入前等TXE。

```c
/* 轮询发送；不要在接收ISR中做长时间等待 */
static void USART1_SendByte(uint8_t value)
{
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET) {}
    USART_SendData(USART1, value);
}

static void USART1_WaitComplete(void)
{
    while (USART_GetFlagStatus(USART1, USART_FLAG_TC) == RESET) {}
}
```

第一函数保证DR可写；最后需要确认线路发完时再调用第二函数。轮询会阻塞当前程序，适合简单示范；若外设/时钟配置出错，循环也可能一直等待。

### 10.3.5 函数USART_ReceiveData

标准库实际原型为`uint16_t USART_ReceiveData(USART_TypeDef *USARTx);`，返回接收到的数据字。课件写成u8不宜照搬，因为外设支持9位数据字。8N1时可显式转成`uint8_t`；读取前应判断RXNE，读DR通常会清RXNE。

### 10.3.6 函数USART_GetFlagStatus

`FlagStatus USART_GetFlagStatus(USART_TypeDef *USARTx, uint16_t flag);`查询指定状态，返回SET/RESET。它不等于查询“CPU一定已经响应了中断”，也不自动读取接收数据。

### 10.3.7 函数USART_ClearFlag

`void USART_ClearFlag(USART_TypeDef *USARTx, uint16_t flag);`仅用于该函数支持的软件清除标志，例如TC、LBD、CTS、RXNE；错误/空闲标志须按读SR再读DR的序列处理。接收时直接清RXNE而不取数据可能丢掉数据；正常应读取DR。

 **【易错点】** 不能为了“清中断”任意把TXE或全部SR位传给ClearFlag，更不能只在RX中断里清TC就认为接收事件处理完了。

### 10.3.8 函数USART_ITConfig

`void USART_ITConfig(USART_TypeDef *USARTx, uint16_t source, FunctionalState state);`控制RXNE、TXE、TC、IDLE、PE、LBD、CTS、ERR等允许的中断源。`USART_ITConfig`使用`USART_IT_...`，状态查询使用相应`USART_FLAG_...`，不要混用宏类别。

### 10.3.9 函数USART_GetITStatus

`ITStatus USART_GetITStatus(USART_TypeDef *USARTx, uint16_t source);`结合中断源状态和对应使能检查是否满足中断条件。与GetFlagStatus相比，它还关心相应中断使能；ISR仍需按各事件的要求清除状态或读写DR。

## 10.4 项目实例

### 10.4.1 项目分析

 **【重点·课件任务】** PC把时、分、秒三个数值发送给STM32，STM32显示时间；主程序延续数码管动态扫描、按键调整和TIM6计秒，并增加USART1。

1. 开启GPIOA和USART1时钟。
2. 默认映射：PA9为TX，复用推挽输出；PA10为RX，浮空输入。
3. 设置9600、无校验、8位、1停止位，无硬件流控，发送接收都使能。
4. 用中断接收时，配置USART1_IRQn、开启RXNE中断，再开始工作。
5. 主循环保持显示；接收函数按协议解释数据，校时时避免与TIM6进位交错。

 **【基础补充·连接】** 使用匹配3.3V逻辑的USB转串口时，适配器TX接PA10/RX，适配器RX接PA9/TX，并接GND。不要把RS-232正负电压接口直接接MCU逻辑引脚；板上已有转换电路时按板卡接口连接，不要重复跨接。

课件正文“96 N 8 1”应结合设置图理解为 **9600、8N1** ，不是96 baud。DTR/RTS是否关联复位取决于开发板电路；课件把二者设False来避免其板卡复位，本项目同样关闭流控并按接线检查。

### 10.4.2 项目实施

 **一、理解课件原程序。** 加入`stm32f10x_usart.c`，新建USART.C/USART.H，把串口初始化声明加入头文件。main依次初始化显示、EXTI、TIM6、USART1，在循环里动态扫描。

课件ISR使用静态`unsigned char k`，每次接收先k++，按k%3分配hour/minute/second，回发k并等待TXE。这个例子展示按到达次序解释三个字节，但要看清以下边界：

- 开头清TC并不能证明RXNE已发生；应先辨别接收来源，再读取DR。
- k是8位，255之后回到0；0%3仍是0，会出现连续两次落入“秒”分支的问题。作为总接收计数器可回绕，作为三字段状态应单独使用0、1、2循环变量。
- 直接逐字节修改时间会出现尚未收齐的中间状态；未检查时<24、分秒<60，也没有丢字节后的重同步机制。
- ISR里等待发送会延长中断处理；回发1、2、3是接收计数，不是把“收到的时分秒”原样回显。

 **二、完整初始化与三字节接收讲解示例。** 以下继续使用9600、8N1、USART1和无包头的三个数值字节。补全范围检查、0～2状态循环、错误清除和主循环统一校时，属于教学整理，不冒充课件原码。接口中的`use_irq=0`用于轮询测试，`use_irq=1`用于校时项目。

```c
/* usart_review.h */
#ifndef USART_REVIEW_H
#define USART_REVIEW_H
#include "stm32f10x.h"
void Usart1_Init(uint8_t use_irq);
void USART1_ApplyTime(void);
#endif
```

```c
/* usart_review.c */
#include "usart_review.h"
extern volatile uint8_t hour, minute, second;

static volatile uint8_t pending_hour, pending_minute, pending_second;
static volatile uint8_t time_ready = 0U;

void Usart1_Init(uint8_t use_irq)
{
    GPIO_InitTypeDef gpio;
    USART_InitTypeDef serial;
    NVIC_InitTypeDef nvic;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA |
                          RCC_APB2Periph_USART1, ENABLE);
    gpio.GPIO_Pin = GPIO_Pin_9;
    gpio.GPIO_Speed = GPIO_Speed_50MHz;
    gpio.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_Init(GPIOA, &gpio);
    gpio.GPIO_Pin = GPIO_Pin_10;
    gpio.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &gpio);

    USART_StructInit(&serial);
    serial.USART_BaudRate = 9600U;
    serial.USART_WordLength = USART_WordLength_8b;
    serial.USART_StopBits = USART_StopBits_1;
    serial.USART_Parity = USART_Parity_No;
    serial.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    serial.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_Init(USART1, &serial);

    USART_ITConfig(USART1, USART_IT_RXNE,
                   (use_irq != 0U) ? ENABLE : DISABLE);
    nvic.NVIC_IRQChannel = USART1_IRQn;
    nvic.NVIC_IRQChannelPreemptionPriority = 0U;
    nvic.NVIC_IRQChannelSubPriority = 0U;
    nvic.NVIC_IRQChannelCmd = (use_irq != 0U) ? ENABLE : DISABLE;
    NVIC_Init(&nvic); /* main统一设置优先级分组2 */
    USART_Cmd(USART1, ENABLE);
}

void USART1_IRQHandler(void)
{
    static uint8_t packet[3];
    static uint8_t pos = 0U;
    uint32_t status = USART1->SR; /* 先读SR */
    uint16_t data;

    if ((status & (USART_SR_ORE | USART_SR_NE |
                   USART_SR_FE | USART_SR_PE)) != 0U)
    {
        data = (uint16_t)USART1->DR; /* 再读DR，完成错误清除序列 */
        (void)data;
        pos = 0U; /* 出错后丢弃未完成的一组 */
        return;
    }
    if ((status & USART_SR_RXNE) == 0U) return;
    data = (uint16_t)USART1->DR; /* 读出数据并清RXNE */
    packet[pos++] = (uint8_t)data;
    if (pos == 3U)
    {
        pos = 0U;
        if ((packet[0] < 24U) && (packet[1] < 60U) &&
            (packet[2] < 60U))
        {
            pending_hour = packet[0];
            pending_minute = packet[1];
            pending_second = packet[2];
            time_ready = 1U; /* 只发布完整时间，ISR不等待发送 */
        }
    }
}

void USART1_ApplyTime(void)
{
    uint32_t saved = __get_PRIMASK();
    __disable_irq();
    if (time_ready != 0U)
    {
        hour = pending_hour;
        minute = pending_minute;
        second = pending_second;
        time_ready = 0U;
    }
    __set_PRIMASK(saved); /* 恢复原中断屏蔽状态 */
}
```

 **代码解释：** GPIO模式对应USART硬件功能；USART_Init设置串口参数；RXNE和NVIC两处使能共同建立接收中断。ISR每次只读一个字节，pos按0、1、2循环；收齐且范围有效才发布待应用值。主循环在短临界区内一起更新三个变量，避免被TIM6秒进位或新串口数据打断；不要用`volatile`代替临界区。

这个示例只保存最近一组完整时间；主循环处理前又到一组时，新值会覆盖旧待处理值。无帧头、长度、校验和超时，丢一个字节会使后续三字节分组错位；演示时复位后从完整三字节开始发送。复杂同步协议并非课件提供的内容。

```c
/* 串口校时项目main.c：替换第9章计时main.c */
#include "dsgshow.h"
#include "exti_keys.h"
#include "timer_review.h"
#include "usart_review.h"

volatile uint8_t hour = 9U, minute = 30U, second = 25U;

int main(void)
{
    uint8_t h, m, s;
    uint32_t saved;
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    DsgShowInit();
    EXTIInit();
    TIM6Init();
    Usart1_Init(1U);
    while (1)
    {
        USART1_ApplyTime();
        saved = __get_PRIMASK();
        __disable_irq();
        h = hour; m = minute; s = second;
        __set_PRIMASK(saved);
        DsgShowTime(h, m, s);
        Key_AdjustTask();
    }
}
```

复用第5章delay、第7章dsgshow、第8章exti_keys和第9章timer_review。只保留本例main.c；EXTI、TIM6和USART1的同名ISR各一份。工程需有CMSIS、启动/时钟初始化、GPIO/RCC/EXTI/TIM/USART及misc等相应库文件。代码以STM32F103ZET6和上述课件接线为条件。

 **三、先做最小轮询回显测试。** 若要先确认串口接线与参数，可使用下列独立main.c，配合上面的usart_review.c/.h。它不显示时间，不使用接收中断；收到一个字节就原样回发，便于把串口链路与校时协议分开排查。

```c
/* 独立轮询回显main.c，不与校时main.c同时加入工程 */
#include "usart_review.h"
/* 满足usart_review.c中的外部变量声明，本测试不使用其校时功能 */
volatile uint8_t hour = 0U, minute = 0U, second = 0U;

int main(void)
{
    uint16_t value;
    Usart1_Init(0U);
    while (1)
    {
        if (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) != RESET)
        {
            value = USART_ReceiveData(USART1);
            while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET) {}
            USART_SendData(USART1, value);
        }
    }
}
```

 **四、上位机与串口助手。** 课件上位机采用Visual Basic 6.0：窗体载入查找并初始化可用串口；DTREnable/RTSEnable设False；PC端定时器每秒更新当前时间标签；发送时把时、分、秒作为二进制数值放到数组三个元素中；接收事件取字节数组、转换后显示。课件未给出完整VB源码，这里整理流程，不伪造其完整窗体程序。

串口助手设置9600、8位、1停止位、无校验、无流控，选择开发板对应COM端口；HEX发送/HEX显示，关闭自动附加换行等额外字节。

| 想设置的时间 | HEX发送区输入 | 实际字节 | 常见误解 |
| --- | --- | --- | --- |
| 10:18:30 | `0A 12 1E` | 0x0A、0x12、0x1E | 12和1E是十六进制数，不是十进制12和文本1E |
| 23:59:58 | `17 3B 3A` | 十进制23、59、58 | 23小时不是发送0x23 |
| 8:05:09 | `08 05 09` | 三个数值字节 | 不是字符串`08:05:09`的8个字符 |

 **【基础补充·数值与字符】** HEX模式下输入`0A`发送一个0x0A字节；文本模式输入`0A`发送ASCII 0x30和0x41。HEX输入区中的空格通常只是分隔符，不会发送0x20；是否发送回车换行要看助手选项。课件允许分三次发送或一次发送三字节，协议的关键仍是实际接收顺序与字节数。

 **五、调试和易错点。** 先用最小回显确认COM、TX/RX交叉与共地、逻辑电平、波特率和格式；再检查GPIOA/USART1时钟、PA9/PA10模式、RXNE/NVIC及ISR名称；最后核对HEX发送、三字节顺序、合法范围和是否多发了换行。显示乱码不一定是波特率错，也可能把二进制计数误当成字符显示。

## 本章公式和代码复习清单

| 必会内容 | 自检 |
| --- | --- |
| 串行/并行、同步/异步、三种双工 | 能按线路、时序与同时性分别判断 |
| 异步帧 | 起始低、停止高、LSB先发；8N1共10位 |
| 波特率计算 | 9600时104.167μs/位，约1.04167ms/字节 |
| BRR计算 | 72MHz、9600得到USARTDIV=468.75、BRR=0x1D4C |
| USART时钟 | USART1看PCLK2；USART2/3看PCLK1；不乘TIM的倍频 |
| TXE/TC/RXNE | 可继续写、线路发完、接收可读，三者分别解释 |
| 标志清除 | RXNE读DR；错误/IDLE先SR后DR；不统一套ClearFlag |
| 库函数与头文件 | ReceiveData返回uint16_t；SendData单次写一个数据字 |
| 三字节校时 | 能把10:18:30转换为0A 12 1E，并解释pos循环 |
| 主循环与ISR | ISR短、范围检查、整组发布；校时与进位用短临界区 |

 **自测答案：** 9600、8E1的物理帧为1起始+8有效数据+1校验+1停止，共11位，三字节约3.4375ms；STM32F1对应9位字长+偶校验。TXE=1不推出TC=1；HEX发送0x31是数值49，也恰好是ASCII字符“1”，其含义由应用协议决定。
