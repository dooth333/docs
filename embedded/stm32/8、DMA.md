# 1、概述

## 1、简介

- DMA（Direct Memory Access）直接存储器存取，DMA外设可以直接访问STM32内部存储器的，比如运行内存SRAM，程序存储器flash和寄存器等。

- DMA可以提供外设和存储器或者存储器（SRAM、flash）和存储器之间的高速数据传输，无须CPU干预，节省了CPU的资源

- 12个独立可配置的通道： DMA1（7个通道）， DMA2（5个通道）（通道：数据转运的路径）

- 每个通道都支持软件触发和**特定**的硬件触发（存储器到存储器一般用软件触发，而外设到存储器一般要硬件触发，因为外设不是一直转运而要有一定的时机，比如AD转运时要等AD转换完成后再通过硬件触发）（**特定**意识是，你使用不同的硬件触发源对应的通道是不一样的，比如DA触发有对应的通道与之相对应）

- STM32F103C8T6 DMA资源：DMA1（7个通道）

## 2、存储器映像

![image-20220815103226664](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220815103226664.png)

- ROM：只读存储器，掉电不丢失
- RAM：随机存储器，掉电丢失
- RAM中，运行内存SRAM就是我们定义变量、数组、结构体的地方

## 3、DMA框图

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220816100128391.png" alt="image-20220816100128391" style="zoom:80%;" />

- M3内核是CPU，其他的都可以看作存储器，其中flash是主闪存，SRAM是运行内存，其他外设都可以看作寄存器，也就是一种SRAM的存储器（寄存器可以看作一种特殊的存储器，cpu可以对其进行读写，并且每个寄存器都可以控制外设电路状态，所以寄存器是连接软件和硬件的桥梁，软件读写寄存器就相当于控制硬件的执行）
- 所以DMA转运可以简单归于，从某个地址取内容，再放到另一个地址
- 为了高效的访问，设置了**矩阵总线**，总线左边是**主动单元**，拥有存储器的访问权，右边这些是**被动单元**，只能被左边主动单元读写。
- DCode总线专门用于访问Flash，系统总线是用于访问其他东西的
- DMA1和DMA2也都有对应的总线
- 仲裁器：在通道使用发生冲突时，根据优先级决定使用的先后
- AHB从设备，时DMA自身的寄存器，连接到AHB上，所以DMA即是主动单元又是被动单元
- DMA请求：DMA硬件触发源
- Flash只能读，不能把DMA转运地址设为Flash地址，不能写（只用配置Flash接口控制器对其进行写入）

## 4、DMA基本结构图

![image-20220816102243661](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220816102243661.png)

- 注意DMA时，只能从Flash中读出，不能往Falsh写入

- 起始地址：有外设端金和存储器端，决定数据从哪里来到哪里去
- 数据宽度：指定一次转运要按多大数据宽度进行转运，（字节Byte（8位uint8_t）、半字HalfWord（16位uint16_t）、字Word（32位uint2_t））
- 地址是否自增：为一次转运完成后，下一次转运地址是否要移到下一个位置上（一般发射端寄存器不用自增，否者下次就跑到别的寄存器了。接收端存储器，需要自增每转运一个数据后往后挪一下，防止数据被覆盖）
- 外设寄存器不一定只能存外设的地址，Flash和SRAM地址都能写，只与地址有关，与名字无关
- 传输计数器：用于设置转运几次的，是一个自减计数器，每转运1次减1，到0时DMA不在进行数据转运，并且自增的地址也会恢复到起始地址
- 自动重装器：计数器减0后是否自动回到初值再次开始。（不重装就是单次模式，重装就是循环模式）
- 最下面的时触发控制，有M2M参数控制软件触发还是硬件触发
- 这里的软件触发与以往软件触发（调用某个函数就触发一次）不同，这个是以最快的速度，连续不断的触发DMA，最快速度把传输计数器清零
- 软件触发和循环模式不能同时使用，否者DMA停不下来
- 软件触发一般是，存储器到存储器，而硬件触发时ADC、串口、定时器等等(一般与外设有关，有一定时机)
- 开关控制：就是使能
- 传输计数器写值时必须先关闭DMA再写入

## 5、其他

### 1、DMA请求

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220816105323511.png" alt="image-20220816105323511" style="zoom: 80%;" />

- 每个通道的硬件触发源不同，若使用某个硬件触发源，一定要通过上图找到对应的通道
- 若是软件触发，就可以随意选择通道
- 每个通道都有多个触发源，如何确定使用哪一个触发源呢？取决于你把哪个外设DMA开启了（CMD），若多个都开启了，由于是个或门，多个都可以进行触发。
- 然后优先级判断

