# STM32中断

### 概述：

1、包括68个可屏蔽中断通道，包含EXTI、TIM、ADC、USART、SPI、I2C、RTC等多个外设

2、使用NVIC统一管理中断，每个中断通道都拥有**16个可编程的优先等级**，可对优先级进行分组，进一步设置抢占优先级和响应优先级

3、分布图：

![image-20220805092913902](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220805092913902.png)

![image-20220805093007056](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220805093007056.png)

![image-20220805093023349](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220805093023349.png)

![image-20220805093035528](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220805093035528.png)

- 其中灰色的是内核中断，比较高深一般不用。
- WWDG为看门狗中断，用来检测程序运行状态的中断，比如程序卡死会跳入这个中断。、
- PVD中断，发现电压不足时进入该中断。
- 其中EXTI1~EXTI4 、EXTI9_5、EXTI5_10都是外部中断。
- 后面地址，是中断跳转的地址

### NVIC：

#### 作用：

1、用于分配中断优先级。

#### NVIC优先级分组：

- NVIC的中断优先级由优先级寄存器的4位（0~15）决定（**值越小中断优先级越高**），这4位可以进行切分，分为高n位的**抢占优先级**和低4-n位的**响应优先级**。

- 抢占优先级高的可以中断嵌套（特别紧急可以打断正在看病的病人），响应优先级高的可以优先排队（紧急但不是非常紧急的，优先排队等正在看的开完后立刻进去看），抢占优先级和响应优先级均相同的按**中断号**（上表中的第一列的号码）排队。

- 我们可选的分组方式：

  | **分组方式** | **抢占优先级**  | **响应优先级**  |
  | ------------ | --------------- | --------------- |
  | 分组0        | 0位，取值为0    | 4位，取值为0~15 |
  | 分组1        | 1位，取值为0~1  | 3位，取值为0~7  |
  | 分组2        | 2位，取值为0~3  | 2位，取值为0~3  |
  | 分组3        | 3位，取值为0~7  | 1位，取值为0~1  |
  | 分组4        | 4位，取值为0~15 | 0位，取值为0    |



## 1、EXIT外部中断

### 1、概述：

1、作用：•EXTI（Extern Interrupt）外部中断：EXTI可以监测指定GPIO口的电平信号，当其指定的GPIO口产生电平变化时，EXTI将立即向NVIC发出中断申请，经过NVIC裁决后即可中断CPU主程序，使CPU执行EXTI对应的中断程序

2、支持的触发方式：上升沿/下降沿/双边沿（上升和下降沿都触发）/软件触发（引脚啥事没有，程序执行一句代码进行中断）

3、支持的GPIO口：所有GPIO口，但相同的Pin不能同时触发中断。如：GPIOA0、GPIOB0、GPIOC0这几个的Pin都为0只能选A、B、C其中一个用于EXTI，不能同时使用。**原因是**：AFIO中断引脚选择作用的结果。

4、通道数：16个GPIO_Pin，外加PVD输出、RTC闹钟、USB唤醒、以太网唤醒 （20个，主要学习16个引脚的）

5、触发响应方式：中断响应（执行中断函数）/事件响应（触发事件用于触发其他外设）

6、**基本结构：**

![image-20220805101449578](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220805101449578.png)



- 防止太占用NVIC就把EXTI9`~`EXTI5放入了一个通道，把EXTI15`~`EXTI10放入了同一个通道。
- 右下角20路到其他外设就是用于上述事件响应。

### 2、红外计次

#### 1、对设式红外传感器：

- 被遮挡D0输出0，未遮挡输出1

#### 2、外部中断配置：

- 通过基本结构图来配置：

##### 1、配置RCC：

把涉及到的引脚时钟打开。

##### 2、配置GPIO

把端口设置为输入模式。

- 可以去参考手册8.1.11看外设的GPIO配置表。

- ```
  EXTI输入线： 外部中断输入 浮空输入或带上拉输入或带下拉输入
  ```

  

##### 3、配置AFIO：

把我们用到的那一路GPIO连接到后面的EXTI

```C
void GPIO_AFIODeInit(void);//用于复位AFIO
void GPIO_PinLockConfig(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);//用于锁定GPIO配置，防止意外更改
void GPIO_EventOutputConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);//用于配置AFIO事件输出功能（不常用）
void GPIO_EventOutputCmd(FunctionalState NewState);//用于配置AFIO事件输出功能（不常用）
void GPIO_PinRemapConfig(uint32_t GPIO_Remap, FunctionalState NewState);//用于引脚重映射，GPIO_Remap：为重映射的方式，NewState：新的状态（重要）
void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);//配置AFIO的数据选择器，来选择我们想要中断的引脚（重要）
void GPIO_ETH_MediaInterfaceConfig(uint32_t GPIO_ETH_MediaInterface);//关于以太网（用不到）

```

