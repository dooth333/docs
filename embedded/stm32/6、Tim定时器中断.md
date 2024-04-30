# TIM定时器

## 1、概述：

### 1、简介：

- TIM（Timer）定时器，定时器可以对输入的时钟进行计数，并在计数值达到设定值时触发中断

- 16位计数器、预分频器、自动重装寄存器的时基单元，在72MHz计数时钟下可以实现最大59.65s的定时

- 不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能

- 根据复杂度和应用场景分为了高级定时器、**通用定时器**、基本定时器三种类型

### 2、分类：

| **类型**   | **编号**               | **总线** | **功能**                                                     |
| ---------- | ---------------------- | -------- | ------------------------------------------------------------ |
| 高级定时器 | TIM1、TIM8             | APB2     | 拥有通用定时器全部功能，并额外具有重复计数器、死区生成、互补输出、刹车输入等功能 |
| 通用定时器 | TIM2、TIM3、TIM4、TIM5 | APB1     | 拥有基本定时器全部功能，并额外具有**内外时钟源选择**、**输入捕获**、**输出比较**、**编码器接口**、**主从触发模式**等功能 |
| 基本定时器 | TIM6、TIM7             | APB1     | 拥有定时中断、主模式触发DAC的功能                            |

- STM32F103C8T6定时器资源：**TIM1、TIM2、TIM3、TIM4**

### 3、原理：

#### 1、基本定时器

![image-20220807094947199](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807094947199.png)

- 基本定时器的时钟只能是内部时钟，频率为单片机主频72MHz
- 然后到PSC分频器，若不分频就是72MHz，若二分频就是72 / 2 = 36MHz, 若三分频就是72 / 3= 24MHz。而且（实际分频系数 = 预分频器的值 + 1）比如：PSC = 1，就是2分频。最大可以65536分频。
- CNT为计数器，值可以从0到65535
- 自动重装寄存器：可以写入我们的计数目标。（当计数器值等于自动重装值时，产生中断）
- 图中右下角，UI：代表触发中断，U: 代表进入事件。
- 主模式触发DAC的功能：直接不通过中断触发，通过事件触发，这样可以避免主程序频繁被中断。把更新事件U通过主模式映射到TRGO，然后TRGO直接去触发DAC

#### 2、通用定时器

![image-20220807100724711](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807100724711.png)

- 中间基本结构与基本定时器一样。

- 通用计数器的计数模式不止有向上计数模式（一直往上加）一种了，还有：向下计数模式（从重装值向下减，减到0中断）、中央对齐模式（先向上自增，到重装值，中断，然后再从重装值递减，减到0申请中断）。

- 上半部分为内外时钟源选择和主从模式触发

  - 内外时钟源选择：从TIMx_ETR接一个方波时钟，然后再极性选择等，然后输入滤波电路，然后兵分两路：1、直接通过ETRF，通过触发控制器直接到时钟CK，被称为**外部时钟模式2**。2、通过TRGI，通过从模式控制器当时钟，被称为外部时钟模式1。
  - ![image-20220807102209413](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807102209413.png)

  - ITR0、ITR1这四个都是来自其他定时器，就比如本定时器可以通过TRGO输出

- 下半部分，右边为输出比较电路，有四个通道对应CH1到CH4，可以用于输出PWM波形和驱动电机
- 下半部分，左边为输入捕获电路，也是对应CH1到CH4的四个引脚
- 下半部分，中间为比较电路，由输入捕获和输出比较共同作用，因为他两个不能同时进行，所以他俩共用引脚，和寄存器。



#### 3、高级定时器

![image-20220807105848998](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807105848998.png)

- 多一个重复次数计数器，可以实现每隔几个周期进行一次中断或事件





![image-20220807150733699](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807150733699.png)



#### 4、预分频器时序

![image-20220807151403319](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807151403319.png)

- 计数器计数频率：CK_CNT = CK_PSC / (PSC + 1)
- 防止分频器更新造成错误，设计了**缓冲寄存器**，即这个中断周期运行完，下一个周期分频器才发生改变
- 缓冲寄存器可以根据需求自己选择是否开关