### 2、数据宽度与对齐

![image-20220818105449668](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220818105449668.png)

- 若数据宽度一样，就一个一个对应转运。例如：表中的8-8、16-16、32-32
- 若目标宽度比源端宽度大，就在目标宽度前面补0。（高位补0）例如：表中的8-16、8-32、16-32
- 若目标宽度比源端宽度小，一次读多个把高位舍弃。（高位舍弃）例如：表中的16-8、32-8、32-16

### 3、数据转运+DMA

![image-20220818110336896](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220818110336896.png)

- 数据宽度：都是8个字节
- 两边都需要自增 
- 选择计数器选7，因为是7位
- 这个是存储器到存储器，使用软件触发
- 通道可以任意选择

### 4、ADC扫描模式+DMA

![image-20220818115146639](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220818115146639.png)

- 数据宽度：都是16个字节
- AD这边不自增，存储器站点自增
- 选择计数器选7，因为是7位
- 这个是寄存器到存储器，使用硬件触发
- 通道不能任意选择了，要根据图查看对应的

# 2、代码

## 1、关于地址

- 通过`&`符号获取地址时，要在最前面加一个强制转换`(uint32_t)&aa`
- const相当于常量，被const修饰的变量只能读不能写，在stm32中const修饰的常量存储在flash中`const uint8_t aa = 0x06;`
- 创建的普通变量在运行内存SRAM中`uint8_t aa = 0x06;`
- 对于不会改变的大数组可以在前面加const放在flash中，节约SRAM内存
- 外设寄存器地址是固定的，也可以直接取地址查看`(uint32_t)&ADC1->DR`手册可以查看

## 2、相关函数

1、全部

```c
//恢复加载配置
void DMA_DeInit(DMA_Channel_TypeDef* DMAy_Channelx);

//初始化
void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct);

//结构体初始化
void DMA_StructInit(DMA_InitTypeDef* DMA_InitStruct);

//DMA使能
void DMA_Cmd(DMA_Channel_TypeDef* DMAy_Channelx, FunctionalState NewState);

//中断输出使能
void DMA_ITConfig(DMA_Channel_TypeDef* DMAy_Channelx, uint32_t DMA_IT, FunctionalState NewState);

//DMA设置当前计数据寄存器（给传输计数器写数据）
void DMA_SetCurrDataCounter(DMA_Channel_TypeDef* DMAy_Channelx, uint16_t DataNumber); 
**给计数器赋值之前一定先把DMA失能
  * @param  DMAy_Channelx: where y can be 1 or 2 to select the DMA and （选怎DMA和通道）
  *         x can be 1 to 7 for DMA1 and 1 to 5 for DMA2 to select the DMA Channel.
  * @param  DataNumber: The number of data units in the current DMAy Channelx（给计数器赋的值）
  *         transfer.
    
//DMA获取当前数据寄存器（返回传输计数器的值，看还有多少位没有转运）
uint16_t DMA_GetCurrDataCounter(DMA_Channel_TypeDef* DMAy_Channelx);

//获取标志位状态
FlagStatus DMA_GetFlagStatus(uint32_t DMAy_FLAG);
**参数，只列举了其中四个其他可以去函数定义查看
  *     @arg DMA1_FLAG_GL1: DMA1 Channel1 global flag. 				全局标志位
  *     @arg DMA1_FLAG_TC1: DMA1 Channel1 transfer complete flag. 	转运完成标志位
  *     @arg DMA1_FLAG_HT1: DMA1 Channel1 half transfer flag. 		转运过半标志位
  *     @arg DMA1_FLAG_TE1: DMA1 Channel1 transfer error flag. 		转运错误标志置位

//清除标志位状态
void DMA_ClearFlag(uint32_t DMAy_FLAG);
**参数和上面DMA_GetFlagStatus参数一样

//获取中断状态
ITStatus DMA_GetITStatus(uint32_t DMAy_IT);

//清除中断挂起位
void DMA_ClearITPendingBit(uint32_t DMAy_IT);
```

2、初始化结构体

