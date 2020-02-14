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



# resource

disbutions:
RedHat:http://www.rehat.com
Fedora:http://fedoraproject.org
Mandriva:http://www.rmandriva.com
Novell SuSE:http://www.novell.com/linux
CentOS:http://www.centos.org

Ubuntu:http://www.ubuntu.com
Debian:http://www.debian.org
Slackware:http://www.slackware.com
Gentoo:http://www.gentoo.org



鳥哥的 Linux 私房菜 ：http://linux.vbird.org



# 启动与运行

## BIOS

BIOS（Basic Input Output System）是一套程序，是系统在开机的时候首先会去读取的一个小程序，控制着开机时各项硬件参数的取得，与开机设备的选择等等
BIOS原本写死到主板上面的一个内存芯片中， 使用ROM，没有通电时也能够将数据记录下来
现在的 BIOS 通常是写入类似闪存 （flash） 或EEPROM 中，既可以做修改，很多主板官网有BIOS 的更新程序。

- CMOS
  BIOS 程序载入 CMOS 的信息，并且借由 CMOS 内的设置值取得主机的各项硬件设置：
  - CPU 与周边设备的沟通频率
  - 开机设备的搜寻顺序
  - 硬盘的大小与类型
  -  系统时间
  - 各周边总线的是否启动 Plug and Play （PnP, 随插即用设备） 
  -  各周边设备的 I/O 位址
  - 与 CPU 沟通的 IRQ 岔断等等的信息


- POST(Power-on Self Test)

  在取得这些信息后，BIOS 还会进行开机自我测试 （Power-on Self Test, POST） [1]。 然后开始执行硬件侦测的初始化，并设置 PnP 设备，之后再定义出可开机的设备顺序，接下来就会开始进行开机设备的数据读取了。由于我们的系统软件大多放置到硬盘中嘛！所以 BIOS 会指定开机的设备好让我们可以读取磁盘中的操作系统核心文件。 但由于不同的操作系统他的文件系统格式不相同，因此我们必须要以一个开机管理程序来处理核心文件载入 （load） 的问题， 因此这个开机管理程序就被称为 Boot Loader 了。那这个 Boot Loader 程序安装在哪里呢？就在开机设备的第一个扇区（sector） 内，也就是我们一直谈到的 MBR （Master Boot Record, 主要开机记录区）。那你会不会觉得很奇怪啊？既然核心文件需要 loader 来读取，那每个操作系统的 loader 都不相同， 这样的话 BIOS 又是如何读取 MBR 内的 loader 呢？很有趣的问题吧！其实 BIOS 是通过硬件的 INT 13 中断功能来读取 MBR 的，也就是说，只要 BIOS 能够侦测的到你的磁盘（不论该磁盘是 SATA 还是 SAS 接口），那他就有办法通过 INT 13 这条信道来读取该磁盘的第一个扇区内的 MBR 软件啦！[2]这样 boot loader 也就能够被执行啰

  执行硬件检测的初始化，并配置PnP设备
  定义出可启动的设备顺序
  开始进行启动设备的数据读取

  依据设置启动第一个可启动的设备，读取该设备的MBR
  BIOS通过硬件的INT13 中断功能读取MBR(硬盘的主引导分区，即硬盘的第一个扇区)
  MBR中存放的是Bootloader，它被加载至内存



## Bootloader
composition：主程序+配置文件
locations：MBR, boot sector
MBR：硬盘主引导分区，第一个可启动的设备的first sector，一台啊电脑的MBR只有一个
boot sector：引导扇区，每个文件系统所在分区的1st sector都会保留,一台啊电脑的boot sector可以有多个
由上可知启动硬盘的排布：[MBR]+[bootloader+文件系统1]+[bootloader+文件系统2]
操作系统通常默认安装一份loader到他根目录所在的文件系统


多操作系统中的bootloader
一方面，()
Linux可以选择将他的bootloader安装到MBR或者是它的根目录所在文件系统
由于Windows的bootloader不具有转交功能，因此不能使用Windows的loader来加载Linux的bootloader


每种操作系统需要自己专属的bootloader来加载kernel文件
由于MBR容量很小，bootloader的程序代码执行与设置值加载分成两个阶段

提供菜单：用户可以选择不同的启动选项，多重引导
转交其他loader：将引导装在功能转交给其他loader负责
加载kernel文件：直接指向可启动的程序去来开始操作系统，kernel文件一般都是压缩文件，因此在使用前要将它解压

step1: bootloader主程序，这个主程序安装在MBR
step2: bootloader运行时，加载所有配置文件与相关环境参数文件，包括文件系统定义与主要配置文件menu.lst，一般来说这些配置文件都在/boot





- CONF - /boot/grub/menu.lst
  硬盘代号：从零开始计数,(hd0,0) 第一块硬盘的第一个分区

>default = 0 #与title对照，表示使用第一个title选项来启动，defaut表示如果在都秒时间结束前都没有按键，则默认使用此title选项
>timeout= 5 #启动时会读秒，等待按键，超时使用默认title选项，0不读秒使用默认选项，-1不读秒直接进入菜单
>hiddenmenu #启动时是否显示菜单



- 指定内核启动
  root:内核文件所在分区，例如

root (hd0,0) #表示第一块硬盘的第一个分区，即/dev/hda1

kernel:内核文件名 内核参数
root=LABEL=/1 #由于启动过程需要挂在根目录，这里表示根目录所在分区
initrd:RAM Disk的文件名

- 使用chain loader 转交控制权

- 如何重新读取MBR内的loader
  MBR读取的时整块硬盘的第一个sector，

- grub
  安装grub主程序

>grub #进入grub shell
>root (hd0,0) #
>find /grub/stage1 #注意boot是独立的，因此使用 "find /boot/grub/stage1" 是找不到的
>find /vmlinuz-2.6.18-92.e15 #查找内核
>setup (hd0) #安装grub主程序到MBR
>setup (hd0,0) #安装grub主程序到boot sector
>quit



>grub install 
默认安装地址/boot/grub
bootloader 有两个stage，而配置文件要放置到适当的地方
grub install 就是安装配置文件，包括文件系统定义文件和menu.lst
这些配置文件会在启动时被读取



- initrd
  显然kernel modules 位于磁盘根目录，所以在启动过程中必须先要挂载根目录，且为了避免影响磁盘文件系统，启动过程中根目录是read-only
  因为需要使用到磁盘，如SATA磁盘，所以在这之前先得加载SATA磁盘的驱动程序，而SATA磁盘的驱动程序又在/lib/modules，
  这时需要用到InitialRAM Disk，位于/boot/initrd
  该文件能够通过bootloader来加载到内存中，它也是压缩文件，需要解压缩，到内存中
  该文件会在内存中仿真成一个根目录，且此仿真在内存中的文件系统能够提供一个可执行的程序，
  通过该程序加载启动过程中所最需要的kernel模块，通常就是USB,RAID,LVM,SCSI等文件系统与磁盘接口的驱动程序
  载入完成后重新调用/sbin/init来开始后续正常的启动流程

需要使用到initrd的occation
根目录所在磁盘：SATA、USB、SCSI
根目录所在文件系统：LVM、RAID、"not traiditional fs"
有其他在内核加载时提供的模块

重制initrd
>mkinitrd -v initrd_$ (uname -r) $ (uname -r)

## Kernel

内核模块

内核重新检测一次硬件，而不移动会使用BIOD检测到的硬件信息
kernel开始检测硬件和加载去驱动程序
硬件驱动成功后，kernel主动调用init进程

内核：/boot/vmlinuz  #ls --format=single-column -F/boot
内核解压缩：RAMDisk：/boot/initrd
内核模块：/lib/modules/version/kernel
内核源码：/usr/src/linux 安装才会有，默认不安装
内核版本：/proc/version
系统内核功能：/proc/sys/kernal

内核模块(内核驱动)：
/lib/modules
显然kernel modules 位于磁盘根目录，所以在启动过程中必须先要挂载根目录，且为了避免影响磁盘文件系统，启动过程中根目录是read-only
因为需要使用到磁盘，如SATA磁盘，所以在这之前先得加载SATA磁盘的驱动程序，而SATA磁盘的驱动程序又在/lib/modules，
这时需要用到InitialRAM Disk，位于/boot/initrd
该文件能够通过bootloader来加载到内存中，它也是压缩文件，需要解压缩，到内存中
该文件会在内存中仿真成一个根目录，且此仿真在内存中的文件系统能够提供一个可执行的程序，
通过该程序加载启动过程中所最需要的kernel模块，通常就是USB,RAID,LVM,SCSI等文件系统与磁盘接口的驱动程序
载入完成后重新调用/sbin/init来开始后续正常的启动流程



arch:与硬件平台有关的选项，例如CPU等级
crypto:
drivers:
fs:
lib:
net:
sound:

>lsmod #查看已加载的内核模块

模块的依赖性，module1 is used by module2
那么如果要加载module2，那么得先加载module1

>modinfo mii  #查看模块
>insmod /lib/modules/$(uname -r) /kernel/fs/cifs/cifs.ko #手动加载模块，需要完整的文件名
>rmmod cifs #del

>modprobe cifs #手动加载cifs模块
>modprobe -r cifs #del
>modprobe -c  #show all modules
>modprobe -l  #show modules in /lib/modules/$(uname -r) /kernel

CONFIG - /etc/modprobe.conf
用于指定系统内硬件所使用的模块，这个文件通常系统时可以自行产生的，所以不必手动处理
如果系统检测到错误的驱动程序或者是想使用更新的驱动程序来对应相关的硬件配备时，就得要自行处理一下这个文件了



## systemd