#### 5、计数器时序图

![image-20220807151542161](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807151542161.png)

- 产生中断后，手动复位中断更行标志位
- 计数器溢出频率（每个中断周期分之一）：CK_CNT_OV = CK_CNT / (ARR + 1)    =  CK_PSC / (PSC + 1) / (ARR + 1)  （ARR是自动重装载值）
- 其中更新中断标志时也有缓冲寄存器（即这个中断周期运行完，下一个周期分频器才发生改变），也可以自己设置开关

#### 6、RCC时钟树

![image-20220807152708886](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807152708886.png)

- 左边是时钟的产生电路，右边是时钟的分配电路

## 2、相关函数

### 1、关于Time的全部函数：

```c
void TIM_DeInit(TIM_TypeDef* TIMx);//回复缺省配置
void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct);//时基单元初始化，用来配置上图的时基单元的（重要）
void TIM_TimeBaseStructInit(TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct);//对结构体变量赋默认值
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState);//使能计数器
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState);//使能中断输出信号

//时钟选择部分：

void TIM_InternalClockConfig(TIM_TypeDef* TIMx);//选择内部时钟（不写默认也是选择内部时钟，不写也行）
void TIM_ITRxExternalClockConfig(TIM_TypeDef* TIMx, uint16_t TIM_InputTriggerSource);//选择ITR其他定时器的时钟
void TIM_TIxExternalClockConfig(TIM_TypeDef* TIMx, uint16_t TIM_TIxExternalCLKSource,uint16_t TIM_ICPolarity, uint16_t ICFilter);//选择TIX其他定时器的时钟（参数：TIMx ; 具体输入的引脚； 输入的极性； 输入的滤波器）
void TIM_ETRClockMode1Config(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity,uint16_t ExtTRGFilter);//选择ETR通过外部时钟模式1输入的时钟（参数：TIMx ；外部触发预分频器；输入的极性； 输入的滤波器）
void TIM_ETRClockMode2Config(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter);//选择ETR通过外部时钟模式2输入的时钟（参数：TIMx ；外部触发预分频器；输入的极性； 输入的滤波器）
void TIM_ETRConfig(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter);//用于单独来配置ETR引脚的预分频器、极性、滤波器

//其他常用的函数
void TIM_PrescalerConfig(TIM_TypeDef* TIMx, uint16_t Prescaler, uint16_t TIM_PSCReloadMode);//单独配置预分频值	Prescaler：要写入的预分频值 TIM_PSCReloadMode：写入的模式（是否使用缓冲器）
void TIM_CounterModeConfig(TIM_TypeDef* TIMx, uint16_t TIM_CounterMode);//用来改变计数器的计数模式
void TIM_ARRPreloadConfig(TIM_TypeDef* TIMx, FunctionalState NewState);//自动重装器预装功能配置（选择有预装还是无预装）NewState：Enable/DisEnable
void TIM_SetCounter(TIM_TypeDef* TIMx, uint16_t Counter);//给计数器写入一个值
void TIM_SetAutoreload(TIM_TypeDef* TIMx, uint16_t Autoreload);//设置自动重装值
uint16_t TIM_GetCounter(TIM_TypeDef* TIMx);//获取当前计数器的值
uint16_t TIM_GetPrescaler(TIM_TypeDef* TIMx);//获取当前预分频器的值


FlagStatus TIM_GetFlagStatus(TIM_TypeDef* TIMx, uint16_t TIM_FLAG);//获取标志位
void TIM_ClearFlag(TIM_TypeDef* TIMx, uint16_t TIM_FLAG);//清除标志位
ITStatus TIM_GetITStatus(TIM_TypeDef* TIMx, uint16_t TIM_IT);//在中断获取标志位
void TIM_ClearITPendingBit(TIM_TypeDef* TIMx, uint16_t TIM_IT);//在中断清除标志位
```

