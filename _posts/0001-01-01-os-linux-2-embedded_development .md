---
layout: post
os: true
comments: true
title:  "linux"
excerpt: "..."
tag:
- os
- linux


---



# Get Source

```
wget ftp.gnu.org/gnu/binutils/binutils-2.34.tar.gz
wget ftp.gnu.org/gnu/glibc/glibc-2.31.tar.gz
wget ftp.gnu.org/gnu/gcc/gcc-9.2.0/gcc-9.2.0.tar.gz

wget ftp.gnu.org/gnu/gmp/gmp-6.2.0.tar.bz2
wget ftp.gnu.org/gnu/mpc/mpc-1.1.0.tar.gz
wget ftp.gnu.org/gnu/mpfr/mpfr-4.0.2.tar.gz

wget mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.5.3.tar.xz



tar -xzvf *.tar.gz





GMP:
ftp://gcc.gnu.org/pub/gcc/infrastructure/gmp-4.3.2.tar.bz2
tar -xjvf gmp-4.3.2.tar.bz2


tar -xvf *.tar
tar -xzvf *.tar.gz *.tar.z *.tar.tgz

```

构建交叉编译环境

1. 下载工具源码包和补丁，准备内核头文件，创建工作目录等
2. 编译binutils
3. 编译辅助编译器
4. 编译glibc库
5. 编译生成完整的GCC编译器。重新配置GCC功能，使其支持C、C++
6. gdb



- Binutils

GNU Binutils工具链，用于构造和使用二进制

Binutils工具链必不可少，并且与GCC紧密相连

下载地址：ftp://ftp.gnu.org/gnu/binutils/

- 组成部件
  - as：GNU的汇编器
  - ld：GNU的链接器
  - add2line：降低至转换成文件名或行号对，以便调试程序
  - ar：从文件中创建、修改、扩展文件
  - gasp：汇编宏处理器
  - objcopy：使用GNU BSD库，可以把目标文件的内容从一种文件格式复制到另一种格式的目标文件中
  - readelf
  - ranlib
  - size
  - strings
  - strip：放弃所有符号连接
  - c++fit：
  - gprof



- GCC（GNU Compiler Collection）

  - 编译过程
    - 预处理（Pre-Process）
    - 编译（Compiling）
    - 汇编（Assembling）
    - 链接（Linking）



- 使用gcc工具进行编译

  - gcc编译命令
  - 编译命令的选项
    - 常用编译选项
      - -c
      - -S
      - -e
      - -v
      - -x
      - -L
      - -static/
      - -o
    - 出错检擦和警告
      - -pedantic
      - -w：禁止输出警告消息
      - -Werror：将所有警告转换成错误
      - -Wall：显示所有警告消息
    - 代码优化
      - -O
      - -O2
    - 调试分析
      - -g
      - -pg
      - 



- GLIBC（Linux编程库）



- 系统调用





**添加新的系统调用：**

- 修改kernel/sys.c，增加服务例程代码
- 编辑文件下面两个文件，从已有的内核程序中增加到新的函数的连接
  - /usr/src/linux/include/asm-i386/unistd.h
  - /usr/src/linux/arch/i386/kenel/syscall_table.S
- 重新编译内核
- 测试



- Linux线程库









```
sudo gedit /etc/profile   #在文件末尾添加交叉编译工具的路径，如： 
PATH=$PATH:/home/pi/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin
source /etc/profile   #更新环境变量	
echo $PATH   #检查环境变量是否成功修改
arm-linux-gnueabihf-gcc -v   #检查交叉编译工具是否成功配置

sudo apt-get update
sudo apt-get install kernel-package
sudo apt-get install libncurses5-dev
```

- 关于kernel-package

   安装选项选择： install the package maintainer's version







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



