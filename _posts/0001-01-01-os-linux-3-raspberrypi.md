---
layout: post
os: true
comments: true
title:  "树莓派学习"
excerpt: "..."
tag:
- os
- linux
- raspberry


---

# Reference

RPi U-Boot ：<https://elinux.org/RPi_U-Boot>

U-boot：<http://www.denx.de/wiki/U-Boot/>



# HOST开发环境

## 交叉编译工具

- 命名规则：arch -vendor -os -(gnu)eabi
  - arch – 体系架构，如ARM，MIPS（通过交叉编译工具生成的可执行文件或系统镜像的运行平台或环境）
  - vendor – 工具链提供商
  - os – 目标操作系统（host主要操作平台，也就是编译时的系统）
  - eabi – 嵌入式应用二进制接口（Embedded Application Binary Interface）
    - abi：二进制应用程序接口(Application Binary Interface (ABI) for the ARM Architecture)。描述应用程序（或者其他类型）和操作系统之间或其他应用程序的低级接口。
    - eabi：嵌入式ABI。嵌入式应用二进制接口指定了文件格式、数据类型、寄存器使用、堆积组织优化和在一个嵌入式软件中的参数的标准约定。开发者使用自己的汇编语言也可以使用 EABI 作为与兼容的编译器生成的汇编语言的接口 。abi是计算机上的，eabi是嵌入式平台上（如ARM，MIPS等）。
  - gnueabi：用于 armel 架构
  - gnueabihf：用于 armhf 架构，armel 和 armhf 这两种架构在对待浮点运算采取了不同的策略（有 fpu 的 arm 才能支持这两种浮点运算策略）。其实这两个交叉编译器只不过是 gcc 的选项 -mfloat-abi 的默认值不同。gcc 的选项 -mfloat-abi 有三种值 soft、softfp、hard（其中后两者都要求 arm 里有 fpu 浮点运算单元，soft 与后两者是兼容的，但 softfp 和 hard 两种模式互不兼容）：
    soft： 不用fpu进行浮点计算，即使有fpu浮点运算单元也不用，而是使用软件模式。
    softfp： armel架构（对应的编译器为 arm-linux-gnueabi-gcc ）采用的默认值，用fpu计算，但是传参数用普通寄存器传，这样中断的时候，只需要保存普通寄存器，中断负荷小，但是参数需要转换成浮点的再计算。
    hard： armhf架构（对应的编译器 arm-linux-gnueabihf-gcc ）采用的默认值，用fpu计算，传参数也用fpu中的浮点寄存器传，省去了转换，性能最好，但是中断负荷高。

- 版本
  - arm-none-eabi-gcc

    GNU；ARM架构32位芯片；交叉编译 ARM 架构的裸机系统，包括 ARM Linux 的 boot、kernel，不适用编译 Linux 应用程序，一般适合 ARM7、Cortex-M 和 Cortex-R 内核的芯片使用，所以不支持那些跟操作系统关系密切的函数，比如fork(2)，他使用的是 newlib 这个专用于嵌入式系统的C库。

  - arm-linux-gnueabihf-gcc

    Linaro 公司；基于GCC；ARM架构32位芯片；交叉编译系统中所有环节的代码，包括裸机程序、u-boot、Linux kernel、filesystem和App应用程序。

  - aarch64-linux-gnu-gcc

    Linaro 公司；基于GCC；ARMv8架构64位芯片；交叉编译裸机程序、u-boot、Linux kernel、filesystem和App应用程序。

  - arm-none-linux-gnueabi-gcc

    Codesourcery 公司（目前已被Mentor收购）；基于GCC；使用Glibc库；ARM架构32位芯片；交叉编译系统中所有环节的代码，包括裸机程序、u-boot、Linux kernel、filesystem和App应用程序。主要用于基于ARM架构的Linux系统。其浮点运算非常优秀。一般ARM9、ARM11、Cortex-A 内核，带有 Linux 操作系统的会用到。

  - arm-none-elf-gcc

    Codesourcery 公司（目前已经被Mentor收购）；基于GCC；ARM架构32位芯片；如ARM7、ARM9、Cortex-M/R芯片程序。

  - arm-eabi-gcc

    Android ARM 编译器。

  - armcc

    ARM 公司；功能和 arm-none-eabi 类似；编译裸机程序（u-boot、kernel），但是不能编译 Linux 应用程序。armcc一般和ARM开发工具一起，Keil MDK、ADS、RVDS和DS-5中的编译器都是armcc，所以 armcc 编译器都是收费的（爱国版除外，呵呵~~）。

  - arm-none-uclinuxeabi-gcc

    用于uCLinux，使用Glibc

  - arm-none-symbianelf-gcc

    用于symbian





