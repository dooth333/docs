# 1、概述

## 1、简介：

### 1、通信接口：

- 通信的目的：将一个设备的数据传送到另一个设备，扩展硬件系统

- 通信协议：制定通信的规则，通信双方按照协议规则进行数据收发

| **名称** | **引脚**             | **双工** | **时钟** | **电平** | **设备** |
| -------- | -------------------- | -------- | -------- | -------- | -------- |
| USART    | TX、RX               | 全双工   | 异步     | 单端     | 点对点   |
| I2C      | SCL、SDA             | 半双工   | 同步     | 单端     | 多设备   |
| SPI      | SCLK、MOSI、MISO、CS | 全双工   | 同步     | 单端     | 多设备   |
| CAN      | CAN_H、CAN_L         | 半双工   | 异步     | 差分     | 多设备   |
| USB      | DP、DM               | 半双工   | 异步     | 差分     | 点对点   |

-   全双工  ：通信双方同时进行双向通信
- 单工：数据只能从一个设备到另一个设备，不能反着

- 电平：单端：引脚的电平都是对GND的电压差（所以双方要共地）；差分：靠两个引脚的电压差来传输信号（可以抗干扰，传输距离比较远）

## 2、串口通信：

### 1、简介：

- 串口是一种应用十分广泛的通讯接口，串口成本低、容易使用、通信线路简单，可实现两个设备的互相通信

- 单片机的串口可以使单片机与单片机、单片机与电脑、单片机与各式各样的模块互相通信，极大地扩展了单片机的应用范围，增强了单片机系统的硬件实力

### 2、硬件电路：

- 简单双向串口通信有两根通信线（发送端TX和接收端RX），GND线也一定要接，因为串口是单端的

- TX与RX要交叉连接

- 当只需单向的数据传输时，可以只接一根通信线

- 当电平标准不一致时，需要加电平转换芯片

![image-20220909164320636](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909164320636.png)

### 3、电平标准：

- 电平标准是数据1和数据0的表达方式，是传输线缆中人为规定的电压与数据的对应关系，串口常用的电平标准有如下三种：

  - TTL电平：+3.3V或+5V表示1，0V表示0

  - RS232电平：-3~-15V表示1，+3~+15V表示0

  - RS485电平：两线压差+2~+6V表示1，-2~-6V表示0（差分信号）

### 4、串口参数及时序：

- 波特率：串口通信的速率（异步通信约定的通信速率） 1000bps就是1s发一千位

- 起始位：标志一个数据帧的开始，固定为低电平（因为空闲时为高电平，下降沿提示开始）

- 数据位：数据帧的有效载荷，1为高电平，0为低电平，低位先行

- 校验位：用于数据验证，根据数据位计算得来（想要更精确何以了解CRC校验）
  - 无校验：不校验
  - 奇校验：包括校验位在内的9个数会产生**奇数**个1
  - 偶校验：包括校验位在内的9个数会产生**偶数**个1

- 停止位：用于数据帧间隔，固定为高电平

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909165011201.png" alt="image-20220909165011201"  />

- 数据位为8位

<img src="https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909165041978.png" alt="image-20220909165041978"  />

- 也可以在最后第9位加以一个奇偶校验位，前八位为有效载荷，最后一位为校验位

### 5、串口时序

![image-20220909170756530](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909170756530.png)

- stm32中数据的发送和接收都可以有USART自动完成
- 串口的停止位是可以配置的，可以选择1位、1.5位、2位（如上图）



## 3、USART

### 1、简介：

- USART（Universal Synchronous/Asynchronous Receiver/Transmitter）通用同步/异步收发器

- USART是STM32内部集成的硬件外设，可根据数据寄存器的一个字节数据**自动生成数据帧时序**，从TX引脚发送出去，也可自动接收RX引脚的数据帧时序，拼接为一个字节数据，**存放在数据寄存器里**

- 自带波特率发生器，最高达4.5Mbits/s

- 可配置数据位长度（8/9）、停止位长度（0.5/1/1.5/2）

- 可选校验位（无校验/奇校验/偶校验）

- 支持同步模式（加一个时钟线CLK）、硬件流控制（加一根线接收端可以告诉发送端自己是否准备好，准备好了发送端才发）、DMA、智能卡、IrDA、LIN

- STM32F103C8T6 USART资源： USART1（在APB2总线上）、 USART2（在APB1总线上）、 USART3（在APB1总线上）

### 2、USART框图：

![image-20220909173749198](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909173749198.png)

![image-20220909181137363](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909181137363.png)

