# reference

- 


# project

## src

所使用芯片的开发包，包内含USB源码，以STM32L4为例：https://my.st.com/content/my_st_com/zh/products/embedded-software/mcu-mpu-embedded-software/stm32-embedded-software/stm32cube-mcu-mpu-packages/stm32cubel4.html

## files

|  |      |
| ------------------------------------------------------------ | ---- |
|  STM32Cube_FW_L4_V1.12.0\Middlewares\ST\STM32_USB_Device_Library   | com\usb\src\STM32_USB_Device_Library |
|com\usb\src\STM32_USB_Device_Library\Class\CDC\Src\usbd_cdc_if_template.c| com\usb\user\stm32l433\usbd_cdc_if.c |
|com\usb\src\STM32_USB_Device_Library\Class\CDC\Inc\usbd_cdc_if_template.h| com\usb\user\stm32l433\usbd_cdc_if.h |
|com\usb\src\STM32_USB_Device_Library\Core\Src\usbd_conf_template.c| com\usb\user\stm32l433\usbd_conf.c   |
|  com\usb\src\STM32_USB_Device_Library\Core\Src\usbd_desc_template.c | com\usb\user\stm32l433\usbd_desc.c   |
|com\usb\src\STM32_USB_Device_Library\Core\Inc\usbd_cdc_if_template.c | com\usb\user\stm32l433\usbd_conf.h   |
|com\usb\src\STM32_USB_Device_Library\Core\Inc\usbd_desc_template.c |com\usb\user\stm32l433\usbd_desc.h|
|                                                              |      |

## modify





## keil project

- source

  Drivers\STM32L4xx_HAL_Driver\Src\stm32l4xx_ll_usb.c
  com\usb\src\STM32_USB_Device_Library\Core\Src\usbd_core.c
  com\usb\src\STM32_USB_Device_Library\Core\Src\usbd_ctlreq.c
  com\usb\src\STM32_USB_Device_Library\Core\Src\usbd_ioreq.c
  com\usb\src\STM32_USB_Device_Library\Class\CDC\Src\usbd_cdc.c
  com\usb\user\ *.c
  com\usb\user\stm32l433\ *.c


- include

  com\usb\src\STM32_USB_Device_Library\Class\CDC\Inc
  com\usb\src\STM32_USB_Device_Library\Core\Inc
  com\usb\user
  com\usb\user\stm32l433

- Drivers/STM32L4xx_HAL_Driver

  stm32l4xx_hal_pcd.c
  stm32l4xx_hal_pcd_ex.c
  stm32l4xx_ll_usb.c

- USB时钟

  USB协议规定了48MHz 的USB时钟





# Intro

EP：EndPoint

# Init

- USBD_CDC_If_Init
  - HAL_PWREx_EnableVddUSB()
  - USBD_Init(&USBD_Device, &VCP_Desc, 0)
    - USBD_Device->pClass = NULL;
      USBD_Device->pDesc = pdesc;
      Initialize low level driver 
    - USBD_LL_Init
      - pdev->pData = USBD_Device->pData = &hpcd
      - HAL_PCD_Init
      - HAL_PCDEx_PMAConfig：设置EP，最多设置8个
        - ep_addr：EP地址，0x0x共16个OUT_EP用于接收，0x8x共16个IN_EP用于发射，
        - ep_kind
        - pmaadress
  - USBD_RegisterClass(&USBD_Device, &USBD_CDC)
    - USBD_Device->pClass = USBD_CDC
  - USBD_CDC_RegisterInterface(&USBD_Device, &USBD_CDC_fops)
    - USBD_Device->pUserData->Init = CDC_Itf_Init
      USBD_Device->pUserData->DeInit=  CDC_Itf_DeInit,
      USBD_Device->pUserData->Control=  CDC_Itf_Control,
      USBD_Device->pUserData->Receive=  CDC_Itf_Receive
  - USBD_Start(&USBD_Device)》USBD_LL_Start》HAL_PCD_Start	





HAL_PCD_SetupStageCallback》USBD_LL_SetupStage

### Send