### 2、时基单元初始化结构体

#### 1、`TIM_ClockDivision`

```c
//Specifies the clock division.指定时钟分频和PSC预分频器不同，他用于滤波器时钟的分频设置

#define TIM_CKD_DIV1                       ((uint16_t)0x0000) //1分频（不分频）
#define TIM_CKD_DIV2                       ((uint16_t)0x0100) //2分频
#define TIM_CKD_DIV4                       ((uint16_t)0x0200) //4分频
```

#### 2、`TIM_CounterMode`

```c
#define TIM_CounterMode_Up                 ((uint16_t)0x0000) //向上计数
#define TIM_CounterMode_Down               ((uint16_t)0x0010) //向下计数
#define TIM_CounterMode_CenterAligned1     ((uint16_t)0x0020) //中央对齐
#define TIM_CounterMode_CenterAligned2     ((uint16_t)0x0040) //中央对齐
#define TIM_CounterMode_CenterAligned3     ((uint16_t)0x0060) //中央对齐
```

##### 3、`TIM_Period`

```c
//ARR的自动重装值
```

#### 4、`TIM_Prescaler`

```c
//PSC预分频器的值
```

#### 5、`TIM_RepetitionCounter`

```c
//重复计数器的值，高级计时器才有，通用定时器不用，给0
```

#### 6、计算ARR和PSC

```
CK_CNT_OV =  CK_PSC / (PSC + 1) / (ARR + 1) = 72MHz / (PSC + 1) / (ARR + 1)
1MHz = 1000000Hz 
若：定时为1s，则 ARR = 10000 - 1； PSC = 7200 + 1
注意：ARR和PSC值为0到65535
计算的到的是频率，单位是HZ，化成时间是 1/频率
```

### 3、使能中断函数

```
/**
  * @brief  Enables or disables the specified TIM interrupts.
  * @param  TIMx: where x can be 1 to 17 to select the TIMx peripheral.
  * @param  TIM_IT: specifies the TIM interrupts sources to be enabled or disabled.
  *   This parameter can be any combination of the following values:
  *     @arg TIM_IT_Update: TIM update Interrupt source
  *     @arg TIM_IT_CC1: TIM Capture Compare 1 Interrupt source
  *     @arg TIM_IT_CC2: TIM Capture Compare 2 Interrupt source
  *     @arg TIM_IT_CC3: TIM Capture Compare 3 Interrupt source
  *     @arg TIM_IT_CC4: TIM Capture Compare 4 Interrupt source
  *     @arg TIM_IT_COM: TIM Commutation Interrupt source
  *     @arg TIM_IT_Trigger: TIM Trigger Interrupt source
  *     @arg TIM_IT_Break: TIM Break Interrupt source
  * @note 
  *   - TIM6 and TIM7 can only generate an update interrupt.
  *   - TIM9, TIM12 and TIM15 can have only TIM_IT_Update, TIM_IT_CC1,
  *      TIM_IT_CC2 or TIM_IT_Trigger. 
  *   - TIM10, TIM11, TIM13, TIM14, TIM16 and TIM17 can have TIM_IT_Update or TIM_IT_CC1.   
  *   - TIM_IT_Break is used only with TIM1, TIM8 and TIM15. 
  *   - TIM_IT_COM is used only with TIM1, TIM8, TIM15, TIM16 and TIM17.    
  * @param  NewState: new state of the TIM interrupts.
  *   This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState)
```



## 3、代码

### 1、定时器定时中断

主要根据下图配置：

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220807150733699.png" alt="image-20220807150733699"  />



#### 1、RCC开启时钟：

```
RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,ENABLE);
```



#### 2、选着时基单元的时钟源：

（定时中断选着内部时钟源）

```
TIM_InternalClockConfig(TIM2);
```

#### 3、配置时基单元：

```c
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 10000 - 1;
	TIM_TimeBaseInitStructure.TIM_Prescaler = 7200 - 1;
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2,&TIM_TimeBaseInitStructure);
```