- 发送数据寄存器TDR和接收数据寄存器RDR使用同一个地址，类似于51的SBUF。
  - TDR只写，RDR只读
- TDR和RDR下面的移位寄存器把数据寄存器中的数据一位一位移进去，刚好和串口协议的数据位对应
- 当你在TDR中写入数据时，硬件检测到了写入数据，他就会检查当前移位寄存器是不是有数据正在移位，如果没有写入TDR的数据会被全部移到发送移位寄存器准备发送（当数据从TDR移动到移位寄存器时，会置标志位TXT（TX Empty），如果该标志位置1，就可以写入下一个数据了）

- 然后移位寄存器在发送控制器控制下向右移位（低位先行），然后到TX端口输出
- 接收过程与发送类似，RX的数据一位一位的到**接收数据寄存器**然后一下全部转到**RDR**中，转移时也会置一个标志为（RXNE，接收数据寄存器非空），当RXNE为1把数据读走
- 发送时添加帧头和帧尾，接收时去除帧头和帧尾，有电路自动帮我们执行

### 3、引脚

![image-20220909181635277](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909181635277.png)

### 4、USART基本结构

![image-20220909182015740](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909182015740.png)

### 5、数据帧

![image-20220909182345611](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909182345611.png)

- 由图可知，不管时选8位还是9位最后一位都是可以设置成奇偶校验位的，这样就可能一个数据帧有效载荷只有9个或7个了不符合，一般不用
- 一般9位字长有校验，8位无校验，这样有效载荷刚好一字节

![image-20220909182757455](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220909182757455.png)

- 停止位可以选择，但一般用一位停止位

### 6、起始位侦测

![image-20220910092411928](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910092411928.png)

- 多次采样判断噪声，发现噪声会置标志位（NE）

### 7、数据采样

![image-20220910092715392](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910092715392.png)

- 起始位检测成功后，开始数据采样
- 数据采样是一帧数据采样三次，若三次数据不一样则按照2：1的比例确认数据帧，并且把噪声会置标志位（NE）置1

### 8、波特率发生器

- 发送器和接收器的波特率由波特率寄存器BRR里的DIV确定

- 计算公式：波特率 = fPCLK2/1 / (16 * DIV)

![image-20220910093007774](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910093007774.png)

- PCLK1 = 72MHz，PCLK2 = 36MHz
- 如：9600的CLK1的DIV计算，DIV = 72M / 9600 / 16
- DIV_Mantissa[11:0]放DIV（二进制）整数部分，DIV_Fraction[3:0]放DIV（二进制）小数部分
- 库函数配置时直接配置波特率就可以，会自动计算

### 9、数据模式：

- HEX模式/十六进制模式/二进制模式：以原始数据的形式显示

- 文本模式/字符模式：以原始数据编码后的形式显示

![image-20220910110952665](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910110952665.png)

![image-20220910111021893](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910111021893.png)



# 2、代码

### 1、全部函数：