本次主要用`void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);`

```c
/**
  * @brief  Selects the GPIO pin used as EXTI Line.配置AFIO的数据选择器，来选择我们想要中断的引脚（重要）
  * @param  GPIO_PortSource: selects the GPIO port to be used as source for EXTI lines.
  *   This parameter can be GPIO_PortSourceGPIOx where x can be (A..G).参数1，为GPIO_PortSourceGPIOx （x为A到G）
  * @param  GPIO_PinSource: specifies the EXTI line to be configured.
  *   This parameter can be GPIO_PinSourcex where x can be (0..15).参数2：为GPIO_PinSourcex（x为0到15的pin值）
  * @retval None
  */
void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource)
{
  uint32_t tmp = 0x00;
  /* Check the parameters */
  assert_param(IS_GPIO_EXTI_PORT_SOURCE(GPIO_PortSource));
  assert_param(IS_GPIO_PIN_SOURCE(GPIO_PinSource));
  
  tmp = ((uint32_t)0x0F) << (0x04 * (GPIO_PinSource & (uint8_t)0x03));
  AFIO->EXTICR[GPIO_PinSource >> 0x02] &= ~tmp;
  AFIO->EXTICR[GPIO_PinSource >> 0x02] |= (((uint32_t)GPIO_PortSource) << (0x04 * (GPIO_PinSource & (uint8_t)0x03)));
}

```

##### 4、配置EXTI：

选择**触发方式**（上升沿、下降沿、双边沿），选择**触发方式**，是**中断响应**还是事件响应。

关于EXTI的函数：

```c
void EXTI_DeInit(void);//把EXTI的配置都清除，恢复成上电的默认状态
void EXTI_Init(EXTI_InitTypeDef* EXTI_InitStruct);//通过结构体EXTI_InitStruct初始化EXTI外设（重要）
void EXTI_StructInit(EXTI_InitTypeDef* EXTI_InitStruct);//可以个参数传递的结构体变量赋一个默认值
void EXTI_GenerateSWInterrupt(uint32_t EXTI_Line);//软件触发外部中断，EXTI_Line：中断线

//在主函数用
FlagStatus EXTI_GetFlagStatus(uint32_t EXTI_Line);//可以获取指定的标志位是否被置1
void EXTI_ClearFlag(uint32_t EXTI_Line);//可以对被置1的标志位进行清除
//在中断函数中用
ITStatus EXTI_GetITStatus(uint32_t EXTI_Line);//可以在中段函数里获取指定的标志位是否被置1
void EXTI_ClearITPendingBit(uint32_t EXTI_Line);//可以在中段函数里对被置1的标志位进行清除
```

`EXTI_Lines `：

```c
/** @defgroup EXTI_Lines 
  * @{
  */
#define EXTI_Line0       ((uint32_t)0x00001)  /*!< External interrupt line 0 */
#define EXTI_Line1       ((uint32_t)0x00002)  /*!< External interrupt line 1 */
#define EXTI_Line2       ((uint32_t)0x00004)  /*!< External interrupt line 2 */
#define EXTI_Line3       ((uint32_t)0x00008)  /*!< External interrupt line 3 */
#define EXTI_Line4       ((uint32_t)0x00010)  /*!< External interrupt line 4 */
#define EXTI_Line5       ((uint32_t)0x00020)  /*!< External interrupt line 5 */
#define EXTI_Line6       ((uint32_t)0x00040)  /*!< External interrupt line 6 */
#define EXTI_Line7       ((uint32_t)0x00080)  /*!< External interrupt line 7 */
#define EXTI_Line8       ((uint32_t)0x00100)  /*!< External interrupt line 8 */
#define EXTI_Line9       ((uint32_t)0x00200)  /*!< External interrupt line 9 */
#define EXTI_Line10      ((uint32_t)0x00400)  /*!< External interrupt line 10 */
#define EXTI_Line11      ((uint32_t)0x00800)  /*!< External interrupt line 11 */
#define EXTI_Line12      ((uint32_t)0x01000)  /*!< External interrupt line 12 */
#define EXTI_Line13      ((uint32_t)0x02000)  /*!< External interrupt line 13 */
#define EXTI_Line14      ((uint32_t)0x04000)  /*!< External interrupt line 14 */
#define EXTI_Line15      ((uint32_t)0x08000)  /*!< External interrupt line 15 */
#define EXTI_Line16      ((uint32_t)0x10000)  /*!< External interrupt line 16 Connected to the PVD Output */
#define EXTI_Line17      ((uint32_t)0x20000)  /*!< External interrupt line 17 Connected to the RTC Alarm event */
#define EXTI_Line18      ((uint32_t)0x40000)  /*!< External interrupt line 18 Connected to the USB Device/USB OTG FS
                                                   Wakeup from suspend event */                                    
#define EXTI_Line19      ((uint32_t)0x80000)  /*!< External interrupt line 19 Connected to the Ethernet Wakeup event */
                                          
#define IS_EXTI_LINE(LINE) ((((LINE) & (uint32_t)0xFFF00000) == 0x00) && ((LINE) != (uint16_t)0x00))
#define IS_GET_EXTI_LINE(LINE) (((LINE) == EXTI_Line0) || ((LINE) == EXTI_Line1) || \
                            ((LINE) == EXTI_Line2) || ((LINE) == EXTI_Line3) || \
                            ((LINE) == EXTI_Line4) || ((LINE) == EXTI_Line5) || \
                            ((LINE) == EXTI_Line6) || ((LINE) == EXTI_Line7) || \
                            ((LINE) == EXTI_Line8) || ((LINE) == EXTI_Line9) || \
                            ((LINE) == EXTI_Line10) || ((LINE) == EXTI_Line11) || \
                            ((LINE) == EXTI_Line12) || ((LINE) == EXTI_Line13) || \
                            ((LINE) == EXTI_Line14) || ((LINE) == EXTI_Line15) || \
                            ((LINE) == EXTI_Line16) || ((LINE) == EXTI_Line17) || \
                            ((LINE) == EXTI_Line18) || ((LINE) == EXTI_Line19))

```

