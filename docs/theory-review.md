# STM32 理论复习 · 第 1～10 章

课件配套板使用 STM32F103ZET6。读代码时，先找操作对象，再从参数和电路判断这一步的效果。

```c
GPIO_ResetBits(GPIOC, GPIO_Pin_0);  // 把 PC0 输出置低，点亮阳极接 3.3 V 的 L1
```
