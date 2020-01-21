USB主机模块可用于实现主要的USB类
• 大容量存储类 （MSC）
• 人机接口鼠标与键盘类 （HID）
• 通信设备类 （CDC）
• 音频类 （AUDIO）
• 媒体传输协议类 （MTP）


USB CDC类(communications device class)可用于设备与主机之间的USB通信。有了CDC，再也不需要USB-TTL转接板啦，数据传输也更快。

平台：STM32F405
内容：HAL库与STD库的USB CDC类实验
实验效果：设备和电脑通过USB接口通信，完美替代之前的串口

HAL库实验
建立工程

CubeMX中加入USB_OTG_FS,选择Device Only。MiddleWares中选择communications device class IP。

如图配置时钟，USB部分需要48M时钟。







在Project->Settings中把Heap Size调大，因为USB HAL固件库中使用了malloc，如果按照默认配置将导致USB设备无法正常驱动。
Project->Generate Code产生代码。

代码分析
(只关心收发数据函数怎么用时直接看总结，跳过这一部分)

void MX_USB_DEVICE_Init(void)

```
void MX_USB_DEVICE_Init(void)
{
  USBD_Init(&hUsbDeviceFS, &FS_Desc, DEVICE_FS);

  USBD_RegisterClass(&hUsbDeviceFS, &USBD_CDC);

  USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);

  USBD_Start(&hUsbDeviceFS);
}
```

MX_USB_DEVICE_Init()的功能是对USB的初始化。直接看
USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);

```
uint8_t  USBD_CDC_RegisterInterface  (USBD_HandleTypeDef   *pdev, 
                                      USBD_CDC_ItfTypeDef *fops)
{
  uint8_t  ret = USBD_FAIL;
  
  if(fops != NULL)
  {
    pdev->pUserData= fops;
    ret = USBD_OK;    
  }
  
  return ret;
}
```

函数功能是将USBD_HandleTypeDef类型的结构体pdev的成员pUserData赋值为USBD_CDC_ItfTypeDef类型的结构体fops。
USBD_HandleTypeDef数据类型的定义如下：

```
typedef struct _USBD_HandleTypeDef
{
  uint8_t                 id;
  uint32_t                dev_config;
  uint32_t                dev_default_config;
  uint32_t                dev_config_status; 
  USBD_SpeedTypeDef       dev_speed; 
  USBD_EndpointTypeDef    ep_in[15];
  USBD_EndpointTypeDef    ep_out[15];  
  uint32_t                ep0_state;  
  uint32_t                ep0_data_len;     
  uint8_t                 dev_state;
  uint8_t                 dev_old_state;
  uint8_t                 dev_address;
  uint8_t                 dev_connection_status;  
  uint8_t                 dev_test_mode;
  uint32_t                dev_remote_wakeup;

  USBD_SetupReqTypedef    request;
  USBD_DescriptorsTypeDef *pDesc;
  USBD_ClassTypeDef       *pClass;
  void                    *pClassData;  
  void                    *pUserData;    
  void                    *pData;    
} USBD_HandleTypeDef;
```

可以看出pUserData是一个void*指针。
USBD_CDC_ItfTypeDef数据类型的定义如下：

```
typedef struct _USBD_CDC_Itf
{
  int8_t (* Init)          (void);
  int8_t (* DeInit)        (void);
  int8_t (* Control)       (uint8_t, uint8_t * , uint16_t);   
  int8_t (* Receive)       (uint8_t *, uint32_t *);  

}USBD_CDC_ItfTypeDef;
```

USBD_CDC_ItfTypeDef有四个成员，分别是四个函数指针。
下面来看用这两个结构体类型声明的变量。
usbd_device.c文件中声明了

```
USBD_HandleTypeDef hUsbDeviceFS;
```

usbd_cdc_if.c文件中声明了

```
USBD_CDC_ItfTypeDef USBD_Interface_fops_FS = 
{
  CDC_Init_FS,
  CDC_DeInit_FS,
  CDC_Control_FS,  
  CDC_Receive_FS
};
```

看到这里就明白了void MX_USB_DEVICE_Init(void)函数对USBD_HandleTypeDef hUsbDeviceFS这个结构体进行了初始化。而USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);初始化了其中的一个成员。
具体功能就是ST预留了函数接口，用户只需要修改USBD_Interface_fops_FS中的四个函数。

```
USBD_CDC_ItfTypeDef USBD_Interface_fops_FS = 
{
  CDC_Init_FS,
  CDC_DeInit_FS,
  CDC_Control_FS,  
  CDC_Receive_FS
};
```

这四个函数的函数体在usbd_cdc_if.c文件中。
USB固件库中已经调用好了这四个函数，用户只需要在函数体中添加代码实现功能即可。无需调用函数。固件库中具体的调用方法举例：
usbd_cdc.c文件中的