`EXTI_LineCmd`: `ENABLE `or `DISABLE `

` EXTI_Mode`:

```c
typedef enum
{
  EXTI_Mode_Interrupt = 0x00,//中断模式
  EXTI_Mode_Event = 0x04//事件模式
}EXTIMode_TypeDef;
```

`EXTI_Trigger`:

```c
typedef enum
{
  EXTI_Trigger_Rising = 0x08,//上升沿
  EXTI_Trigger_Falling = 0x0C,  //下降沿
  EXTI_Trigger_Rising_Falling = 0x10//双边沿
}EXTITrigger_TypeDef;
```

##### 5、配置NVIC

给中断选择合适的优先级。

关于NVIC的函数：

```c
void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup);//用于中断分组，NVIC_PriorityGroup：中断分组的方式
void NVIC_Init(NVIC_InitTypeDef* NVIC_InitStruct);//初始化
//不常用
void NVIC_SetVectorTable(uint32_t NVIC_VectTab, uint32_t Offset);//设置中断向量表
void NVIC_SystemLPConfig(uint8_t LowPowerMode, FunctionalState NewState);//系统低功耗配置
```

```C
/**
  * @brief  Configures the priority grouping: pre-emption （抢占优先级）priority and subpriority.（响应优先级）
  * @param  NVIC_PriorityGroup: specifies the priority grouping bits length. 
  *   This parameter can be one of the following values:
  *     @arg NVIC_PriorityGroup_0: 0 bits for pre-emption priority
  *                                4 bits for subpriority
  *     @arg NVIC_PriorityGroup_1: 1 bits for pre-emption priority
  *                                3 bits for subpriority
  *     @arg NVIC_PriorityGroup_2: 2 bits for pre-emption priority
  *                                2 bits for subpriority
  *     @arg NVIC_PriorityGroup_3: 3 bits for pre-emption priority
  *                                1 bits for subpriority
  *     @arg NVIC_PriorityGroup_4: 4 bits for pre-emption priority
  *                                0 bits for subpriority
  * @retval None
  */
void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup)
```

`NVIC_InitTypeDef NVIC_InitStructure`结构体配置

`NVIC_IRQChannel//中断源配置`