- 





# HOST Config

```
sudo apt-get update
sudo apt-get install kernel-package
sudo apt-get install libncurses5-dev

sudo gedit /etc/profile   #在文件末尾添加交叉编译工具的路径，如： 
export PATH=$PATH:/home/pi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin
source /etc/profile   #更新环境变量	
echo $PATH   #检查环境变量是否成功修改
arm-linux-gnueabihf-gcc -v   #检查交叉编译工具是否成功配置


```

- 关于kernel-package

   安装选项选择： install the package maintainer's version



# config

```
pi@raspberrypi:~ $ uname -a > ~/Documents/uname   #获取当前树莓派系统的版本
pi@raspberrypi:~ $ sudo modprobe configs   #获取当前树莓派系统的配置文件
pi@raspberrypi:~ $ sudo scp /proc/config.gz meidongwen@HOST_IP: ~/Documents/raspberry/backup/ 
pi@raspberrypi:~ $ sudo scp ~/Documents/uname meidongwen@HOST_IP: ~/Documents/raspberry/backup/ 
cd ~/Documents/raspberrypi/backup/
zcat config.gz > .config
ls -a   #查看 .config文件，这是一个隐藏文件
cp .config .../software/os/linux/kernel/linux-rpi-4.19.y/

cd .../software/os/linux/kernel/linux-rpi-4.19.y/
find ./ -name "*bcm*defconfig*"
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig   #获取树莓派默认内核配置模板，注意不能使用sudo来make
ls -a   #查看 .config文件，这是一个隐藏文件
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig   #基于默认内核配置模板，修改配置
#如果对内核配置错误，或者想重新配置，需要执行这句指令进行内核配置文件清理：sudo make distclean 
#sudo apt-get install bison -y  #if not found
#sudo apt-get install flex  #if not found

```
（1）从树莓派获取的 .config文件（2）内核生成的 .config 文件，两者二选一，置于 linux-rpi-4.19.y/ 目录下



# kernel

```
cd .../software/os/linux/kernel/linux-rpi-4.19.y/
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage   #添加选项-j2，-j4，-j6等，表示使用多少个线程进行编译
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- dtbs 
sudo scripts/mkknlimg arch/arm/boot/zImage ./kernelx.img //将zImage格式转成树莓派需要的img格式
#sudo apt-get install libssl-dev #if no openssl/bio.h

sudo cp kernelx.img /media/meidongwen/boot/  #将img复制到SD卡
sudo cp arch/arm/boot/dts/*.dtb /media/meidongwen/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /media/meidongwen/boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /media/meidongwen/boot/overlays/

sudo vim media/meidongwen/boot/config.txt
#在最后一行输入：  kernel=kernelx.img

```



# modules

```
cd .../software/os/linux/kernel/linux-rpi-4.19.y/
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules 
mkdir modules
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=modules modules_install

sudo cp -r /media/meidongwen/system/lib ~/Documents/raspberrypi/backup/
sudo rm -r /media/meidongwen/system/lib
sudo cp -r modules/lib /media/meidongwen/system/

```


# firmware