- Init - No.1 Process
EXEC - /sbin/init
CONF - /etc/inittab
CONF - /etc/sysconfig/*

init依据inttab设置的处理流程：

1. 取得runlevel，即默认执行等级的相关等级

2. 使用etc/rc.d/rc.sysinit进行系统初始化

3. 例如runlebel=5，那么只执行 5:5:wait:/etc/rc.d/rc5，其他行略过
   规定启动的默认run level 是纯文本的3号或者是巨头图形界面的5号，可经由 id:5:initdefault 中的数字来决定
   若要关闭图形界面，则可以将5改成3

4. 设置[ctrl]+[alt]+[del] 这组的组合键的功能

   ca::ctrlaltdel:/sbin/shutdown -t3-r now 
   将这一行批注掉可取消[ctrl]+[alt]+[del] 重启系统的功能

5. 设置不断电系统的pf,pr两种机制

6. 启动mingetty的6个终端机 tty1~tty6
   如果不想启动6个终端机，可以将 6:2345:respawn:/sbin/mingetty tty6  关闭多个，但至少启动一个

7. 最终以 etc/X11/perfdm-nodaemon 启动图形界面



- 系统初始化
EXEC - /etc/rc.d/rc.sysinit  #called in etc/inittab: 'si::sysinit:/etc/rc.d/rc.sysinit'
CONF - /etc/sysconfig

1. 取得网络环境与主机类型
   读取网络配置文件/etc/sysconfig/network，取得主机名和默认网关等网络环境

2. 测试与挂载内存设备/proc  
   测试与挂载USB设备/sys/kernal
   USB设备不一定有，但是会检查，若有则会主动加载USB的驱动程序，并挂在USB的文件系统

3. 决定是否启动SELinux

4. 启动系统的随机数生成器
   随机数生成器帮助系统进行一些密码加密盐酸功能

5. 设置终端机console的字体

6. 设置欢迎界面 textbanner

7. 设置系统时间与失去设置，
   读入 /etc/sysconfig/clock

8. 接口设备的检测与PnP(Plug and Play)参数测试
   根据keernel在启动时检测的结果 (/proc/sys/kernel/modprobe)开始进行 ide/scsi/网络/音效 等接口设备的检测，
   以及利用已加载的kernel modules 进行PnP设备的参数测试

9. 用户自定义模块加载
   /etc/sysconfig/modules/*.modules

10. 加载kernel的相关设置
    /etc/sysctl.conf

11. 设置主机名与初始化电源模块ACPI

12. 初始化软件磁盘阵列
    /etc/mdadm.conf

13. 初始化LVM的文件系统功能

14. 以fsck检验磁盘文件系统
    filesystem check

15. 进行磁盘配额quota的转换(非必要)

16. 重新以可读可写的模式挂载系统磁盘

17. 启动quota功能
    因此我们不需要自定义quotaon的操作

18. 启动系统伪随机数生成器 pseudo-random

19. 清除启动过程中的临时文件

20. 将启动相关信息加载 /var/log/dmesg 文件中



- 启动服务
执行run-level的各个服务的启动(script)
依据/etc/inittab中run level的设置值，决定启动的服务选项
15:5:wait:/etc/c.d/rc 5
由/etc/rc.d/rc 5 可以取得/etc/rc5/d/ 这个目录来准备处理相关的脚本程序
找到/etc/rc5.d/K??*开头的文件，进行 /etc/rc5.d/K??*stop的操作
找到/etc/rc5.d/S??*开头的文件，进行 /etc/rc5.d/S??*start的操作



## 开机启动项


1. /etc/rc.local

    编辑文件 /etc/rc.local，在文件末尾（exit 0之前）加上你开机需要启动的程序或执行的命令即可，（执行的程序需要写绝对路径，添加到系统环境变量的除外），如：

    ```
    touch /var/lock/subsys/local
    /etc/init.d/mysqld start #mysql开机启动
    /etc/init.d/nginx start #nginx开机启动
    /etc/init.d/php-fpm start #php-fpm开机启动
    /etc/init.d/memcached start #memcache开机启动
    /usr/local/thttpd/sbin/thttpd  -C /usr/local/thttpd/etc/thttpd.conf
    exit 0
    ```

2. /etc/profile.d/

    自己写一个shell脚本（.sh文件），放到目录 /etc/profile.d/ 下，系统启动后就会自动执行该目录下的所有shell脚本。

3. /etc/init.d/

   将启动文件cp到目录 /etc/init.d/下，编辑该启动文件，在文件前面添加如下三行代码
    ```
    #!/bin/sh                #
    #chkconfig: 35 20 80     #分别代表运行级别，启动优先权，关闭优先权
    #description: balabala   #自己随便发挥
    ```
   进入bash，用chkconfig --add 命令将/etc/init.d中的启动脚本软连接到 /etc/rc.d/rc3.d （rc0.d ... rc6.d)  0-6个运行级别对应相应的目录。都是/etc/init.d 中启动脚本的软连

   ```
    chkconfig --add 脚本文件名 
   ```

   









## 加载终端机
mingetty
终端机模拟程序，启动login进程，等待用户登录

# 查看系统资源
​	free -m    内存使用情况(MB)
​	uname -a   系统的基本信息
​	uptime     系统启动时间
​	netstat    跟踪网络
​	dmesg      内核信息
​	vmstat     系统资源变化







### 网络信息

- Proto ： 网络的数据报协议
 - tcp
 - udp 
- Recv-Q ： 非由用户进程连接到此socket的复制的总字节数
- Send-Q ： 非由用户进程连接到此socket的复制的总字节数
- LocalAdress
- ForeignAddress
- State：连接状态
 - ESTABLISHED：建立
 - LISTEN：监听 
- RefCnt
- Flags：连接的标识
- Type ： socket访问类型
- State ： 
 - CONNECTED
- Path :















# 磁盘&文件

### 磁盘结构

- 柱面(Cylinder) :
- 扇区(Sector) ： 512bytes，

### 文件系统

- 第一扇区，将硬盘的第一个扇区独立出来
 - MBR(446bytes)：主引导分区
 - 分区表(64bytes)：指定该磁盘的各个分区的柱面范围，最多有4条分区记录，每条分区记录包含起始柱面序号和结束柱面序号，即每条记录占用 16bytes = 8bytes * 2
- 分区(prartition)：由若干连续的的柱面组成，分区与文件系统相互对应
 - primary：主分区，最多可以有3个，
 - extended：扩展分区，最多只能有1个，
 - logical：逻辑分区，由扩展分区分出来，因此在分区表中没有记录，分区序号1-4已经预留给了主分区和扩展分区，因此逻辑分区的分区序号从5开始，IDE硬盘的扩展分区最多可以制作59个逻辑分区(5号到63号)，SATA硬盘的扩展分区最多可以制作11个逻辑分区(5号到15号)
- 分区组成
 - boot sector：启动扇区，将分区的第一个扇区独立出来
 - Journaling file system：日志文件系统，万一数据记录的过程中发生问题，那么系统只要检查日志记录块就可以知道哪个文件发生问题，针对该问题来还一致性的检查即可，而不必针对整个文件去检查
 - super block：超级块，每个块组都有一个超级块，但是只第一个块组的超级块发挥作用，其余块组的超级块作备份，而超级块记录的又是整个分区的信息，所以写在这个位置
     - datablock/inode的大小、总量、使用量、剩余量
     - 文件系统信息：time-mount、last-time-write、last-time-fsck、validbit(=0未被挂载，=1已挂载)
 - block group：块组， 
     - super block：超级块，记录本块组所属分区的信息
 	 - file system description ： 文件系统描述，记录本块组的信息
         - 本块组，起始block号码、结尾block号码  
         - 本块组，每个区段的起始block号码、结尾block号码  
 	 - datablock ： 大小固定，一般为1KB或2KB或4KB，每个block都有编号，以方便inode记录，文件小于block则多余空间不使用，文件大于block则占用多个block
 	 - InodeTable：
	 	     - Inode ：一个文件对应一个Inode，每个Inode占用128bytes
		 	         - 文件属性记录区 ： 文件访问模式；文件的所有者和组；文件大小；创建时间；最近一次读取时间；最近修改时间；文件特性标志；文件的真正内容指向
             - 文件block号码 ： 一个blcok号码占用4bytes，假设block大小为1KB
                 - 12个直接记录，12个block号码，对应datablock数目=12
                 - 1个间接记录，对应block存储256个直接记录，对应datablock数目=256*12
                 - 1个双间接记录，对应block存储256个间接记录，对应datablock数目=256*256*12
                 - 1个三间接记录，对应block存储256个双间接记录，对应datablock数目=256*256*256*12
 	 - block bitmap ： 对照表， 
	 	     - 使用与未使用的block的号码  
 	 - Inode bitmap ： 对照表
	 	     - 使用与未使用的Inode的号码

### 文件系统与目录树的关系
linux中一切皆文件，接下来看看各式各样的“东西”是怎样以文件的形式存在的

- 文件读取：通过挂载的信息找到挂载点的inode号码，通常一个文件系统的最顶层的Inode号码由2开始，此时就能够得到根目录的Inode内容，并依据该Inode读取根目录的block内容，即文件名数据，再一层一层往下读到正确的文件名
系统读取文件时，先找到该文件的Inode，分析权限
- 文件写入，
 - 先确定用户对于要添加文件的目录是否具有w或x的权限
 - 根据Inode bitmap找到未使用的Inode号码，向Inode写入文件属性
 - 根据block bitmap找到未使用的block号码，向Inode写入文件block号码
 - 更新：Inode bitmap、block bitmap、super block
 - 在日志记录块中记录某个文件准备要写入的信息，
 - 写入文件的权限与数据
 - 更新meta data数据
 - 再日志记录块中完成文件的记录

### 文件系统挂载

- 挂载
 - 将某一个磁盘的某一个分区，挂载到某个目录上，这个目录称为挂载点，它将成为进入该分区(文件系统)的入口。
- 挂载点

- 单一文件系统不应该被重复挂载在不同的挂载点
- 单一目录不应该重复挂载多个文件系统
- 作为挂载点的目录理论上应该都是空目录



- 开机挂载:/etc/fstab
 - Device
 - Mount point
 - fil 






​	df -aT    显示所有分区，以及它们的挂载点
​	fdisk -l  显示所有的磁盘，以及它们的分区(文件系统)

###Intialize
建立自定义的分区结构

1. /boot, ext3, 100M, 强制为主分区
2. /, ext3, 10000M

##View

Disk /dev/hda:
/dev/hda1: No.1-IDE-Disk,No.1-pratition
/dev/hda2: No.1-IDE-Disk,No.2-pratition

Disk /dev/hdb
/dev/hdb1: No.2-IDE-Disk,No.1-pratition

Disk /dev/sda
/dev/sda1: No.1-SATA-Disk,No.1-pratition

3 disks in all = 2*IDE + 1*SATA
hda-2*pratitions, hdb-1*pratitions, sda-1*partitions,

Then choose one disk to config its partition

##Add
在初始配置中，我们将某一个硬盘挂载在了根目录上，
(1)swap partition

```
fdisk /dev/sda
- command=p #Check
- command=n, first sector=default, last sector= +256M
- command=t, HexCode=82 #82 is the ID of swap
- command=p #Check
- command=w #launch 

partprobe  #refresh
mkswap /dev/sda3
free 
swapon /dev/sda3
free
swapon -s
```

(2)user partition

```
fdisk /dev/sda  #Disk(non-number-followed), Partition(number-followed)
Input:p #partition table of this disk
Input:d #delete user partition, reconfig then
Input:p #check
Input:n #add user1 partition
Input:w #launch the configuration above, input q to abandon and quit
mke2fs -b -i -L -cj /dev/sda5  #filesystem for new partition
mount #check
mount /dev/sda5 /home/user1 
```

###设置开机挂载
/etc/fstab

###特殊设备loop挂载





###通配符

通配符用来代替命令中的文件名参数

- - 任意数量字符
- ? 单个字符
- [characters] 任意一个属于字符集的字符
- [!characters] 任意一个不属于字符集的字符
- [[:class:]] 任意一个属于指定字符类中的字符

字符类

- [:alnum:] 任意一个字母或数字
- [:alpha:] 任意一个字母
- [:digit:] 任意一个数字
- [:lower:] 任意一个小写字母
- [:upper:] 任意一个大写字母

示例

- - 所有文件
- g* ：以g开头，任一文件
- b*.txt ：以b开头，中间任意多个字符，.txt结尾，任一文件
- Data??? :Data开头，后面紧跟3个字符
- [abc]* :a或b或c开头，任一文件
- BACK.[0-9][0-9][0-9] BACK.开头，后面紧跟3个数字
- [[:upper:]]* : 任意一个大写字母开头，任一文件
- [![:alpha:]]* : 不以字母开头，任一文件
- *[[:lower:]123] : 以小写字母或1或2或3结尾，任一文件







### 目录结构

/etc、/bin、/dev、/lib、/sbin 这5个目录一定要和根目录放在同一个分区

.

- 可分享的(sharable) ： 可以分享给其他系统挂在使用的目录，
- 不可分享的 ： 
- 不变的(static)
- 可变的


FHS针对目录树架构仅定义出三层目录下面应该放什么数据
每层目录下所应该放置的目录也都有特定的规定

- / ： 根目录，与开机系统有关 ，建议根目录所在分区应该越小越好，且应用程序所安装的软件最好不要与根目录放在同一个分区内，
- /usr ： UNIX software resource ，UNIX操作系统软件资源，sharable，static，放置所有系统默认的软件，因此系统安装好时，这个目录会占用最多的存储
   - /usr/X11R6/ ：   

   - /usr/bin/ ： 

   - /usr/include/ ： 

   - /usr/lib/ ： 

   - /usr/local/ ： 

   - /usr/sbin/ ： 

   - /usr/share/ ：

      - /usr/share/doc 

   - /usr/src ：  

- /var ： variable，与系统运作过程有关，主要针对常态性变动的文件，系统运行后会渐渐占据越来越多的存储,
   - /var/cache/ ： 应用程序本身运行过程中会产生的一些暂存文件
   - /var/lib/ ： 
   - /var/lock/ ： 
   - /var/log/ ： 日志文件
     - /var/log/cron
     - /var/log/dmesg
     - /var/log/lastlog
     - /var/log/maillog
     - /var/log/message
     - /var/log/mail/
     - /var/log/messages
     - /var/log/secure
     - /var/log/wtmp
     - /var/log/faillog
     - /var/log/httpd/
     - /var/log/news
     - /var/log/sammba
   - /var/mail/ ： 
   - /var/run/ ： 
   - /var/spool/ ： 

.

 - /bin ：系统
    - 启动和运行所需的程序(二进制文件)，是在单用户维护模式下还能够被操作的命令，如cat、chmod、chown、date、mv、mkdir、cp、bash等
 - /boot  放置开机会使用的文件
 - /dev  任何设备与接口设备都是以文件的形式存在于这个目录中的
     - /dev/null/
     - /dev/zero/
     - /dev/tty/
     - /dev/lp*/
     - /dev/hd*/
     - /dev/sd*/
 - /etc ： 系统主要的配置文件，一般这个目录下的个文件属性可以允许一般用户查阅，但是只有root才能修改，建议不要防止可执行文件(binary)在这个目录中
     - /etc/init.d/
     - /etc/xinetd.d/
     - /etc/X11/
     - /etc/
     - /etc/
     - /etc/
     - /etc/
     - /etc/
 - /home：用户文件夹
 - /lib ： 开机时会用到的函数库，以及/bin或/sbin目录中的命令会调用的函数库，所谓的函数库指的是某些命令必须通过它们才能顺利地完成程序
 - lost+found
 - /media ： 
 - /mnt ： 
 - /opt ： 
 - /proc ： 
 - /root ： 
 - /sbin ： 命令，开机、修复、还原系统
 - /tmp ： 
 - 