USBD_CDC_Transmit》 USBD_CDC_TransmitPacket》 USBD_LL_Transmit》 HAL_PCD_EP_Transmit》 USB_EPStartXfer 

USB_EPStartXfer：启动一个端点传输，发送时ep->is_in == 1U

- USB_WritePMA：向发送寄存器填入数据
  - n：写入发送寄存器(16 bit)的数据的个数，(发送数据字节数+1) / 2
  - 发送寄存器地址为pdwVal ，pdwVal = (uint16_t *)(BaseAddr + 0x400U + ((uint32_t)wPMABufAddr * PMA_ACCESS))：
- PCD_SET_EP_TX_CNT：sets counter for the tx/rx buffer.
- SET_EP_TX_STATUS
- ep->doublebuffer判断是否使用双缓冲区，若使用，如果DTOG_TX置位则访问DBUF1，否则访问DBUF0，关于STM32中USB的双缓冲机制大家可以去阅读STM32F103C8T6参考手册的USB章节，里面会有详细讲解，我们知道STM32的USB提供了8个双向端点，而如果一个端点使能了某一方向的双缓冲机制，则TX和RX这两块区域就都由同一方向管理，可以看下分组缓冲区的的示例，重点可以看下双缓冲模式下IN端点3的缓冲区，此时，RX区域由TX_1接管了，在代码中由用于指定了TX_0和TX_1的地址，并通过PCD_SET_EP_DBUFx_CNT()指定这段缓冲中待发送数据的长度，而追溯PCD_SET_EP_DBUFx_CNT()的定义，就发现了问题，可以看到这两个IN端点的缓冲长度设置居然是使用的同一个接口-PCD_EP_TX_CNT((USBx), (bEpNum)) = (uint32_t)(wCount);所以这就是问题的所在，



### Receive

HAL_PCD_IRQHandler》 PCD_EP_ISR_Handler 》 HAL_PCD_DataOutStageCallback 》USBD_LL_DataOutStage 》  USBD_CDC_DataOut 》 CDC_Itf_Receive 》 USBD_CDC_ReceivePacket 》 USBD_LL_PrepareReceive》 HAL_PCD_EP_Receive》 USB_EPStartXfer

- PCD_EP_ISR_Handler 

  - ISTR & USB_ISTR_CTR != 0U：an endpoint has successfully completed a transaction; using DIR and EP_ID bits software can determine which endpoint requested the interrupt

  - epindex = (uint8_t)(wIstr & USB_ISTR_EP_ID)：产生该中断的EP的序号，

  - wEPVal = PCD_GET_ENDPOINT(hpcd->Instance, epindex)

  - 接收触发

    - ep = &hpcd->OUT_ep[epindex];
    - PCD_GET_EP_RX_CNT
    - USB_ReadPMA

  - 发送触发

    - ep = &hpcd->IN_ep[epindex];
    - ep->xfer_count = PCD_GET_EP_TX_CNT(hpcd->Instance, ep->num)
    - ep->xfer_buff += ep->xfer_count

  - USBD_LL_PrepareReceive 

  - ep_addr：指定EP Number

    pbuf：指针，指向用于接收数据的缓冲数组，

    size：CDC_DATA_FS_OUT_PACKET_SIZE

  - ​	

- USB_EPStartXfer：启动一个端点传输，接收时ep->is_in ==0U

  - Set RX buffer count
    - 不使用双缓冲：PCD_SET_EP_RX_CNT
    - 使用双缓冲：PCD_SET_EP_DBUF_CNT
  - PCD_SET_EP_RX_STATUS

- CDC_Itf_Receive

- 每次USB控制器收到数据后都会调用这个回调函数，从USB收到的数据就存放在参数表指向的数据缓冲区，参数还指明了收到的数据长度。CDC_Itf_Receive()函数在接收标记完这些数据之前别急着返回，不然你可能就摸不到剩下的数据了。

- USBD_CDC_ReceivePacket：复位OUT端点接收缓冲区，CDC_Itf_Receive()函数在接收完数据之后要调用该函数复位缓冲区。

- 注意，CDC_Itf_Receive是USB中断回调函数，应尽快返回，只能在函数内对数据做简单的标记或处理

