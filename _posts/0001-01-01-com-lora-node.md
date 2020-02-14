---
layout: post
com: true
comments: true
title:  "lora node"
excerpt: "..."
tag:
- com
- lora

---



reference

- lora alliance ：https://lora-alliance.org/resource-hub/lorawanr-specification-v11
- 

# project

- src

  LoRaMac node ：https://github.com/Lora-net/LoRaMac-node



- files

  |      |      |
  | ---- | ---- |
  |   LoRaMac-node-master\src   |   software\com\lora\src   |
  |      |   software\com\lora\user   |

- modify

  - 修改冲突，SUCESS改成LoRa_SUCCESS，避免与stm32l4xx.h冲突
  
      @utilities.h
  
      @i2c.c 
  
      @gps.c 
  
  - 添加LoRaMac状态获取接口
  
    ```LoRaMac.c
    @LoRaMac.c
    +#include "radio.h"  
    
    +uint32_t LoRaMacGetState(void)
    +{
    + return (uint32_t)LoRaMacState;
    +}
    
    
    @LoRaMac.h
    +uint32_t LoRaMacGetState(void);
    ```
  
  - 添加sx1276唤醒窗口（截取部分的SX1276Init函数的代码）
  
      ```sx1276.h
      @sx1276.h
      +void SX1276Wakeup(void);
      
      
      @sx1276.c
      +void SX1276Wakeup(void)  \\meidongwen
      +{
      +    uint8_t i;
      +    bool     PublicNetwork;	
      +    SX1276Reset( );
      +    RxChainCalibration( );
      +    SX1276SetOpMode( RF_OPMODE_SLEEP );
      +    SX1276IoIrqInit( DioIrq );
      +    for( i = 0; i < sizeof( RadioRegsInit ) \ sizeof( RadioRegisters_t ); i++ )
      +    {
      +        SX1276SetModem( RadioRegsInit[i].Modem );
      +        SX1276Write( RadioRegsInit[i].Addr, RadioRegsInit[i].Value );
      +    }
      +    
      +    SX1276SetModem( MODEM_FSK );
      +    SX1276.Settings.State = RF_IDLE;		
      +        \\ Random seed initialization
      +    srand1( Radio.Random( ) );
      +    
      +    PublicNetwork = true;
      +    SX1276SetPublicNetwork( PublicNetwork );
      +    SX1276SetSleep( );	
      +}
      
      ```
  
  
  - 修正接收窗口打开时间的补偿值计算（RegionCommonComputeRxWindowParameters）
  
    ```
    @RegionCommon.c
    -*windowOffset = ( int32_t )ceil( ( 4.0 * tSymbol ) - ( ( *windowTimeout * tSymbol ) \ 2.0 ) - wakeUpTime );
    +*windowOffset = ( int32_t )ceil( ( 4.0 * 0 ) - ( ( *windowTimeout * 0 ) \ 2.0 )   - wakeUpTime );
    ```
  
  
  - 修改默认的发送通道（RegionCN470InitDefaults）
  
    ```
    @RegionCN470.c
    -ChannelsDefaultMask[0] = 0xFFFF;
    -ChannelsDefaultMask[1] = 0xFFFF;
    -ChannelsDefaultMask[2] = 0xFFFF;
    -ChannelsDefaultMask[3] = 0xFFFF;
    -ChannelsDefaultMask[4] = 0xFFFF;
    -ChannelsDefaultMask[5] = 0xFFFF;
      
    +#ifdef LORA_KIWI_MODE
    +ChannelsDefaultMask[0] = 0x00FF;  \\00-15
    +ChannelsDefaultMask[1] = 0x0000;  \\16-31 
    +ChannelsDefaultMask[2] = 0x0000;  \\32-47 
    +ChannelsDefaultMask[3] = 0x0000;  \\48-63 
    +ChannelsDefaultMask[4] = 0x0000;  \\64-79
    +ChannelsDefaultMask[5] = 0x0000;  \\80-95
    +#elif defined( LORA_PICO_MODE )
    +ChannelsDefaultMask[0] = 0x0003;  \\00-15
    +ChannelsDefaultMask[1] = 0x0000;  \\16-31 
    +ChannelsDefaultMask[2] = 0x0000;  \\32-47 
    +ChannelsDefaultMask[3] = 0x0000;  \\48-63 
    +ChannelsDefaultMask[4] = 0x0000;  \\64-79
    +ChannelsDefaultMask[5] = 0x0000;  \\80-95						
    +#elif defined( LORA_CLAA_MODE )
    +ChannelsDefaultMask[0] = 0x0000;  \\00-15
    +ChannelsDefaultMask[1] = 0x0000;  \\16-31 
    +ChannelsDefaultMask[2] = 0x0000;  \\32-47 
    +ChannelsDefaultMask[3] = 0xF000;  \\48-63 
    +ChannelsDefaultMask[4] = 0x000F;  \\64-79
    +ChannelsDefaultMask[5] = 0x0000;  \\80-95							
    +#else						
    +ChannelsDefaultMask[0] = 0xFFFF;
    +ChannelsDefaultMask[1] = 0xFFFF;
    +ChannelsDefaultMask[2] = 0xFFFF;
    +ChannelsDefaultMask[3] = 0xFFFF;
    +ChannelsDefaultMask[4] = 0xFFFF;
    +ChannelsDefaultMask[5] = 0xFFFF;
    +#endif
    
    ```

  - 修改接收窗口频点（RegionCN470RxConfig）
  
    ```
    @RegionCN470.c
    -frequency = CN470_FIRST_RX1_CHANNEL + ( rxConfig->Channel % 48 ) * CN470_STEPWIDTH_RX1_CHANNEL;
      
    +#ifdef LORA_CLAA_MODE
    +frequency = CN470_FIRST_RX1_CHANNEL + ( rxConfig->Channel % 51 ) * CN470_STEPWIDTH_RX1_CHANNEL;
    +#else			
    +frequency = CN470_FIRST_RX1_CHANNEL + ( rxConfig->Channel % 48 ) * CN470_STEPWIDTH_RX1_CHANNEL;
    +#endif	
    ```

  - CLAA_MODE需要回避修改通道频点的请求(RegionCN470LinkAdrReq)
    ```
    @RegionCN470.c
                    {\\ Trying to enable an undefined channel
                        status &= 0xFE; \\ Channel mask KO
                    }
                }
    
    -           channelsMask[linkAdrParams.ChMaskCtrl] = linkAdrParams.ChMask;
    +#ifdef LORA_CLAA_MODE
    +            channelsMask[linkAdrParams.ChMaskCtrl] = 0xffff;
    +#else												
                channelsMask[linkAdrParams.ChMaskCtrl] = linkAdrParams.ChMask;
    +#endif	
    ```

  - 接收窗口   

    ```
    @RegionCN470.h
    PICO_MODE - Receive delay 1
    +#ifdef LORA_PICO_MODE
    +#define CN470_RECEIVE_DELAY1                        2000
    +#else
     #define CN470_RECEIVE_DELAY1                        1000
    +#endif
    
    PICO_MODE - Receive delay 2
    +#ifdef LORA_PICO_MODE
    +#define CN470_RECEIVE_DELAY2                        3000
    +#else
     #define CN470_RECEIVE_DELAY2                        2000
    +#endif
    
    
    PICO_MODE - Second reception window channel frequency definition.
    
    +#ifdef LORA_PICO_MODE
    +#define CN470_RX_WND_2_FREQ                         471725000
    +#else
     #define CN470_RX_WND_2_FREQ                         505300000
    +#endif
    
    ```