### 文件类型

linux文件没有扩展名，但仍然会使用如 *.sh 的文件名，这只是为了便于识别



- 普通文件(-)

 - 纯文本文件(ASCII) ： 内容可以直接读取(cat命令)，是数字和字母等，需要由用户进行设置的文件都属于这种类型

 - 二进制文件(binary) ： 系统其实仅认识且仅执行二进制文件，linux中的可执行文件(scripts、文字批处理文件不算)就是这种格式，如cat命令就是一个binary file

 - 数据格式文件(data): 有些进程会读取data file，如用户登陆时会将登陆数据记录在/var/log/wtmp文件内，使用last命令读取，不能使用cat

- 目录(d)

- 链接文件(l)

  - 硬链接

    - 文件组成：1-内容数据，2-初始链接  
    - 索引节点：每一个数据对应一个索引节点，硬链接都是指向某一个索引节点  
    - 链接文件：可以理解为文件的文件名，因此创建文件时其实就已经默认创建了一个硬链接，而再创建硬链接就是创建额外的名称 
    - 大部分的文件操作命令，如果使用的是硬链接作为参数，那么操作的却是文件数据本身，但是rm命令是一个例外，当它删除一个链接文件时，只是删除该链接文件，而该链接文件对应的目标文件仍然存在
    - 硬链接的不足:
    - 不能跨越物理设备，即不能引用自身文件系统之外的文件，即不能引用其他磁盘分区的文件，
    - 不能引用目录 

  - 符号链接

    foo为符号链接文件
    每一个要使用foo-2.6文件的程序，先调用foo文件，再通过foo文件找到foo-2.6
    那么当foo-2.6做了修改，实际是产生了一个新的文件，并赋予了它新的文件名foo-2.7
    那么对于所有要使用foo-2.7的程序，我们不必对每个程序都修改
    由于它们都调用了foo，所以只要修改foo的指向，令他指向foo-2.7即可

    符号链接可以指向目录

- 字符设备文件(c) ： 通常位于/dev目录， 一些串行端口的接口设备，如键盘鼠标，这些设备的特点时一次性读取，不能截断输出   

- 块设备文件(b) ： 通常位于/dev目录， 存储数据，已提供系统随机访问的接口设备，如硬盘、软盘  

- 套接字(s)：位于/var/run 数据接口文件，通常用于网络上的数据连接，

- 管道(p):FIFO，用于解决多个进程同时访问一个文件所造成的错误问题


- 





- 日志文件

    - 日志文件的产生方式：
      1. 软件开发商自信股份定义写入，如www的apache
      2. 日志文件管理服务，如CentOS的syslogd

    - 记录内核信息的日志文件服务：klogd

    - syslogd

      /etc/syslog.conf #规定什么服务的什么等级信息，以及需要被记录在哪里
      服务名称.信息等级   信息记录的文件名或设备主机

      服务名称：
      auth
      cron
      daemon
      kern
      lpr
      mail
      news
      syslog
      uer
      uucp
      local0~local7

      信息等级，数值越大等级越高
      1.info
      2.notice
      3.warning
      4.err
      5.crit
      6.alert
      7.emerg
      特殊等级.debug
      特殊等级.none

      kern.* :表示只要是kern的信息，全部会被记录下来
      mail.info：表示只要是mail的信息，并且该信息的等级大于或等于info是时，就会被记录下来
      mail.=info：表示只要是mail的信息，并且该信息的等级等于info是时，就会被记录下来
      mail.!info：表示只要是mail的信息，并且该信息的等级不等于info是时，就会被记录下来

    - logrotate

      日志文件轮转，讲旧的日志文件更改名字，新的日志文件写入，旧日志保留一段时间后删除，



#基本命令
which -a ls
help cd
man cd








### 文件权限

 - 字符表示 ：9个字符，3个一组，分别顺序表示owner、group members、others 的权限
 - 数字表示 ：3个数字，如740，owner=111=rwx,group=100=r--,others=000=---
- 硬链接数目
- owner name
- group name
- 文件大小 单位bytes
- 修改日期
- 文件名
  	

注意默认权限的数值的含义有所不同，如023，表示000 010 010，那么
文件的默认权限 = 110 110 110 - 000 010 011 = 110 100 100 = rw- r-- r--
目录的默认权限 = 111 111 111 - 000 010 011 = 111 101 100 = rwx r-x r--