```c
//恢复加载配置
void USART_DeInit(USART_TypeDef* USARTx);
//USART初始化
void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct);
//结构体初始化
void USART_StructInit(USART_InitTypeDef* USART_InitStruct);
//同步时钟输出初始化
void USART_ClockInit(USART_TypeDef* USARTx, USART_ClockInitTypeDef* USART_ClockInitStruct);
//同步时钟输出结构体初始化
void USART_ClockStructInit(USART_ClockInitTypeDef* USART_ClockInitStruct);
//USART开关
void USART_Cmd(USART_TypeDef* USARTx, FunctionalState NewState);

//中断输出控制（使能或禁用指定的USART中断。）
void USART_ITConfig(USART_TypeDef* USARTx, uint16_t USART_IT, FunctionalState NewState);
**
  *     @arg USART_IT_CTS:  CTS change interrupt (not available for UART4 and UART5)
  *     @arg USART_IT_LBD:  LIN Break detection interrupt
  *     @arg USART_IT_TXE:  Transmit Data Register empty interrupt
  *     @arg USART_IT_TC:   Transmission complete interrupt
  *     @arg USART_IT_RXNE: Receive Data register not empty interrupt	读取数据寄存器非空标标志，收到数据时
  *     @arg USART_IT_IDLE: Idle line detection interrupt
  *     @arg USART_IT_PE:   Parity Error interrupt
  *     @arg USART_IT_ERR:  Error interrupt(Frame error, noise error, overrun error)
      
//开启USART到DMA的触发通道
void USART_DMACmd(USART_TypeDef* USARTx, uint16_t USART_DMAReq, FunctionalState NewState);
//发送数据（写DR寄存器）
void USART_SendData(USART_TypeDef* USARTx, uint16_t Data);
//接收数据（读DR寄存器）
uint16_t USART_ReceiveData(USART_TypeDef* USARTx);

//标志位的函数
//获取标志位
FlagStatus USART_GetFlagStatus(USART_TypeDef* USARTx, uint16_t USART_FLAG);
**第二个参数为要获取的标志位名称：
  *     @arg USART_FLAG_CTS:  CTS Change flag (not available for UART4 and UART5)
  *     @arg USART_FLAG_LBD:  LIN Break detection flag
  *     @arg USART_FLAG_TXE:  Transmit data register empty flag		发送数据寄存器空标志位（判断是否发送完成，1表示发送完成）由手册25.6.1描述，该位不用手动清0，再下次写的时候会自动清0
  *     @arg USART_FLAG_TC:   Transmission Complete flag
  *     @arg USART_FLAG_RXNE: Receive data register not empty flag	读取数据寄存器非空标标志（有数据时为1，并且使用读寄存器数据函数时会自动置0）
  *     @arg USART_FLAG_IDLE: Idle Line detection flag
  *     @arg USART_FLAG_ORE:  OverRun Error flag
  *     @arg USART_FLAG_NE:   Noise Error flag
  *     @arg USART_FLAG_FE:   Framing Error flag
  *     @arg USART_FLAG_PE:   Parity Error flag
      
void USART_ClearFlag(USART_TypeDef* USARTx, uint16_t USART_FLAG);
ITStatus USART_GetITStatus(USART_TypeDef* USARTx, uint16_t USART_IT);
//清除标志位
void USART_ClearITPendingBit(USART_TypeDef* USARTx, uint16_t USART_IT);
**参数和GET一样

//特殊功能的函数不常用
void USART_SetAddress(USART_TypeDef* USARTx, uint8_t USART_Address);
void USART_WakeUpConfig(USART_TypeDef* USARTx, uint16_t USART_WakeUp);
void USART_ReceiverWakeUpCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_LINBreakDetectLengthConfig(USART_TypeDef* USARTx, uint16_t USART_LINBreakDetectLength);
void USART_LINCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_SendBreak(USART_TypeDef* USARTx);
void USART_SetGuardTime(USART_TypeDef* USARTx, uint8_t USART_GuardTime);
void USART_SetPrescaler(USART_TypeDef* USARTx, uint8_t USART_Prescaler);
void USART_SmartCardCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_SmartCardNACKCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_HalfDuplexCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_OverSampling8Cmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_OneBitMethodCmd(USART_TypeDef* USARTx, FunctionalState NewState);
void USART_IrDAConfig(USART_TypeDef* USARTx, uint16_t USART_IrDAMode);
void USART_IrDACmd(USART_TypeDef* USARTx, FunctionalState NewState);
```

### 2、结构体配置：

```c
//波特率
USART_InitStructure.USART_BaudRate = ;
**直接写数值就可以，比如9600
    
//硬件流控制
USART_InitStructure.USART_HardwareFlowControl = ;
**把参数名复制然后Ctrl+Alt+空格
	*#define USART_HardwareFlowControl_None       ((uint16_t)0x0000)	不使用流控
	*#define USART_HardwareFlowControl_RTS        ((uint16_t)0x0100)	只使用RTS
	*#define USART_HardwareFlowControl_CTS        ((uint16_t)0x0200)	只使用CTS
	*#define USART_HardwareFlowControl_RTS_CTS    ((uint16_t)0x0300)	CTS和RTS都使用
    
//串口模式
USART_InitStructure.USART_Mode = ;
**如果继续要发送也需要接收把他们或一下USART_Mode_Tx|USART_Mode_Rx
    #define USART_Mode_Rx                        ((uint16_t)0x0004)	接收模式
	#define USART_Mode_Tx                        ((uint16_t)0x0008)	发送模式
    
//校验位
USART_InitStructure.USART_Parity = ;
**
    #define USART_Parity_No                      ((uint16_t)0x0000)	无校验
	#define USART_Parity_Even                    ((uint16_t)0x0400)	奇校验
	#define USART_Parity_Odd                     ((uint16_t)0x0600)	偶校验
    
//停止位
USART_InitStructure.USART_StopBits = ;
**
    #define USART_StopBits_1                     ((uint16_t)0x0000)	1位
	#define USART_StopBits_0_5                   ((uint16_t)0x1000)	0.5位
	#define USART_StopBits_2                     ((uint16_t)0x2000)	2位
	#define USART_StopBits_1_5                   ((uint16_t)0x3000)	1.5位
    
//字长
USART_InitStructure.USART_WordLength = ;
**
    #define USART_WordLength_8b                  ((uint16_t)0x0000)	8位
	#define USART_WordLength_9b                  ((uint16_t)0x1000)	9位
```