# IDE

- KEIL

  - source

      com\lora\src\mac\LoRaMac.c
      com\lora\src\mac\LoRaMacCrypto.c
      com\lora\src\mac\region\Region.c
      com\lora\src\mac\region\RegionCN470.c
      com\lora\src\mac\region\RegionCommon.c
      com\lora\src\radio\sx1276\sx1276.c
      com\lora\src\system\ *.c
      com\lora\src\system\crypto\ *.c
      com\lora\user\ *.c
      com\lora\user\stm32l433\ *.c

  - include

    ```
    ;..\..\com\lora\src\boards;..\..\com\lora\src\mac;..\..\com\lora\src\mac\region;..\..\com\lora\src\radio;..\..\com\lora\src\radio\sx1276;..\..\com\lora\src\system;..\..\com\lora\src\system\crypto;..\..\com\lora\user;..\..\com\lora\user\mcu
    ```
  
  - define
  
    REGION_CN470
    LORA_ * _MODE
  
  - 禁止勾选UseMicroLIB









# JoinReq-OTAA

## PrepareFrame 

OTAA请求 

- [0] :  0x00   

- [1-8] :  LoRaMacAppEui  

- [9-16] :  LoRaMacDevEui  

- [17-18] : LoRaMacDevNonce  

- [19-22] : MIC

  

OTAA建立在上下行都通的情况下。

- 网关
  检查网页的节点设置

- 参数
  - DevEUI
  - AppEUI
  - AppKey

- 频率
  接收频率和接收速率
  OnRadioTxDone -> TimerStart( &RxWindowTimer1 )
  OnRxWindow1TimerEvent -> RxWindowSetup( freq , datarate )

- Datarate
  函数AlternateDatarate
  OTAA失败原因：datarate需要达到DR_0(SF12)才能实现OTAA连接
  而原程序是以DR_5(SF7)开始尝试，不断增加，直至DR_0(SF12)，所以总是要等10min钟才能入网成功

- 接收窗口

  - 窗口打开时间
    OnRadioTxDone  -  TimerSetValue( &RxWindowTimer1, RxWindow1Delay )
    发送完毕后进入OnRadioTxDone 函数，设置并启动RxWindowTimer1，即窗口1，RxWindow1Delay 是溢出时间，即发送完毕后，经过RxWindow1Delay 的时间，打开接收窗口1，
   RxWindow1Delay在ScheduleTx函数中设置：

  ```LoRaMac.c
      if( IsLoRaMacNetworkJoined == false )
      {
          RxWindow1Delay = LoRaMacParams.JoinAcceptDelay1 + RxWindow1Config.WindowOffset;
          RxWindow2Delay = LoRaMacParams.JoinAcceptDelay2 + RxWindow2Config.WindowOffset;
      }      
  ```

  RxWindow1Config.WindowOffset在ScheduleTx函数中计算得到：
  ScheduleTx  -  RegionComputeRxWindowParameters 
  RegionCN470ComputeRxWindowParameters  -  RegionCommonComputeRxWindowParameters

  windowOffset = ( int32_t )ceil( ( 4.0 * tSymbol ) - ( ( *windowTimeout * tSymbol ) / 2.0 ) - wakeUpTime )
  ceil返回大于或者等于指定表达式的最小整数
  RegionCN470ComputeRxWindowParameters函数设置tSymbol=0, 因此windowOffset=- wakeUpTime = - SX1276GetWakeupTime()


  - 窗口持续时间

- 报文 


  - LoRaMacBuffer[0] :  0x00   
  - LoRaMacBuffer[1-8] :  LoRaMacAppEui  
  - LoRaMacBuffer[9-16] :  LoRaMacDevEui  
  - LoRaMacBuffer[17-18] : LoRaMacDevNonce  
  - LoRaMacBuffer[19-22] : MIC



## TxConfig

            LoRaMacDevEui = mlmeRequest->Req.Join.DevEui;
            LoRaMacAppEui = mlmeRequest->Req.Join.AppEui;
            LoRaMacAppKey = mlmeRequest->Req.Join.AppKey;
            ;;


mlmeReq.Req.Join.DevEui = LoRa_Params.DevEui

mlmeReq.Req.Join.AppEui = LoRa_Params.AppEui;

mlmeReq.Req.Join.AppKey = LoRa_Params.AppKey;

- Datarate = LORAWAN_DEFAULT_DATARATE
  - LoRa_Join：mlmeReq.Req.Join.Datarate = LORAWAN_DEFAULT_DATARATE;
  - LoRaMacMlmeRequest：LoRaMacParams.ChannelsDatarate = RegionAlternateDr( LoRaMacRegion, mlmeRequest->Req.Join.Datarate )
  - RegionCN470AlternateDr：LoRaMacParams.ChannelsDatarate = mlmeRequest->Req.Join.Datarate

  

- OnRadioRxDone

  - IsLoRaMacNetworkJoined = true;





# Uplink



## PrepareFrame 

Frame：Preamble + PHDR + PHDR_CRC + PHYPayload + CRC

PHYPayload = MHDR(1Byte) + MACPayload(8~23+n) + MIC(4B)

- MHDR = MType(3bit) + RFU(3bit) + Major(2bit)
  - MType ：000->Join Request，010->Unconfirmed Data Up，100->Confirmed Data Up
  - RFU：
  - Major：