```
static uint8_t  USBD_CDC_Init (USBD_HandleTypeDef *pdev, 
                               uint8_t cfgidx)
{
  uint8_t ret = 0;
  USBD_CDC_HandleTypeDef   *hcdc;
  ...
  ...
 ((USBD_CDC_ItfTypeDef *)pdev->pUserData)->Init();
  ...
  ...
}    
```

调用了函数 int8_t CDC_Init_FS(void)
pdev是USBD_HandleTypeDef *指针，pUserData是USBD_HandleTypeDef类型结构体中的成员。这个成员被注册为了一个USBD_CDC_ItfTypeDef类型的结构体。而Init()是这个结构体中的成员。
((USBD_CDC_ItfTypeDef *)pdev->pUserData)->Init();这一句话其实就是CDC_Init_FS这个函数的地址。
下面来看

```
USBD_CDC_ItfTypeDef USBD_Interface_fops_FS = 
{
  CDC_Init_FS,
  CDC_DeInit_FS,
  CDC_Control_FS,  
  CDC_Receive_FS
};
```

这四个函数的功能。
只要会使用这四个函数，就可以用stm32与pc进行usb通信了。

int8_t CDC_Init_FS(void)

```
static int8_t CDC_Init_FS(void)
{ 
  /* USER CODE BEGIN 3 */ 
  /* Set Application Buffers */
  USBD_CDC_SetTxBuffer(&hUsbDeviceFS, UserTxBufferFS, 0);
  USBD_CDC_SetRxBuffer(&hUsbDeviceFS, UserRxBufferFS);
  return (USBD_OK);
  /* USER CODE END 3 */ 
}
```

CDC初始化函数，设置了收发Buffer。

int8_t CDC_Control_FS  (uint8_t cmd, uint8_t* pbuf, uint16_t length)

```
static int8_t CDC_Control_FS  (uint8_t cmd, uint8_t* pbuf, uint16_t length)
{ 
  /* USER CODE BEGIN 5 */
  switch (cmd)
  {
  case CDC_SEND_ENCAPSULATED_COMMAND:
 
    break;

  case CDC_GET_ENCAPSULATED_RESPONSE:
 
    break;

  case CDC_SET_COMM_FEATURE:
 
    break;

  case CDC_GET_COMM_FEATURE:

    break;

  case CDC_CLEAR_COMM_FEATURE:

    break;

  /*******************************************************************************/
  /* Line Coding Structure                                                       */
  /*-----------------------------------------------------------------------------*/
  /* Offset | Field       | Size | Value  | Description                          */
  /* 0      | dwDTERate   |   4  | Number |Data terminal rate, in bits per second*/
  /* 4      | bCharFormat |   1  | Number | Stop bits                            */
  /*                                        0 - 1 Stop bit                       */
  /*                                        1 - 1.5 Stop bits                    */
  /*                                        2 - 2 Stop bits                      */
  /* 5      | bParityType |  1   | Number | Parity                               */
  /*                                        0 - None                             */
  /*                                        1 - Odd                              */ 
  /*                                        2 - Even                             */
  /*                                        3 - Mark                             */
  /*                                        4 - Space                            */
  /* 6      | bDataBits  |   1   | Number Data bits (5, 6, 7, 8 or 16).          */
  /*******************************************************************************/
  case CDC_SET_LINE_CODING:   
    
    break;

  case CDC_GET_LINE_CODING:     

    break;

  case CDC_SET_CONTROL_LINE_STATE:

    break;

  case CDC_SEND_BREAK:
 
    break;    
    
  default:
    break;
  }

  return (USBD_OK);
  /* USER CODE END 5 */
}
```

CDC控制命令处理，列举了主机有可能向设备发送的一些命令。没有具体的处理过程，需要用户自己编写。其中包括串口参数的设置，要做串口转USB通信的话需要修改这里。只是为了用USB与PC通信则不用管这里。每个命令具体的意思需要查询CDC类手册。

int8_t CDC_Receive_FS (uint8_t* Buf, uint32_t *Len)

```
static int8_t CDC_Receive_FS (uint8_t* Buf, uint32_t *Len)
{
  /* USER CODE BEGIN 6 */
  USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &Buf[0]);
  USBD_CDC_ReceivePacket(&hUsbDeviceFS);
  return (USBD_OK);
  /* USER CODE END 6 */ 
}
```

接收函数，Buf为接收缓存。这个缓存实际上就是CDC_Init_FS()中设置的UserRxBufferFS[]数组。这个全局数组的定义在usbd_cdc_if.c文件中。Len为接收到数据的长度。这个变量不是全局的，需要用户声明变量把这个传出去。
HAL库源码已经完成了对CDC_Receive_FS ()的调用，代码在usb_device.c文件中

