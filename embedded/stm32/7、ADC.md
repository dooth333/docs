## 1、概述：

### 1、简介：

- ADC（Analog-Digital Converter）模拟-数字转换器，而DAC为数字-模拟转换器（数字信号转换为模拟信号但是而我们通常用PWM来转换）
- ADC可以将引脚上连续变化的模拟电压转换为内存中存储的数字变量，建立模拟电路到数字电路的桥梁

- 12位（分辨率）逐次逼近型ADC，1us转换时间（转换频率为1MHz）

- 输入电压范围：0到3.3V，转换结果范围：0到4095

- 18个输入通道，本系列最多，可测量16个外部（GPIO口）和2个内部信号源（内部温度传感器和内部参考电压）

- 规则组和注入组两个转换单元

- 模拟看门狗自动监测输入电压范围（可以实现当温度或光敏等各种值高于或低于某个特定值时会进入中断）

- STM32F103C8T6 ADC资源：**ADC1、ADC2，10个外部输入通道**

### 2、原理：

#### 1、逐次逼近型ADC

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811092421895.png" alt="image-20220811092421895" style="zoom:80%;" />

- 这是个0809ADC芯片与STM32的ADC模块不同，便于理解原理
- IN0到IN7为8个输入通道，这个8个通道通过ADDA、ADDB、ADDC这三个进行译码选择。
- 然后拿通道输入的**未知编码的电压**，与**DAC已知编码的电压**通过**比较器**进行大小判断
- 通过比较值进行调整DAC值，直到两值相等，得到的DAC值就是外部电压的编码数据了，而DAC的调节过程就是通过**逐次逼近寄存器SAR**完成的，一般通过二分法进行寻找
- EOC是转换结束信号，START是开始转换信号，VREF+、VREF-为参考电压，它可以决定最大值255是5v还是3.3v
- ADCCLK为ADC时钟，用于驱动逐次比较

#### 2、ADC框图

![image-20220811094012473](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811094012473.png)

- 这个才为STM32的ADC框图
- **最左边**是ADC的输入通道，ADCx_IN1到15为16个GPIO口输入通道，和两个内部的通道，内部温度传感器、  V~REFINT~(内部参考电压) 
- 然后输入到达模拟多路通道开关，指定我们想要选择的通道，再往右进入多路选择的输出，进入到模数转换器（执行逐次比较的功能）
- 然后把逐次比较的结果放入（注入通道/规则通道）数据寄存器里，通过读取寄存器的值得到转换结果。
- 普通ADC多路通道只选一个，而这可以同时选中多个，并且分了两个组（注入通道（最多4个）/规则通道（最多16个））
- **规则通道**：虽然一次可以选择16个通道，但是数据寄存器**一次只能存一个结果**（即16位），若不想同一组前面的数据被覆盖就要用到**DMA转运数据**
- **注入通道**：相当于vip席位，一次可以选择4个通道，并且注入通道数据寄存器**一次可以存4个结果**（即4 * 16），**不会被覆盖**。

- **左下角**为触发转换的部分，触发方式分为两种：1、为软件触发，通过调用代码触发。2、硬件触发，即本图左下角的触发。
- **上面**为注入组触发源，**下面**为规则组触发源。TIM1_CH1这些都是由定时器来的，因为DAC需要每隔一段时间转换一次，所以要连定时器，但频繁进中断会影响程序，可以通过更新事件避免这些。
- 也可以通过外部中断引脚EXTI_11进行触发中断
- **左上角**V~REF+~、V~REF-~为ADC的参考电压，决定了ADC输入电压的范围；V~DDA~（+）、V~SSA~（-）为参考电压的供电引脚
- **右下角**，ADCCLK为ADC时钟，用于驱动逐次比较。来自ADC预分频器，ADCCLK最大14MHz，而选择2、4分频都超范围了，只有选择6分频12MHz，8分频9MHz
- <img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811102406436.png" alt="image-20220811102406436" style="zoom:80%;" />
- 上面，模拟看门狗，可以存一个阈值高限和低限，如果打开并且指定了看门的通道，超出阈值范围会申请模拟看门狗的中断到NVIC
- 最上面的EOC是规则组完成信号，JEOC是注入组完成的信号