#### 4、配置输出中断控制：

允许更新中断输出到NVIC，也就是使能中断

```c
TIM_ITConfig(TIM2,TIM_IT_Update,ENABLE);
```

#### 5、配置NVIC：

在NVIC中打开定时器中断的通道，并分配一个优先级。

```c
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
	NVIC_Init(&NVIC_InitStructure);
```

#### 6、运行控制：

（使能定时器）

```
TIM_Cmd(TIM2,ENABLE);
```

#### 7、写中断函数：

```c
void TIM2_IRQHandler(void)
{
	if(TIM_GetITStatus(TIM2,TIM_IT_Update) == SET)
	{

		TIM_ClearITPendingBit(TIM2,TIM_IT_Update);
	}
}
```

### 2、定时器外部中断

选择外部定时器模式2

```
/**
  * @brief  Configures the External clock Mode2
  * @param  TIMx: where x can be  1, 2, 3, 4, 5 or 8 to select the TIM peripheral.
  * @param  TIM_ExtTRGPrescaler: The external Trigger Prescaler.
  *   This parameter can be one of the following values:外部触发预分频器，我们不用这个分频
  *     @arg TIM_ExtTRGPSC_OFF: ETRP Prescaler OFF.不分频
  *     @arg TIM_ExtTRGPSC_DIV2: ETRP frequency divided by 2.
  *     @arg TIM_ExtTRGPSC_DIV4: ETRP frequency divided by 4.
  *     @arg TIM_ExtTRGPSC_DIV8: ETRP frequency divided by 8.
  * @param  TIM_ExtTRGPolarity: The external Trigger Polarity.
  *   This parameter can be one of the following values:
  *     @arg TIM_ExtTRGPolarity_Inverted: active low or falling edge active.低电平或下降沿有效
  *     @arg TIM_ExtTRGPolarity_NonInverted: active high or rising edge active.高电平或上升沿有效
  * @param  ExtTRGFilter: External Trigger Filter.
  *   This parameter must be a value between 0x00 and 0x0F 外部触发滤波器，值为0x00 到0x0F（不用设置为0x00）
  * @retval None
  */
void TIM_ETRClockMode2Config(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, 
                             uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter)
```



# 输出比较模块

## 1、概述：

### 1、简介：

- OC（Output Compare）输出比较，输出比较可以通过比较CNT与CCR寄存器值的关系，来对输出电平进行置1、置0或翻转的操作，用于输出一定频率和占空比的PWM波形

- 每个高级定时器和通用定时器都拥有4个输出比较通道

- 高级定时器的前3个通道额外拥有死区生成和互补输出的功能

### 2、原理：

#### 1、输出比较通道(通用)：

![image-20220808155441312](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808155441312.png)

- CNT和CCR1比较后，控制oc1ref（参考信号）的高低电平
- 然后oc1ref可以到主模式里，但是不常用，经常走下面这一路
- 通过控制CC1P控制信号是否取反（极性选择）
- 然后到输出使能电路，控制是否输出
- 最后由OC1引脚输出，也就是CH1，要去引脚定义表看是哪个GPIO

### 2、输出比较模式：

| **模式**         | **描述**                                                     |
| ---------------- | ------------------------------------------------------------ |
| 冻结             | CNT=CCR时，REF保持为原状态                                   |
| 匹配时置有效电平 | CNT=CCR时，REF置有效电平                                     |
| 匹配时置无效电平 | CNT=CCR时，REF置无效电平                                     |
| 匹配时电平翻转   | CNT=CCR时，REF电平翻转                                       |
| 强制为无效电平   | CNT与CCR无效，REF强制为无效电平                              |
| 强制为有效电平   | CNT与CCR无效，REF强制为有效电平                              |
| PWM模式1         | 向上计数：CNT<CCR时，REF置有效电平，CNT≥CCR时，REF置无效电平  向下计数：CNT>CCR时，REF置无效电平，CNT≤CCR时，REF置有效电平 |
| PWM模式2         | 向上计数：CNT<CCR时，REF置无效电平，CNT≥CCR时，REF置有效电平  向下计数：CNT>CCR时，REF置有效电平，CNT≤CCR时，REF置无效电平 |