```
static uint8_t  USBD_CDC_DataOut (USBD_HandleTypeDef *pdev, uint8_t epnum)
{      
  USBD_CDC_HandleTypeDef   *hcdc = (USBD_CDC_HandleTypeDef*) pdev->pClassData;
  
  /* Get the received data length */
  hcdc->RxLength = USBD_LL_GetRxDataSize (pdev, epnum);
  
  /* USB data will be immediately processed, this allow next USB traffic being 
  NAKed till the end of the application Xfer */
  if(pdev->pClassData != NULL)
  {
     //这一行完成调用
    ((USBD_CDC_ItfTypeDef *)pdev->pUserData)->Receive(hcdc->RxBuffer, &hcdc->RxLength);

    return USBD_OK;
  }
  else
  {
    return USBD_FAIL;
  }
}
```

用户只需要在这个函数中添加代码，将数据和数据长度复制到自己的接收缓存中即可。而不需要在自己的程序中调用这个函数。
以下举一个简单的例子：

```
uint8_t my_RxBuf[100];
uint32_t my_RxLength;
static int8_t CDC_Receive_FS (uint8_t* Buf, uint32_t *Len)
{
  /* USER CODE BEGIN 6 */
    memcpy(my_RxBuf,Buf,*Len);
    my_RxLength=*Len;
  USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &Buf[0]);
  USBD_CDC_ReceivePacket(&hUsbDeviceFS);
  return (USBD_OK);
  /* USER CODE END 6 */ 
}
```

发送函数也定义在usbd_cdc_if.c文件中：
```
uint8_t CDC_Transmit_FS(uint8_t* Buf, uint16_t Len)
{
  uint8_t result = USBD_OK;
  /* USER CODE BEGIN 7 */ 
  USBD_CDC_HandleTypeDef *hcdc = (USBD_CDC_HandleTypeDef*)hUsbDeviceFS.pClassData;
  if (hcdc->TxState != 0){
    return USBD_BUSY;
  }
  USBD_CDC_SetTxBuffer(&hUsbDeviceFS, Buf, Len);
  result = USBD_CDC_TransmitPacket(&hUsbDeviceFS);
  /* USER CODE END 7 */ 
  return result;
}
```

Buf表示待发送数据的指针，Len表示长度。这个函数使用很简单，发送时直接调用即可。例如：
uint8_t my_TxBuf[10];
for(i=0;i<10;i++)
   my_TxBuf[i]=i;
while(CDC_Transmit_FS(my_TxBuf, 10)!=USBD_OK)
{}

总结
STM32 CDC类HAL库的使用：

使用CubeMx建立工程，注意修改HeapSize。
usbd_cdc_if.c中CDC_Receive_FS()是接收函数。这个函数不需要调用。直接在函数中添加代码把接受到的数据和数据长度复制到自己定义的接收缓存。
usbd_cdc_if.c中CDC_Transmit_FS()是发送函数。要发送时调用这个函数,需要传入待发送数据的指针和长度。


STD库实验
建立工程
如下图添加文件








代码分析
使用函数
USBD_Init(&USB_OTG_dev,USB_OTG_FS_CORE_ID,&USR_desc,&USBD_CDC_cb,&USR_cb);
初始化USB。
这里注意，usbd_cdc_if_template文件对应于HAL库中的usbd_cdc_if.c。而usbd_cdc_vcp.c文件是用usbd_cdc_if_template修改得到的。这个文件不属于USB库。实际上直接使用usbd_cdc_if_template也是可以的。我之所以使用usbd_cdc_vcp.c文件是因为我下载的官方例程就这么写的，懒得再改~
usbd_cdc_vcp.c中声明了VCP_fops 。
CDC_IF_Prop_TypeDef VCP_fops = 
{
  VCP_Init,
  VCP_DeInit,
  VCP_Ctrl,
  VCP_DataTx,
  VCP_DataRx
};

与HAL库中不同的是，这里可以注册五个fops。将发送函数也注册到了fops中。数据收发函数分别是
uint16_t VCP_DataRx(uint8_t* Buf, uint32_t Len)
uint16_t VCP_DataTx(uint8_t* Buf, uint32_t Len)
与HAL库中的使用方法完全一致，这里就不再写了。
因为发送函数加了static，所以使用时需要声明:
extern CDC_IF_Prop_TypeDef VCP_fops
调用时:
VCP_fops.pIf_DataTx(my_Tx_Buf,10);
当然直接拿掉static也可以。
实验效果
下载stm官方VCP驱动,官网资料编号stsw-stm32102。按照readme.txt安装驱动。
连接设备，可以正常使用。
打开串口调试助手，不需要管波特率等配置，可以正常收发数据。36k的图像秒传，达到了预期效果。

作者：posedge_clk
链接：https://www.jianshu.com/p/82f277d0fe2b
来源：简书
简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。