---
layout: post
com: true
comments: true
title:  "modbus"
excerpt: "..."
tag:
- com
- modbus


---



# reference

The Modbus Organization：http://www.modbus.org/tech.php\



# project

- src

  FreeModbus：https://sourceforge.net/projects/freemodbus.berlios/files/latest/download

- files

  |  |      |
  | ------------------------ | ---- |
  | freemodbus-v1.5.0\modbus | software\com\modbus\src\modbus|
  |                          | software\com\modbus\user|
  |                          |      |

  注：编写user文件时，不要include modbus源码中的头文件，modbus源码中的头文件只供其内部源文件使用。

- modify

  - 添加发送报文的CRC校验 @mbrtu.c
    ```
    pucSndBufferCur[usSndBufferCount++] = ( UCHAR )( usCRC16 & 0xFF );  //meidongwen
    pucSndBufferCur[usSndBufferCount++] = ( UCHAR )( usCRC16 >> 8 );    //meidongwen
    ```


# ide
- keil MDK

  - source 
  - include



  - Undefined symbol __aeabi_assert (referred from mbascii.o)
    使用Keil工具时，MicroLib不支持assert()，在target中钩掉USE MicroLIB



# 通讯机制

Modbus协议是一项应用层报文传输协议，包括ASCII、RTU、TCP三种报文类型。

标准的Modbus协议物理层接口有RS232、RS422、RS485和以太网接口，采用master/slave方式通信。

使用RTU模式，消息发送至少要以3.5个字符时间的停顿间隔开始。在网络波特率下多样的字符时间，这是最容易实现的(如下图的T1-T2-T3-T4所示)。传输的第一个域是设备地址。可以使用的传输字符是十六进制的0...9,A...F。网络设备不断侦测网络总线，包括停顿间隔时间内。当第一个域（地址域）接收到，每个设备都进行解码以判断是否发往自己的。在最后一个传输字符之后，一个至少3.5个字符时间的停顿标定了消息的结束。一个新的消息可在此停顿后开始。

整个消息帧必须作为一连续的流转输。如果在帧完成之前有超过1.5个字符时间的停顿时间，接收设备将刷新不完整的消息并假定下一字节是一个新消息的地址域。同样地，如果一个新消息在小于3.5个字符时间内接着前个消息开始，接收的设备将认为它是前一消息的延续。这将导致一个错误，因为在最后的CRC域的值不可能是正确的。一典型的消息帧如下所示： 



FreeModbus协议栈通过淳口中断接收一帧数据，用户需在串口接收中断中回调prvvUARTRxISR()函数；

在第一阶段中eMBInit()函数中赋值pxMBFrameCBByteReceived = xMBRTUReceiveFSM,发生接收中断时,最终调用xMBRTUReceiveFSM函数对数据进行接收；

当主机发送一帧完整的报文后，3.5T定时器中断发生，定时器中断最终回调xMBRTUTimerT35Expired函数;

至此：从机接收到一帧完整的报文，存储在ucRTUBuf[MB_SER_PDU_SIZE_MAX]全局变量中，定时器禁止，接收机状态为空闲；

3：解析报文机制




- big-Endian





## 基础

- xMBPortEventPost(eMBEventType) ：eQueuedEvent = eMBEventType
- xMBPortEventGet ： eEvent = eQueuedEvent
- eMBPoll：switch(eEvent )

- 



## 初始化

接收状态，直至接收到完整的一帧




- vMBPortTimersEnable
- pxMBPortCBTimerExpired( xMBRTUTimerT35Expired )

  - eRcvState = STATE_RX_IDLE;
- 
- 

- 初始化
  - eMBInit

    - eMBRTUInit
      - xMBPortSerialInit
      - xMBPortTimersInit
    - xMBPortEventInit

  - eMBEnable

    - pvMBFrameStartCur( eMBRTUStart )
      - eRcvState = STATE_RX_INIT;
      - vMBPortSerialEnable( TRUE, FALSE );
      - vMBPortTimersEnable(  )：启动定时器，3.5s后进入定时器中断函数

  - pxMBPortCBTimerExpired( xMBRTUTimerT35Expired ) ，eRcvState == STATE_RX_INIT

    - xMBPortEventPost( EV_READY )：eQueuedEvent = EV_READY 
    - vMBPortTimersDisable
    - eRcvState = STATE_RX_IDLE



## 接收


- Frame Receiving：接收状态，直至接收到完整的一帧

- ucRTUBuf：接收缓存

- UARTx_IRQHandler ：接收到第一个 Byte 的数据

  - pxMBFrameCBByteReceived( xMBRTUReceiveFSM ) 

    - xMBPortSerialGetByte：获取该1 Byte 数据
    - eRcvState == STATE_RX_IDLE
    - usRcvBufferPos = 0：first  Byte 
    - ucRTUBuf[usRcvBufferPos++] = ucByte ：储存该1 Byte 数据
    - eRcvState = STATE_RX_RCV：接下来进入接收循环，直至发生定时器溢出，即接收完成完整的一帧数据
    - vMBPortTimersEnable：启动3.5s 定时器，在接收状态，要求接收两个字节的时间间隔为 ~3.5T