```
#ifdef STM32F10X_MD
  ADC1_2_IRQn                 = 18,     /*!< ADC1 and ADC2 global Interrupt                       */
  USB_HP_CAN1_TX_IRQn         = 19,     /*!< USB Device High Priority or CAN1 TX Interrupts       */
  USB_LP_CAN1_RX0_IRQn        = 20,     /*!< USB Device Low Priority or CAN1 RX0 Interrupts       */
  CAN1_RX1_IRQn               = 21,     /*!< CAN1 RX1 Interrupt                                   */
  CAN1_SCE_IRQn               = 22,     /*!< CAN1 SCE Interrupt                                   */
  EXTI9_5_IRQn                = 23,     /*!< External Line[9:5] Interrupts                        */
  TIM1_BRK_IRQn               = 24,     /*!< TIM1 Break Interrupt                                 */
  TIM1_UP_IRQn                = 25,     /*!< TIM1 Update Interrupt                                */
  TIM1_TRG_COM_IRQn           = 26,     /*!< TIM1 Trigger and Commutation Interrupt               */
  TIM1_CC_IRQn                = 27,     /*!< TIM1 Capture Compare Interrupt                       */
  TIM2_IRQn                   = 28,     /*!< TIM2 global Interrupt                                */
  TIM3_IRQn                   = 29,     /*!< TIM3 global Interrupt                                */
  TIM4_IRQn                   = 30,     /*!< TIM4 global Interrupt                                */
  I2C1_EV_IRQn                = 31,     /*!< I2C1 Event Interrupt                                 */
  I2C1_ER_IRQn                = 32,     /*!< I2C1 Error Interrupt                                 */
  I2C2_EV_IRQn                = 33,     /*!< I2C2 Event Interrupt                                 */
  I2C2_ER_IRQn                = 34,     /*!< I2C2 Error Interrupt                                 */
  SPI1_IRQn                   = 35,     /*!< SPI1 global Interrupt                                */
  SPI2_IRQn                   = 36,     /*!< SPI2 global Interrupt                                */
  USART1_IRQn                 = 37,     /*!< USART1 global Interrupt                              */
  USART2_IRQn                 = 38,     /*!< USART2 global Interrupt                              */
  USART3_IRQn                 = 39,     /*!< USART3 global Interrupt                              */
  EXTI15_10_IRQn              = 40,     /*!< External Line[15:10] Interrupts                      */
  RTCAlarm_IRQn               = 41,     /*!< RTC Alarm through EXTI Line Interrupt                */
  USBWakeUp_IRQn              = 42      /*!< USB Device WakeUp from suspend through EXTI Line Interrupt */  
#endif /* STM32F10X_MD */  
```

`NVIC_IRQChannelPreemptionPriority`//抢占优先级

`NVIC_IRQChannelSubPriority`//响应优先级

如果选的是分组2，则这两个的值都为0到3

| **分组方式** | **抢占优先级**  | **响应优先级**  |
| ------------ | --------------- | --------------- |
| 分组0        | 0位，取值为0    | 4位，取值为0~15 |
| 分组1        | 1位，取值为0~1  | 3位，取值为0~7  |
| 分组2        | 2位，取值为0~3  | 2位，取值为0~3  |
| 分组3        | 3位，取值为0~7  | 1位，取值为0~1  |
| 分组4        | 4位，取值为0~15 | 0位，取值为0    |

`NVIC_IRQChannelCmd` : `ENABLE` or `DISABLE` 是否使能

#### 3、代码

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
uint16_t CountSensor_Count;

void CountSensor_Init(void)
{
	//配置时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB,ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE);//开启AFIO的时钟他也属于APB2
	//EXTI和NVIC外设，时钟一直处于打开状态
	
	//配置GPIO
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_14;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB,&GPIO_InitStructure);
	
	//配置AFIO
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB,GPIO_PinSource14);
	
	//配置EXTI
	EXTI_InitTypeDef EXTI_InitStruct;
	EXTI_InitStruct.EXTI_Line = EXTI_Line14 ;
	EXTI_InitStruct.EXTI_Mode = EXTI_Mode_Interrupt;
	EXTI_InitStruct.EXTI_LineCmd = ENABLE;
	EXTI_InitStruct.EXTI_Trigger = EXTI_Trigger_Rising;
	EXTI_Init(&EXTI_InitStruct);


	//配置NVIC
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);//整个工程分一次组就行
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel = EXTI15_10_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_Init(&NVIC_InitStructure);
	
}

uint16_t CountSensor_Get(void)
{
	return CountSensor_Count;
}

//中断函数的名字都是固定的，可以在stratup_stm32f10x_md.s中查看IRQHandler结尾的
void EXTI15_10_IRQHandler(void)
{
	//判断进来的是否为EXTI14
	if(EXTI_GetITStatus(EXTI_Line14) == SET)
	{
		
		CountSensor_Count++;
		//清除中断标志位
		EXTI_ClearITPendingBit(EXTI_Line14);
	}
}
```



#### 注意：

- 不要在中断函数中执行时间过长的函数，比如delay啥的
- 不要在中断函数和主函数执行相同的函数，如两个都调用Oled也容易引起显示错误或乱码

### 3、旋转编码器

#### 1、旋转编码器：

可以测速度和旋转方向，方向可以通过看B的方波是比A提前90度还是滞后90度。

#### 2、代码：



