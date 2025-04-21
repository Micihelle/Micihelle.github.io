---
layout: post
title: STM32-learning：串口通信
date: '2025-04-20 20:03:02 +0800'
---



## 0x01 通信接口

对于最小系统板，需要准备USB转串口(CH340)模块；

USB也是一种串口：全称为(Universal Serial Bus)通用串行总线

TTL串口：

- TX（transmit）
- RX（receive）
- GND的作用：共地，意味着选择参考系，去相对值高低电平，确保两设备正常通信
- 注意在使用系统板的时候有共用VCC，所以这里不需要考虑单独引出VCC。

处理器与外部设备通信方式：并行通信和串行通信



- 并行通信：数据各个位同时传输
- 串行通信：数据按位顺序传递

串行通信：

- 单工：数据只能从一个设备到一个设备
- 半双工（某一个时刻只允许从一个方向传递到另外一个方向）
- 全双工   （这里定义方式都跟管道差不多）：发送线路和接受线路不会互相影响

![image.png](/assets/img/2025-04-20/image.png)

<!--![图片]({{ "/assets/img/2025-04-20/image.png" | relative_url }}) !-->


串行通信的通信方式：

- 同步通信：什么时候进行数据的位的传输主要取决于时钟信号，带时钟同步信号传输。
- 异步通信：UART(不带时钟同步信号，自然的需要对通信双方做约定→通信协议）

常见的串行通信接口：

![image.png](/assets/img/2025-04-20/image%201.png)

- 这里I2C和SPI都有单独的时钟线，所以他们是同步的，接收方可以在时钟信号的指引下进行采样
- GND的作用就是参考系的作用，引脚的高低电平都是相对于GND的电压差

![image.png](/assets/img/2025-04-20/image%202.png)

STM32串口通信接口：

- UART:通信异步收发器
- USART

UART异步通信引脚连接：

- 发送端TX（transmit）
- 接收端RX（receive）
- GND的作用：共地，意味着选择参考系，去相对值高低电平，确保两设备正常通信

![image.png](/assets/img/2025-04-20/image%203.png)

![image.png](/assets/img/2025-04-20/image%204.png)

![image.png](/assets/img/2025-04-20/image%205.png)

![image.png](/assets/img/2025-04-20/image%206.png)

查阅串口1的发送引脚：→ STM32官方提供的芯片手册，直接去翻相关的硬件结构图也能找到

![image.png](/assets/img/2025-04-20/image%207.png)

电平标准：

- 电平标准是数据1和数据0的表达方式，是传输线缆中人为规定的电压与数据的对应关系，串口常用的电平标准有如下三种：

- **TTL电平：+3.3V或+5V表示1，0V表示0**
- RS232电平：-3~-15V表示1，+3~+15V表示0
- RS485电平：两线压差+2~+6V表示1，-2~-6V表示0（差分信号）

**串口参数及时序：如何用1和0来组成我们想要发送的一个字节数据**

- 波特率：由于串口进行异步通信，所以发送和接受需要约定好速率，串口通信过程的传递速度主要由波特率决定
- 起始位：标志一个数据帧的开始，固定为低电平（STM32系统串口通信是怎么开始的？）
- 串口时序波形图（探头GND接在负极，探头接发送设备的TX引脚）:TX引脚输出定时翻转的高低电平，RX引脚定时读取定时引脚的高低电平；每个字节的的数据加上起始位、停止位、可选的校验位打包为数据帧，依次输出在TX引脚，另一端RX引脚以此接受，完成数据的传递，串口通信。

STM32串口异步通信标准例程：

![image.png](/assets/img/2025-04-20/image%208.png)

位8采用奇偶校验码(Parity Check)进行校验：(只能检测出奇数个错误，见图中案例，低8位用作数据位，校验位置最高位）

![image.png](/assets/img/2025-04-20/image%209.png)

串口通信寄存器库函数相关配置：

cudeide配置：(STM32F103RC、DEBUG::JTAG 5pins)

USART(Universal Synchronous / Asynchronous Receiver & Transmitter) （TTL串口使用的是异步通信） 

![image.png](/assets/img/2025-04-20/image%2010.png)

![image.png](/assets/img/2025-04-20/image%2011.png)

将USART2 转化为TTL串口模式

![image.png](/assets/img/2025-04-20/image%2012.png)

出现了波特率参数：（反映每秒传送的码元数量，即每秒多少次高低电平信号

默认情况下，TTL串口每传送一个字节(1 byte = 8bit)，还需要添加一位起始位和一位停止位，这意味着每传送1字节信息需要10bit。（这里的parity表示校验位，Stop BITS表示停止位）

![image.png](/assets/img/2025-04-20/image%2013.png)

通信两设备需要保持相同的波特率才能进行正常通信

## 0x02 USART串口外设

- USART（Universal Synchronous/Asynchronous Receiver/Transmitter）通用同步/异步收发器
- USART是STM32内部集成的硬件外设，可根据数据寄存器的一个字节数据自动生成数据帧时序，从TX引脚发送出去，也可自动接收RX引脚的数据帧时序，拼接为一个字节数据，存放在数据寄存器里

> USART是串口通信的硬件支持电路，发送部分将字节数据自动转化为协议规定的波形从TX引脚发送出去，接收部分自动接收RX引脚的波形，按照协议的规定，解码为一个字节数据，存放在数据寄存器里
> 
- 自带波特率发生器，最高达4.5Mbits/s
- 可配置数据位长度（8/9）、停止位长度（0.5/1/1.5/2）停止位长度决定了间隔
- 可选校验位（无校验/奇校验/偶校验）（注意波特率发生器、数据位、校验位等参数都通过配置寄存器或者标准库结构体赋值完成）
- 支持同步模式、硬件流控制、DMA、智能卡、IrDA、LIN

- STM32F103C8T6 USART资源： USART1、 USART2、 USART3

从软件的角度理解USART框图 ：

![image.png](/assets/img/2025-04-20/image%2014.png)

发送移位寄存器里面的数据会向右移位从TX引脚发送（串口数据中的低位先行特性）；移位过程中TDR会进行等待

控制部分：发送器控制用于控制发送移位寄存器的工作；接收器控制用于控制接受移位寄存器的动作；硬件数据流控（如何防止数据覆盖的问题？

下列引脚在硬件流控模式中需要： 

- nRTS(Request To Send): 发送请求，若是低电平，表明USART准备好接收数据 (告诉别人我当前能否接收）
- nCTS(Clear to Send): 清除发送，若是高电平，在当前数据传输结束时阻断下一次的数据发送。 
(接受其他设备的nRTS信号
- 自适应波特率实现：SCLK+软件控制
- 唤醒单元：多设备串口通信（进程唤醒）

在STM23-USART中波特率发生器到底对发送器时钟和接收器时钟产生了什么样的作用？

- 对齐：波特率发生器会对系统时钟进行分频，得到一个与预设波特率相匹配的时间基准。这个时间基准决定了每个数据位的宽度，也就是数据传输的速率。
- 对多种波特率通信的支持：用户可通过设置USART_BRR（Baud Rate Register）来调整分频因子，从而支持多种波特率的通信。
- 控制发生器时钟；控制接收器时钟
- USART外设作用：能够根据STM32字节数据翻转高低电平

![image.png](/assets/img/2025-04-20/image%2015.png)

GPIO口复用输出（这些都涉及标志位的设计，可以考虑重写中断函数进入中断来实现快速读取和保存数据的效果）

![image.png](/assets/img/2025-04-20/image%2016.png)

思考-信号到底是怎么控制波形输出的？

数据帧-数据采样问题：起始位侦测、采样位置对齐策略

- 抗干扰能力

波特率寄存器（16倍分频采样时钟）

## 0x03 串口发送 && 串口发送+接收

一根线可以有多个输入，但是只能有一个输出，复用输入和普通输入的区别就在于是否只有一个输入

```python
/**
  * @brief  Transmits single data through the USARTx peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  Data: the data to transmit.
  * @retval None
  */
void USART_SendData(USART_TypeDef* USARTx, uint16_t Data)
{
  /* Check the parameters */
  assert_param(IS_USART_ALL_PERIPH(USARTx));
  assert_param(IS_USART_DATA(Data)); 
    
  /* Transmit Data */
  USARTx->DR = (Data & (uint16_t)0x01FF); // 0000 0001 1111 1111 无关高位清零
}
```

数据模式：

- HEX模式/十六进制模式/二进制模式：以原始数据的形式显示
- 文本模式/字符模式：以原始数据编码后的形式  （通过字符集译码，也可以在发送的时候直接发送字符然后再进行编码后发送）

多次按下reset后的接收效果

![image.png](/assets/img/2025-04-20/image%2017.png)

**问题：示波器抓取串口发送接收信号的波形**

该段程序实现将会遇到的问题：下载程序之后，PC端接收到了123450000….012345000…的循环发送信息；

```c
void Serial_SendNumber(uint32_t Number,uint8_t Length)
{
	uint8_t i;
	for (i=Length-1;i>=0;i--)
	{
		Serial_SendByte(Number/Serial_Pow(10,i) %10 + 0x30);  // 以10进制从高位到低位依次发送;根据ASCI编码方式设置偏移量
	
	}
}

```

![image.png](/assets/img/2025-04-20/image%2018.png)

解决思路：进入仿真模式设置断点进行调试可发现，由于对局部变量i采用了无符号类型进行定义(≥0)，所以一直无法退出循环，进行123450000….012345000…的循环发送，

- 直接改用int型或其他有符号数据类型

![image.png](/assets/img/2025-04-20/image%2019.png)

![image.png](/assets/img/2025-04-20/image%2020.png)

![image.png](/assets/img/2025-04-20/image%2021.png)

printf可变参数封装代码改进建议？

对于串口发送接收，可采用查询和中断两种方法，现在考虑**利用中断实现串口发送接收数据，实现流程为：**

开启USART中断

文档里面的USART中断请求居然没有列出RXNE事件标志的使能位，但是却存在于中断映像图中

![image.png](/assets/img/2025-04-20/image%2022.png)

- USART与其他外设的协同工作：这句话是什么意思？ → 后面学过DMA之后回来复习的时候整理

> 仅当使用DMA接收数据时，才使用这个标志位。
> 

![image.png](/assets/img/2025-04-20/image%2023.png)

思考：在STM32系统中是否会出现中断函数混用的现象？比如原本应用于USART1接收数据的中断服务函数被用于USART2接收数据，这是否有人为方面重写出更严谨的中断控制函数和控制服务函数的需求？

思考-中断服务函数中数据和标志位的暂存和转发：这样写的意义是什么？更方便主程序的读取吗？是不是可以考虑直接写个宏定义调用中断标志位呢？ →  

```c
uint8_t Serial_GetRxFlag(void)
{
	//读后自动清除
	if (Serial_RxFlag==1)
	{
		Serial_RxFlag = 0;
		return 1;
	}
	return 0;
}

uint8_t Serial_GetRxData(void)
{
	return Serial_RxData;
}

void USART1_IRQHandler(void)
{
	if(USART_GetFlagStatus(USART1,USART_IT_RXNE) == SET)  
	{	//数据和标志位暂存	
		Serial_RxData = USART_ReceiveData(USART1);
		Serial_RxFlag = 1;  
		USART_ClearITPendingBit(USART1,USART_IT_RXNE);
	}
}
```

在中断服务函数中实现数据和标志位的转存，然后就可以在主程序中改写成我们自己的函数，也是一种编程技巧工程技巧  → 方便后续添加其他功能拓展，然后可以从有限自动机的视角去思考更加复杂，涉及更多状态和模型的工程项目。后面的串口收发HEX&&文本数据包就有这一视角的简单案例。

```c
	while(1)
	{
//		if (USART_GetFlagStatus(USART1,USART_FLAG_RXNE)==SET)   //收到数据
//		{
//			RxData = USART_ReceiveData(USART1);	
//			OLED_ShowHexNum(1,1,RxData,2);
//		}	
			if (Serial_GetRxFlag() == 1)  //在中断服务函数中转存的标志位
			{
				RxData = Serial_GetRxData(); //在中断服务函数中转存的数据
				Serial_SendByte(RxData);
				OLED_ShowHexNum(1,8,RxData,2);
			}
	} 
```

## 0x04 USART串口数据包

基于前面单字节接收，做出多字节数据包形式的传输

串口接收多字节数据包;

数据包格式规定及数据包收发：可以利用数据包的最高位进行分割；不过本小节的实验考虑额外添加包头包尾来以易于管理

![image.png](/assets/img/2025-04-20/image%2024.png)

串口收发HEX数据包测试案例思考：

- 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF  (测试语义，期望输出，实际输出..）；如果考虑利用列表结构进行可变含包头包尾的包长管理是不是就能更方便地实现了？其实这里可能有点想当然了，我们要的是数据包格式可变，而不是内容可变，直接上列表可能有点浪费内存和性能 → 有无针对单片机开发的 支持标准库 或者是支持一些基本数据结构的 mico库提供 [Embedded Template Library](https://www.etlcpp.com/)

**启示-后续要求：利用状态机模型，画出G45-G55项目的完整状态迁移图**

![image.png](/assets/img/2025-04-20/image%2025.png)

## 0x05 串口收发HEX数据包&串口收发文本数据包

状态机编程：状态选择、状态执行、状态转移条件

尝试下载程序发现OLED没有正常显示，对比课程示例代码发现使用错了库函数，更改过来以后依然没有正常显示，串口发送数据正常但是OLED显示异常

```c
if(USART_GetFlagStatus(USART1,USART_IT_RXNE) == SET) 
```

利用仿真进行调试发现原本应该用于缓存区发送载荷数据的 `Serial_RxPacket[]` 未得到赋值，可能是标志位未能进行正确传递。

是不是应该再发送数据的时候设置标志位？为什么采用了同样的实现方式，在9-2能够跑通呢

无法在OLED上显示串口发送的数据，直接调用课程示例代码也没有实现该效果

暂时跳过了，主要是课程示例的代码也用不了然后这个STLINK接口不太稳定；或者晚点换块板子调

是否应该考虑更改串口发送数据呢？  暂时不太想调了，看看电子元器件或者定时器吧；不太理解为什么无法接收到中断标志位的清除，是否有可能是接口不太稳定？但是【9-2】上能够实现同样的功能 (电脑端发送的数据可以直接打印在OLED上）

另外，这个keil有个很麻烦的bug就是每次都无法退出仿真调试；需要将整个软件关闭重新打开

尝试检查一下接收数据的功能实现代码：

尝试一下标准实力的代码：只能进行串口发送，无法将发送的数据转存后做串口接收；OLED上无法显示接收到的数据

**最后发现是因为一直没有在PC端给串口发送数据，所以一直无法正常接收**

**复盘思考：如果想要实现(脑袋中想当然的效果)串口本身发送数据并接收数据，应该如何连线？**

当前的接线方式只能走通过串口发送数据到PC端，PC端发送数据到串口（如果没有串口助手要如何在PC端发送数据到串口上面呢？



注意在本代码的接收数据中断服务函数中，RxPacket同时被写入又同时被读出（中断服务函数中以次写入，主程序中以次读出），可能会出现数据包混用的现象，具体怎么处理还是得看应用场景。

[9-4]示例：串口发送文本数据包效果检验：发送字符点亮/关闭小灯；

## 0x06 串口下载程序

1.在keil中配置.hex文件的生成

2.配置BOOT引脚，让STM32执行BootLoade程序

![image.png](/assets/img/2025-04-20/image%2026.png)

- 配置BOOT引脚：跳线帽切换BOOT引脚配置，然后reset重新让STM32重新读取BOOT引脚。进入BOOTLOADER程序以后，STM32会不断接收USART1的数据，刷新到主闪存

如果无法搜索到串口可以尝试在设备管理器卸载端口后重新连接；

程序通过Bootloader成功刷新到主闪存，目前由于STM32仍在执行bootloader程序，所以无法观察到LED闪烁现象；需要再次进行boot引脚的修改（跳线帽切换）然后reset，可观察到LED闪烁现象

![image.png](/assets/img/2025-04-20/image%2027.png)

串口下载原理：（手机刷机，电脑重装系统，辅助主程序进行自我更新）

![image.png](/assets/img/2025-04-20/image%2028.png)

BOOT引脚配置与启动模式（windows-bios）

![image.png](/assets/img/2025-04-20/image%2029.png)

手动切换跳线帽过于麻烦了，如何提升效率？：

- STM32一键下载电路设计方案
- 利用串口的限制端口做GPIO → 注意对比大板子上的相关硬件电路设计

问题-STM32系统板上的reset按键做了哪些事情？

- 程序计数器（PC）跳转到复位向量地址
- ..

## 遗留问题
0250416-遗留问题：

后面需要在正点原子板上进行的效果展示：利用数码管反映串口数据包的接受和发送（或者直接先借用一下OLED模块吧）；利用数码管反映按键控制效果 （先把中断串口定时器这块学好）
- 思考-考虑实现一个用RUST语言写的MicroLIB?/或者用C/CPP复现MicroLIB?

## 参考资料

- 【【正点原子】 手把手教你学STM32入门教学视频单片机 嵌入式  之 F103-基于新战舰V3/精英/MINI板-串口通信原理】[https://www.bilibili.com/video/BV1Lx411Z7Qa?p=29&vd_source=137987af132b72c0f5ec9a7598b1ca2d](https://www.bilibili.com/video/BV1Lx411Z7Qa?p=29&vd_source=137987af132b72c0f5ec9a7598b1ca2d)
- 【STM32入门教程-2023版-USART串口协议 && USART串口外设】[https://www.bilibili.com/video/BV1th411z7sn?p=25&vd_source=137987af132b72c0f5ec9a7598b1ca2d](https://www.bilibili.com/video/BV1th411z7sn?p=25&vd_source=137987af132b72c0f5ec9a7598b1ca2d)