- UARTx_IRQHandler ：接收到第二个 Byte 的数据

  - pxMBFrameCBByteReceived( xMBRTUReceiveFSM ) 
    - xMBPortSerialGetByte：获取该1 Byte 数据
    - eRcvState == STATE_RX_RCV
    - ucRTUBuf[usRcvBufferPos++] = ucByte：储存该1 Byte 数据
    - vMBPortTimersEnable：启动3.5s 定时器，在接收状态，要求接收两个字节的时间间隔为 ~3.5T

- UARTx_IRQHandler ：。。。。。。。。。。。。。。。

- pxMBPortCBTimerExpired( xMBRTUTimerT35Expired )：定时器溢出，表明距离上次接收到一个字节，已经经过了3.5T的时间，完整的一帧数据已接收完成

  - eRcvState == STATE_RX_RCV：这是该数据帧最后一个Byte对应的 UARTx_IRQHandler 保留的状态
  - xMBPortEventPost( EV_FRAME_RECEIVED )
    - eQueuedEvent = EV_FRAME_RECEIVED
    - xEventInQueue = TRUE
  - vMBPortTimersDisable：定时器停止工作
  - eRcvState = STATE_RX_IDLE：在下一个数据来临之前，处于接收空闲状态

- eMBPoll

  - xMBPortEventGet
    - eEvent = eQueuedEvent =EV_FRAME_RECEIVED 
    - xEventInQueue = FALSE
  - peMBFrameReceiveCur( eMBRTUReceive )：ucRTUBuf储存了此次接收到的一个完整数据帧，从中取得如下信息
    - ucRcvAddress：从机地址
    - ucMBFrame：PDU单元
    - usLength：PDU长度
  - xMBPortEventPost( EV_EXECUTE )：判断从机地址地是否一致，若一致，上报报文解析事EV_EXECUTE
    - eQueuedEvent = EV_EXECUTE
    - xEventInQueue = TRUE

- eMBPoll

  - xMBPortEventGet

    - eEvent = eQueuedEvent =EV_EXECUTE 
    - xEventInQueue = FALSE

  - FuncHandle：根据功能码，调用对应的功能函数，执行相应的功能。

  - 判断ucRcvAddress，如果不是广播地址(0x00)，就发送回复报文

  - peMBFrameSendCur( eMBRTUSend )：发送回复报文，

    - eRcvState = STATE_RX_IDLE

- FuncHandle - eMBFuncReportSlaveID

- FuncHandle - eMBFuncReadInputRegister

- FuncHandle - eMBFuncReadHoldingRegister

- FuncHandle - eMBFuncWriteMultipleHoldingRegister

- FuncHandle - eMBFuncWriteHoldingRegister

- FuncHandle - eMBFuncReadWriteMultipleHoldingRegister

- FuncHandle - eMBFuncReadCoils

- FuncHandle - eMBFuncWriteCoil

- FuncHandle - eMBFuncWriteMultipleCoils

- FuncHandle - eMBFuncReadDiscreteInputs

  

  

  成功接收到完整的一帧数据

  - Modbus_Board_UARTx_IRQHandler：接收1 Byte 数据

  - > pxMBFrameCBByteReceived( xMBRTUReceiveFSM ) - STATE_RX_IDLE

    - xMBPortSerialGetByte：获取该1 Byte 数据
    - ucRTUBuf[usRcvBufferPos++] = ucByte ：储存该1 Byte 数据

    在第二阶段，从机接收到一帧完整的报文后，上报“接收到报文”事件，eMBPoll函数轮询，发现“接收到报文”事件发生，调用peMBFrameReceiveCur函数，此函数指针在eMBInit被赋值eMBRTUReceive函数，最终调用eMBRTUReceive函数，从ucRTUBuf中取得从机地址、PDU单元和PDU单元的长度，然后判断从机地址地是否一致，若一致，上报“报文解析事件”EV_EXECUTE,(xMBPortEventPost( EV_EXECUTE ));“报文解析事件”发生后，根据功能码，调用xFuncHandlers[i].pxHandler( ucMBFrame, &usLength )对报文进行解析，此过程全部在eMBPoll函数中执行；

    - eMBPoll - EV_FRAME_RECEIVED
      - 
    - eMBPoll - EV_EXECUTE
      - 



xMBRTUReceiveFSM - STATE_RX_IDLE：

- usRcvBufferPos = 0 ：
- ucRTUBuf[usRcvBufferPos++] = ucByte：获取的数据存入ucRTUBuf
- eRcvState = STATE_RX_RCV：
- vMBPortTimersEnable：启动3.5s 定时器，



 ## 发送



