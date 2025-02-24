# 0、作业内容

1. 理解什么是 ADC 及其工作原理
2. 理解 DMA 的原理和使用
3. 通过设计程序，使得 CPU 和 DMA 同时工作（DMA 双缓冲机制）
4. 初步设计实现程序架构，懂得如何使用静态分析工具
5. 发现没有线程同步时的弊端

# 一、写在开头

1. ## **数据手册、参考手册、****hal****库用户手册！！（看懂硬件结构对理解代码至关重要！）**

# 二、知识学习

1. ## ADC硬件原理

   1. ###  并联比较型

   ![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2MyMTdmOTQ1YmY5NWJjYjdmOTI5YmE5Njg0ODcyYWJfMVc2VXRZbWZ4V0pYb1JoQkE0SHE5VEY4TzdOOTd5Q1dfVG9rZW46TkphWmJVUzRrb0QyWTl4dzQ4TGMyOXpFbmJmXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

   1. ###  逐次比较型

      ###   需要n个时钟周期，完成n-bit的量化，分辨率为Vref/2^n,中高转换精度。

   ![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=M2NjOThlMzNlMzU1ZTM3MjE1MWU3MTUwMDI1MjlkN2ZfZVJXWE1qZnlYaDY5U0FVbHdndG9TZGFlZm9od3pEOFVfVG9rZW46SUJ6S2JlVjJRb1VtMGR4Qll1QmNzcGxYbnVlXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. ## ADC使用

1. 在stm32Cubemx中配置为手动触发、只采集1次时，需要在hal库中找到开启ADC的函数，写在任务中。每次采集都要开启ADC。
2. 状态寄存器

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=OGVlM2ZkY2ViZGNmMGQ5ZTJlNmMxYzBmOTE5ZTE1ZGZfQ0FSSmNoZnYzbFVZSXpBYjVIVlpsbmtZSFg1NjhaOTdfVG9rZW46UTkyM2JvODI0b0dMTU14anN2WWNzYjFxbkRnXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. ## DMA 的原理和使用

1. ### 原理

直接将某个数据通过串口发送，需要CPU参与，将数据从SRAM → CPU → 外设数据寄存器（例如串口）

datasheet

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=OWY0MWRjMjY0Y2Q3NGRkZjc2MmQ1ZTZkZDA1MzZlZTRfRGpuSDFVbjBZV3QwV2tacUpyTGR2cjQ1bVRHMDB5SXRfVG9rZW46TG95QWI4MG80b1E2Slh4ajFVV2NuMFZibmFkXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

而使用DMA，DMA可以直接从SRAM搬运数据，转移数据时不需要经过CPU。

**配置：**

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=YTg0ZWUwYzkwODkzZmMyZmNkNDMyOTlmMDFkNWNmZmZfZkVWTmVTR2hSbHRVUDZhbGxYcmN1TXhjZ0kwcEpLSThfVG9rZW46WElRYmI2aERTb1pCc2l4ZVN2bWNNUkpnbnVoXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

DMA_Sram+: 内存地址是否自增

DMA_Peri+：外设地址是否自增

1. ### 代码

1. 双ADC同步模式下，ADC2的中断由ADC1完成中断触发，无需开启ADC2的全局中断，无需配置ADC2的DMA（使用DMA时）
2. 双ADC模式下，不能使用相同的采集通道？
3. 使用1个DMA（半满发送，相当于两个缓冲区），双ADC同步采集，即ADC1采集完触发ADC2采集，采集的数据为16位，因此，存储的内容如下

| 高地址内存（32位） | ……   | ADC2（高16位） ADC1（低16位） | ADC2（高16位） ADC1（低16位） | ADC2（高16位） ADC1（低16位） | 起始地址 低地址内存（32位） |
| ------------------ | ---- | ----------------------------- | ----------------------------- | ----------------------------- | --------------------------- |
|                    |      |                               |                               |                               |                             |

1. datasheet引脚

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=NTkzNzVkMmM1OGUzMDhjMDBlNzQxYTVlOGZhMDFlM2ZfOUdKMENqOWVwM1d1NlVyT0RSeW9iVTdzQ0IzdW1nY3hfVG9rZW46UlFSNWJYQ0FLb0pNSXZ4Nm4xYWNhZVFZblRoXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. ## ADC  -  DMA

HAL库中已经给出了配置ADC-DMA、开启ADC-DMA的函数；

1. DMA时钟

   1. ```C
        * Enable DMA controller clock
        */
      void MX_DMA_Init(void)
      {
      
        /* DMA controller clock enable */
        __HAL_RCC_DMA2_CLK_ENABLE();
      
        /* DMA interrupt init */
        /* DMA2_Stream0_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA2_Stream0_IRQn, 5, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream0_IRQn);
      
      }
      ```

2. ADC配置

   1. ```C
        ADC_ChannelConfTypeDef sConfig = {0};
      
        /* USER CODE BEGIN ADC1_Init 1 */
      
        /* USER CODE END ADC1_Init 1 */
      
        /** Configure the global features of the ADC (Clock, Resolution, Data Alignment and number of conversion)
        */
        hadc1.Instance = ADC1;
        hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
        hadc1.Init.Resolution = ADC_RESOLUTION_12B;
        hadc1.Init.ScanConvMode = DISABLE;
        hadc1.Init.ContinuousConvMode = DISABLE;
        hadc1.Init.DiscontinuousConvMode = DISABLE;
        hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
        hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
        hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
        hadc1.Init.NbrOfConversion = 1;
        hadc1.Init.DMAContinuousRequests = DISABLE;
        hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
        if (HAL_ADC_Init(&hadc1) != HAL_OK)
        {
          Error_Handler();
        }
      
        /** Configure for the selected ADC regular channel its corresponding rank in the sequencer and its sample time.
        */
        sConfig.Channel = ADC_CHANNEL_1;
        sConfig.Rank = 1;
        sConfig.SamplingTime = ADC_SAMPLETIME_3CYCLES;
        if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK)
        {
          Error_Handler();
        }
        /* USER CODE BEGIN ADC1_Init 2 */
      
        /* USER CODE END ADC1_Init 2 */
      
      }
      ```

3. 其中`HAL_ADC_Init(&hadc1)`调用了`HAL_ADC_MspInit`，

   1. ```C
      void HAL_ADC_MspInit(ADC_HandleTypeDef* adcHandle)
      {
      
        GPIO_InitTypeDef GPIO_InitStruct = {0};
        if(adcHandle->Instance==ADC1)
        {
        /* USER CODE BEGIN ADC1_MspInit 0 */
      
        /* USER CODE END ADC1_MspInit 0 */
          /* ADC1 clock enable */
          __HAL_RCC_ADC1_CLK_ENABLE();
      
          __HAL_RCC_GPIOA_CLK_ENABLE();
          /**ADC1 GPIO Configuration
          PA1     ------> ADC1_IN1
          */
          GPIO_InitStruct.Pin = GPIO_PIN_1;
          GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
          GPIO_InitStruct.Pull = GPIO_NOPULL;
          HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
      
          /* ADC1 DMA Init */
          /* ADC1 Init */
          hdma_adc1.Instance = DMA2_Stream0;
          hdma_adc1.Init.Channel = DMA_CHANNEL_0;
          hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
          hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
          hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
          hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
          hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
          hdma_adc1.Init.Mode = DMA_NORMAL;
          hdma_adc1.Init.Priority = DMA_PRIORITY_LOW;
          hdma_adc1.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
          if (HAL_DMA_Init(&hdma_adc1) != HAL_OK)
          {
            Error_Handler();
          }
      
          __HAL_LINKDMA(adcHandle,DMA_Handle,hdma_adc1);
      
        /* USER CODE BEGIN ADC1_MspInit 1 */
      
        /* USER CODE END ADC1_MspInit 1 */
        }
      }
      ```

4. 以上都由stm32Cubemx配置好。

5. 开启ADC及DMA，使用ADC hal库中的

 `HAL_ADC_Start_DMA(&hadc1, buffer1, BUFFER_SIZE);`

由于设置了手动开启，每次只采集1次数据，所有每次采集数据，都要使用此函数。

1. ## 邮箱（橱窗）

FreeRTOS中的邮箱是一个长度为1的队列，写入时使用xQueueOverwrite()，覆盖式写入；查看使用xQueuePeek()；（中断中使用带FromISR）；查看使用，不会删除数据；写入一次后永远能读出数据。

1. ## 任务通知

1. **xTaskNotifyWait**

`xTaskNotifyWait` 用于一个任务等待来自其他任务或中断的通知。

#### **原型：**

```
BaseType_t xTaskNotifyWait(uint32_t ulBitsToClearOnEntry,uint32_t ulBitsToClearOnExit,uint32_t *pulNotificationValue, ``    TickType_t xTicksToWait);
```

#### **参数说明：**

- `ulBitsToClearOnEntry`：进入函数时需要清除的任务通知值的位，常用 `0xFFFFFFFF` 来清除所有位。
- `ulBitsToClearOnExit`：退出函数时需要清除的任务通知值的位。
- `pulNotificationValue`：指向一个变量，用于接收任务通知的 32 位值（实际传递的数据）。
- `xTicksToWait`：等待通知的超时时间（以系统滴答为单位）。

1. **xTaskNotifyFromISR**

`xTaskNotifyFromISR` 用于从中断向任务发送通知。

#### **原型：**

```
BaseType_t xTaskNotifyFromISR( ``    TaskHandle_t xTaskToNotify,uint32_t ulValue, ``    eNotifyAction eAction, ``    BaseType_t *pxHigherPriorityTaskWoken);
```

#### **参数说明：**

- `xTaskToNotify`：需要通知的任务句柄。

- `ulValue`：传递的 32 位值。

- `eAction`：任务通知的行为类型，可选值如下：

  - `eNoAction`：不改变任务通知值，只通知任务。
  - `eSetBits`：设置通知值中的某些位（按位或）。
  - `eIncrement`：将通知值加 1。
  - `eSetValueWithOverwrite`：覆盖通知值为 `ulValue`。
  - `eSetValueWithoutOverwrite`：如果通知值未被读取，则不覆盖。

- `pxHigherPriorityTaskWoken`：指向一个变量，如果唤醒了一个优先级更高的任务，则此变量被置为 `pdTRUE`。

  - ```C
    void ISR_Handler(void)
    {
        BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
        // 通知任务，并传递值 123
        xTaskNotifyFromISR(xTaskHandle, 123, eSetValueWithOverwrite, &xHigherPriorityTaskWoken);
    
        // 如果唤醒了更高优先级的任务，则触发上下文切换
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
    ```

# 三、实验内容

代码：https://gitee.com/jingezhengaoxing/t714-f4-project.git

1. 【线程间同步漏洞】ADC+DMA在RTOS下高速采集数据完整性和实时性的架构设计

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=MGU5NTRkZjhiY2JjMGE4MjBmY2Q1ODAxMzZmYmMzMzdfc2FURU1Qb3hpd3ZBb3hoNkVYYlZkYm5wV1ZFcEFHc0pfVG9rZW46RlVISGJqbHQ1bzk4dnp4NXJGVWNSTjZmbjhCXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. RTT输出内容不全问题：在RTT配置文件中修改上行缓冲区大小/改为阻塞式（默认为跳过塞不下的数据）。

1. 电压转换：根据ADC配置中的`hadc1.Init.Resolution = ADC_RESOLUTION_12B;`可知有效位为12bit，即最大为2^12-1=4095，根据硬件电路知参考电压为0-3.3V，因此实际电压为 采集值 *3.3V /4095。

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=ODk2ZmQ5NTIwOGY1N2UxZDhiNWQzZDhmNDk0NWU2NDFfaEVEV1J2S0FTTlYzZ2d6TjlaazdCZVN3VmFDbzJiUkJfVG9rZW46VmNGTWI1T3c3b2RneER4MElyMWNrQkxmbmVmXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. 通过Understand分析adc采集任务流程图

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=M2Y3NjQ1ZGVlNjY5MjBjOWJlYWE2NTMwZTNhMzVkMmNfc0NrRDRvOWREemdEMVgzYzhid1NHNGdNM0ZwbGllb0dfVG9rZW46S0FtbGJpVXpTb3dUQ054QzljSmMzNlo2bnpnXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=YmNjMmQ2OTliNmE1ZmY3ZmE0MTJhZTA3NjQxOGFiMzJfektMd0lSUHN5VVR0U2JMTDhlb0dGOWRabm1wZUllWjJfVG9rZW46SzFtbGJmTUZTbzdCZUR4V2xhSWNpVVBmbkpPXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDRhN2I3YjUxMDI0NzRmYTNjM2E5YTIyM2NhOGMzYzVfa2lPWjYwVHl1a0lJVXFGWlQyc29wSU1VTUVsUGhWVFlfVG9rZW46SzV6cGJFd2pHb1Vsem54MEgyTmNvSnMzbkJoXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

# 四、面试问题准备

1. ## 邮箱和队列有什么区别？

发送数据大小：邮箱只有一个32位数据作为信息空间，而队列可任意设置；

写入方式：邮箱可以覆盖写入，可以操作数据位；队列在空间满时无法写入；

**每个任务在创建时都有自己的邮箱(Notify时）**，而队列邮箱需要单独创建

邮箱是直接通知到任务的（Notify时），而队列邮箱则需要任务去访问

1. ## 邮箱和信号量有什么区别？

- 如果是基于Notify（任务通知）实现的邮箱，每个任务在创建时都有自己的邮箱，而信号量需要

单独创建

- 邮箱是直接通知到任务的，而信号量则需要任务去访问，通过邮箱解除RTOS 任务阻塞状态的速

度比信号量相快 45%，且使用的 RAM 更少

- 邮箱可以作为二值信号量用来同步与互斥，也可以作为计数信号量计数，而信号量只能完成其本

身功能

1. ## 邮箱是否需要额外资源（比如队列，需要创建队列）？

不需要。在创建任务的同时也创建了任务通知。

1. ## 回顾什么是 data 段，什么是 bss 段，什么是堆、什么是栈，并在 stm32f4 中，在数据手册中，找到这些区域的地址段

- .data段：用于存储已初始化的全局变量和静态变量，位于RAM中，程序启动时，这些变量从

Flash复制到RAM中，即在程序加载时分配内存和初始化值

- .bss段：用于存储未初始化的全局变量和静态变量，位于RAM中，程序启动时，BSS段中的变量会自动初始化为零，不会占用Flash空间
- 堆：动态分配内存的区域，程序在运行时可以随意分配和释放
- 栈：用于存储局部变量和函数调用信息的内存区域

1. ## ADC 的 ScanConvMode 参数什么意思？如果我只有一个通道，需要 Enable 吗？

    多通道循环检测。不需要。

1. ## ContinuousConvMode 参数的含义？

    一直检测，不停。与单次检测区分。

1. ## EOCSelection 参数的含义？

是标志位，end of conversion。用于选择在合适置位E0C标志位，比如**是一个通道采集完后置位标志位，还是所有通道都转换完后置位标志位**。置位标志位后会通知NVIC**进行中断悬起**，然后CPU再响应NVIC进入异常，进行中断服务。

1. ## （选做：中阶面试话题）关于采样时间，为什么越长越准确？能从 ADC 的构造解释一下吗？

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=YTkwYmZlYzg0NjdmYTQ0OGEwOGNhN2E2YjExMTY5MWRfYU91QVpFUVMyOFlNelFObzIzQmhkZFNXNHJ4ZmZFdWRfVG9rZW46WTNpN2J1QXBjb3N2VlB4U0pKMGNVUURkbnRkXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

使用逐次比较时，实际上是通过二分法进行检测的，每次二次都会使分辨率更高，不过也因此需要在前一次的检测基础上进行下一次二分检测，每次进行电压比较都需要时间稳定DAC的电压（由单片机产生的比较电压），而比较后的值都存放在SAR寄存器中，每次比较完成都会移动到下一位进行比较。

采样时间长：DAC输出更稳定、比较器反应更准确、减少噪声、电路元件稳定

使用1个DMA（半满发送，相当于两个缓冲区），双ADC同步采集，即ADC1采集完触发ADC2采集，采集的数据为16位，因此，存储的内容如下，对吗？

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=MTk4OWUxNjQ3ZTU5NzQ0YTViZDg4OGI5N2Q0NDI4NDZfV2o4dkt1NDNaRGlNR1VtR2JTVWRVM2MyMklsS3VZNm9fVG9rZW46QU5weGJQUnRTb0c5U1V4RjdxMGNlRmZWbmZoXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

这里每次进入任务都会配置一次，难道不会导致DMA传输的数据重新从buffer的0处开始存储吗

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=MTEyMDQ5YTgyYzQyZjQyNjc0YzE3YzMzNzI3OWZlZDVfdjM0eTRWczVCU1Q3a1RvV2dOUWFvc0lSR0dZd2ZPR1VfVG9rZW46UW80VmJ0dkZTb2pmM214U2lhSGM4dkZZblVjXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

其他（已解决）

1. 单片机处理速度上限多数时候是不是取决于CPU频率，而AHB总线速度一般远大于需要传送的数据频率？
   1.  AHB 总线速度通常远快于数据传输需求，CPU 频率更可能成为性能瓶颈。
2. 通过定时器可以触发ADC转换，但是不需要开启TIMer的全局中断，意思是定时周期到了之后，自动触发ADC转换，而且不会经过CPU？
   1.  定时器可以直接触发 ADC 转换，且不需要全局中断或 CPU 干预。
3. 双ADC模式下，不能使用相同的采集通道？
   1.  双 ADC 模式不能使用相同的采集通道，否则会导致资源冲突。

1. 当使用1个DMA缓冲区存储数据，采集32次数据触发一次DMA中断，并直接在中断中RTT输出，但发现总是输出不全，一般输出到第25个数据就开始新一轮的输出了。如果采集的数据少一些就能输出完全。是和之前的printf串口打印一样也被后面的打印字符吞了吗？
   1.  将RTT缓冲区配置为更大后，能够输出完整。所以应该是因为缓冲区不够。
   2. 
   3.  RTT缓冲区是在哪里？上限是多少？
   4.  SRAM，根据内存容量而定。

一般解决方法：

 **扩大 RTT 缓冲区**：如你所述，通过将 RTT 缓冲区配置为更大后，能够输出完整数据，因此你可以增加缓冲区大小，避免数据被覆盖。

 **优化输出频率**：如果可能，减少输出的频率或分批输出数据，避免过量数据堆积。

 **将输出移至主循环**：避免在中断中直接进行大量的输出操作。可以在中断中将数据传递到队列或信号量中，在主任务中批量输出

1. 为什么ADC每次采集的第一个数据都要比后面的数据大一些？

**电容充电效应：**

- ADC 通道的输入端有一个采样保持电容，这个电容用于存储输入电压值。在开始采样之前，如果该电容未完全充电，初始采样的值会偏高，随后随着电容的充电，采样值会趋于稳定。
- 如果输入电压的变化较快或存在噪声，首次采样时的值可能会比后续的稳定值大。

**多通道切换：**

- 如果在多通道模式下切换 ADC 通道，可能会有类似的现象。在切换通道后，采样保持电容需要一定的时间来稳定当前通道的输入电压，导致第一次采样的结果偏高。

**输入信号不稳定：**

- 如果 ADC 的输入信号本身不稳定，特别是输入信号的上升或下降速度过快，也会导致 ADC 采集的第一个值与后续值有所不同。

#### **如何解决：**

- **增加采样时间**：可以通过增加 ADC 的采样时间，使得采样保持电容有足够时间来稳定。可以通过 `ADC_SAMPLETIME` 配置来增加采样时间。
- **空采样**：在正式采集数据之前，进行一次空采样，以使电容稳定。
- **调整输入信号**：确保 ADC 输入信号稳定，减少噪声。

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=M2ZlZTJjNTlkMWI4YWU4ZWZjMjYzZmUxMGFlYjNmNTJfSUxTZGpYcEpGa0h0UE81ZjM4dzBTazlrQ3JFeXlRQkxfVG9rZW46QTJ6SWJmZHZTb3VNcTB4UUJqZWM2RUhCbk5kXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

1. 是不是几乎所有的配置（例如ADC通道、DMA通道、中断等）都可以在状态/控制寄存器中看到？而ADC和DMA配置后的关联还是要看数据手册对吗？

是的，关联关系如图（**Reference manual**）

![img](https://ncnoby04sj0b.feishu.cn/space/api/box/stream/download/asynccode/?code=N2E1NDNiMmEyZGE1ZWI5MzQxMzk3MTBiMDE0ZWVkMWJfTGNsTUdxMU52aUpGWWs5dGZOY3ZQTzVDMVVnbUZGb1VfVG9rZW46U0NRamJzSDhrb0d0MUx4SDJpbGNOUEhlbkRoXzE3NDA0MDk2MTA6MTc0MDQxMzIxMF9WNA)

队列能否覆盖写入？

默认情况下 **不能覆盖写入**，当队列已满时，再尝试写入队列会返回 `errQUEUE_FULL` 错误，写入操作会失败。