#### 3、基本结构图

![image-20220811103412576](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811103412576.png)

#### 4、数据细节

##### 1、输入通道

| **通道** | **ADC1**     | **ADC2** | **ADC3** |
| -------- | ------------ | -------- | -------- |
| 通道0    | PA0          | PA0      | PA0      |
| 通道1    | PA1          | PA1      | PA1      |
| 通道2    | PA2          | PA2      | PA2      |
| 通道3    | PA3          | PA3      | PA3      |
| 通道4    | PA4          | PA4      | PF6      |
| 通道5    | PA5          | PA5      | PF7      |
| 通道6    | PA6          | PA6      | PF8      |
| 通道7    | PA7          | PA7      | PF9      |
| 通道8    | PB0          | PB0      | PF10     |
| 通道9    | PB1          | PB1      |          |
| 通道10   | PC0          | PC0      | PC0      |
| 通道11   | PC1          | PC1      | PC1      |
| 通道12   | PC2          | PC2      | PC2      |
| 通道13   | PC3          | PC3      | PC3      |
| 通道14   | PC4          | PC4      |          |
| 通道15   | PC5          | PC5      |          |
| 通道16   | 温度传感器   |          |          |
| 通道17   | 内部参考电压 |          |          |

- ADC1和ADC2引脚相同，一般不用ADC2，
- 本芯片只有通道1到10加16和17，其他没有，也没有ADC3

##### 2、规则组四种转换模式：

###### 1、单次转换，非扫描模式

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811104621642.png" alt="image-20220811104621642" style="zoom:80%;" />

- 相当于每次只转换一个，只有第一个有效。而其转换结束后，下一次转换需要重新触发才能开始

###### 2、连续转换，非扫描模式

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811104838913.png" alt="image-20220811104838913" style="zoom:80%;" />

- 一次也只转换一个，但是与上一次不同的是，上一次转换完成后不会停止，而是立刻开始下一轮转换

###### 3、单次转换，扫描模式

![image-20220811105224283](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811105224283.png)

- 一次转换多个，但要用DMA处理，一组转换结束后，下次转换要重新触发

###### 4、连续转换，扫描模式

![image-20220811153039647](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811153039647.png)

- 一次转换多个，但要用DMA处理，一组转换结束后，下次转换不用重新触发，直接开始下次转换

##### 3、触发控制

![image-20220811153452026](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811153452026.png)

##### 4、数据对齐

![image-20220811153753207](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811153753207.png)

- 由于我们的ADC结果是12位的数据，而寄存器为16位的，右对齐为左边高位0（经常用，这样16位的值和12位的一样），左对齐为右边低4位补0

##### 5、转换时间

- AD转换的步骤：采样，保持，量化，编码

- STM32 ADC的总转换时间为： TCONV = 采样时间 + 12.5个ADC周期

- 例如：当ADCCLK=14MHz，采样时间为1.5个ADC周期：  TCONV = 1.5 + 12.5 = 14个ADC周期 = 1μs

##### 6、校准

- ADC有一个内置自校准模式。校准可大幅减小因内部电容器组的变化而造成的准精度误差。校准期间，在每个电容器上都会计算出一个误差修正码(数字值)，这个码用于消除在随后的转换中每个电容器上产生的误差

- 建议在每次上电后执行一次校准，（不需要了解过程，只要会用代码校准就行）

- 启动校准前， ADC必须处于关电状态超过至少两个ADC时钟周期

### 3、电路图