```
cd /media/meidongwen/boot/
sudo cp *.elf *.bin ~/Documents/raspberrypi/backup/
sudo rm *.elf *.bin

cd .../software/os/linux/firmware/raspberrypi/firmware-master/boot
sudo cp bootcode.bin fixup.dat fixup_cd.dat start.elf /media/meidongwen/boot/
cd ../hardfp/opt/
sudo cp -r vc/ /media/meidongwen/system/opt/
```



# bootloader


```
#if your gcc is older than 6.0 and is not supported
vim .../software/os/linux/bootloader/u-boot/u-boot-master/arch/arm/config.mk
    #else
    #archprepare: checkgcc6
    endif
     
    #checkgcc6:
    #   @if test "$(call cc-name)" = "gcc" -a \
    #           "$(call cc-version)" -lt "0600"; then \
    #       echo '*** Your GCC is older than 6.0 and is not supported'; \
    #       false; \
    #   fi


cd .../software/os/linux/bootloader/u-boot/u-boot-master
source /etc/profile
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- rpi_3_defconfig V=1
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```



# 交叉调试环境 - GDB



# 在主机上开发

- 串口

  



# 空白SD卡

将SD卡插入主机

```
df -h   #查看文件系统信息和SD卡的挂载点,如 /dev/sdb1 和 /media/meidongwen/3FC9-C721
sudo umount /media/meidongwen/3FC9-C721   #在格式化SD卡之前需要卸载SD卡

sudo fdisk /dev/sdb
Command (m for help): p   #显示硬盘分割情形
Command (m for help): d   #删除硬盘分割区属性
Partition number (1-Ｎ): 1   #如果SD卡上有第一个分区,则一一删除
Command (m for help): d   
Partition number (1-Ｎ): 2   #删除2分区
Command (m for help): d   
No partition is defined yet!   #所有分区删除完毕
Command (m for help): p   #删除完毕后查看,此时已经没有分区

Command (m for help): n   #建立新的分区
Select (default p): p   #主分区
Partition number(1-4,default 1):１  #选分第１个分区
First sector (2048 - 15556607,default 2048):2048   #第一个区从2048开始
Last sector, +sectors or +size{K,M,G,T,P} (2048 - 15556607,default 15556607): +64M
Command (m for help): t   #修改分区类型
Partition number(1,2,default 2): 1   
Hex Code (type L to list all codes): L   #下面显示的分区选项对应的代号
Hex Code (type L to list all codes): b   #FAT32
Command (m for help): a   #激活第一分区的bootable标志
Partition number(1,2,default 2): 1  

Command (m for help): n   #建立新的分区
Select (default p): p   #主分区
Partition number(2-4,default 2):１  #选分第2个分区
First sector (133120 - 15556607,default 133120):133120   #第2个区从133120开始
Last sector, +sectors or +size{K,M,G,T,P} (133120 - 15556607,default 15556607): 15556607
Command (m for help): t   
Partition number(1,2,default 2): 2   
Hex Code (type L to list all codes): L
Hex Code (type L to list all codes): 83   #LINUX

Command (m for help): p   #查看结果
Command (m for help): w   #保存并结束

partprobe   #通知系统分区表的变化

sudo mkfs.vfat -f 32 -n boot /dev/sdb1   #格式化分区,boot是分区的名字
sudo mkfs.ext3 -L system /dev/sdb2   #格式化分区,system是分区的名字

sudo mount /dev/sdb1 /media/meidongwen/boot
sudo mount /dev/sdb2 /media/meidongwen/system

安装modules：
sudo cp kernelx.img /media/meidongwen/boot/  #将img复制到SD卡
sudo cp arch/arm/boot/dts/*.dtb /media/meidongwen/boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /media/meidongwen/boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /media/meidongwen/boot/overlays/

sudo cp -r modules/lib/ /media/meidongwen/system/

sudo umount mnt/fat32
sudo umount mnt/ext4

```