## 2、PWM

### 1、简介：

- PWM（Pulse Width Modulation）脉冲宽度调制

- 在具有惯性的系统中，可以通过对一系列脉冲的宽度进行调制，来等效地获得所需要的模拟参量，常应用于电机控速等领域

- PWM参数：  频率 = 1 / TS      占空比 = TON / TS      分辨率 = 占空比变化步距

### 2、PWM基本结构：

![image-20220808161310140](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808161310140.png)

![image-20220808162012227](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808162012227.png)

### 3、参数计算：

- 黄线：ARR（进入中断的数），红线：CCR（比较时用的数），PSC为设置的预分频器系数

- PWM频率： Freq = CK_PSC / (PSC + 1) / (ARR + 1)（和计数器更新频率相同）

- PWM占空比： Duty = CCR / (ARR + 1)  （这里时CNT=CCR时即为无效电平，所以CCR不用加1，因为从零开始，ARR要加1）

- PWM分辨率： Reso = 1 / (ARR + 1)

## 3、其他原件

### 1、舵机

- 舵机是一种根据输入PWM信号占空比来控制输出角度的装置

- 输入PWM信号要求：周期为20ms，高电平宽度为0.5ms~2.5ms

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808164850401.png" alt="image-20220808164850401" style="zoom: 67%;" />

![image-20220808164949171](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808164949171.png)

### 2、直流电机及驱动

- 直流电机是一种将电能转换为机械能的装置，有两个电极，当电极正接时，电机正转，当电极反接时，电机反转

- 直流电机属于大功率器件，GPIO口无法直接驱动，需要配合电机驱动电路来操作

- TB6612是一款双路H桥型的直流电机驱动芯片，可以驱动两个直流电机并且控制其转速和方向

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808165109275.png" alt="image-20220808165109275" style="zoom: 50%;" />

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220808165136084.png" alt="image-20220808165136084" style="zoom: 67%;" />

## 4、代码

### 1、常用函数：

```c
//通过结构体配置输出比较单元（重要）
void TIM_OC1Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);
void TIM_OC2Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);
void TIM_OC3Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);
void TIM_OC4Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);



void TIM_OCStructInit(TIM_OCInitTypeDef* TIM_OCInitStruct);//给输出比较赋默认值，先赋初始值就算用不到也可避免突然用的而没有值的错误

//配置强制输出模式（强制输出高电平或低电平）（了解，不常用）
void TIM_ForcedOC1Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction);
void TIM_ForcedOC2Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction);
void TIM_ForcedOC3Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction);
void TIM_ForcedOC4Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction);


//配置CCR寄存器的预装功能（即可以配置事件之后再生效）（了解，不常用）
void TIM_OC1PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC2PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC3PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC4PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);

//配置快速使能（了解，不常用）
void TIM_OC1FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast);
void TIM_OC2FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast);
void TIM_OC3FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast);
void TIM_OC4FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast);

//（了解，不常用）
void TIM_ClearOC1Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear);
void TIM_ClearOC2Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear);
void TIM_ClearOC3Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear);
void TIM_ClearOC4Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear);

//用于单独设置输出比较的极性（带N的时高级定时器里互补通道的函数）（在通过结构体初始化时也能设置极性）
void TIM_OC1PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity);
void TIM_OC1NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity);
void TIM_OC2PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity);
void TIM_OC2NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity);
void TIM_OC3PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity);
void TIM_OC3NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity);
void TIM_OC4PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity);

//单独修改输出使能参数的
void TIM_CCxCmd(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_CCx);
void TIM_CCxNCmd(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_CCxN);

//选择输出比较模式
void TIM_SelectOCxM(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_OCMode);

//用于更改CCR寄存器值的函数
void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1);
void TIM_SetCompare2(TIM_TypeDef* TIMx, uint16_t Compare2);
void TIM_SetCompare3(TIM_TypeDef* TIMx, uint16_t Compare3);
void TIM_SetCompare4(TIM_TypeDef* TIMx, uint16_t Compare4);
```



