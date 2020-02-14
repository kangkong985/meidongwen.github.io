---
ayout: post
os: true
comments: true
title:  "armMBed"
excerpt: "..."
tag:
- com
- canopen



---



# Quick Start 例程-温度模块

- 添加工程：新建文件夹作为工作区，在文件夹中添加以下内容：
  - mbed-os
  - mbed-lora-radio-drv  
  - mbed\mbed-os-example-lorawan  
  - main.cpp   
  - mbed_app.json  

- 初始化：进入工程文件夹，右键，Git Bash here`
  ```bash
  mbed new . --create-only 
  mbed target NUCLEO_L433RC_P
  mbed toolchain GCC_ARM
  mbed config GCC_ARM_PATH  "C:\Program Files (x86)\GNU Tools ARM Embedded\6 2017-q2-update\bin"
  mbed config --list 
  ```


- 编译
  进入工程文件夹，右键，Git Bash here，
  `mbed compile`

- 下装
  打开Jflash，新建一个工程，芯片选择EFMLG330 256kb
  找到编译生成的二进制文件，位于BUILD文件夹
  将*.hex文件拖入Jflash
  Target->connect
  Target->ProductionPrograming

  

  

# arm Mbed 开发步骤


- 开发环境
  - python：https://www.python.org/downloads/release/python-2713
  - GCC_ARM Toolchain：https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads，使用version6
  -  git：<https://git-scm.com>  (安装)
  - mbed-cli:https://mbed-media.mbed.com/filer_public/7f/46/7f46e205-52f5-48e2-be64-8f30d52f6d75/mbed_installer_v041.exe (安装)
  -  J-LINK 程序烧写工具 (安装)

- 创建工程文件夹
  - 新建一个文件夹，用于存放整个工程。**注：文件夹名称不能包含空格**

  - 添加操作系统文件夹：mbed-os，git clone https://github.com/ARMmbed/mbed-os

  - 添加sx1276 驱动程序文件夹：mbed-lora-radio-drv，

  - 添加驱动应用文件：lora_radio_helper.h、mbedtls_lora_config.h、trace_helper.cpp、trace_helper.h

  - 添加配置文件：mbed_app.json，见附件

  - 添加主文件：main.cpp

- 初始化

  ```bash
  mbed new . --create-only
  mbed target NUCLEO_L433RC_P
  mbed toolchain GCC_ARM
  mbed config GCC_ARM_PATH  "C:\Program Files (x86)\GNU Tools ARM Embedded\6 2017-q2-update\bin"
  mbed config --list
  ```

  

  

  ## 配置

  总配置文件mbed_app.json仅有一个，位置在最上层的目录中

  库配置文件mbed_lib.json有多个， mbed中包含许多库，每一个库都含有一个mbed_app.json文件，用于配置对应的库

   

  mbed_app.json包含两个部分：config和target_overrides

  **“config****”：{**

  "parameter1"：{"value":"****"}

  "parameter2"：{"value":"****"}

  …………

  **}**

  value用于表示参数的数值

  **“target_overrides****”：{**

  ​       **"\*": {**

  ​                     "parameter1"： "****"

  ​           "parameter2"： "****"

  ​           …………

  ​       **}**

  ​       **"** target2**": {**

  ​                     "parameterI"： "****"

  ​           "parameterII"： "****"

  ​           …………

  ​       **}**

  ​       **"** target3**": {**

  ​                     "parameter_1"： "****"

  ​           "parameter_2"： "****"

  ​           …………

  ​       **}**

     **……………………**

  **}**

   

  l  "*"部分的设置应用于所有的targets，" target1"部分的设置仅应用于target1，" target2"部分的设置仅应用于target2，……

  l  "*"部分的设置会覆盖" target1"、" target2"、" target3"……的配置

  mbed_lib.json中的配置都可以出现在mbed_app.json文件的**target_overrides**中，并且被其覆盖，因此，所有的配置都在mbed_app.json文件中编写即可，不用管各个mbed_lib.json文件

  l  配置方法：

  查看库的mbed_lib.json文件， 

  举例说明

  ：

  lora

  库的配置

  mbed_lib.json

  文件中：

  

  

  lora库的mbed_lib.json文件中：其中"name"部分表示库名，"config"部分表示配置项

  相应的在mbed_app.json文件中的配置：格式：**库名**.配置项





查看配置**

mbed compile --config

​    

## 编译

进入工程文件夹Git Bash here 

mbed compile

 

系统执行编译。

编译完成后生成mbed_config.h文件，位置在BUILD文件夹中，它包含了所有的配置参数，仅用于查看，不能对其做修改

编译完成后生成bin文件，位置在BUILD文件夹中 ，使用J-LINK烧写bin文件地址0x08000000

 





# LoRa参数配置

"lora.device-eui": "{0x70, 0xb3, 0xd5, 0x31, 0xc0, 0x00, 0x01, 0x53 }",

"lora.device-address": "0x28ad9153",                            

"lora.appskey": "{0xe0, 0x8c, 0x28, 0xff, 0xcf, 0x7a, 0x47, 0xff, 0x7a, 0xb5, 0xeb, 0x00, 0xbd, 0x00, 0x01, 0x53}",           

"lora.nwkskey": "{0x3a, 0x1e, 0xbd, 0x00, 0x3a, 0xc8, 0x8c, 0x00, 0xc0, 0xc5, 0x1e, 0xff, 0xea, 0xd2, 0x65, 0x53}",

##  



# arm Mbed API

## GPIO

DigitalIn Din1(PB_0, PullNone);  // PullUp/PullDown/PullNone/OpenDrain

DigitalOut Dout1 (PB_1);  

AnalogIn  Ain1(PA_0);

AnalogOut  Aout1 (PA_5);

int main() 

{

​    bool din1= Din1.read();

​    int ain1= Ain1.read();

​    Dout1.write(1);  //0/1

​    Aout1.write(0.5); //0.0f(0v) - 1.0f(3.3v)

​    wait(3); //3s

​    Aout1.write_u16(0x1111); //0x0000(0v) - 0xFFFF(3.3v)

}

## 串口

Serial Uart1(PB_8, PB_9, 19200);   // tx, rx

uint8_t data[100];

int main() 

{

  Uart1.[format](http://os.mbed.com/docs/v5.9/mbed-os-api-doxy/classmbed_1_1_serial_base.html#a768f1e9fd9e14c787f693745fa2538a4) (8,None,1);  // number of bits in a word/ parity / number of stop bits

  Uart1.printf("Hello World\n");

  while(Uart1.readable()) //  if there is a character available to read

  {

​    Uart1.read(data,10,NULL, SERIAL_EVENT_RX_COMPLETE, SERIAL_RESERVED_CHAR_MATCH)

​    Uart1.write(data,10,NULL, SERIAL_EVENT_RX_COMPLETE, SERIAL_RESERVED_CHAR_MATCH)

  }

}

## SPI

SPI Spi1(PC_1, PC_2, PC_3, PD_0); // mosi, miso, sclk

DigitalOut cs(PD_0); 

 

{

  Spi1.[format](http://os.mbed.com/docs/v5.9/mbed-os-api-doxy/classmbed_1_1_serial_base.html#a768f1e9fd9e14c787f693745fa2538a4) (8,3);  // number of bits per [SPI](https://os.mbed.com/docs/v5.9/mbed-os-api-doxy/classmbed_1_1_s_p_i.html) frame(4-16) / Clock polarity and phase mode (0-3)

  Spi1.frequency(1000000);

  cs.write(1); 

  cs.write(0);  

  Spi1.write(0x8F);

  cs.write(1);

}

## I2C

I2C I2c1(PD_1,PD_2);   // SDA,SCL

const int addr_read = 0x48;    //读地址 0x40<<1+0x1

const int addr_write = 0x90;    //写地址 0x40<<1+0x0

 

int main() 

{

​    char cmd[2]={0x01,0x02};

​    i2c.write(addr_write, cmd, 2);

​    wait(0.5);

​    i2c.write(addr_write, cmd, 1);

​    i2c.read( addr_read, cmd, 2);

}