```c
	DMA_InitTypeDef DMA_InitStructure;

//外设站点的起始地址
	DMA_InitStructure.DMA_PeripheralBaseAddr = ;
**这个参数一般不是固定值，通过函数参数传入
    
//外设站点的数据宽度
	DMA_InitStructure.DMA_PeripheralDataSize = ;
**参数
	#define DMA_PeripheralDataSize_Byte        ((uint32_t)0x00000000)  字节对应的的uint8_t（8位）
	#define DMA_PeripheralDataSize_HalfWord    ((uint32_t)0x00000100)  半字对应的的uint16_t（16位）
	#define DMA_PeripheralDataSize_Word        ((uint32_t)0x00000200)  字对应的的uint32_t（32位）
    
//外设站点的是否自增
	DMA_InitStructure.DMA_PeripheralInc = ;
**参数
    #define DMA_PeripheralInc_Enable           ((uint32_t)0x00000040)  自增
	#define DMA_PeripheralInc_Disable          ((uint32_t)0x00000000)  不自增
    
//存储器站点的起始地址
	DMA_InitStructure.DMA_MemoryBaseAddr = ;
**这个参数一般不是固定值，通过函数参数传入
    
//存储器站点的数据宽度
	DMA_InitStructure.DMA_MemoryDataSize = ;
**参数：
    #define DMA_MemoryDataSize_Byte            ((uint32_t)0x00000000)	字节对应的的uint8_t（8位）
	#define DMA_MemoryDataSize_HalfWord        ((uint32_t)0x00000400)	半字对应的的uint16_t（16位）
	#define DMA_MemoryDataSize_Word            ((uint32_t)0x00000800)	字对应的的uint32_t（32位）
    
//存储器站点是否递增
	DMA_InitStructure.DMA_MemoryInc = ;
**参数
    #define DMA_MemoryInc_Enable               ((uint32_t)0x00000080)	自增
	#define DMA_MemoryInc_Disable              ((uint32_t)0x00000000)	不自增
    
//传输的方向
	DMA_InitStructure.DMA_DIR = ;
**参数
    #define DMA_DIR_PeripheralDST              ((uint32_t)0x00000010)	外设站点作为目的地（存储器到外设站点）
	#define DMA_DIR_PeripheralSRC              ((uint32_t)0x00000000)	外设站点作为数据源头（外设站点到存储器）

//缓存区大小（传输计数器）
	DMA_InitStructure.DMA_BufferSize = ;
**以数据单元指定缓存区大小，要传输几次0~65535
**一般也是通过函数参数传入

//传输模式（是否自动重装）
	DMA_InitStructure.DMA_Mode = ;
**如果在选定的通道上配置了存储器到存储器数据传输，则无法使用循环（自动重装）模式，就是自动从装和软件触发不能同时使用
**参数
    #define DMA_Mode_Circular                  ((uint32_t)0x00000020)	自动重装模式（循环模式）
	#define DMA_Mode_Normal                    ((uint32_t)0x00000000)	正常模式

//是否是存储器到存储器（软件触发还是硬件触发）
	DMA_InitStructure.DMA_M2M = ;
**参数
    #define DMA_M2M_Enable                     ((uint32_t)0x00004000)	软件触发
	#define DMA_M2M_Disable                    ((uint32_t)0x00000000)	硬件触发

//优先级（按照参数要求给你一个优先级就行）
	DMA_InitStructure.DMA_Priority = ;
**参数
    #define DMA_Priority_VeryHigh              ((uint32_t)0x00003000)	非常高
	#define DMA_Priority_High                  ((uint32_t)0x00002000)	高
	#define DMA_Priority_Medium                ((uint32_t)0x00001000)	中等
	#define DMA_Priority_Low                   ((uint32_t)0x00000000)	低

```

3、初始化函数

```c
DMA_Init();
/**
  * @brief  Initializes the DMAy Channelx according to the specified
  *         parameters in the DMA_InitStruct.
  * @param  DMAy_Channelx: where y can be 1 or 2 to select the DMA and 
  *   x can be 1 to 7 for DMA1 and 1 to 5 for DMA2 to select the DMA Channel.
  * y选择1或2确认是DMA1还是DMA2；x选择通道，DMA1对应通道1到7，DMA2对应通道1到5
  * @param  DMA_InitStruct: pointer to a DMA_InitTypeDef structure that
  *         contains the configuration information for the specified DMA Channel.
  * @retval None
  */
void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct)
```

4、开启DMA触发信号

```
void ADC_DMACmd(ADC_TypeDef* ADCx, FunctionalState NewState);
```



## 3、初始化

![image-20220816102243661](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220816102243661.png)

1、第一步：RCC开启DMA时钟

2、第二步： 调用DMA——Init初始化参数

3、开关控制 DMA_Cmd

4、注意：要使用的是硬件触发，要记得调用XXX_DMACmd,开启触发信号输出

5、注意：数组名就是地址，就不用再取地址