- 特殊权限

- 文件隐藏属性	
  	chattr +A file1  设置隐藏权限  
  +
  -
  =
  A
  S
  a
  c
  d
  i
  s
  u	





### 文件操作



|                            |                                                   |
| -------------------------- | ------------------------------------------------- |
| **目录操作**--------------------- | ------------------------------------------------------------- |
| pwd                        | 显示当前工作目录                                  |
| ls -lthF                   | 显示当前工作目录中的所有文件和目录,按修改时间排序 |
| cd /home/Documents         | 更改当前工作目录，使用绝对路径                    |
| cd ../Downloads            | 更改当前工作目录，使用相对路径                    |
| cd ~                       | 当前用户的主目录，/home/*                         |
| cd -                       | 回到上一个工作目录                                |
|                            |                                                   |
| mkdir dir1                 |                                                   |
| mkdir dir1 dir2 dir3       | 创建目录，同时多个                                |
| mkdir -p dir1/dir11/dir111 | 创建目录，若没有dir1和dir11，则一起创建           |
| mkdir -m 711 dir1          | 创建目录，强制权限rwx--x--x                       |
|                            |                                                   |
| cp -r file1 dir1           | file1复制到dir1目录                               |
| cp -r file1                | file1复制到当前目录                               |
| cp -r file1 file2 dir1     | file1、file2 复制到dir1目录中                     |
|  |  |
|**创建文件**--------------------|-------------------------------------------------------------|
|touch ~/Documents/dir2/file1|指定目录|
| ln file1 link1 | 创建硬链接文件link1，指向file1 |
| cp -l file1 link1 | 创建硬链接文件link1，指向file1 |
| ln -s file1 link | 创建符号链接 |
| cp -s file1 link1 | 创建符号链接 |
|      |      |
| **文件操作**-------------- | --------------------------------------------------------------- |
| cp -ai file1 dir1 | 复制，file1到dir1目录 |
| cp -ai file1 | 复制，file1到当前目录 |
| cp -ai file1 file2 dir1 | 复制，file1、file2 到dir1目录中 |
| cp link1 | 复制，链接文件link1对应的源文件，到当前目录 |
| cp -d link1 | 复制，链接文件link1，到当前目录 |
| mv file1 dir1 | 剪切 |
| rm -r file2 | 删除 |
| | |
| **查找文件**---------------------- | ----------------------------------------------------------------- |
| whereis file1 | #b, m, s, u,    查找 |
| locate file |#i, r,            查找  |
| find /home -mtime 0 | 查找 |
| | |
| | |
| **查看文件**--------------------- | ---------------------------------------------------------------- |
| file file1 | 查看说明 |
| cat file1 | 从第一行开始显示 |
| tac file1 | 从最后一行开始显示 |
| nl file1 | 显示时输出行号 |
| more file1 | 一页一页地显示 |
| head -6 file1 | 显示头6行 |
| tail -5 file1 | 只看结尾5行 |
| od file1 | 以二进制读取 |
| less file1 | 查看，文本文件的内容 |
|  |  |
| **文件权限**----------------------------- | ------------------------------------------------------------------- |
| umask | 默认权限显示 |
| umask 022 | 默认权限设置 |
|  |  |
|  |  |
|  |  |
| chgrp -R group2 file1 | 修改所属用户组 |
| chown -R user2 file1 | 修改所属用户 |
| groupmod -g 201 -n group2 $group_name | 修改group1为group2 |
| chmod -R 730 file1 | 修改权限,owner=111=rwx,group=011=-wx,others=000=--- |
| chmod -R u=rwx,g=rx,o=r file1 | 修改权限,owner=rwx,group=r-x,others=r-- |
| chmod -R a+w file1 | 修改权限,owner=rwx,group=rwx,others=rw- |
| chmod -R u-x,g-w,o+x file1 | 修改权限,owner=rw-,group=r-x,others=rwx |
|  |  |
|  |  |
|  |  |








# 进程

### 进程查看
|                            |                                            |
| :------------------------- | ------------------------------------------ |
| ps -l                      | 查看当前的bash，的子进程                   |
| jobs -l                    | 查看当前的bash，的后台进程                 |
| ps aux                     | 查看所有进程                               |
| ps -lA                     | 查看所有进程                               |
|                            |                                            |
| top -d 2                   | 持续检测进程状态，每两秒钟更新一次         |
| top -b -n 2 > /tmp/top.txt | 执行2批次，结果输出至文件                  |
| pstree -A                  | 查看目前系统所有进程树                     |
| pstree -Aup                | 查看目前系统所有进程树，同时显示PID和Users |


ps aux | egrep ' (cron|syslog) '  查看与cron, syslog 这两个服务有关的进程

### 进程修改
​    renice 10 18625 调整进程(PID=18625)的NI值

### 工作管理
​	tar -zpcf /tmp/etc.tar.gz /etc >/tmp/log.txt 2>&1 &  直接丢给后台执行
​	[ctrl+z]  当前bash的前台进程，进入后台，并等待
​	fg %2     当前bash的2号job（bg & Stopped），bg->fg，Stopped->Running
​	bg %3     当前bash的3号job（bg & Stopped），Stopped->Running
​	bg 3      PID = 3的进程   （bg & Stopped），Stopped->Running

### 进程管理
​    kill -l          查看可以使用哪些signal ，signal用数字1,2,3……表示，表示对执行什么样的处置
​	kill -signal %2  当前bash的2号job
​    kill -signal 2   PID = 2的进程
​	killall -signal  httpd 所有以httpd启动的进程

- 1 = SIGHUP : 重新启动
- 2 = SIGINT : 中断，相当于[Ctrl+c]
- 9 = SIGKILL : 强制中断，会有残余文件
- 15 = SIGTERM : 正常终止
- 17 = SIGSTOP : 暂停，相当于[Ctrl+z]



### 进程的触发

process = PID + 权限 + 属性 + code + data  
process作为一个完整的内存块，存在于内存中。  
但是在初始情况下，内存中是不存在进程的，某个进程被触发后，其对应的位于磁盘的program，被加载到内存中，再加上其它东西，成为一个进程  
打开的终端，即bash，即/bin/bash，是一个program，不同的用户打开这个程序时，取得的PID是不同的，这个PID是根据用户的UID/GID来的
bash作为一个进程，也带有PID，而PID又是由UID产生的，所以每个user登录后取得的shell的PID不同，这就实现了Linux的多用户环境
下面通过bash执行的任何命令都是这个bash衍生出来的，这些被执行的命令就是这个bash的子进程
当执行子进程时，原本的bash处于暂停状态，若要返回原bash，只有将子进程结束

子进程会继承父进程的环境变量，但不继承父进程的自定义变量

 - fork&exec  父进程产生子进程
  - fork：复制parent progress得到一个一摸一样的进程temporary progress，它和parent的唯一区别是PID不同，并且还会多一个PPID参数  
  - exec：temporary progress，以exec的方式加载要执行的程序，最终成为一个子进程


当前bash的子进程，又称为job  

- foreground :前台进程，由用户控制与执行，数量只有一个，  
- background :后台进程，自行运行，数量可以多个，不能使用[ctrl+c]终止，不能等待terminal/shell 输入    

在命令后面加 & 符号，直接丢给后台执行  
命令执行结果直接放在/tmp/log.txt以免打扰到前台工作

### 进程属性

- TTY ： 该进程在哪个终端机上运行，登陆者的终端机位置
 - ? ： 与终端机无关
 - pts/n ： 远程登录的动态终端接口
- USER / UID ： 该进程所属的用户
- START ： 被触发启动的时间点
- CMD ： 造成此程序的触发进程的命令是什么  
- PID ： 进程产生后就会获得一个PID，进程由不同的用户打开，PID不同；
- PPID ： parent process PID

-------------
- ADDR ： 指出进程在内存的哪个部分
 - - ： 这是一个running的进程，
- F ：  Flag
 - 4 ： 此进程的权限是root，
 - 1 ： 此进程仅可以fork，不能exec  
- PRI/NI ： Priority/Nice，进程优先级，数值越小优先级越高
- PRI ： 进程优先级，数值越小优先级越高,内核动态调整，用户无法干预
- NI ： 取值范围[-20,19], PRI(new)≈PRI(old)+NI, NI的数值仅仅是影响PRI，而并不能准确的改变进程的优先级，因为PRI的值是内核动态调整的，最终的PRI取值将经过系统的分析决定
- S / STAT ：State
 - R ： Runing
 - S ： Sleep且可以被唤醒
 - D ： Sleep且不可被唤醒(通常是在等待I/O)
 - T ： Stop后台暂停或traced(除错)状态
 - Z ： Zombie僵尸状态，该进程已经被终止但无法被删除至内存外
- WCHAN ： 表示目前进程是否在运行中
 - - ： 表示正在运行

-------------
- C / %CPU ： CPU使用率
- SZ / %MEM ： 物理内存占用率
- VSZ ： 虚拟内存使用量（KB）
- RSS ： 固定内存使用量（KB）
- TIME ： 使用掉的CPU的时间，实际花费CPU的时间，而不是系统时间

### 进程管理signal

- 1 = SIGHUP ： 重新启动
- 2 = SIGINT ： 中断，相当于[Ctrl+c]
- 9 = SIGKILL ： 强制中断，会有残余文件
- 15 = SIGTERM ： 正常终止
- 17 = SIGSTOP ： 暂停，相当于[Ctrl+z]















# daemon

- daemon & process  
- 启动与管理方式
 - 自行启动(stand-alone) ： 不必通过其他机制的管理，daemon启动并加载到内存后，作为一个常驻进程，一直在执行，一直占用内存与系统资源，持续提供服务，因此对于发生客户端请求时，响应速度快。example:httpd, vsftpd,
 - 统一管理 ： 这一类daemon我称为civil-daemon，它们都受一个super-daemon来管理，super-daemon是一个stand-alone-daemon，它负责管理各项服务，当有客户端请求某个civil-daemon时，super-daemon会先工作，它会触发该civil-daemon,example:talnet
- 启动脚本 /etc/init.d/*
- 初始化环境配置文件 /etc/sysconfig/*

### super-daemon   

- 对应进程 ：xinetd  
- 默认配置文件 ：/etc/xinetd.conf 
 - log_type
 - log_on_failure
 - log_on_success
 - cps
 - instance
 - per_source
 - v6only
 - groups
 - umask
- 配置文件 ： /etc/xinetd.d/*
 - Disable
 - id
 - Server
 - server_args
 - User
 - group
 - socket_type
 - protocol
 - Wait
 - instances
 - per_source
 - Cps
 - log_type
 - log_on_success
 - log_on_failure
 - env
 - Port
 - redirect
 - includedir
 - bind
 - interface
 - only_form
 - no_access
 - access_time
 - umask
- 查看daemon及其端口号 ： /etc/service 
- 启动脚本 ： /etc/init.d  
- 初始化环境配置文件 ： /etc/sysconfig.d  

### civil-daemon

 - 各服务各自的配置文件 ： /etc 
 - 各服务产生的数据库 ： var/lib 
 - 各服务的程序的PID记录处 ： var/run 
 - etc/init.d/crond restart



### crontab 
系统服务，默认启动，因为linux系统上面本身就有一些需要循环执行的操作

>vi /etc/cron.allow #限制使用crontab的账号，写入可以使用crontab的账号
>>one line one user: 
>>另外还有一个文件 /etc/cron.deny，写入禁止使用crontab的账号，功能相同但allow文件优先级较高，因此不用管它

>cd /var/spool/cron/user1 



>cat cat /var/spool/cron/user1 #某用户使用crontab命令创建工作调度后，该项工作就会被记录到上述文件夹中，文件夹以该用户的用户名来命名
>crontab执行的每一项工作都记录在该文件中

>cat /etc/crontab  #查看系统crontab
>>MAILTO=root #当该文件中的某个例行性工作命令发生错误时，会将错误信息或屏幕显示的信息传递给谁
>>

>vi /etc/crontab  #编辑系统crontab
>>MAILTO=/dev/null #取消输出项，直接送进垃圾箱
新建一个目录=/root/runcron，将每隔五分钟执行的"可执行文件"放入该目录中
>*/5 * * * root run-parts /root/runcron #系统每隔五分钟执行 /root/runcron 中的所有可执行文件
>*/5 * * * root /bin/mrtg /etc/mrtg/mrtg.org #系统每隔五分钟执行单个程序