## 2、流程：

### 1、初始化流程

#### 1、开启时钟：

- 开启用到的USART和GPIO的时钟

#### 2、GPIO初始化：

- 把TX配置成复用输出，RX配置成输入（上拉输入或浮空输入）

#### 3、配置USART

- 通过结构体配置

#### 4、开启开关控制

- 如果你只需要发送功能

#### 5、配置中断

- 需要接收功能：在开启USART之前，加上ITConfig和NVIC代码

### 2、发送：

```
/**
  * @brief  串口发送一个字节的函数
  * @param  Byte 要发送的字节
  * @retval None
  */
void Serial_SendByte(uint8_t Byte){
	
	USART_SendData(USART1,Byte);
	//USART_FLAG_TXE 发送数据寄存器空标志位（判断是否发送完成，1表示发送完成）由手册25.6.1描述，该位不用手动清0，再下次写的时候会自动清0
	while(USART_GetFlagStatus(USART1,USART_FLAG_TXE) == RESET);
}
```



## 3、Printf到串口输出

### 1、方法1：重定向fputc

- 要重定向，只能一个串口使用

1、Keil用printf函数，先设置，打开keil的精简库

![image-20220910141744068](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimgimage-20220910141744068.png)

2、`#include "stdio.h"`导包

3、重写printf底层函数

```c
int fputc(int ch, FILE *f){
	Serial_SendByte(ch);
	return ch;
}
```

### 2、方法2：

- 不涉及重定向，每个串口都可以用

```
	char String[100];
	//指定字符串的内容
	sprintf(String,"number=%d\r\n",666);
	Serial_SendString(String);
```

3、封装sprintf

```
头文件
#include <stdarg.h>

函数
/**
  * @brief  封装sprintf，用于打印到串口
	* 和java未知变量个数类似
  * @param  
  * @retval None
  */
void Serial_Printf(char *format, ...){
	char String[100];
	va_list arg;
	va_start(arg,format);
	vsprintf(String,format,arg);
	va_end(arg);
	Serial_SendString(String);
}

使用方法：
Serial_Printf("a=%d,B=%d\r\n",1111,222);
```

### 3、中文显示

#### 1、utf-8:

- 把串口也调成UTF-8
- 如果中文keil编译保存添加`--no-multibyte-chars`
- ![image-20220910152327842](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910152327842.png)

#### 2、GB2312

- keil设置中把文字编码改为GB2312
- 注意改完后要把原来的汉字删除，关闭文件重新打开，文字发生变化才是改变过来了





# 3、数据包

## 1、简介：

### 1、HEX数据包：

![image-20220910164651406](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910164651406.png)

- 适合一些数据，如陀螺仪模块数据、温湿度传感器数据
- 缺点：灵活性不足，载荷容易和包头包尾重复

### 2、文本数据包：

![image-20220910164721154](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910164721154.png)

- 优点：数据直观易理解，适合输入指令人机交互
- 解析效率低，对于一些数据更麻烦

## 2、收发流程：

### 1、发送：

- 发送比较简单，直接调用我们写过的代码实现即可

### 2、HEX数据包接收：

- 数据长度确定

![image-20220910165332883](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910165332883.png)

- 状态机方法，根据接收到不同的数据进入不同的状态，如上面的S为0，1，2代表三种不同状态

### 3、文本数据包接收

- 数据长度不确定

![image-20220910170007533](https://cdn.jsdelivr.net/gh/dooth333/note_image/imgimage-20220910170007533.png)

- 与上面不一样的就是s=1时也要兼顾一下检测包尾的`\r`,s=2用于检测另一个包尾`\n`
- 如果只有一个包尾可以少一种状态



## 3、代码：

### 1、HEX收发数据包

- 问题：Serial_RxPacket被同时写入和读出，可能由于读取的太慢还没完全读完就已经更新数据了，照成前几位和后几位数据不一致
  - 解决方法：可以加个判断，读取完了再让中断接收下一个数据包
  - 对于那种传感器的的值，由于他们相邻数据包具有连续性，混一起影响也不大
  - 可以参考文本的处理方法Flag把get和set分开（但是如果频率过快这样可能会造成数据包丢失）