整个报文帧必须以连续的字符流发送，如果两个字符之间的空闲大于1.5个字符时间，则报文帧认为不完整，应该被接收点丢弃。

 注意：RTU接收驱动程序的实现，由于T1.5和T3.5的定时，隐含着大量对中断的管理。在高通信速率下，这导致CPU负担家中，因此，在<=19200bps时，这两个定时必须严格遵守；对于>19200bps的情形，应该使用2个定时的固定值，建议字符间的超时时间t1.5为750us；帧间超时时间为1.75ms。

实际使用的时候，我觉得T1.5都没有必要关注。据我用示波器观察，在一个帧之内，字节和字节之间也就是停止位和起始位，字节和字节是紧密连接在一起的。所以我在程序里面就没有对T1.5进行处理。

而在帧与帧之间的T3.5则需要程序处理。因为RTU模式没有起始符和结束符，两个数据包之间只能靠时间间隔来区分。但是仪表在工厂实际的使用过程中，一般都是间隔40ms、50ms，甚至更长时间读一次数据，这之间的间隔完全超过了T3.5。



假设现在波特率是9600bps，要发送读取数据的请求：01 03 00 1e 00 01 e4 0c

每个字节包含一个起始位和一个停止位，这样发送这串命令要花费时间8*10/9600 = 8.3ms

即第一帧发送时间是8.3ms

而T3.5的时间是 3.5*10/9600 = 3.65ms

所以第二帧数据开始发送的时间至少是第12ms开始（8.3+3.65 = 11.95ms）

下面是这段程序的时间线。如果ModbusInterval在串口中断里面赋值为1的话，那么从10ms就要开始处理数据，那么处理完后，可能还没到12ms就可以接收下一帧数据。这在时间间隔上不满足T3.5的要求；所以还是赋值为2最为合适。

- eMBMasterReqReadHoldingRegister
  - vMBMasterGetPDUSndBuf(&ucMBFrame) ： ucMBFrame = ucMasterRTUSndBuf
  - 填充 ucMBFrame 数组，相当于填充 ucMasterRTUSndBuf
  - xMBPortEventPost( EV_FRAME_SENT )
- eMBPoll
  - vMBMasterGetPDUSndBuf( &ucMBFrame )：ucMBFrame = ucMasterRTUSndBuf
  - peMBFrameSendCur ：参数是ucMBFrame ，相当于ucMasterRTUSndBuf
- eMBRTUSend
  - eRcvState == STATE_RX_IDLE
  - pucSndBufferCur = ( UCHAR * ) pucFrame - 1; ：pucSndBufferCur  = ucMasterRTUSndBuf
  - usSndBufferCount = 1;
  - pucSndBufferCur[MB_SER_PDU_ADDR_OFF] = ucSlaveAddress;
- pxMBFrameCBTransmitterEmpty( xMBRTUTransmitFSM )
  - eSndState = STATE_TX_XMIT
  - usSndBufferCount != 0
  - xMBPortSerialPutByte( ( CHAR )*pucSndBufferCur );
  -  pucSndBufferCur++：准备发送下一个字节
  - usSndBufferCount-- ：用于判断
- pxMBFrameCBTransmitterEmpty( xMBRTUTransmitFSM )
  - eSndState = STATE_TX_XMIT
  - usSndBufferCount = 0
  - xMBPortEventPost( EV_FRAME_SENT );
  - vMBPortSerialEnable( TRUE, FALSE );
  - eSndState = STATE_TX_IDLE;



# 通讯报文

成对的Request与Response的Function code相同

- ADU( 应用数据单元 )：Additional address(1 Byte)  + PDU(1~253 Bytes) + Error check(2 Bytes)

  - Additional address：从机地址
  - PDU( 协议数据单元 )：Function code(1 Byte) + Data(0~252 Bytes)
  - Error check：CRC校验

  



## 0x01：Read Coils



## 0x02：Read Discrete Inputs



## 0x03：Read Holding Registers

  - Request
      - Function code：0x03
      - Data：Starting_Address(2 Bytes) + Register_Quantity(2 Bytes)
          - Starting_Address：0x0000 ~ 0xFFFF
          - Register_Quantity：1~125

  - Response
      - Function code：0x03
      - Data：Byte_Count (1 Byte) + Register_Value(Byte_Count Bytes)
          - Byte_Count： 2 * Register_Quantity，因为每个寄存器2Byte
          - Register_Value：
## 0x04：Read Input Registers
## 0x05：Write Single Coil

- Request
  - Function code：0x05
  - Data：Starting_Address(2 Bytes) + Register_Quantity(2 Bytes)
    - Output Address：0x0000 ~ 0xFFFF
    - Output Value：开关量，0x0000 或者 0xFF00
- Response
  - Function code：0x05
  - Data：Starting_Address(2 Bytes) + Register_Quantity(2 Bytes)
    - Output Address：0x0000 ~ 0xFFFF
    - Output Value：开关量，0x0000 或者 0xFF00



## 0x06：Write Single Register
## 0x0F：Write Multiple Coils.
## 0x10：Write Multiple registers
## 0x14：Read File Record
## 0x16：Mask Write Register
## 0x17：Read/Write Multiple registers
## 0x2B：Read Device Identification