- MACPayload = DevAddr(4B)+FCtr(1B)+FCnt(2B)+FOpts(0~15B)+FPort(1B)+FRMPayload(nB)
  - FHDR
    - DevAddr （4Bytes）
    - FCtrl（1Byte）= ADR + ADRACKReq + ACK + RFU + FOptsLen
      - FOptsLen（4bits）：指示FOpts的长度，范围0~15Bytes
    - FCnt（2Bytes）：报文序号
    - FOpts（0..15Bytes）
  - FPort（1Byte）
    - 0x00：FRMPayload contains MAC commands only
    - 0x01~0xDF：由应用层决定
    - 0xE0：
    - 0xE1~0xFF：reserved
  - FRMPayload（n Byte）：加密数据，保证总长度不超过最大值，using AES with a key length of 128 bits

- MIC（4Bytes）

  - 计算范围：msg = MHDR | FHDR | FPort | FRMPayload

  - 计算方法：`cmac = aes128_cmac(NwkSKey, B0 | msg)`

    B0 = 49 00 00 00 00 + Dir +DevAddr + FCntUp or FCntDown + 00 + len(msg)

    Dir =  0 for uplink frames and 1 for downlink frames.

    MIC = cmac[0..3]

    



## Config Channel/Freq

- Channel/Freq

    - 默认设置：@RegionCN470.c，RegionCN470InitDefault 函数

- RegionCN470NextChannel
  - RegionCommonCountChannels
  - CountNbOfEnabledChannels  确定使用的通道，函数返回可用的通道总数

- McpsIndication：
  - LoRaMacMibSetRequestConfirm：LoRaMacParams.ChannelsDatarate = verify.DatarateParams.Datarate;



## SX1276SetTxConfig

- TxPower & MaxPower & PaSelect
  - LoRaMacInitialization
    - RegionGetPhyParam
      - LoRaMacParamsDefaults.ChannelsTxPower = RegionGetPhyParam = CN470_DEFAULT_TX_POWER;
      - LoRaMacParamsDefaults.AntennaGain = RegionGetPhyParam = CN470_DEFAULT_MAX_EIRP;
      - LoRaMacParamsDefaults.MaxEirp = RegionGetPhyParam = CN470_DEFAULT_ANTENNA_GAIN;
    - ResetMacParameters 
      - LoRaMacParams.ChannelsTxPower = LoRaMacParamsDefaults.ChannelsTxPower;
      -  LoRaMacParams.MaxEirp = LoRaMacParamsDefaults.MaxEirp
      - LoRaMacParams.AntennaGain = LoRaMacParamsDefaults.AntennaGain
  - SendFrameOnChannel
    - txConfig.TxPower = LoRaMacParams.ChannelsTxPower;
    - txConfig.MaxEirp = LoRaMacParams.MaxEirp;
    - txConfig.AntennaGain = LoRaMacParams.AntennaGain;
  - RegionTxConfig(RegionCN470TxConfig) ：
    - txPower = txConfig.TxPower
    - maxEirp = txConfig->MaxEirp
    - txPowerIndex  = txPowerLimited = LimitTxPower = MAX( txPower, maxBandTxPower )
    - antennaGain = txConfig->AntennaGain
    - phyTxPower = RegionCommonComputeTxPower = floor( ( maxEirp - ( txPowerIndex * 2U ) ) - antennaGain )
  - Radio.SetTxConfig：phyTxPower = RegionCommonComputeTxPower( txPowerLimited, txConfig->MaxEirp, txConfig->AntennaGain );
  - SX1276SetRfTxPower：
    - PaSelect：paConfig = ( paConfig & RF_PACONFIG_PASELECT_MASK ) | SX1276GetPaSelect( SX1276.Settings.Channel );
    - MaxPower：paConfig = ( paConfig & RF_PACONFIG_MAX_POWER_MASK ) | 0x70;
    - TxPower （PaSelect=PA_BOOST）：
      - TxPower > 17 ：paConfig = ( paConfig & RF_PACONFIG_OUTPUTPOWER_MASK ) | ( uint8_t )( ( uint16_t )( power - 5 ) & 0x0F );
      - TxPower<=17 ：paConfig = ( paConfig & RF_PACONFIG_OUTPUTPOWER_MASK ) | ( uint8_t )( ( uint16_t )( power - 2 ) & 0x0F );
      - TxPower = 17-(15-OutputPower)
    - SX1276Write( REG_PACONFIG, paConfig );
    - SX1276Write( REG_PADAC, paDac )