`TIM_OCInitTypeDef`结构体

```c
TIM_OCInitTypeDef TIM_OCInitStructure;

//加一句初始值
void TIM_OCStructInit(TIM_OCInitTypeDef* TIM_OCInitStruct);//给输出比较赋默认值，先赋初始值就算用不到也可避免突然用的而没有值的错误
```

`TIM_OCMode`

```
//输出比较模式
#define TIM_OCMode_Timing                  ((uint16_t)0x0000)冻结模式
#define TIM_OCMode_Active                  ((uint16_t)0x0010)相等时置有效电平
#define TIM_OCMode_Inactive                ((uint16_t)0x0020)相等时置无效电平
#define TIM_OCMode_Toggle                  ((uint16_t)0x0030)相等时电平翻转
#define TIM_OCMode_PWM1                    ((uint16_t)0x0060)
#define TIM_OCMode_PWM2                    ((uint16_t)0x0070)

```

`TIM_OCPolarity`

```
//设置输出比较的极性
#define TIM_OCPolarity_High                ((uint16_t)0x0000)不反转
#define TIM_OCPolarity_Low                 ((uint16_t)0x0002)反转
```

`TIM_OutputState`

```
//设置输出比较的使能
#define TIM_OutputState_Disable            ((uint16_t)0x0000)失能
#define TIM_OutputState_Enable             ((uint16_t)0x0001)使能
```

`TIM_Pulse`

```
//设置CCR值
参数为0x0000 到 0xFFFF
```

其他那几的都是不常用的，带N的都是高级定时器互补通道的函数

### 2、步骤：

#### 1、RCC开启时钟：

把TIM外设和GPIO外设的时钟打开

#### 2、配置时基单元

不需要打开中断（中断使能），也不再需要NVIC配置，但需要保留启动定时器

#### 3、配置输出比较单元

包括：CCR的值、输出比较模式、极性选择、输出使能

#### 4、配置GPIO

![image-20220810114134544](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220810114134544.png)

- 引脚对应关系对应
- 重定义功能那一栏有的可以通过AFIO完成

把GPIO对应的GPIO初始化为**复用推挽输出**，为社么使用复用推挽输出？普通的开漏和推挽输出，引脚的控制权来于输出数据寄存器，而使用复用开漏、推挽就由定时器控制了

### 3、LED呼吸灯

#### `PWM.c`

```c
#include "stm32f10x.h"                  // Device header

void PWM_Init(void)
{
//1、RCC开启时钟：
	//把TIM外设和GPIO外设的时钟打开
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,ENABLE);


//2、配置时基单元
	TIM_InternalClockConfig(TIM2);
	
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 100-1;//ARR
	TIM_TimeBaseInitStructure.TIM_Prescaler = 720-1;//PSC
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2,&TIM_TimeBaseInitStructure);
	
	
	TIM_OCInitTypeDef TIM_OCInitStructure;
	
	//先赋初始值就算用不到也可避免突然用的而没有值的错误
	TIM_OCStructInit(&TIM_OCInitStructure);
	//然后更改需要用的值
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable ; 
	TIM_OCInitStructure.TIM_Pulse = 0; //CCR
	TIM_OC1Init(TIM2,&TIM_OCInitStructure);
	


//3、配置输出比较单元
//包括：CCR的值、输出比较模式、极性选择、输出使能
	

// 4、配置GPIO
//把GPIO对应的GPIO初始化为复用推挽输出

	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	TIM_Cmd(TIM2,ENABLE);
	
}

void PWM_SetCompare1(uint16_t Compare)
{
	TIM_SetCompare1(TIM2,Compare);
}
```

