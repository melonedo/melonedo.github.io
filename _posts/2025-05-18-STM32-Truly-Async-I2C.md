---
layout: post
title: STM32 真正异步 I2C
date: 2025-05-18 00:22:00 +0800
tags:
- 编程
- 嵌入式
render_with_liquid: false
---

翻浏览记录看到这个，记录一下免得忘了。

## 问题

STM32 的 HAL 库中 I2C 的驱动在进入 I2C 前有一步是检查 I2C 总线是否被占用：

```c
// https://github.com/STMicroelectronics/stm32f1xx-hal-driver/blob/38d14024a9d3f6802506c3ec7a4308563760e54e/Src/stm32f1xx_hal_i2c.c#L1744-L1765
HAL_StatusTypeDef HAL_I2C_Master_Transmit_IT(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint8_t *pData, uint16_t Size)
{
  __IO uint32_t count = 0U;

  if (hi2c->State == HAL_I2C_STATE_READY)
  {
    /* Wait until BUSY flag is reset */
    count = I2C_TIMEOUT_BUSY_FLAG * (SystemCoreClock / 25U / 1000U);
    do
    {
      count--;
      if (count == 0U)
      {
        // ...
        return HAL_BUSY;
      }
    }
    while (__HAL_I2C_GET_FLAG(hi2c, I2C_FLAG_BUSY) != RESET);
    // ...
  }
  // ...
}
```

可以看到，这一步中，驱动会等待固定的`I2C_TIMEOUT_BUSY_FLAG`（=25）时间。根据其他部分的代码，含义为 25ms，这里按照每次循环 25 个时钟周期算，再除了一个 25。

不过再看一下这个函数的命名，这个函数以`_IT`结尾，理应迅速返回，实际上当 I2C 总线上有其他主设备时，会持续等待直到总线空闲才返回。这意味着使用该函数时有可能等待非常长的时间，影响其他功能运行。所有的 I2C 函数，包括中断处理时，均可能等待 25ms。

当总线上没有其他的主设备时，这个位应该和驱动中自己的状态位同步，不需要等待。一个很常见的情况是 I2C 没有设置上拉，导致总线一直被占用，也会在这里报错总线忙。

## 解决

Stackoverflow 上有人也有同样的疑问：https://stackoverflow.com/questions/72840119/stm32-hal-how-to-read-i2c-memory-in-truly-non-blocking-mode

这实际上是 STM32 的硬件限制导致的。表示总线占用的 BTF 位没有中断，只能通过轮询的方法管理，需要通过 RTOS 线程或是事件循环来管理。

这是该问题下的一个解决方案：https://gist.github.com/HTD/e36fb68488742f27a737a5d096170623