- bandwidth
  - RegionTxConfig(RegionCN470TxConfig) ： bandwidth = 0 
  - Radio.SetTxConfig：
    - bandwidth +=7
    - SX1276Write( REG_LR_MODEMCONFIG1, bandwidth 

- datarate（SpreadingFactor）

  - datarate = LoRaMacParams.ChannelsDatarate;
  - LoRaMacParams.ChannelsDatarate ：

    - ResetMacParameters：LoRaMacParams.ChannelsDatarate = LoRaMacParamsDefaults.ChannelsDatarate
    - ProcessMacCommands：LoRaMacParams.ChannelsDatarate = linkAdrDatarate;
    - LoRaMacMcpsRequest：LoRaMacParams.ChannelsDatarate = verify.DatarateParams.Datarate;
    - OnMacStateCheckTimerEvent：LoRaMacParams.ChannelsDatarate = phyParam.Value;
  - LoRa_Send 
    - mcpsReq.Req.Confirmed.Datarate = LORAWAN_DEFAULT_DATARATE;
    - mcpsReq.Req.Unconfirmed.Datarate = LORAWAN_DEFAULT_DATARATE;
  - LoRaMacMcpsRequest
    - datarate = mcpsRequest->Req.confirmed.Datarate;  
    - datarate = mcpsRequest->Req.Unconfirmed.Datarate;
    - phyParam.Value = CN470_TX_MIN_DATARATE = DR_0
    - LoRaMacParams.ChannelsDatarate = verify.DatarateParams.Datarate  = datarate = MAX( datarate, phyParam.Value );
  - SendFrameOnChannel：txConfig.Datarate = **LoRaMacParams.ChannelsDatarate**
  - RegionTxConfig(RegionCN470TxConfig)：phyDr = DataratesCN470[txConfig->Datarate];
  - Radio.SetTxConfig：SX1276Write( REG_LR_MODEMCONFIG2 ,  phyDr

- coderate（ErrorCoding）

  - RegionTxConfig(RegionCN470TxConfig)：coderate = 1
  - Radio.SetTxConfig：SX1276Write( REG_LR_MODEMCONFIG1,coderate

- preambleLen

  - RegionTxConfig(RegionCN470TxConfig)：preambleLen = 1
  - Radio.SetTxConfig：SX1276Write( REG_LR_PREAMBLEMSB,  preambleLen

- PayloadLength & fixLen & ImplicitHeaderOn

  - RegionTxConfig(RegionCN470TxConfig)：fixLen= false

  - Radio.Send：SX1276Write( REG_LR_PAYLOADLENGTH, size );

  - Radio.SetTxConfig

    - SX1276Write( REG_LR_MODEMCONFIG1, &RFLR_MODEMCONFIG1_IMPLICITHEADER_MASK |  fixLen 

      fixLen= false ->

- MaxPayloadLength

  - SendFrameOnChannel：txConfig.PktLen = LoRaMacBufferPktLen
  - RegionTxConfig(RegionCN470TxConfig)：max = txConfig->PktLen
  - Radio.SetMaxPayloadLength ：SX1276Write( REG_LR_PAYLOADMAXLENGTH, max );

- PktLen = LoRaMacBufferPktLen;

- CrcOn

  - RegionTxConfig(RegionCN470TxConfig)：crcOn = 1
  - Radio.SetTxConfig：SX1276Write( REG_LR_MODEMCONFIG2,crcOn

- freqHopOn & hopPeriod

  - RegionTxConfig ：  freqHopOn= 0 ，hopPeriod= 0 
  - Radio.SetTxConfig：
    - if( SX1276.Settings.LoRa.FreqHopOn == true )
    - SX1276.Settings.LoRa.HopPeriod = hopPeriod
    - SX1276Write( REG_LR_HOPPERIOD, SX1276.Settings.LoRa.HopPeriod )

- MaxEirp = LoRaMacParams.MaxEirp

- AntennaGain = LoRaMacParams.AntennaGain

  

- rxContinuous & rxSingle

  - SX1276SetTx >> SX1276SetOpMode( RF_OPMODE_TRANSMITTER ) >> Radio.Tx
    - rxContinuous = SX1276.Settings.LoRa.RxContinuous;
    - rxContinuous == true  -> SX1276SetOpMode( RFLR_OPMODE_RECEIVER );
    - rxContinuous ==false  -> SX1276SetOpMode( RFLR_OPMODE_RECEIVER_SINGLE );



## LoRaMacState

- LORAMAC_TX_RUNNING标志在OnMacStateCheckTimerEvent函数中消除
    但要求进入**OnMacStateCheckTimerEvent** 函数时要求满足 LoRaMacFlags.Bits.MacDone == 1
    下面三个函数执行**LoRaMacFlags.Bits.MacDone = 1** ：

    1. **OnRadioTxTimeout(RadioEvents->TxTimeout)** 

       OnRadioTxTimeout由SX1276OnTimeoutIrq函数调用，SX1276OnTimeoutIrq是TxTimeoutTimer的溢出函数

       执行发送时，SendFrameOnChannel  -  SX1276Send(Radio.Send)  -  SX1276SetTx
       SX1276Send函数先调用 SX1276SetTxConfig(Radio.SetTxConfig)，再调用 SX1276SetTx
       SX1276SetTx启动TxTimeoutTimer
       SX1276SetTxConfig的timeout参数设置溢出时间，本程序设置为3s

       正常运行时，不使用本步骤将MacDone置1，因为发送成功时进入sx1276发送成功中断SX1276OnDio0Irq，SX1276OnDio0Irq函数先停止TxTimeoutTimer，再进入OnRadioTxDone，

    2. **OnRadioRxTimeout(RadioEvents->RxTimeout)**

       OnRadioRxTimeout由SX1276OnTimeoutIrq函数调用，SX1276OnTimeoutIrq是RxTimeoutTimer的溢出函数

       发送成功时，OnRadioTxDone  -  OnRxWindow1TimerEvent - RxWindowSetup -  SX1276SetRx(Radio.Rx)
       SX1276SetRx函数启动RxTimeoutTimer
       RxWindowSetup函数执行时，设置其rxContinuous参数为false，则溢出时间设置为LoRaMacParams.MaxRxWindow = MAX_RX_WINDOW = 3s

       正常运行时，不使用本步骤将MacDone置1，因为接收超时的情况下，SX1276SetRx函数执行后紧接着就进入SX1276OnDio1Irq，而SX1276OnDio1Irq函数会停止RxTimeoutTimer。

    3. **OnRadioRxTimeout(RadioEvents->RxTimeout)**

       OnRadioRxTimeout由SX1276OnDio1Irq调用，SX1276OnDio1Irq是sx1276芯片接收超时中断

       发送成功时，OnRadioTxDone  -  OnRxWindow2TimerEvent - RxWindowSetup -  SX1276SetRx(Radio.Rx)
       执行SX1276SetRx函数控制sx1276芯片执行接收，若没有接收到数据，则触发接收超时中断
       注意，两个接收窗口，即OnRxWindow1TimerEvent和OnRxWindow2imerEvent，都会调用SX1276SetRx，从而触发接收超时中断，但是SX1276OnDio1Irq函数执行OnRadioRxTimeout的条件要求RxSlot == 1，因此仅在第二个接收窗口所触发的接收超时中断中才会进入OnRadioRxTimeout，即若安装标准LoRaWan协议，在发送后过2s进入OnRadioRxTimeout

       正常情况下，就是通过本步骤将MacDone置1

    4. **OnRadioRxDone**   
       没有接收时，不使用本步骤将MacDone置1





## 发送数据

- LoRaMacMcpsRequest

  - LoRaMacState：如果是LORAMAC_TX_RUNNING或LORAMAC_TX_DELAYED，则直接返回LORAMAC_STATUS_BUSY

    



- Send
  - fCtrl
  - PrepareFrame( macHdr, &fCtrl, fPort, fBuffer, fBufferSize );
  - ScheduleTx( false )


- ScheduleTx

  - CalculateBackOff( LastTxChannel )
  - nextChan
  - RegionNextChannel
  - RegionComputeRxWindowParameters（RegionCN470ComputeRxWindowParameters）

    - 计算RxWindow1Config.WindowOffset(接收窗口打开时间补偿值)
    - windowOffset = ( int32_t )ceil( ( 4.0 * tSymbol ) - ( ( *windowTimeout * tSymbol ) / 2.0 ) - wakeUpTime )
      ceil返回大于或者等于指定表达式的最小整数
      RegionCN470ComputeRxWindowParameters函数设置tSymbol=0, 因此windowOffset=- wakeUpTime = - SX1276GetWakeupTime()
  - RxWindow1Delay & RxWindow2Delay

    - 设置接收窗口的打开时间: RxWindow1Delay
      - 若本次发送是OTAA请求 : RxWindow1Delay= LoRaMacParams.JoinAcceptDelay1 + RxWindow1Config.WindowOffset
      - 若本次发送是UpLink : RxWindow1Delay= LoRaMacParams.ReceiveDelay1+ RxWindow1Config.WindowOffset
  - SendFrameOnChannel(RegionCN470NextChannel)

- SendFrameOnChannel

  - Radio.Send(SX1276Send)
    - 启动TxTimeoutTimer
  - LoRaMacState |= LORAMAC_TX_RUNNING;



- 发送失败 - SX1276OnTimeoutIrq

  SX1276OnTimeoutIrq  -  RadioEvents.TxTimeout(OnRadioTxTimeout)

  如果发送失败，则不会执行下面的步骤，接下来会进入TxTimeoutTimer的溢出函数，即发送超时
  如果发送成功，则继续执行下面的步骤

- 发送成功 - SX1276OnDio0Irq

  - 停止TxTimeoutTimer

- OnRadioTxDone

  - Radio.Sleep
  - 启动RxWindowTimer1
  - 启动RxWindowTimer2
- OnMacStateCheckTimerEvent

# Downlink

LoRa报文：
DownLink：Preamble + PHDR + PHDR_CRC + PHYPayload


使用RxSlot标志 来标记接收窗口1和接收窗口2



执行SX1276SetRx函数，表示

RxSlo=0时，表明还未进入接收窗口2

RxSlo=1时，表明已进入接收窗口2

发送成功后，进入OnRadioTxDone，启动RxWindowTimer1，RxWindowTimer2
经过1s打开RxWindow1，进入OnRxWindow1TimerEvent
再经过1s打开RxWindow2，进入OnRxWindow2TimerEvent
OnRxWindowTimerEvent  -  RxWindowSetup  -  SX1276SetRx(Radio.Rx)

- RxWindowSetup

- SX1276SetRx(Radio.Rx)

- SX1276OnDio1Irq ，sx1276接收超时中断

  执行SX1276SetRx函数时，sx1276芯片执行接收，若没有接收到数据，则触发接收超时中断
  sx1276芯片执行接收的持续时间很短，所以接收超时的情况下，SX1276SetRx函数执行后紧接着就进入SX1276OnDio1Irq，若两个接收窗口都没有接收到数据，那么就会先后执行SX1276SetRx，就会先后进入两次SX1276OnDio1Irq

- RxTimeoutTimer - SX1276OnTimeoutIrq，软件定时器接收超时溢出

  SX1276SetRx会设置和启动RxTimeoutTimer，上面已经说到SX1276SetRx函数执行后紧接着就进入SX1276OnDio1Irq，而SX1276OnDio1Irq函数会停止RxTimeoutTimer，因此，一般不会进入RxTimeoutTimer的溢出函数



## RxWindow-CLASS_A

-  接收窗口1：OnRxWindow1TimerEvent
  - RxSlot = RX_SLOT_WIN_1 ：窗口标记
  - RegionRxConfig
  - RxWindowSetup - Radio.Rx(SX1276SetRx)
    - 启动TxTimeoutTimer
    - 触发接收超时中断 - SX1276Write( REG_DIOMAPPING1
    - 触发后，SX1276立刻会在DIO1引脚产生电平变化，触发EXTI1的中断

- 接收窗口2 - OnRxWindow2TimerEvent

  - RxSlot = RX_SLOT_WIN_2 ：窗口标记
  - RegionRxConfig(RegionCN470RxConfig)
    - Radio.SetChannel
    - Radio.SetRxConfig
  - RxWindowSetup - Radio.Rx(SX1276SetRx)
    - 启动TxTimeoutTimer
    - 触发接收超时中断 - SX1276Write( REG_DIOMAPPING1
    - 触发后，SX1276立刻会在DIO1引脚产生电平变化，触发EXTI1的中断

## RxWindow-CLASS_C

- RxWindow1与CLAA_A相同
- RxWindow2在发送成功后直接打开，并且一直处于打开状态，直到再次有数据发送



## 接收数据

- OnRadioRxDone

如果在接收窗口1成功接收到数据，则不会执行下面的步骤，而是执行OnRadioRxDone函数
如果在接收窗口1没有接收到数据，则继续执行下面的步骤

- OnMacStateCheckTimerEvent
- LoRaMacPrimitives->MacMcpsIndication( &McpsIndication )
  - xTaskNotifyFromISR(lora_task_handle, DL_RECEIVED,
- 





## 接收超时

- 中断SX1276OnDio1Irq



- 接收超时处理函数 - OnRadioRxTimeout

  OnRadioRxTimeout由SX1276OnDio1Irq调用，SX1276OnDio1Irq是sx1276芯片接收超时中断

  发送成功时，OnRadioTxDone  -  OnRxWindow2TimerEvent - RxWindowSetup -  SX1276SetRx(Radio.Rx)
  执行SX1276SetRx函数控制sx1276芯片执行接收，若没有接收到数据，则触发接收超时中断
  注意，两个接收窗口，即OnRxWindow1TimerEvent和OnRxWindow2imerEvent，都会调用SX1276SetRx，从而触发接收超时中断，但是SX1276OnDio1Irq函数

- OnRadioRxTimeout

  执行OnRadioRxTimeout的条件RxSlot == 1，因此仅在第二个接收窗口所触发的接收超时中断中才会进入OnRadioRxTimeout，即若安装标准LoRaWan协议，在发送后过2s进入OnRadioRxTimeout

  正常情况下，就是通过本步骤将MacDone置1

- OnRadioRxTimeout由SX1276OnTimeoutIrq函数调用，SX1276OnTimeoutIrq是RxTimeoutTimer的溢出函数
  发送成功时，OnRadioTxDone  -  OnRxWindow1TimerEvent - RxWindowSetup -  SX1276SetRx(Radio.Rx)
  SX1276SetRx函数启动RxTimeoutTimer
  RxWindowSetup函数执行时，设置其rxContinuous参数为false，则溢出时间设置为LoRaMacParams.MaxRxWindow = MAX_RX_WINDOW = 3s

  正常运行时，不使用本步骤将MacDone置1，因为接收超时的情况下，SX1276SetRx函数执行后紧接着就进入SX1276OnDio1Irq，而SX1276OnDio1Irq函数会停止RxTimeoutTimer。

    



# Timer

- TimerEvent_t

    ```timer.h
    typedef struct TimerEvent_s
    {
        uint32_t Timestamp;         //! Current timer value
        uint32_t ReloadValue;       //! Timer delay value
        bool IsRunning;             //! Is the timer currently running
        void ( *Callback )( void ); //! Timer IRQ callback function
        struct TimerEvent_s *Next;  //! Pointer to the next Timer object.
    }TimerEvent_t;
    
    ```

- TimerInit

    - obj->Timestamp = 0：溢出值
    - obj->ReloadValue = 0：重载值
    - obj->IsRunning = false：初始化时默认停止
    - obj->Callback = callback：回调函数
    - obj->Next = NULL：在TimerList( 链表 )中的下一个timer

- TimerSetValue

  - obj->Timestamp = value：设置溢出值
  - obj->ReloadValue = value：设置重载值

- TimerInsertNewHeadTimer

  - 更新TimerListHead

  - 更新定时器溢出时间：TimerSetTimeout(TimerListHead) ：设定定时器下次溢出所需时间，溢出进入定时器中断

    传递的参数是一个TimerEvent_t结构体变量，该变量的Timestamp值，就是溢出时间

    实际调用该函数时，传递的参数肯定是TimerListHead，因为TimerListHead的Timestamp肯定是最小的，即TimerListHead是所有定时器中最早溢出的。

- TimerGetValue

  - 用于获取距离上次TimerSetTimeout，经过的时间

    有时，TimerSetTimeout设置了很短的timeout时间，短到小于1ms，这时要注意补足，否则永远停留在等待这1ms中。

## TimerStart

- 时刻1：value=0，TimerListHead启动时调用函数TimerSetTimeout，设置溢出时间
- 时刻2：value=elapsedTime ，当前时间，调用TimerStart ，启动Current_Timer
- 时刻3：value = TimerListHead->Timestamp，TimerListHead溢出

- 时刻1 < 时刻2 < 时刻3
- remainingTime ：当前时间距离TimerListHead溢出还需多少时间，时刻3 - 时刻2 = TimerListHead->Timestamp - elapsedTime
- obj->Timestamp：当前时间距离Current_Timer溢出还需多少时间
  - obj->Timestamp > remainingTime：Current_Timer相较TimerListHead更晚溢出，
  - TimerInsertTimer( obj, remainingTime )：正常插入到TimerList
    - aggregatedTimestamp ：Pre_Timer，距离当前时间，的溢出时间，第一次就是TimerListHead，即remainingTime
    - aggregatedTimestampNext：Next_timer，距离当前时间，的溢出时间
    - aggregatedTimestampNext= aggregatedTimestamp + cur->Timestamp = aggregatedTimestamp + TimerListHead->Next->Timestamp
      - aggregatedTimestampNext > obj->Timestamp  ，本timer相较Next_timer更早溢出
        - obj->Timestamp -= aggregatedTimestamp：更新其溢出时间，减去aggregatedTimestamp
        - 即，溢出时间是Pre_Timer溢出之后，再经过obj->Timestamp的时间
        - 插入到Pre_Timer后面，Next_timer前面
      - aggregatedTimestampNext > obj->Timestamp ，本timer相较Next_timer更晚溢出
        - 向后继续寻找：Next_Timer变为Pre_Timer，Next_Next_Timer变为Next_Timer
  - obj->Timestamp < remainingTime：Current_Timer相较TimerListHead更早溢出，
  - TimerInsertNewHeadTimer：Current_Timer成为新的TimerListHead，它将最早溢出

## TimerStop

- TimerStop -  Stop the Head On going
  - 时刻1：value=0，该timer启动时调用函数TimerSetTimeout，设置溢出时间，
  - 时刻2：value=elapsedTime ，当前时间，调用TimerStop ，停止timer
  - 时刻3：value=obj->Timestamp，该timer的溢出时间
  - 时刻1 < 时刻2 < 时刻3
  - remainingTime ：当前时间距离该timer溢出还需多少时间， 时刻3 - 时刻2 = obj->Timestamp - elapsedTime
  - 设置下一个timer
    - TimerListHead = TimerListHead->Next;
    - TimerListHead->Timestamp += remainingTime;
    - TimerListHead->IsRunning = true;
    - TimerSetTimeout( TimerListHead );
- TimerStop -  Stop the Head not going yet
  - 
- TimerStop -  Stop an object within the list



## TimerIrqHandler

- 时刻1：value=0，该timer启动时调用函数TimerSetTimeout，设置溢出时间，

- 时刻2：value =  elapsedTime >= obj->Timestamp，当前时间，该timer溢出，调用TimerIrqHandler

- 时刻1 < 时刻2 

- Timer未溢出

  - elapsedTime：**当前时间** 距离 **上次TimerIrqHandler** 经过的时间 ，单位：ms
  - elapsedTime = TimerGetValue( ) =  RtcGetElapsedAlarmTime = **Current_Time - TimerContext** 
  - Timer未溢出：elapsedTime < TimerListHead->Timestamp：时钟还没有进行到TimerListHead的溢出时刻
    - TimerListHead->IsRunning = false：TimerListHead状态修改
    - TimerListHead->IsRunning = true：TimerListHead状态恢复
    - TimerSetTimeout：TimerContext = Current_Time，记录本次TimerIrqHandler的时间点
  - Timer溢出：elapsedTime >= TimerListHead->Timestamp：时钟已经运转超过了TimerListHead的溢出时刻

  

  

  - elapsedTime：**当前时间** 距离 **上次TimerIrqHandler** 经过的时间 ，单位：ms
  - elapsedTime = TimerGetValue( ) =  RtcGetElapsedAlarmTime = **Current_Time - TimerContext** 
  - elapsedTime < TimerListHead->Timestamp：时钟还没有进行到TimerListHead的溢出时刻
  - TimerListHead->IsRunning = false：TimerListHead状态修改
  - TimerListHead->IsRunning = true：TimerListHead状态恢复
  - TimerSetTimeout：TimerContext = Current_Time，记录本次TimerIrqHandler的时间点

  

- 比较 elapsedTime 与 TimerListHead->Timestamp

  - elapsedTime较大：时钟已经运转超过了TimerListHead的溢出时刻：
  - elapsedTime较小：：

- TimerEvent

  - 注意：TimerEvent均在中断函数中被调用

TimerGetValue：



timer.c  -  TimerSetTimeout( TimerEvent_t *obj )
HasLoopedThroughMain = 0;
obj->Timestamp = RtcGetAdjustedTimeoutValue( obj->Timestamp );
RtcSetTimeout( obj->Timestamp );    





# ADR

对于固定的节点，适合使用ADR ON

但对于移动的节点，准确来说，是通讯环境会变化的节点，适合ADR OFF，

因为移动的过程中，可能出现因为通讯环境太恶劣而断连的情况，节点发送时会测试通讯环境而使用合适的daterate，而不会直接使用DR_0(SF12)，以此来节省电源消耗，但如果通讯环境很差，调节到DR_0仍然不能正常通讯，那么节点就会进入睡眠状态，经过一段时间后在醒来发送报文，如果依旧通讯环境很差，那么会再次进入睡眠，且睡眠的时间会一次比一次长。



# Duty Cycle

REGION_CN470：RegionCN470.h，默认CN470_DUTY_CYCLE_ENABLED为0，关闭

REGION_EU433：RegionEU433.h，默认CN470_DUTY_CYCLE_ENABLED为1，打开，





# Class C

切换到CLASS_C的方法：只需要修改
Class_C模式下持续打开接收窗口的途径：
首先令OnRxWindow2TimerEvent函数调用RxWindowSetup时，是进入RXCONTINUOUS模式，这样就可以保证，一旦调用OnRxWindow2TimerEvent，之后就一直处于持续接收的模式。
接下来就是何时调用OnRxWindow2TimerEvent的问题：注意与CLASS_A/B 不同的是，CLASS_C不在TxDone中打开OnRxWindow2TimerEvent
1.初始状态：调用LoRaMacMibSetRequestConfirm, mibSet->Type=LoRaMacDeviceClass
2.工作状态：虽然是持续接收的模式，但仍要保留发送功能，而在发送状态下是不能接收的，
因此要保证发送LoRa报文后，一定可以返回到持续接收的模式，即在发送之后可能出现的所有情况中都要调用OnRxWindow2TimerEvent，共有如下几种情况：
(1)发送超时-OnRadioTxTimeout
(2)发送成功(TxDone)，RX1接收超时-OnRadioRxTimeout
(3)发送成功(TxDone)，RX1接收成功-OnRadioRxDone




```

```

# FAQ

2019-02-25：

发现问题：上行失败，KiWi网页不显示数据

分析：SX1276Send 正常执行OnRadioTxDOne正常执行

查看KiWi网关的接收日志，发现网关正常接收数据显然问题是网站没有正常显示数据

解决：联系KiWi技术人员

------

2019-03-07

Rx sensitivities

------

2019-04-08

adc battery measure

Vrefint 
内部参考电压 
查看datasheet

------

2019-04-09

天线的设计与改进：

理想情况：无线设备 = 内部电路 + 天线 ，其中，内部电路和天线都是50Ω匹配电阻   
但如果内部电路或天线不满足50Ω电阻匹配，则需要加匹配网络进行纠正：  
单独取出内部电路或天线，测量链接点的阻抗，显然不在smith chart的中心  
按顺序增加以下电路：  

1. 并联电容或电阻  
2. 串联电容或电阻  
3. 并联电容或电阻  
   然后引出新的链接点。

案例：  
lora-module产品在实验室测试发现二倍频超标，  

1. 天线采用螺旋弹簧天线  
2. 内部电路假设已做好50阻抗匹配  
3. 天线与内部电路之间有一个PI型匹配电路  
   我们打算保留天线和内部电路不变，仅修改PI型匹配电路的参数，来减少二倍频超标  
   即换算到simith chart时，470MHZ的点尽量靠近圆心，940MHZ的点处于圆图的边缘
   使用罗德施瓦茨的ZNB网络分析仪  
   断开内部电路和天线  
   单独取出天线，测量阻抗ZL  
   若是在电路板上测量。测量线的屏蔽线接电路板的射频地

我是使用ZV-Z135进行校准的，测量时机器接测量电缆，但是测量电缆没法直接接到电路板上的，所以我是先将测量电缆接一根我自己做的测量线，测量线另一端再焊到我的电路板的上，但是我校准时是只有测量电缆的，所以测出来的结果是不准的，即多了那根测量线，但是测量线又没办法接在测量电缆上一起校准，因为测量线的另一端是裸线需要焊接到电路板上的
解决办法：使用ZNB校准完毕后，连接那根测量线，在使用offset embeded，是ZNB内置的补偿功能，然后再将测量线接入电路板进行测量。

使用simith软件时，加入选择PI型匹配电路的元件时，选择的元件应该处于，变化速度缓慢平缓的区域

------

2019-04-12

CLAA MODE 



发现问题：OTAA入网失败

分析：

------

2019-04-16

OTAA失败原因：datarate需要达到DR_0(SF12)才能实现OTAA连接

而原程序是以DR_5(SF7)开始尝试，不断增加，直至DR_0(SF12)，所以总是要等10min钟才能入网成功

解决：

------

2019-05-06

发现问题：上行发送若干数据包后，停止发送

------

2019-05-08

- 分析1：
  LoRaMacMcpsRequest  >> return LORAMAC_STATUS_BUSY
  LoRaMacState & LORAMAC_TX_RUNNING =  LORAMAC_TX_RUNNING 
  **OnMacStateCheckTimerEvent** 函数执行  *LoRaMacState &= ~LORAMAC_TX_RUNNING*
  进入**OnMacStateCheckTimerEvent** 函数不满足 LoRaMacFlags.Bits.MacDone == 1
  下面三个函数执行**LoRaMacFlags.Bits.MacDone = 1** ：
  1. **OnRadioRxTimeout(RadioEvents->RxTimeout)**  -  SX1276OnDio1Irq  - MCU_Port_EXTI1_IRQHandler  -  EXTI1_IRQHandler 
     该中断由sx1276接收超时触发
  2. **OnRadioTxTimeout(RadioEvents->TxTimeout)**  - SX1276OnTimeoutIrq  -  SX1276SetTx(Radio.Tx) - TimerStart(TxTimeoutTimer)   -  SX1276Send(Radio.Send)  -  SendFrameOnChannel  -  ScheduleTx  -  Send  -  LoRaMacMcpsRequest
  3. **OnRadioRxDone**

------

2019-05-09

- 推理1：
  出于某种原因，节点发送一个报文后，没有进入上述3个地方中的任意一个，于是总是处于 *LoRaMacFlags.Bits.MacDone = 1*的状态，之后也无法发送报文，在正常发送的时候，由于本程序没有设置接收功能，程序是通过OnRadioRxTimeout将MacDone 置1，
  整理发送过程：唤醒 - OnMacStateCheckTimerEvent - 发送 - OnRadioRxTimeout - 睡眠

  发送完毕后会会调用SX1276SetTx( txTimeout )，其中txTimeout 就是OnRadioRxTimeout 的延时执行的时间
  SX1276SetTx  - SX1276SetTxConfig  -  SendFrameOnChannel
  timeOut作为SX1276SetTxConfig函数的参数被设置，在当前程序中直接设置为3e3，即3000ms，那么
  发送完毕后，经过1s打开RxWindow1，再经过1s打开RxWindow2，再经过1s就是sx1276接收超时的中断到来
  如果打开RxWindow2后，在sx1276接收超时中断到来之前就进入睡眠，睡眠模式下sx1276处于断电状态，会错过OnRadioRxTimeout，唤醒时由于MacDone = 0，OnMacStateCheckTimerEvent也无法消除LORAMAC_TX_RUNNING标志，发送失败，程序也将不会进入OnRadioRxTimeout和OnRadioTxTimeout，可见，一旦错过了某次OnRadioRxTimeout，之后将永远无法执行发送。

- 现象1：在debug模式下，在SX1276SetTx(txTimeout)处设置断点，手动控制发送，发现没有出现上行中断的的现象，因此怀疑是发送间隔太短导致，此时设置的alive_period是20s，于是将其延长至30s和1min仍然在发送20次后中断，说明与发送间隔无关，但该现象却可以用来验证推理，在SX1276SetTx处停留时，MCU虽然停止运转，但sx1276仍在工作，因此有了足够的时间等到OnRadioRxTimeout 。

- 验证1：如果该推论成立，那么显然低功耗是重要的cause，因此取消低功耗功能来验证，修改FreeRTOSConfig.h文件
  #define  configUSE_TICKLESS_IDLE   0
  验证失败，异常中断仍然存在，但发生在成功发送200多个上行包文后

- 验证2：打开低功耗功能，发送报文后等待一段时间，以保证有足够的时间留给sx1276成功触发接收超时，进入

```C
  OnRadioRxTimeout函数。
  task_lora.c  -  Task_LoRa 
  case SEND_REQUEST:
  {
          DEBUG("SEND_REQUEST");
          LoRa_Port_SendRequest();
  ++++++++vTaskDelay(3000); 
          xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
  验证失败，上行中断仍会出现
```

------

2019-05-10

- 验证3：

  ```C
  task_lora.c  -  Task_LoRa
  case SEND_REQUEST:
  {
          DEBUG("SEND_REQUEST");
          LoRa_Port_SendRequest();
  +++++++++DelayMs(3000); 
          xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
  ```

  验证失败，验证两次，两次均在上行发送431个数据包后，发生上行中断  

------

2019-05-11

- 验证4：

  ```C
  task_lora.c  -  Task_LoRa
  case SEND_REQUEST:
  {
          DEBUG("SEND_REQUEST");
          LoRa_Port_SendRequest();
  +++++++++DelayMs(4000); 
          xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite); 
  ```

  验证成功，上行中断异常不再出现



- 分析2：关于异常中断前成功发送的次数？
  异常中断总是在成功上行发送20个左右的数据包后出现



------

2019-05-13

- 总结：
  使用低功耗模式时应该确保所有事物处理完毕后再进入睡眠，且要保证在睡眠时不会发生重要的事件而被错过

- 解决：

  ```C
  task_lora.c  -  Task_LoRa
  case SEND_REQUEST:
  {
          DEBUG("SEND_REQUEST");
          LoRa_Port_SendRequest();
  +++++++++DelayMs(4000); 
          xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
  ```

------

2019-05-14

验证5：

```C
修改timeOut
```

------

2019-05-16

验证6：

```C
void Modbus_Task(void)
{
	uint32_t i;	
	uint32_t State;
  eMBInit(MB_RTU, 0x0A, 1, 115200, MB_PAR_NONE);  
  eMBEnable();  	
	while(1)
	{
		xTaskNotifyWait( 0x00,0xffffffff,&State,portMAX_DELAY );  
        for(i=0;i<0x1FFFFF;i++)
		{
			eMBPoll();
		}
	}
}

	task_lora.c  -  Task_LoRa
	case SEND_REQUEST:
	{
	        DEBUG("SEND_REQUEST");
	        LoRa_Port_SendRequest();
	++++++++xTaskNotify(modbus_task_handle, NULL, eSetValueWithOverwrite); 
	        xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
```

2019-05-14

发现问题：task_algo的心跳包文时间间隔不准确，比预期的间隔长1~2秒

分析：

该问题是解决20190506-uplink abort之后产生的，即：

```C
task_lora.c  -  Task_LoRa
case SEND_REQUEST:
{
        DEBUG("SEND_REQUEST");
        LoRa_Port_SendRequest();
+       DelayMs(4000); 
        xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
```

心跳包文时间间隔=Task_Algo_Alive_Time：

```C
task_lora.c  -  Task_LoRa
ulNotificationValue = xTaskNotifyWait(0x00,0xffffffff,&event,Task_Algo_Alive_Time);
```

正常来讲，上面的xTaskNotifyWait语句，应该准确地每隔Task_Algo_Alive_Time执行一次，即：
mcu由于xTaskNotifyWait超时而被唤醒
间隔时间：Internal_time，唤醒后进入Task_Algo的非超时执行部分(while(1)中的else部分)，执行心跳报文的发送
运行到 xTaskNotifyWait处，立即开始计时
额外时间：Extral_time，开始计时后再执行一些其他程序，假设这些程序消耗了Extral_time的时间
额外时间过后，mcu进入睡眠，并且设定睡眠时间为Task_Algo_Alive_Time - Extral_time
关键是要保证Internal_time接近于0，准确地每隔Task_Algo_Alive_Time执行一次上面的xTaskNotifyWait语句

现在在debug模式下测得Internal_time有1~2s，显然是alive time inaccurate的原因了
现在的情况：发送心跳包文之后至少延时示4000ms，程序才会再次执行到xTaskNotifyWait处。

解决：

```C
task_lora.c  -  Task_LoRa
case SEND_REQUEST:
{
        DEBUG("SEND_REQUEST");
        LoRa_Port_SendRequest();
-       DelayMs(4000); 
        xTaskNotify(task_led_handle, 0, eSetValueWithOverwrite);
        
        
        
```