>crontab -e #编辑cat /var/spool/cron文件
进入编辑器:
>>one line one process: min hour date month week command
>>0 3,6 * * * command #每一天的 3:00 & 6:00
>>20 8-12 * * * command #每一天的8:20 & 9:20 & 10:20 & 11:20 & 12:20 这5个时间点都执行
>>*/5 * * * * command #每隔5分钟执行一次

星号(*): 代表任何时候都能接受
逗号(,): 代表或
- 代表一段时间范围
、

>contab -u user1  #root用户使用该命令，为用户user1新建或删除crontab工作调度

>crontab -l #编辑工作内容



cat /etc/crontab


anacron:
anacron以一天、七天、一个月为期去检测系统未进行的crontab任务
anacron会读取时间记录文件timestamps，分析现在的时间与时间记录文件所记载的上次执行anacron的时间，比较后若发现有区别，那就是在某些时刻没有执行crontab
此时anacron就会执行未进行的crontab任务

anacron运行的时间点通常有两个
1.系统开机期间
2.写入crontab的调度中

>ll /etc/cron*/*ana*
>cat /etc/anacron







# 日志文件
/var/log/cron
/var/log/dmesg
/var/log/lastlog
/var/log/maillog
/var/log/message
/var/log/mail/
/var/log/messages
/var/log/secure
/var/log/wtmp
/var/log/faillog
/var/log/httpd/
/var/log/news
/var/log/sammba






日志文件的产生方式：
1.软件开发商自信股份定义写入，如www的apache
2.日志文件管理服务，如CentOS的syslogd

记录内核信息的日志文件服务：klogd

## syslogd

/etc/syslog.conf #规定什么服务的什么等级信息，以及需要被记录在哪里
服务名称.信息等级   信息记录的文件名或设备主机

服务名称：
auth
cron
daemon
kern
lpr
mail
news
syslog
uer
uucp
local0~local7

信息等级，数值越大等级越高
1.info
2.notice
3.warning
4.err
5.crit
6.alert
7.emerg
特殊等级.debug
特殊等级.none

kern.* :表示只要是kern的信息，全部会被记录下来
mail.info：表示只要是mail的信息，并且该信息的等级大于或等于info是时，就会被记录下来
mail.=info：表示只要是mail的信息，并且该信息的等级等于info是时，就会被记录下来
mail.!info：表示只要是mail的信息，并且该信息的等级不等于info是时，就会被记录下来

## logrotate
日志文件轮转，讲旧的日志文件更改名字，新的日志文件写入，旧日志保留一段时间后删除，



## 终端窗口 tty1~tty7
Linux默认提供了6个终端窗口和一个图形界面，也可能是别的配置，可以自行查看
之后的讲解中所使用的命令都是在上述的终端窗口输入，不要进入图形界面
不同的电脑打开终端的方式可能不同，一般又如下几种方式
ctrl+alt+F1~F6
Fn+ctrl+alt+F1~F6
它们之间相互独立，如果其中一个死机了，可以切换到其他窗口
























echo $PATH

## 压缩与打包
gzip -d file1
bzip2 -d file1
tar -jcpv -f /root/etc.tar.bz2 /etc #打包etc文件夹,并保存在root文件夹中
tar -jxv -f /etc.tar.bz2 /root/etc.tar.bz2 #解压







# 用户管理

## 查看用户信息

- 用户信息

  - cat /etc/passwd

    cat /etc/passwd | grep meidongwen

    grep 'meidongwen' /etc/passwd /etc/shadow /etc/group /etc/gshadow     #查看用户meidongwen的所有信息 

    每一行代表一个账号，每行由7个字段组成，以冒号“ : ”作为字段的分隔符

    1. 账号名称
    2. 密码：使用x作替代，这是为了保护密码，实际数据在 /etc/shadow文件中
    3. UID：账户ID
       - 0：root
       - 1~499：系统账号
       - 500~65535：自定义账户
    4. GID：账户所属用户组的ID，
    5. 使用者信息说明栏
    6. 主文件夹：用户的主文件夹，即该账户的活动空间，账户登陆后会自动进入该账户的主文件夹中
       - root 的主文件夹： /root 
       - 自定义账户的主文件夹： /home/“账户名称”
    7. Shell：
       - 当使用者登陆系统后就会取得一个 Shell 来与系统的核心沟通以进行使用者的操作任务。该字段指定该用户默认使用的 shell 
       - 该字段如果是/sbin/nologin，那么意味着该账号帐号无法取得shell 环境的登陆动作 

- 用户密码

  - cat /etc/shadow 

    cat /etc/shadow | grep meidongwen

    每一行代表一个账号，每行由7个字段组成，以冒号“ : ”作为字段的分隔符

    1. 账号名称
    2. 密码
    3. 最近更改密码的日期
    4. 密码更改的最小时间间隔
    5. 密码更改的最大时间间隔
    6. 密码更改警告时间
    7. 密码过期后的宽限时间
    8. 账号失效日期
    9. 保留

- 用户组

  - cat /etc/group

    每一行代表一个用户组，每行由4个字段组成，以冒号“ : ”作为字段的分隔符

    1. 用户组名称： 基本上需要与第三字段的 GID 对
       应。
    2. 用户组密码： 使用x作替代，这是为了保护密码，实际数据在 /etc/shadow文件中
    3. GID： 群组 ID ，对应 /etc/passwd 第四个字段使用的 GID 对应的群组名
    4. 该用户组支持的用户名称：某个帐号要加入此群组时，该帐号的名称会填入这个字段

  - groups  

    查看该账户支持的用户组，其中输出的第一个用户组是本帐户的effective_group  

    

- 用户组

  - cat /etc/gshadow 
  - cat /etc/gshadow 
  - 该文件记录GID与组名的对应
    1. 用户组名称
    2. 密码列，开头!表示无合法密码，即无用户组管理员
    3. 用户组管理员账号
    4. 用户组支持的账号名称

idname=meidongwen
uid=
initial_group=group1
group=
info="my account"
home=  #使用绝对路径
shell=/bin/bash
inactive=-1 #0密码过期后立刻失效，-1密码永不失效

passwd_lastchange
passwd_expires
passwd_inactive
account_inactive
passwd_change_min
passwd_change_max

group_name = group1
group_gid = 200



gpasswd 

## 用户管理

- 添加用户

  - useradd -m meidongwen   #
  - passwd meidongwen   #请root给予新账户密码  

- 用户切换

  - su root   #切换到root用户，输入root密码
  - su meidongwen   #切换到用户meidongwen，需要输入该用户的密码

- 用户组管理

  - newgrp users  

    假设users是本帐户支持的一个用户组，该命令将users切换为本帐户的effective_group  

  - groupadd -g 200 group1    #添加用户组，名称为group1，用户组的ID为=200

- 密码管理

  - sudo passwd root   #设定root密码
  - passwd   #修改本帐户密码  

- 删除账户

  - userdel -r meidongwen

  

sudo passwd --unlock root

sudo passwd --lock root  锁定root账户



useradd -u $uid -g $initial_group -G $group -M -m -c $info -d $home -r -s $shell -e -f $inactive $idname
         



# 网络

- ifconfig





# daemon

- daemon & process  
- 启动与管理方式
- 自行启动(stand-alone) : 不必通过其他机制的管理，daemon启动并加载到内存后，作为一个常驻进程，一直在执行，一直占用内存与系统资源，持续提供服务，因此对于发生客户端请求时，响应速度快。example:httpd, vsftpd,
- 统一管理 : 这一类daemon我称为civil-daemon，它们都受一个super-daemon来管理，super-daemon是一个stand-alone-daemon，它负责管理各项服务，当有客户端请求某个civil-daemon时，super-daemon会先工作，它会触发该civil-daemon,example:talnet
- 启动脚本 /etc/init.d/*
- 初始化环境配置文件 /etc/sysconfig/*

### super-daemon   

- 对应进程 ：xinetd  
- 默认配置文件 ：/etc/xinetd.conf 
- log_type
- log_on_failure
- log_on_success
- cps
- instance
- per_source
- v6only
- groups
- umask
- 配置文件 ： /etc/xinetd.d/*
- Disable
- id
- Server
- server_args
- User
- group
- socket_type
- protocol
- Wait
- instances
- per_source
- Cps
- log_type
- log_on_success
- log_on_failure
- env
- Port
- redirect
- includedir
- bind
- interface
- only_form
- no_access
- access_time
- umask
- 查看daemon及其端口号 : /etc/service 
- 启动脚本 : /etc/init.d  
- 初始化环境配置文件 : /etc/sysconfig.d  

### civil-daemon

- 各服务各自的配置文件 : /etc 
- 各服务产生的数据库 : var/lib 
- 各服务的程序的PID记录处 : var/run 
- etc/init.d/crond restart



### crontab 
系统服务，默认启动，因为linux系统上面本身就有一些需要循环执行的操作

> vi /etc/cron.allow #限制使用crontab的账号，写入可以使用crontab的账号
>
> > one line one user: 
> > 另外还有一个文件 /etc/cron.deny，写入禁止使用crontab的账号，功能相同但allow文件优先级较高，因此不用管它

> cd /var/spool/cron/user1 



> cat cat /var/spool/cron/user1 #某用户使用crontab命令创建工作调度后，该项工作就会被记录到上述文件夹中，文件夹以该用户的用户名来命名
> crontab执行的每一项工作都记录在该文件中

> cat /etc/crontab  #查看系统crontab
>
> > MAILTO=root #当该文件中的某个例行性工作命令发生错误时，会将错误信息或屏幕显示的信息传递给谁

> vi /etc/crontab  #编辑系统crontab
>
> > MAILTO=/dev/null #取消输出项，直接送进垃圾箱
> > 新建一个目录=/root/runcron，将每隔五分钟执行的"可执行文件"放入该目录中
> > */5 * * * root run-parts /root/runcron #系统每隔五分钟执行 /root/runcron 中的所有可执行文件
> > */5 * * * root /bin/mrtg /etc/mrtg/mrtg.org #系统每隔五分钟执行单个程序