#### `main.c`

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "PWM.h"
uint16_t i;
int main(void)
{
	PWM_Init();
	OLED_Init();
	OLED_ShowString(1,3,"HelloWorld!");
	while (1)
	{
		for(i = 0; i<= 100; i++)
		{
			PWM_SetCompare1(i);
			Delay_ms(10);
		}
		for(i = 0; i<= 100; i++)
		{
			PWM_SetCompare1(100 - i);
			Delay_ms(10);
		}
	}
}
```

#### 端口重映射：

案例，由引脚表可以看出TIM2_CH1默认在PA0上，可以重映射到PA15上

```c
//重映射要使用AFIO，先开启AFIO时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO,ENABLE);
//重定义函数
GPIO_PinRemapConfig(GPIO_PartialRemap1_TIM2,ENABLE);
//PA15上电已经默认为调试端口JTDI，若当作普通GPIO要先关闭调试端口的复用
//解除调试端口
GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable,ENABLE);
```

```c
//引脚重定义函数
void GPIO_PinRemapConfig(uint32_t GPIO_Remap, FunctionalState NewState);
//参数1，选择重定义的方式
//此次方式1即可，部分重映射，部分包括CH1的PA15和CH2的PB3即为：GPIO_PartialRemap1_TIM2


//SWJ是SWD和JTAG两种调试方式，不要乱用，用了容易把STlink下载不能使用
  *     @arg GPIO_Remap_SWJ_NoJTRST      : Full SWJ Enabled (JTAG-DP + SW-DP) but without JTRST	解除JTRST引脚复用   PA13、PA14
  *     @arg GPIO_Remap_SWJ_JTAGDisable  : JTAG-DP Disabled and SW-DP Enabled  解除JTAG引脚复用PA15、PB3、PB4
  *     @arg GPIO_Remap_SWJ_Disable      : Full SWJ Disabled (JTAG-DP + SW-DP) 把SWD和JTAG引脚全部解除   PA13、PA14、PA15、PB3、PB4
```

![image-20220810175503356](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220810175503356.png)

重映射方式可通过手册查看

![image-20220810174058974](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220810174058974.png)

### 4、PWM驱动舵机

ARR：20k - 1 ； PSC： 72 - 1  ；周期为20ms ；CCR为1对应0.001





# 输入捕获

```
void TIM_ICInit(TIM_TypeDef* TIMx, TIM_ICInitTypeDef* TIM_ICInitStruct);//结构体初始化输入捕获，只能配置一个通道
void TIM_PWMIConfig(TIM_TypeDef* TIMx, TIM_ICInitTypeDef* TIM_ICInitStruct);//结构体初始化输入捕获,但是可以快速配置两个通道
void TIM_ICStructInit(TIM_ICInitTypeDef* TIM_ICInitStruct);//给输入捕获结构体赋初始值
void TIM_SelectInputTrigger(TIM_TypeDef* TIMx, uint16_t TIM_InputTriggerSource);//选择输入触发源
void TIM_SelectOutputTrigger(TIM_TypeDef* TIMx, uint16_t TIM_TRGOSource);//选择输出触发源TRGO
void TIM_SelectSlaveMode(TIM_TypeDef* TIMx, uint16_t TIM_SlaveMode);//选择从模式
//分别单独配置通道1、2、3、4的分频器
void TIM_SetIC1Prescaler(TIM_TypeDef* TIMx, uint16_t TIM_ICPSC);
void TIM_SetIC2Prescaler(TIM_TypeDef* TIMx, uint16_t TIM_ICPSC);
void TIM_SetIC3Prescaler(TIM_TypeDef* TIMx, uint16_t TIM_ICPSC);
void TIM_SetIC4Prescaler(TIM_TypeDef* TIMx, uint16_t TIM_ICPSC);
//分别读取四个通道的CCR
uint16_t TIM_GetCapture1(TIM_TypeDef* TIMx);
uint16_t TIM_GetCapture2(TIM_TypeDef* TIMx);
uint16_t TIM_GetCapture3(TIM_TypeDef* TIMx);
uint16_t TIM_GetCapture4(TIM_TypeDef* TIMx);
```

结构体

```
TIM_Channel 选择通道
#define TIM_Channel_1 
#define TIM_Channel_2 
#define TIM_Channel_3 
#define TIM_Channel_4 