![image-20220811154957626](https://cdn.jsdelivr.net/gh/dooth333/note_image/img/image-20220811154957626.png)

## 2、函数

### 1、步骤：

#### 1、开启RCC时钟：

- 包括ADC和GPIO的时钟，ADCCLK的时钟也需要配置

#### 2、配置GPIO：

- 把需要的GPIO配置成模拟输入的模式
- 选`GPIO_Mode_AIN`,`AIN`模式下GPIO口无效，所以AIN位ADC专属模式

#### 3、配置多路开关：

- 把左边通道接到右边的规则组列表里

- ```
  void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime)
  ```

#### 4、配置ADC转换器：

```
void ADC_Init(ADC_TypeDef* ADCx, ADC_InitTypeDef* ADC_InitStruct);//Init初始化
```

#### 5、如果需要：

- 可以配置看门狗，如果需要中断就在中断输出控制里配置，然后再配置NVIC

#### 6、开关控制：

- 开启ADC_Cmd函数

- ```c
  void ADC_Cmd(ADC_TypeDef* ADCx, FunctionalState NewState);//给ADC上电，即开关控制
  ADC_Cmd(ADC1,ENABLE);
  ```

#### 7、ADC校准

```c
void ADC_ResetCalibration(ADC_TypeDef* ADCx);//复位校准
FlagStatus ADC_GetResetCalibrationStatus(ADC_TypeDef* ADCx);//获取复位校准状态
void ADC_StartCalibration(ADC_TypeDef* ADCx);//开始校准
FlagStatus ADC_GetCalibrationStatus(ADC_TypeDef* ADCx);//获取开始校准状态
```

### 2、相关库函数

```c
//RCC库函数
void RCC_ADCCLKConfig(uint32_t RCC_PCLK2);//配置ADCCLK分频器，APB2为72MHz进行分频
```

```c
void ADC_DeInit(ADC_TypeDef* ADCx);//恢复缺省配置
void ADC_Init(ADC_TypeDef* ADCx, ADC_InitTypeDef* ADC_InitStruct);//Init初始化
void ADC_StructInit(ADC_InitTypeDef* ADC_InitStruct);//结构体初始化
void ADC_Cmd(ADC_TypeDef* ADCx, FunctionalState NewState);//给ADC上电，即开关控制

void ADC_DMACmd(ADC_TypeDef* ADCx, FunctionalState NewState);//开启DMA输出信号

void ADC_ITConfig(ADC_TypeDef* ADCx, uint16_t ADC_IT, FunctionalState NewState);//中断输出控制

//关于校准的函数，再ADC初始化后调用
void ADC_ResetCalibration(ADC_TypeDef* ADCx);//复位校准
FlagStatus ADC_GetResetCalibrationStatus(ADC_TypeDef* ADCx);//获取复位校准状态
void ADC_StartCalibration(ADC_TypeDef* ADCx);//开始校准
FlagStatus ADC_GetCalibrationStatus(ADC_TypeDef* ADCx);//获取开始校准状态

void ADC_SoftwareStartConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);//ADC软件开始转换控制（软件触发）
//SWSTART位由软件设置开始启动，转换开始后硬件马上清除此位
FlagStatus ADC_GetSoftwareStartConvStatus(ADC_TypeDef* ADCx);//ADC获取软件开始转换状态（什么也判断不了，没啥用）

//配置间断函数
void ADC_DiscModeChannelCountConfig(ADC_TypeDef* ADCx, uint8_t Number);//配置间断函数，配置每隔几个通道间断一次
void ADC_DiscModeCmd(ADC_TypeDef* ADCx, FunctionalState NewState);//是否启用间断模式

//ADC规则组通道配置（重要）参数：1、ADCx 2、你想指定的通道 3、序列几的位置 4、指定通道的采样时间
void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime);

void ADC_ExternalTrigConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);//ADC外部触发转换控制
uint16_t ADC_GetConversionValue(ADC_TypeDef* ADCx);//ADC获取转换值（重要）

uint32_t ADC_GetDualModeConversionValue(void);//ADC获取双模式转换值

//配置ADC注入组（没讲）
void ADC_AutoInjectedConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);
void ADC_InjectedDiscModeCmd(ADC_TypeDef* ADCx, FunctionalState NewState);
void ADC_ExternalTrigInjectedConvConfig(ADC_TypeDef* ADCx, uint32_t ADC_ExternalTrigInjecConv);
void ADC_ExternalTrigInjectedConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);
void ADC_SoftwareStartInjectedConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);
FlagStatus ADC_GetSoftwareStartInjectedConvCmdStatus(ADC_TypeDef* ADCx);
void ADC_InjectedChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime);
void ADC_InjectedSequencerLengthConfig(ADC_TypeDef* ADCx, uint8_t Length);
void ADC_SetInjectedOffset(ADC_TypeDef* ADCx, uint8_t ADC_InjectedChannel, uint16_t Offset);
uint16_t ADC_GetInjectedConversionValue(ADC_TypeDef* ADCx, uint8_t ADC_InjectedChannel);

//配置模拟看门狗
void ADC_AnalogWatchdogCmd(ADC_TypeDef* ADCx, uint32_t ADC_AnalogWatchdog);//是否启动模拟看门狗
void ADC_AnalogWatchdogThresholdsConfig(ADC_TypeDef* ADCx, uint16_t HighThreshold, uint16_t LowThreshold);//配置高/低阈值
void ADC_AnalogWatchdogSingleChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel);//配置看门的通道

void ADC_TempSensorVrefintCmd(FunctionalState NewState);//ADC内部传感器参考电压控制(用于开启内部两个通道)
FlagStatus ADC_GetFlagStatus(ADC_TypeDef* ADCx, uint8_t ADC_FLAG);//获取标志位的状态
void ADC_ClearFlag(ADC_TypeDef* ADCx, uint8_t ADC_FLAG);//清除标志位
ITStatus ADC_GetITStatus(ADC_TypeDef* ADCx, uint16_t ADC_IT);//获取中断状态
void ADC_ClearITPendingBit(ADC_TypeDef* ADCx, uint16_t ADC_IT);//清除中断挂起位


FlagStatus ADC_GetFlagStatus(ADC_TypeDef* ADCx, uint8_t ADC_FLAG);//获取标志位的状态，当参数给EOC时，若EOC置1可以判断转换结束
```

```
配置ADC的通道分组
/**
  * @brief  Configures for the selected ADC regular channel its corresponding
  *         rank in the sequencer and its sample time.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  ADC_Channel: the ADC channel to configure. 
  *   This parameter can be one of the following values:每个通道的对应的IO口不同，查上表
  *     @arg ADC_Channel_0: ADC Channel0 selected
  *     @arg ADC_Channel_1: ADC Channel1 selected
  *     @arg ADC_Channel_2: ADC Channel2 selected
  *     @arg ADC_Channel_3: ADC Channel3 selected
  *     @arg ADC_Channel_4: ADC Channel4 selected
  *     @arg ADC_Channel_5: ADC Channel5 selected
  *     @arg ADC_Channel_6: ADC Channel6 selected
  *     @arg ADC_Channel_7: ADC Channel7 selected
  *     @arg ADC_Channel_8: ADC Channel8 selected
  *     @arg ADC_Channel_9: ADC Channel9 selected
  *     @arg ADC_Channel_10: ADC Channel10 selected
  *     @arg ADC_Channel_11: ADC Channel11 selected
  *     @arg ADC_Channel_12: ADC Channel12 selected
  *     @arg ADC_Channel_13: ADC Channel13 selected
  *     @arg ADC_Channel_14: ADC Channel14 selected
  *     @arg ADC_Channel_15: ADC Channel15 selected
  *     @arg ADC_Channel_16: ADC Channel16 selected
  *     @arg ADC_Channel_17: ADC Channel17 selected
  * @param  Rank: The rank in the regular group sequencer. This parameter must be between 1 to 16.对应规则组的16个序列
  * @param  ADC_SampleTime: The sample time value to be set for the selected channel. 
  *   This parameter can be one of the following values:参数越小，转换的越快；参数越大，越稳定（没要求随便选55都行）
  *     @arg ADC_SampleTime_1Cycles5: Sample time equal to 1.5 cycles
  *     @arg ADC_SampleTime_7Cycles5: Sample time equal to 7.5 cycles
  *     @arg ADC_SampleTime_13Cycles5: Sample time equal to 13.5 cycles
  *     @arg ADC_SampleTime_28Cycles5: Sample time equal to 28.5 cycles	
  *     @arg ADC_SampleTime_41Cycles5: Sample time equal to 41.5 cycles	
  *     @arg ADC_SampleTime_55Cycles5: Sample time equal to 55.5 cycles	
  *     @arg ADC_SampleTime_71Cycles5: Sample time equal to 71.5 cycles	
  *     @arg ADC_SampleTime_239Cycles5: Sample time equal to 239.5 cycles	
  * @retval None
  */
void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime)
```

`ADC_InitTypeDef：`

`ADC_Mode：`模式

```
#define ADC_Mode_Independent                       ((uint32_t)0x00000000)  独立模式，就是ADC1和ADC2各转换各的


下面全是双ADC模式，暂时不用
#define ADC_Mode_RegInjecSimult                    ((uint32_t)0x00010000)
#define ADC_Mode_RegSimult_AlterTrig               ((uint32_t)0x00020000)
#define ADC_Mode_InjecSimult_FastInterl            ((uint32_t)0x00030000)
#define ADC_Mode_InjecSimult_SlowInterl            ((uint32_t)0x00040000)
#define ADC_Mode_InjecSimult                       ((uint32_t)0x00050000)
#define ADC_Mode_RegSimult                         ((uint32_t)0x00060000)
#define ADC_Mode_FastInterl                        ((uint32_t)0x00070000)
#define ADC_Mode_SlowInterl                        ((uint32_t)0x00080000)
#define ADC_Mode_AlterTrig                         ((uint32_t)0x00090000)
```

`ADC_DataAlign`:对齐方式

```
#define ADC_DataAlign_Right       右对齐                 ((uint32_t)0x00000000)
#define ADC_DataAlign_Left        左对齐            ((uint32_t)0x00000800)
```

`ADC_ExternalTrigConv`触发控制的触发源选择

```
#define ADC_ExternalTrigConv_T1_CC1                ((uint32_t)0x00000000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T1_CC2                ((uint32_t)0x00020000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T2_CC2                ((uint32_t)0x00060000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T3_TRGO               ((uint32_t)0x00080000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_T4_CC4                ((uint32_t)0x000A0000) /*!< For ADC1 and ADC2 */
#define ADC_ExternalTrigConv_Ext_IT11_TIM8_TRGO    ((uint32_t)0x000C0000) /*!< For ADC1 and ADC2 */

#define ADC_ExternalTrigConv_T1_CC3                ((uint32_t)0x00040000) /*!< For ADC1, ADC2 and ADC3 */
#define ADC_ExternalTrigConv_None    不使用外部触发，使用软件触发              ((uint32_t)0x000E0000) /*!< For ADC1, ADC2 and ADC3 */

#define ADC_ExternalTrigConv_T3_CC1                ((uint32_t)0x00000000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T2_CC3                ((uint32_t)0x00020000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T8_CC1                ((uint32_t)0x00060000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T8_TRGO               ((uint32_t)0x00080000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T5_CC1                ((uint32_t)0x000A0000) /*!< For ADC3 only */
#define ADC_ExternalTrigConv_T5_CC3                ((uint32_t)0x000C0000) /*!< For ADC3 only */
```

以下用用于配置对应上面的四种模式：

`ADC_ContinuousConvMode`连续转换模式

```
ENABLE or DISABLE
```

`ADC_ScanConvMode`扫描转换模式

```
ENABLE or DISABLE
```

`ADC_NbrOfChannel`通道数目（扫描模式需要扫描几个通道）

```
This parameter must range from 1 to 16.
1到16
```



```
//获取标志位的状态
/**
  * @brief  Checks whether the specified ADC flag is set or not.
  * @param  ADCx: where x can be 1, 2 or 3 to select the ADC peripheral.
  * @param  ADC_FLAG: specifies the flag to check. 
  *   This parameter can be one of the following values:
  *     @arg ADC_FLAG_AWD: Analog watchdog flag 看门狗标志位
  *     @arg ADC_FLAG_EOC: End of conversion flag 规则组转换完成标志位
  *     @arg ADC_FLAG_JEOC: End of injected group conversion flag 注入组转换完成标志位
  *     @arg ADC_FLAG_JSTRT: Start of injected group conversion flag 注入组开始转换标志位
  *     @arg ADC_FLAG_STRT: Start of regular group conversion flag	规则组转换完成biao'z
  * @retval The new state of ADC_FLAG (SET or RESET).
  */
FlagStatus ADC_GetFlagStatus(ADC_TypeDef* ADCx, uint8_t ADC_FLAG)
```



## 3、代码

### 1、单通道

### 2、多通道：

- 这是不使用DMA的