> crontab -e #编辑cat /var/spool/cron文件
> 进入编辑器:
>
> > one line one process: min hour date month week command
> > 0 3,6 * * * command #每一天的 3:00 & 6:00
> > 20 8-12 * * * command #每一天的8:20 & 9:20 & 10:20 & 11:20 & 12:20 这5个时间点都执行
> > */5 * * * * command #每隔5分钟执行一次

星号(*): 代表任何时候都能接受
逗号(,): 代表或

- 代表一段时间范围
  、

> contab -u user1  #root用户使用该命令，为用户user1新建或删除crontab工作调度

> crontab -l #编辑工作内容



cat /etc/crontab

anacron:
anacron以一天、七天、一个月为期去检测系统未进行的crontab任务
anacron会读取时间记录文件timestamps，分析现在的时间与时间记录文件所记载的上次执行anacron的时间，比较后若发现有区别，那就是在某些时刻没有执行crontab
此时anacron就会执行未进行的crontab任务

anacron运行的时间点通常有两个
1.系统开机期间
2.写入crontab的调度中

> ll /etc/cron*/*ana*
> cat /etc/anacron







#日志文件
/var/log/cron
/var/log/dmesg
/var/log/lastlog
/var/log/maillog
/var/log/message
/var/log/mail/
/var/log/messages
/var/log/secure
/var/log/wtmp
/var/log/faillog
/var/log/httpd/
/var/log/news
/var/log/sammba





日志文件的产生方式：
1.软件开发商自信股份定义写入，如www的apache
2.日志文件管理服务，如CentOS的syslogd

记录内核信息的日志文件服务：klogd

##syslogd

/etc/syslog.conf #规定什么服务的什么等级信息，以及需要被记录在哪里
服务名称.信息等级   信息记录的文件名或设备主机

服务名称：
auth
cron
daemon
kern
lpr
mail
news
syslog
uer
uucp
local0~local7

信息等级，数值越大等级越高
1.info
2.notice
3.warning
4.err
5.crit
6.alert
7.emerg
特殊等级.debug
特殊等级.none

kern.* :表示只要是kern的信息，全部会被记录下来
mail.info：表示只要是mail的信息，并且该信息的等级大于或等于info是时，就会被记录下来
mail.=info：表示只要是mail的信息，并且该信息的等级等于info是时，就会被记录下来
mail.!info：表示只要是mail的信息，并且该信息的等级不等于info是时，就会被记录下来

##logrotate
日志文件轮转，讲旧的日志文件更改名字，新的日志文件写入，旧日志保留一段时间后删除，















# BASH Shell

目前 Linux （以 CentOS 7.x 为例） 有多少我们可以使用的 shells 呢？ 你可以检查一下 /etc/shells 这个文件，至少就有下面这几个可以用的 shells ，包括 /bin/sh 等于 /usr/bin/sh

/bin/sh （已经被 /bin/bash 所取代）
/bin/bash （就是 Linux 默认的 shell）
/bin/tcsh （整合 C Shell ，提供更多的功能）
/bin/csh （已经被 /bin/tcsh 所取代）

为什么我们系统上合法的 shell 要写入 /etc/shells 这个文件啊？ 这是因为系统某些服务在运行过程中，会去检查使用者能够使用的 shells ，而这些
shell 的查询就是借由 /etc/shells 这个文件

使用者什么时候可以取得 shell 来工作呢？ 我这个使用者默认会取得哪一个 shell ？当登陆的时候，系统就会给一个 shell 让使用者来工作了。 而这个登陆取得的 shell 就记录在 /etc/passwd 这个文件内：

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
```







- 命令与文件补全功能：[tab] 按键

  [Tab] 接在一串指令的第一个字的后面，则为命令补全；
  [Tab] 接在一串指令的第二个字以后时，则为“文件补齐”！
  若安装 bash-completion 软件，则在某些指令后面使用 [tab] 按键时，可以进行“选项/参数的补齐”功能！
  所以说，如果我想要知道我的环境当中所有以 c 为开头的指令呢？就按下“ c [tab] [tab] ”就好了，所以，有事没事，在 bash shell 下面，多按几次[tab] 是一个不错的习惯！

- 命令别名设置功能： （alias）
  假如我需要知道这个目录下面的所有文件 （包含隐藏文件） 及所有的文件属性，那么我就必须要下达“ ls -al ”这样的指令串，有没有更快的取代方式？使用命令别名！例如直接以 lm 这个自订的命令来取代上面的命令，也就是说， lm 会等于ls -al 这样的一个功能，嘿！那么要如何作呢？就使用 alias 即可！你可以在命令行输入 alias就可以知道目前的命令别名有哪些了！也可以直接下达命令来设置别名：

  - alias lm='ls -al'
  - 工作控制、前景背景控制： （job control, foreground, background）

  这部分我们在第十六章 Linux 程序控制中再提及！ 使用前、背景的控制可以让工作进行的更为顺利！至于工作控制（jobs）的用途则更广， 可以让我们随时将工作丢到背景中执行！而不怕不小心使用了 [Ctrl] + c 来停掉该程序！真是好样的！此外，也可以在单一登陆的环境
  中，达到多任务的目的呢！

- 程序化脚本： （shell scripts）
  在 DOS 年代还记得将一堆指令写在一起的所谓的“批处理文件”吧？在 Linux 下面的 shellscripts 则发挥更为强大的功能，可以将你平时管理系统常需要下达的连续指令写成一个文件， 该文件并且可以通过对谈互动式的方式来进行主机的侦测工作！也可以借由 shell 提供的环境变量及相关指令来进行设计，哇！整个设计下来几乎就是一个小型的程序语言了！该scripts 的功能真的是超乎鸟哥的想像之外！以前在 DOS 下面需要程序语言才能写的东西，在Linux 下面使用简单的 shell scripts 就可以帮你达成了！真的厉害！这部分我们在第十二章再来谈！

- 万用字符： （Wildcard）
  除了完整的字串之外， bash 还支持许多的万用字符来帮助使用者查询与指令下达。 举例来说，想要知道 /usr/bin 下面有多少以 X 为开头的文件吗？使用：“ ls -l /usr/bin/X* ”就能够知道啰～此外，还有其他可供利用的万用字符， 这些都能够加快使用者的操作

### bash

- 登陆信息 - /etc/issue
  \d 本地时间日期
  \l 显示第几个终端机接口 
  \m 显示硬件等级
  \n 主机网络名称
  \o domain name
  \r 操作系统版本
  \t 本地端时间
  \s 操作系统名称
  \v 操作系统版本
- 操作环境
  - 执行文件路径变量 ： $PATH
    例如命令ls，其实完整的形式应该是/bin/ls
    为什么可以在任何工作目录输入ls，都会对应到/bin/ls这个绝对路径？
    是因为环境变量PATH
    如执行ls时，系统先按照PATH的设置去每个PATH定义的目录下查询文件名为ls的可执行文件，如果在PATH定义的目录中含有多个ls文件，那么先查询到的先执行
    echo $PATH  查询
    执行文件(.sh)的查找路径，目录与目录使用冒号分隔，查找顺序就按照变量内目录的排列顺序
    PATH=/bin:/sbin:/usr/bin:/usr/sbin:user/local/bin:user/local/sbin:~/bin

不同身份的用户默认的PATH不同，默认能够随意执行的命令也不同
使用绝对路径执行
使用相对路径执行
使用查询PATH来执行

bash启动时读取环境配置文件，以此里规划操作环境
系统配置文件+用户配置文件

login shell：需要取得用户名和密码

- 系统配置文件 - /etc/profile  ，仅用于login shell，
  PATH ：
  MAIL
  USER
  HOSTNAME
  HISTSIZE
- /etc/profile.d/*.sh
  该文件被profile调用
- /etc/sysconfig/i18h
  该文件被profile调用，语系配置文件
- ~/.bash_pofile
  ~/.bash_login
  ~/.profile
  用户配置文件，bash的login shell按顺序读取上面三个文件中的一个，如果第一文件存在，后面两个就不会被读取

> - souce 配置文件名 #修改配置文件后启用生效

- ~/.bashrc
  non-login shell 仅读取该文件
  依据不同的UID规定umask的值
  依据不同的UID规定提示符PS1变量
  调用/etc/profile.d/*.sh的设置
- /etc/man.config
  ~/.bash_history
  历史命令
- ~/.bash_logout
  注销bash后，系统需要做的操作



.

```
type type 查看命令
which -a ls
help cd
mmkdir --help
man 1 cd    显示cd命令的手册的第一部分，其他部分如下
            1. 用户命令
            2. 内核系统调用的程接口
            3. C库函数程序接口
            4. 特殊文件，如设备节点和驱动程序
            5. 文件格式
            6. 游戏和娱乐，如屏保程序
            7. 其他
            8. 系统管理命令
```



### 语系

语系定义文件位置：/etc/stsconfig/i18h
locale -a 查看支持的所有语系
locale 查看语系设置



### I/O重定向

- 命令输入输出
- standard output，程序运行的结果，如ls命令的输出，输出时先将内容发送到文件"stdout"，该文件的内容并不保存在磁盘，而是链接到屏幕上
- standard error，标准状态和错误信息，输出时先将内容发送到文件"stderr"，该文件的内容并不保存在磁盘，而是链接到屏幕上
- standard input，默认情况下，标准输入连接到键盘

可见命令的输出也遵循UNIX一切皆文件的真理  

命令输出的正确结果存到file1，错误结果存到file2，用于存放结果的文件若不存在则会自动创建

```
(cmd) ls -l ./home > ls_output.txt  标准输出重定向
(cmd) 1>file1 2>file2 #文件内容先清空，再存放结果数据
(cmd) 1>>file1 2>>file2 #文件内容不清空，存放再文件末尾
(cmd) 2>/dev/null  #错误结果存放到垃圾桶黑洞
(cmd) 1>>file 2>&1  #正确/错误结果都存放到file文件中
```

< #用文本内容代替命令输入



### 变量

变量类型默认为字符串

```
variable = 1
parameter=$1 #使用参数
myname = meidongwen #创建变量myname，直接赋值
hisname = "second $myname" #双引号，变量可以含空格，符号具有原本功能
hername = 'second $myname' #单引号，变量可以含空格，符号仅为一般字符 
read -p "please key in your name:" -t 30 yourname #创建变量yourname，键盘输入赋值,等待30秒，超时退出
variable = $(cmd) #取得命令的执行结果

myname = ${myname-meidong} #变量存在则不做任何操作，变量不存在则创建并赋值meidong
myname = ${myname:-meidong} #变量存在且不是空字符串则不做任何操作，变量不存在或为空字符串则创建并赋值meidong
path = ${PATH}
path = ${path#/*bin} #删除从开头往后至第一个bin，包括bin
path = ${path##/*sbin:} #删除从开头往后至最后一个sbin:，包括sbin:
path = ${path%:*bin} #删除从末尾往前至第一个bin
path = ${path%%:*bin} #删除从末尾往前至最后最后一个bin
path = ${path%:/*} #删除最后一个
path = ${path/sbin/SBIN} #替换第一个sbin->SBIN
path = ${path//sbin/SBIN} #替换所有个sbin->SBIN

var = $ ((13*3)) #数值运算

echo $myname      #显示变量

export meidongwen #自定义变量变成环境变量
declare -aixr variable #声明变量类型，a数组, i整型, x环境变量, r只读
```



### History

~/.bash_history

> history 10 #列出最近的10命令



### [Tab]

c[Tab],自动补全
c[Tab][Tab]，显示所有以开头的命令



### 转义

\[Enter] 换行继续输入命令

### 变量

set #显示所有变量





*** $meidongwen  使用
unset myname 取消



### 命令别名

```
alias              查看所有别名
type lm            设置前线查看这个别名是否已经被使用
alias lm='cd ../home; ls-al; mkdir dir1; cd -'   设置
unalias lm         取消
```

###用户限制
ulimit 







# Shell Scripts



## 介绍

- 第一行宣告使用的 shell 名称，

  ```
  #!/bin/bash 
  ```

  上例宣告这个文件是一个Bash程序文件，需要由/bin目录下的bash程序来解释执行

  当这个程序被执行时，他就能够载入 bash 的相关环境配置文件 （一般来说就是 non-login shell的 ~/.bashrc）， 并且执行 bash 来使我们下面的指令能够执行（在很多状况中，如果没有设置好这一行， 那么该程序很可能会无法执行，因为系统可能无法判断该程序需要使用什么 shell 来执行）

- 宣告主要环境变量

  ```
  PATH = bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
  LANG = 
  export PATH
  ```

  可让我们这支程序在进行时，可以直接下达一些外部指令，而不必写绝对路径，比较方便

- 注解
  除了第一行的“ #! ”是用来宣告 shell 的之外，其他的“#” 都是“注解”用途，任何加在 # 后面的数据将全部被视为注解文字而被忽略！

- 输入与输出

  ```
  read -p "Please input your first name: " firstname # 提示使用者输入
  read -p "Please input your last name: " lastname # 提示使用者输入
  echo -e "\nYour full name is: ${firstname} ${lastname}" # 结果由屏幕输出
  ```

- 循环语句

  ```
  while [ "${yn}" != "yes" -a "${yn}" != "YES" ]
  do
  read -p "Please input yes/YES to stop this program:" yn
  done
  echo "OK! you input the correct answer."
  
  
  for animal in dog cat elephant
  do
  echo "There are ${animal}s.... "
  done
  ```

  

- 执行成果告知 （定义回传值） 

  /是否记得我们在第十章里面要讨论一个指令的执行成功与否，可以使用 $? 这个变量来观察～ 那么我们也可以利用 exit 这个指令来让程序中断，并且回传一个数值给系统。 在我们这个例子当中，鸟哥使用 exit 0 ，这代表离开 script并且回传一个 0 给系统， 所以我执行完这个 script 后，若接着下达 echo $? 则可得到 0的值喔！ 更聪明的读者应该也知道了，呵呵！利用这个 exit n （n 是数字） 的功能，我们还可以自订错误讯息， 让这支程序变得更加的 smart 呢！



### 判断式

条件语句

```
read -p "Please input （Y/N）: " yn
if [ "${yn}" == "Y" ] || [ "${yn}" == "y" ]; then
echo "OK, continue"
elif [ "${yn}" == "N" ] || [ "${yn}" == "n" ]; then
echo "Oh, interrupt!"
else
echo "I don't know what your choice is"
fi


case ${1} in
"Y"）
echo "OK, continue"
;;
"N"）
echo "Oh, interrupt!"
;;
""）
echo "You MUST input parameters, ex&gt; {${0} someword}"
;;
esac
```

注意：

在中括号 [] 内的每个元件都需要有空白键来分隔；如：[□"$HOME"□==□"$MAIL"□]
在中括号内的变量，最好都以双引号括号起来；
在中括号内的常数，最好都以单或双引号括号起





## 执行

- 设置脚本的权限
  - chmod u+x Script_Name   #只有自己可以执行
  - chmod ug+x Script_Name   #只有自己和同组人员可以执行
  - chmod +x Script_Name   #所有人都可以执行



- 执行

1. 直接指令下达： 假设叫脚本文件是 /home/dmtsai/ shell.sh， 文件必须要具备可读与可执行 （rx） 的权限，然后：
   - 绝对路径：输入命令： /home/dmtsai/shell.sh 
   - 相对路径：工作目录在 /home/dmtsai/ ，输入命令： ./shell.sh
   - 变量“PATH”功能：将 shell.sh 放在 PATH 指定的目录内，例如： ~/bin/
2. 以 bash 程序来执行：通过“ bash shell.sh ”或“ sh shell.sh ”来执行
3. CentOS 默认使用者主文件夹下的 ~/bin 目录会被设置到 ${PATH} 内，所以你也可以将 shell.sh 创建在 /home/dmtsai/bin/ 下面 （ ~/bin 目录需要自行设置） 。此时，若 shell.sh 在 ~/bin 内且具有 rx 的权限，那就直接输入 shell.sh 即可执行该脚本程序！
4. 那为何“ sh shell.sh ”也可以执行呢？这是因为 /bin/sh 其实就是 /bin/bash （链接文件），使用 sh shell.sh 亦即告诉系统，我想要直接以 bash 的功能来执行 shell.sh 这个文件内的相关指令的意思，所以此时你的 shell.sh 只要有 r 的权限即可被执行，而我们也可以利用 sh 的参数，如 -n 及 -x 来检查与追踪 shell.sh 的语法是否正确

read 功能的问题是你得要手动由键盘输入一些判断式。如果通过
指令后面接参数， 那么一个指令就能够处理完毕而不需要手动再次输入一些变量行为！这样
下达指令会比较简单方便



## 变量

- 变量的使用

  在变量前加$

- 变量相关的命令

  - env  #显示所有环境变量

- 系统变量（默认变量）

  - $#：保存程序命令行参数的数目，后接的参数“个数”

  - $?：保存前一个命令的返回值

  - 

  - $*：以

  - $@：代表“ "$1" "$2" "$3" "$4" ”之意，每个变量是独立的（用双引号括起来）；

  - $0：脚本文件名

  - $1：执行script时，后接的第1个参数

  - $2：执行script时，后接的第2个参数 ……

    

- 环境变量

  环境变量就是全局变量

  - HISTSIZE：历史记录数
  - HOME：当前用户主目录的完全路径名
  - HOSTTYPE
  - LANG：
  - MAIL：当前用户的邮件存放目录
  - MACHTYPE
  - OSTYPE
  - PATH：决定了Shell将到哪些目录中寻找命令或程序
  - PS1：主提示符
  - PS2：辅助提示符
  - PWD：
  - RANDOM
  - SHELL：Shell路径名

- 用户变量

  自定义变量是局部变量

  任何不包含空格字符的字符串来作为变量名

 

## 正则表达式

- Regular Expression, RE, 或称为常规表达式
- 用在字串的处理上面的一项“表示式”
- 通过一些特殊字符的排列，用以“搜寻/取代/删除”一列或多列文字字串， 简单的说，正则表达式就是。正则表达式并不是一个工具程序， 而是一个字串处理的标准依据，如果您想要以正则表达式的方式处理字串，就得要使用支持正则表达式的工具程序才行， 这类的工具程序很多，例如 vi, sed, awk 等等。
  正则表达式对于系统管理员来说实在是很重要！因为系统会产生很多的讯息，这些讯息有的重要有的仅是告知， 此时，管理员可以通过正则表达式的功能来将重要讯息撷取出来，并产生便于查阅的报表来简化管理流程。此外， 很多的套装软件也都支持正则表达式的分析，例如邮件服务器的过滤机制（过滤垃圾信件）就是很重要的一个例子。 所以，您最好要了解正则表达式的相关技能，在未来管理主机时，才能够更精简处理您的日常事务！

```
t[ae]st   #tast或test
[^g]00     #前面没有g的oo
[^a-z]00 '  #前面没有a,b,c……z的oo
[0-9] #数字
[a-zA-Z0-9]  #数字或英文
^the   #出现再行首的the
^[a-z]   #开头是小写字符
^[^a-zA-Z]   #开头是非英文字符
\.$   #结尾是小数点
^$  #空白行
```

命令执行：
type -a ls
1.以相对/绝对路径执行命令
2.由alias找到该命令来执行
3.由bash内置的(builtin)命令来执行
4.通过$PATH这个变量的顺序找到第一个命令来执行

命令执行：

> sh bash-script.sh #打开一个新的bash，在新的bash中执行该脚本
> source bash-script.sh #直接在本bash中执行该脚本

默认变量
sh bash-script.sh a b c d  #该脚本执行时，$0=bash-script.sh, $#=4, $@=a b c d, $1=a, $2=b, $3=c, $4=d



## pipe

就如同前面所说的， bash 命令执行的时候有输出的数据会出现！ 那么如果这群数据必需要经过几道手续之后才能得到我们所想要的格式，应该如何来设置？ 这就牵涉到管线命令的问题了 （pipe） ，管线命令使用的是“ | ”这个界定符号！ 另外，管线命令与“连续下达命令”是不一样的呦！ 这点下面我们会再说明。

下面我们先举一个例子来说明一下简单的管线命令。假设我们想要知道 /etc/ 下面有多少文件，那么可以利用 ls /etc 来查阅，不过， 因为 /etc 下面的文件太多，导致一口气就将屏幕塞满了～不知道前面输出的内容是啥？此时，我们可以通过 less 指令的协助，利用：

如此一来，使用 ls 指令输出后的内容，就能够被 less 读取，并且利用 less 的功能，我们就能够前后翻动相关的信息了！很方便是吧？我们就来了解一下这个管线命令“ | ”的用途吧！ 其实这个管线命令“ | ”仅能处理经由前面一个指令传来的正确信息，也就是 standard output 的
信息，对于 stdandard error 并没有直接处理的能力。那么整体的管线命令可以使用下图表示：示意图
，这样的指令才可以是为“管线命令”，

- 注意

  - 管线命令仅会处理 standard output，对于 standard error output 会予以忽略
  - 管线命令必须要能够接受来自前一个指令的数据成为 standard input 继续处理才行。每个管线后面接的第一个数据必定是“指令”，而且这个指令必须要能够接受 standard input 的数据
  - 如 less, more, head, tail 等都是可以接受 standard input 的管线命令，而 ls, cp, mv 等不是管线命令，因为不接受来自 stdin 的数据。 、、

- cut

  - 将一段讯息的某一段给他“切”出来，处理的讯息是以“行”为单位
  - 

- grep

  - 分析一行讯息， 若当中有我们所需要的信息，就将该行拿出来

  - grep [-acinv] [--color=auto] TARGET FILENAME

  - 选项与参数：

    - TARGET ：要搜寻的关键字
    - FILENAME : 搜寻的目标文件
    - -a ：将 binary 文件以 text 文件的方式搜寻数据
    - -c ：计算找到 '搜寻字串' 的次数
    - -i ：忽略大小写的不同，所以大小写视为相同
    - -n ：顺便输出行号
    - -v ：反向选择，亦即显示出没有 '搜寻字串' 内容的那一行！
    - --color=auto ：可以将找到的关键字部分加上颜色的显示喔！

  - 

    

  

- 

cut -d'' -f 
cut -c
grep [acinv] [--color=auto] 'MANPATH' /etc/man.config #将该文件中含有MANPATH的行取出，a,c,i忽略大小写,n输出行号,v反向选择,--color=auto

### Debug









## Shell Script

​	#!/bin/bash  #声明使用bash shell，本脚本执行时将加载bash的相关环境配置文件（~/.bashrc）
​	export PATH
​	echo -e "HELLO World! \a \n"
​	
​	

​	
​	# 文件
​	touch "$myname" #创建文件名
​	test
​	
​	
​	
​	sort
​	
​	wc
​	
​	uniq
​	
​	tee
​	
​	tr
​	col
​	join
​	paste
​	expand
​	
​	split -b 
​	split -l
​	
​	xargs

​		
​	#命令
​	[cmd1] | [cmd2] #执行cmd1后，将cmd1的结果输出给cmd2并执行cmd2，cmd1是可以输出文件的命令，cmd2是可以输入文件的命令，如less、more、head、tail
​	

```
exit 0 #退出并返回0
```

```

```







# 交叉开发环境

## 介绍

因为 1.嵌入式系统的专用性；2嵌入式系统的硬件特殊性；所以，不能安装发行版的Linux系统，需要专门为特定的目标板定制Linux操作系统

这必然需要相应的开发环境，即交叉开发模式：在主机环境下开发，在目标板上运行

- Target（目标板）
- Host（开发主机，宿主）
  - 安装开发工具，编辑、编译目标版的Linux引导程序、内核、文件系统，然后在目标板上运行
  - 工作站：给开发者提供开发工具
  - 服务器：配置启动各种网络服务

对于交叉开发方式，一方面开发者可以在自己熟悉的主机环境下进行程序开发，另一方面又可以真实地在目标板系统上运行和调试程序，这种开发方式将贯穿Linux系统开发的全过程

- Target与Host的连接
  - 串口
  - 以太网
  - USB
  - JTAG
- 文件传输
  - 串口
  - 以太网
  - USB
  - JTAG
  - 移动存储设备



## 使用现成的交叉工具链

- 树莓派

  - `git clone https://github.com/raspberrypi/tools.git`

  - `sudo gedit ~/.bashrc`

  - 在该文件最后加入交叉工具链所在目录，如：`export PATH=$PATH:/home/meidongwen/Downloads/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin`
  - 更新当前控制台所包含的环境变量：`source ~/.bashrc`

  



## 在主机上构建交叉开发环境

1. 下载工具源码包和补丁，准备内核头文件，创建工作目录等
2. 编译binutils
3. 编译辅助编译器
4. 编译glibc库
5. 编译生成完整的GCC编译器。重新配置GCC功能，使其支持C、C++
6. gdb



### Binutils

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



### GCC（GNU Compiler Collection）

- 编译过程
  - 预处理（Pre-Process）
  - 编译（Compiling）
  - 汇编（Assembling）
  - 链接（Linking）



### 使用gcc工具进行编译

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







## make工具与makefile文件

make是一个解释makefile文件中指令的命令工具，通过makefile文件来描述源文件之间的相互关系并自动维护编译工作，告知系统以何种方式编译和链接程序。

- 查找当前目录下的Makefile文件
- 初始化文件中的变量
- 分析Makefile中的所有规则
- 为所有的目标文件创建依赖关系
- 根据依赖关系，决定哪些目标文件要重新生成
- 执行生成指令



makefile的作用就是让编译器知道编译一个文件需要依赖哪些其他文件，同时当那些依赖文件有改动时，编译器会自动发现最终生成文件已经过时而重新编译相应的模块。



- 规则
  - 说明文件之间的依赖关系
  - 告诉make工具如何生成目标文件的命令
- 文件结构
  - 注释：和Shell Script一样，使用“#”符号
  - 目标文件
    - 目标文件（.o）
    - 最终可执行的程序
    - 要执行的动作如“clean”
  - 依赖文件列表：目标文件所
  - 命令列表：make程序执行的动作，



变量

- makefile文件中的变量主要用于以下几个方面
  - 代表以恶文件列表
  - 代表编译命令选项
- 内部变量
  - $@：
  - $<：
  - $^：
  - $?：



## GLIBC（Linux编程库）



### 系统调用





**添加新的系统调用：**

- 修改kernel/sys.c，增加服务例程代码
- 编辑文件下面两个文件，从已有的内核程序中增加到新的函数的连接
  - /usr/src/linux/include/asm-i386/unistd.h
  - /usr/src/linux/arch/i386/kenel/syscall_table.S
- 重新编译内核
- 测试



### Linux线程库

# 开发

scp local_file remote_username@remote_ip:remote_folder



## 交叉调试环境 - GDB





# Bootloader



# Linux内核

## 准备主机环境

- `sudo apt-get update`
- `sudo apt-get install kernel-package`
  - 选择 `install the package maintainer's version`
- `sudo apt-get install libncurses5-dev`
  - 

## 配置