TIM_ICPolarity 触发方式
#define  TIM_ICPolarity_Rising   上升沿
#define  TIM_ICPolarity_Falling  下降沿
#define  TIM_ICPolarity_BothEdge 上升沿和下降沿都触发


TIM_ICSelection	触发信号从哪个引脚输入
#define TIM_ICSelection_DirectTI  直连通道输入
                                  
#define TIM_ICSelection_IndirectTI 交叉通道输入
                                  
#define TIM_ICSelection_TRC 不用      


TIM_ICPrescaler 分频器
#define TIM_ICPSC_DIV1  //不分频
#define TIM_ICPSC_DIV2 
#define TIM_ICPSC_DIV4 
#define TIM_ICPSC_DIV8 


TIM_ICFilter//输入捕获滤波器
/*!指定输入捕获过滤器。此参数可以是介于0x0和0xF之间的数字，数越大滤波效果越好 */
```



触发源选择

```
/**
  * @brief  Selects the Input Trigger source
  * @param  TIMx: where x can be  1, 2, 3, 4, 5, 8, 9, 12 or 15 to select the TIM peripheral.
  * @param  TIM_InputTriggerSource: The Input Trigger source.
  *   This parameter can be one of the following values:
  *     @arg TIM_TS_ITR0: Internal Trigger 0
  *     @arg TIM_TS_ITR1: Internal Trigger 1
  *     @arg TIM_TS_ITR2: Internal Trigger 2
  *     @arg TIM_TS_ITR3: Internal Trigger 3
  *     @arg TIM_TS_TI1F_ED: TI1 Edge Detector
  *     @arg TIM_TS_TI1FP1: Filtered Timer Input 1
  *     @arg TIM_TS_TI2FP2: Filtered Timer Input 2
  *     @arg TIM_TS_ETRF: External Trigger input
  * @retval None
  */
void TIM_SelectInputTrigger(TIM_TypeDef* TIMx, uint16_t TIM_InputTriggerSource)
{
  uint16_t tmpsmcr = 0;
  /* Check the parameters */
  assert_param(IS_TIM_LIST6_PERIPH(TIMx));
  assert_param(IS_TIM_TRIGGER_SELECTION(TIM_InputTriggerSource));
  /* Get the TIMx SMCR register value */
  tmpsmcr = TIMx->SMCR;
  /* Reset the TS Bits */
  tmpsmcr &= (uint16_t)(~((uint16_t)TIM_SMCR_TS));
  /* Set the Input Trigger source */
  tmpsmcr |= TIM_InputTriggerSource;
  /* Write to TIMx SMCR */
  TIMx->SMCR = tmpsmcr;
}
```

选择从模式

```
/**
  * @brief  Selects the TIMx Slave Mode.
  * @param  TIMx: where x can be 1, 2, 3, 4, 5, 8, 9, 12 or 15 to select the TIM peripheral.
  * @param  TIM_SlaveMode: specifies the Timer Slave Mode.
  *   This parameter can be one of the following values:
  *     @arg TIM_SlaveMode_Reset: Rising edge of the selected trigger signal (TRGI) re-initializes
  *                               the counter and triggers an update of the registers.
  *     @arg TIM_SlaveMode_Gated:     The counter clock is enabled when the trigger signal (TRGI) is high.
  *     @arg TIM_SlaveMode_Trigger:   The counter starts at a rising edge of the trigger TRGI.
  *     @arg TIM_SlaveMode_External1: Rising edges of the selected trigger (TRGI) clock the counter.
  * @retval None
  */
void TIM_SelectSlaveMode(TIM_TypeDef* TIMx, uint16_t TIM_SlaveMode)
```

