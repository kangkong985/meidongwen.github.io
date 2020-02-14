---
layout: post
os: true
comments: true
title:  "fatfs"
excerpt: "..."
tag:
- os
- fatfs


---



# reference

Home：http://elm-chan.org/fsw/ff/00index_e.html



# project

## src

http://elm-chan.org/fsw/ff/archives.html



## files



## modify



## Keil MDK





# fatfs FatFs读写SD卡出现FR_NO_FILESYSTEM解决方法

起因
去年做了个GPS路径记录器, 将路径息记录于TF卡上, 上了FatFs系统. 刚开始那会虽然偶尔罢工, 但好歹能工作. 后来没时间也没心情了, 就搁在一边, 没再管. 前几天又找出来, 想着弄稳定了, 过年回家的时候玩一下, 结果发现居然不工作了. 想当初调试的时候折腾的死去活来, 走了无数弯路, 现在说不工作就不工作, 心里各种不服, 加了一堆调试信息输出后, 发现SD卡初始化是正常的, 但在读写的时候返回FR_NO_FILESYSTEM. 于是找出仿真器, 跟踪了一下, 另外祭出神器WinHex, 终于找出问题所在.
环境: 
MPU: MSP430F5418.
SD Card: SanDisk 1G T-Flash.
Interface: 硬件SPI.
FatFs: 0.09
问题现像: 在电脑上读写正常的SD卡, 使用FatFs读写不正常, 返回的错误类型为FR_NO_FILESYSTEM. 
问题原因: 如果你移植时SD卡低层读写函数正确的话, 就不是你的问题. 具体下面会分析.
解决方法: 1. 使用FatFs自带的f_mkfs()函数格式化SD卡, 但注意底层函数的正确(disk_ioctl()), 要不会出现莫名其妙的问题, 比如将你1G的卡格式化为30M.
                  2. 使用FatFs V0.10.


基本知识.
1.从MBR说起.
MBR(Master Boot Record, 主引导记录)位于SD卡的第0扇区(物理), 共512个字节. 其中前446个字节为引导代码, 接下来64个字节为分区表, 再接下来两个字节为签名, 固定为 0×55, 0xAA.
下图是我的SD卡的MBR:

64个字节的分区表分为4组, 每16字节为一组, 每一组描述一个分区. 这16个字节数据的意义在此不做详述, 只须了解偏移为0H, 4H, 8H处的数据即可.
偏移 0x00H 起的一个字节表示分区状态, 0×00表示非活动状态, 0×80表示活动状态. 其它数值无意义.
偏移 0x04H 起的一个字节表示文件系统类型, 常见的有: 01, FAT32; 06, FAT16; 07, NTFS. 0为法值.
偏移 0x08H 起的4个字节数据表示相对扇区数, 即从磁盘开始到该分区开始的偏移量, 以扇区计算. 此数值指向的物理扇区也就是逻辑扇区0. 比如此数值为63, 则第63个物理扇区为逻辑扇区0.
2.DBR(Dos Boot Record, Dos引导记录).
DBR位于逻辑扇区0, 属于FAT文件系统的一部分, 记录着文件系统的相关信息.
类似MBR, DBR也由几部分组成, 如下:
    偏移        字段长度        名称
    0×00        3字节           跳转指令
    0×03        8字节           厂商标志和OS版本号
    0x0B        53字节         BPB(BIOS Parameter Bblock)
    0×40        26字节          扩展BPB
    0x5A        420字节       引导程序代码
    0x1FE       2字节           签名, 0×55, 0xAA
我不打算研究文件系统, 所以这些数据不用去管它, 我们唯一需要了解的是, 当SD卡被格式化为FAT文件系统后, 会在DBR的BPB或者扩展BPB中出现FAT字样.
下图是我的SD卡的DBR:


基本知识到此结束. 如果想进一步研究FAT文件系统, 可参考本文末尾给的的链接.
FatFs检测文件系统的手段.
既然FatFs提示无文件系统, 那FatFs是通过何种手段来检测是否存在文件系统的呢?
在FatFs中有一个check_fs()函数(第1981行 – 第1998行)用来检测FAT文件系统的有效性, 代码如下:
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18	static BYTE check_fs (    /* 0:FAT-VBR, 1:Valid BR but not FAT, 2:Not a BR, 3:Disk error */
            FATFS *fs,    /* File system object */
            DWORD sect    /* Sector# (lba) to check if it is an FAT boot record or not */
)
{
    if (disk_read(fs->drv, fs->win, sect, 1) != RES_OK)    /* Load boot record */
        return 3;
    /* Check record signature (always placed at offset 510 even if the sector size is >512) */
    if (LD_WORD(&fs->win[BS_55AA]) != 0xAA55)
        return 2;

    if ((LD_DWORD(&fs->win[BS_FilSysType]) & 0xFFFFFF) == 0x544146)    /* Check "FAT" string */
        return 0;
    if ((LD_DWORD(&fs->win[BS_FilSysType32]) & 0xFFFFFF) == 0x544146)
        return 0;
     
    return 1;
}

此函数先读取了指定的扇区, 然后执行相应的检查.
BS_FilSysType和BS_FilSysType2均为常数宏定义, 值分别为54和82, 代表了”FAT”字样出现在DBR中的偏移量.
将0×544146拆分为三个8位数据, 转为ASCII码, 即为FAT.
可见FatFs就是通过检测DBR的BPB和扩展BPB中是否存在FAT字样来判断目标文件系统是否为FAT文件系统的.
另外注意函数的返回值:
0为有效的FAT分区;
1为BR(Boot Record)有效但并不是FAT分区.
2为BR无效.
3为读取数据出现错误.

既然SD卡经过正确的格式化, 那么DBR一定不会出错, 所以问题不会在这里.
接下来找到调用check_fs()的地方, 从chk_mounted()函数(第2007行)的2066行开始:
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15	/* Search FAT partition on the drive. Supports only generic partitionings, FDISK and SFD. */
fmt = check_fs(fs, bsect = 0);      /* Load sector 0 and check if it is an FAT-VBR (in SFD) */
if (LD2PT(vol) && !fmt) fmt = 1;    /* Force non-SFD if the volume is forced partition */
if (fmt == 1) {                     /* Not an FAT-VBR, the physical drive can be partitioned */
    /* Check the partition listed in the partition table */
    pi = LD2PT(vol);
    if (pi) pi--;
    tbl = &fs->win[MBR_Table + pi * SZ_PTE];    /* Partition table */
    if (tbl[4]) {                               /* Is the partition existing? */
        bsect = LD_DWORD(&tbl[8]);              /* Partition offset in LBA */
        fmt = check_fs(fs, bsect);              /* Check the partition */
    }
}
if (fmt == 3) return FR_DISK_ERR;
if (fmt) return FR_NO_FILESYSTEM;               /* No FAT volume is found */
解释一下这段代码.
如果没有定义_MULLTI_PARTITION的话, 则LD2PT(vol)值始终为0(参照LDPT的宏定义).
这里我们只有一个分区, 所以此值为0.
fmt = check_fs(fs, bsect = 0)对SD卡的0扇区(物理)执行文件系统有效性检查. 如果SD卡无MBR, 即第0扇区就为DBR,且文件系统正确的话, 那么第二个if将不会再执行.
如果SD卡有MBR, 那么函数必定返回1, 那么下面的函数将会从MBR的分区表中查找到逻辑扇区0(即第一个分区DBR所在的物理扇区), 再次进行文件系统有效性检测. 
逻辑很完美, 但问题恰恰出在这里.
注意这几行代码:
1
2
3
4
5
6
7	pi = LDPT(vol);
if (pi) pi--;
tbl = &fs->win[MBR_Table + pi * SZ_PTE];        // 得到一个分区表项 SE_PTE = 16
if (tbl[4]) {                                   // 如果存在文件系统
    bsect = LD_DWORD(&tbl[8]);                  // 得到分区的DBR所在位置(物理扇区)
    fmt = check_fs(fs, bsect);                  // 执行检测
}

前面说过, LDPT(vol) 的值始终为0, tbl[4](即分区表第4个字节)指的是文件系统类型, 不可为0.
如果SD卡存在MBR, 则从MBR的第一个分区表项中得到第一个分区的相关信息, 如果此分区存在文件系统的话(检查第4个字节得到), 通过分区表项找到分区的DBR位置, 再次进行文件系统有效性检查.
以上就是这段代码的所有的逻辑.
看起来还是没有问题.
是的, 代码是没有问题, 但是SD卡会耍流氓, 不, 应该是格式化SD卡的软件会耍流氓.
一般我们不会给SD卡分区, 所以SD卡只有一个分区. 如果SD卡有MBR(一般都会有), 那意味着将会有4个分区表项. 重点就在这里. 按道理说SD卡分区相关数据应该在MBR的第一个分区表项里面, 但是, 这些数据存在于第二个分区表项或者第三个或者第四个都没有问题, PC都是可以正确认出来的, FatFs不行. 如果分区数据正好在第4个分区表项里, PC识别是没有问题的, FatFs傻乎乎的只会去检查第一个分区表项, 然后就杯具了, 只好报告FR_NO_FILESYSTEM.
如果你仔细看图一的话你就会发现, 我的这张SD卡的分区相关数据正好就在MBR的第4个分区表项里!
更糟糕的时, 就算你使用Windows(我使用的是Win7 32bit sp1)将SD卡再格式化一次, 分区表是不会改变的. 所以只好使用FatFs自带的f_mkfs()再格式化一次了, 或者, 你可以使用WinHex手工更改一下(我就是这么干的), 再或者, 自己动手更改FatFs源代码.
关于我的SD卡为什么会出现这种情况:
记得前几个月给电脑更新了一下BIOS, 正好使用这张SD卡做引导, 所以应该是更改BIOS的程序制作引导磁盘的时候”闯的祸”.
后记:
打算向ELM ChaN反映这个问题, 于是到FatFs的官网转了一圈, 发现FatFs已经更新到V0.10了, 下载下来一看, 此问题已经得到解决, 所以解决此问题又多了一个选择: 使用FatFs V0.10.









# 参数及其取值

## 函数的执行结果
参数：FRESULT
位置：ff.h
名称	Value	说明
FR_OK	0	函数执行成功
FR_DISK_ERR	1	存储设备的硬件错误in the low level disk I/O layer
FR_INT_ERR 	2	Assertion failed
FR_NOT_READY 	3	存储设备无法工作
FR_NO_FILE	4	Could not find the file
FR_NO_PATH 	5	Could not find the path
FR_INVALID_NAME 	6	The path name format is invalid
FR_DENIED	7	Access denied due to prohibited access or directory full
No free contiguous space was found.
Size of the file was not zero.
The file has been opened in read-only mode.
Not allowable file size. (>= 4GiB on FAT volume)
FR_EXIST	8	Access denied due to prohibited access
FR_INVALID_OBJECT 	9	The file/directory object is invalid
FR_WRITE_PROTECTED 	10	存储设备处于写保护状态，不能写入数据
FR_INVALID_DRIVE	11	输入的物理编号是无效的；
FR_NOT_ENABLED	12	The volume has no work area
FR_NO_FILESYSTEM	13	主机中没有该FAT文件系统，需要先创建一个
FR_MKFS_ABORTED 	14	创建过程被意外终止
FR_TIMEOUT 	15	Could not get a grant to access the volume within defined period
FR_LOCKED 	16	The operation is rejected according to the file sharing policy
FR_NOT_ENOUGH_CORE	17	LFN working buffer could not be allocated
FR_TOO_MANY_OPEN_FILES 	18	Number of open files > _FS_LOCK
FR_INVALID_PARAMETER	19	函数输入的参数无效

## 路径名称path name
系统会读取这个字符串，所以输入要按照一定的格式

The format of path name on the FatFs module is similer to the filename specs of DOS/Windos as follows:
"[drive:][/]directory/file"
The FatFs module supports long file name (LFN) and 8.3 format file name (SFN). The LFN can be used when (_USE_LFN != 0). The sub directories are separated with a \ or / in the same way as DOS/Windows API. Duplicated separators are skipped and ignored. Only a difference is that the logical drive is specified in a numeral with a colon. When drive number is omitted, the drive number is assumed as default drive (drive 0 or current drive).
Control characters ('\0' to '\x1F') are recognized as end of the path name. Leading/embedded spaces in the path name are valid as a part of the name at LFN configuration but the space is recognized as end of the path name at non-LFN configuration. Trailing spaces and dots are ignored at both configurations.
In default configuration (_FS_RPATH == 0), it does not have a concept of current directory like OS oriented file system. All objects on the volume are always specified in full path name that follows from the root directory. Dot directory names (".", "..") are not allowed. Heading separator is ignored and it can be exist or omitted. The default drive is fixed to drive 0.
When relative path is enabled (_FS_RPATH >= 1), specified path is followed from the root directory if a heading separator is exist. If not, it is followed from the current directory of the drive set by f_chdir function. Dot names are also allowed for the path names. The default drive is the current drive set by f_chdrive function.
Path name	_FS_RPATH == 0	_FS_RPATH >= 1
file.txt	A file in the root directory of the drive 0	A file in the current directory of the current drive
/file.txt	A file in the root directory of the drive 0	A file in the root directory of the current drive
	The root directory of the drive 0	The current directory of the current drive
/	The root directory of the drive 0	The root directory of the current drive
2:	The root directory of the drive 2	The current directory of the drive 2
2:/	The root directory of the drive 2	The root directory of the drive 2
2:file.txt	A file in the root directory of the drive 2	A file in the current directory of the drive 2
../file.txt	Invalid name	A file in the parent directory
.	Invalid name	This directory
..	Invalid name	Parent directory of the current directory (*)
dir1/..	Invalid name	The current directory
/..	Invalid name	The root directory (sticks the top level)
When option _STR_VOLUME_ID is specified, also pre-defined strings can be used as drive identifier in the path name instead of a numeral. e.g. "sd:file1.txt", "ram:swapfile.dat" and DOS/Windows style drive letter, of course.
Remark: In this revision, double dot name ".." cannot follow the parent directory on the exFAT volume. It will work as "." and stay there.


get_fattime	#if !_FS_READONLY && !_FS_NORTC
ff_convert	#if _USE_LFN != 0  
ff_wtoupper	#if _USE_LFN != 0  
ff_memalloc	#if _USE_LFN == 3   
ff_memfree	#if _USE_LFN == 3  
ff_cre_syncobj	#if _FS_REENTRANT
ff_req_grant	
ff_rel_grant	
ff_del_syncobj	

# 文件系统

## 创建/删除文件系统

### 文件系统类型

_FATFS 	68020
LD2PD(vol) 	/* Get physical drive number */
(VolToPart[vol].pd)   
存在条件： _MULTI_PARTITION=1
LD2PT(vol) 	/* Get partition index */
(VolToPart[vol].pt)
存在条件： _MULTI_PARTITION=1
LD2PD(vol)               

	/* Each logical drive is bound to the same physical drive number */
(BYTE)(vol)
存在条件： _MULTI_PARTITION=0
LD2PT(vol)	/* Find first valid partition or in SFD */
 0
_T(x)	
_TEXT(x)	
f_eof(fp) 	((int)((fp)->fptr == (fp)->obj.objsize))
f_error(fp)	((fp)->err)
f_tell(fp)	((fp)->fptr)
f_size(fp) 	((fp)->obj.objsize)
f_rewind(fp) 	f_lseek((fp), 0)
f_rewinddir(dp) 	f_readdir((dp), 0)
EOF (-1)	

### FATFS

成员	数据类型	说明
fs_type	BYTE类型变量	FS_FAT12	文件系统的类型
 	 	FS_FAT16	
 	 	FS_FAT32	
 	 	FS_EXFAT	
 	 	组合	如FM_FAT|FM_FAT32，那么会自动选择一个符合存储设备容量的类型
drv	BYTE类型变量	表示物理存储设备的编号
n_fats	BYTE类型变量	表示FATs的数量， 只能取值1 或2
wflag	BYTE类型变量	表示win[] flag (b0:dirty)
fsi_flag	BYTE类型变量	表示FSINFO flags (b7:disabled, b0:dirty)
id	WORD类型变量	表示File system mount ID
n_rootdir	WORD类型变量	表示Number of root directory entries (FAT12/16) 
csize	WORD类型变量	表示一个簇中含有的扇区的数目；
ssize	WORD类型变量	表示 /* Sector size (512, 1024, 2048 or 4096) */
存在条件：_MAX_SS != _MIN_SS
lfnbuf	WCHAR类型指针	表示/* LFN working buffer */
存在条件：_USE_LFN != 0
dirbuf	BYTE类型指针	表示/* Directory entry block scratchpad buffer */
存在条件：_FS_EXFAT
sobj	_SYNC_t类型变量	表示/* Identifier of sync object */
存在条件：_FS_REENTRANT
last_clst	DWORD类型变量	表示/* Last allocated cluster */
存在条件：_FS_READONLY=0
free_clst	DWORD类型变量	表示/* Number of free clusters */
存在条件：_FS_READONLY=0
cdir	DWORD类型变量	表示 /* Current directory start cluster (0:root) */
存在条件：_FS_RPATH != 0
cdc_scl	DWORD类型变量	表示/* Containing directory start cluster (invalid when cdir is 0) */
存在条件：_FS_RPATH !=0、_FS_EXFAT =1
cdc_size	DWORD类型变量	表示/* b31-b8:Size of containing directory, b7-b0: Chain status */
存在条件：_FS_RPATH !=0、_FS_EXFAT =1
cdc_ofs	DWORD类型变量	表示/* Offset in the containing directory (invalid when cdir is 0) */
存在条件：_FS_RPATH !=0、_FS_EXFAT =1
n_fatent	DWORD类型变量	表示/* Number of FAT entries (number of clusters + 2) *
fsize	DWORD类型变量	表示/* Size of an FAT [sectors] */
volbase	DWORD类型变量	表示/* Volume base sector */
fatbase	DWORD类型变量	表示/* FAT base sector */
dirbase	DWORD类型变量	表示/* Root directory base sector/cluster */
database	DWORD类型变量	表示/* Data base sector */
winsect	DWORD类型变量	表示/* Current sector appearing in the win[] */
win[_MAX_SS]	BYTE类型数组	



f_mount	输入	FATFS指针	预先创建一个FATFS结构类型变量，
将其地址传递给本参数；这里是将一个空白的传递给本参数
FATFS结构类型变量表示
 	 	 	当某个存储设备不再使用当前挂载的文件系统，使用f_mount函数，第一个参数传递NULL，将储设备中的当前文件系统取消挂载
 	 	TCHAR指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是该文件系统要挂载到的目标存储区域
同一个存储区域的不同分区可以挂载不同的文件系统，这里要指明挂载到哪个存储区域；
 	 	BYTE变量	1立即挂载
0不立即挂载，延迟挂载。
 	输出	FRESULT变量	见《函数执行结果》
 	使用	本函数用于将主机中的一个FAT文件系统挂载到一个存储设备中去；
res1 = f_mount(&fs,"0:",1);   //不管主机中有没有已经建好的文件系统，可以先调用f_mount函数试探一下； 
if(res == FR_NO_FILESYSTEM)   //返回FR_NO_FILESYSTEM，说明主机中没有FAT文件系统；
res2=f_mkfs	                //此时再进行创建，，			
if(res2== FR_OK){             //如果创建成功
res3= f_mount(NULL,"0:",1);   //保险起见，先取消挂载！！	
res3= f_mount(&fs,"0:",1);     //再执行挂载
else{}                     //如果创建不成功……
else if(res_sd!=FR_OK){}     //返回1、3、11，说明文件系统已经存在，但挂载操作出现了错误
else {}                   //返回FR_OK，说明该FAT文件系统是已经存在的；挂载也成功了
		
f_fdisk	输入	BYTE变量	将要进行分区操作的存储设备的编号；
填入diskio.c文件中分配的数字，即设备的物理编号
 	 	DWORD指针	预先创建一个含有4个元素的DWORD类型数组，将该数组的首地址传递到本参数；
这个数组称为partition map table ：
4个元素分别对应4个分区，元素的值表示该分区的存储空间，
如果其数值小于或等于100，表示的是其占有整个存储空间的百分比；
如果其数值大于100，表示的是该分区包含有扇区的个数。
如果其数值等于0，那就跳过这个元素，减少一个分区
 	 	BYTE指针	同f_mkfs函数的第4个参数；(两函数用于同一个设备)
 	输出	FRESULT变量	见《函数执行结果》
 	使用	本函数用于将某个存储设备进行分区操作；
 	 	使用条件	_MULTI_PARTITION=1 ：仅用于FDISK格式的文件系统、仅用于可进行分区的存储设备
_FS_READOLNY = 0：
_USE_MKFS = 1：
 	 	使用限制	1.一个存储设备最多分成4个区域；每个区域可以包含多个簇，每个簇包含多个扇区；
2.本函数要在f_mkfs之前使用，因为




f_mkfs	输入	TCHAR指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是本函数创建的文件系统将要运行的的目标存储区域
 	 	BYTE变量	FM_FAT		FDISK 格式：要对磁盘进行分区
文件系统从第一个分区中的第一个扇区开始创建；
硬件设备：硬盘、SD卡、U盘、MMC卡、CFC 卡
使用条件：_MULTI_PARTITION=1
 	 	 	FM_FAT32		 
 	 	 	FM_EXFAT		 
 	 	 	FM_ANY		 
 	 	 	FM_SFD	SFD格式 (super-floppy disk) ：不对磁盘进行分区(即只有一个分区)；
文件系统从第一个扇区开始创建；
硬件设备：软盘、光盘、Microdrive
使用条件：_MULTI_PARTITION=0
 	 	 	组合	
 	 	DWORD变量	在FAT文件系统中，分配给每个文件的存储空间是固定的，使用本参数设置；
一个文件的实际大小小于分配的存储空间，那么多余的空间就是闲置的；
一个文件的实际大小大于分配的存储空间，那么就会出现错误；
要根据所要存储的文件来合理调节；
单位：字节(bytes)
0：使用默认的分配单位，系统会根据存储设备的容量来自动配置
 	 	void指针	void表示你可以是任意数据类型
根据   数据的数据类型，预先创建一个相应数据类型的数组，数组的容量至少为_MAX_SS
数组名传递给本参数；
本函数执行完毕后，会有一组数据赋给这个数组；
这个数组的值就是
 	 	UINT变量	sizeof work
 	输出	FRESULT变量	见《函数执行结果》
 	使用	本函数用于在某个存储区域中创建一个文件系统

## 相关操作

### 获取文件系统的信息

f_getfree	输入	TCHAR类型
指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是
 	 	DWORD类型
指针	预先创建一个DWORD类型的变量，将该变量的地址传递到本参数；
本函数执行完毕后，会有一个数值赋给这个变量；
这个变量的值就是该存储区域中空闲的簇的数目；
单位：扇区数
 	 	FATFS类型
指针的指针	查询的是某个存储区域中空闲的簇的数目
本参数填入该存储区域的文件系统，即FATFS类型变量的地址的地址；
 	输出	FRESULT类型
变量	见《函数执行结果》
 	使用	本函数用于获得某个存储区域中空闲簇的数目
When FSINFO structure on the FAT32 volume is not in sync, this function can return an incorrect free cluster count. To avoid this problem, FatFs can be forced full FAT scan by _FS_NOFSINFO option.

f_getlabel
	输入	TCHAR类型
指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是要读取序号的目标存储区域
 	 	TCHAR类型
指针	预先创建一个TCHAR类型数组，要求数组最少的元素个数：
_LFN_UNICODE = 0：至少24个元素
_LFN_UNICODE = 1：至少12个元素
将该数组的首地址传递到本参数；传递NULL表示不需要这个信息
TCHAR类型数组可以用来存储一个字符串，本函数执行完毕后，会有一个字符串赋给这个数组；这个字符串就是该存储区域的标签；
 	 	DWORD类型
指针	预先创建一个DWORD类型的变量，将该变量的地址传递到本参数；
本函数执行完毕后，会有一个数值赋给这个变量；
这个变量的值是该存储区域的文件系统的序号；传递NULL表示不需要这个信息
 	输出	FRESULT类型
变量	见《函数执行结果》
 	使用	本函数用于获得某个存储区域的标签
使用条件：_USE_LABEL == 1
f_setlabel	输入	TCHAR类型
指针	输入一个固定格式的字符串，表示一个标签的内容
“2:DATA DISK”，表示给path name为”2:”的存储区域设置标签”DATA DISK”
“3:music1234”，表示给path name为”3:”的存储区域设置标签”DATA DISK”
“haha”，表示给默认存储区域设置标签”haha”
“2:”，表示去除path name为”2”的存储区域的标签
 	输出	FRESULT类型
变量	见《函数执行结果》
 	使用	本函数用于设置某个存储区域的序号
使用条件：
_USE_LABEL == 1
_FS_READONLY == 0 

### 文件系统与存储设备的连接

STA_NOINIT      	
STA_NODISK       	
STA_PROTECT      	
CTRL_SYNC      	
GET_SECTOR_COUNT  	
GET_SECTOR_SIZE    	
GET_BLOCK_SIZE    	
CTRL_TRIM        	
CTRL_POWER     	
CTRL_LOCK   	
CTRL_EJECT       	
CTRL_FORMAT     	
MMC_GET_TYPE     	
MMC_GET_CSD        	
MMC_GET_CID      	
MMC_GET_OCR     	
MMC_GET_SDSTAT   	
ISDIO_READ    	
ISDIO_WRITE  	
ISDIO_MRITE    	
ATA_GET_REV      	
ATA_GET_MODEL     	
ATA_GET_SN         	

### 文件系统功能配置

位置：ffconf.h

_FS_READONLY	0：不使用 disk_write、get_fattime、disk_ioctl 这3个函数
_FS_MINIMIZE	
_USE_STRFUNC	
_USE_FIND	
_USE_MKFS	1：使用FatFs格式化功能(f_mkf函数)
0：不使用格式化功能
_USE_FASTSEEK	
_USE_LABEL	
_USE_FORWARD	
_CODE_PAGE	语言功能选择
936：:支持简体中文，需要把option文件夹中的cc936.c添加到工程

_USE_LFN	长文件名支持，默认不支持长文件名，
2：支持长文件名，并指定使用栈空间为缓冲区
_MAX_LFN	
_LFN_UNICODE	
_STRF_ENCODE	
_FS_RPATH	
_VOLUMES	指定物理设备数量，这里设置为2，包括预留SD卡和SPI Flash芯片
_STR_VOLUME_ID	
_MULTI_PARTITION	
_MIN_SS		指定扇区大小的最小值
SD卡扇区大小一般都为512字节，SPI Flash芯片扇区大小一般设置为4096字节，所以需要把_MAX_SS改为4096：
_MAX_SS	指定扇区大小的最大值
_USE_TRIM	
_FS_NOFSINFO	
_FS_TINY	
_FS_NORTC	
_NORTC_MON	
_NORTC_MDAY	
_NORTC_YEAR	
_FS_LOCK	
_FS_REENTRANT	
_FS_TIMEOUT	
_SYNC_t	
_WORD_ACCESS	

### PARTITION

​	BYTE_pd	存在条件： _MULTI_PARTITION=1
 	BYTE_pt	存在条件： _MULTI_PARTITION=1

### FDID
成员	数据类型	说明
fs	FATFS类型指针
	/* Pointer to the owner file system object */

id	WORD类型变量	表示：/* Owner file system mount ID */

attr	BYTE类型变量	表示：/* Object attribute */

stat	BYTE类型变量	表示：/* Object chain status (b1-0: =0:not contiguous, =2:contiguous (no data on FAT), =3:got flagmented, b2:sub-directory stretched) */
sclust	DWORD类型变量	表示： /* Object start cluster (0:no cluster or root directory) */
objsize	FSIZE_t类型变量	表示： /* Object size (valid when sclust != 0) */

n_cont	DWORD类型变量	表示：/* Size of coutiguous part, clusters - 1 (valid when stat == 3) */
存在条件：_FS_EXFAT=1
c_scl	DWORD类型变量	表示：/* Containing directory start cluster (valid when sclust != 0) */
存在条件：_FS_EXFAT=1
c_size	DWORD类型变量	表示：/* b31-b8:Size of containing directory, b7-b0: Chain status (valid when c_scl != 0) */
存在条件：_FS_EXFAT=1
c_ofs	DWORD类型变量	表示：/* Offset in the containing directory (valid when sclust != 0) */
存在条件：_FS_EXFAT=1
lockid	UINT类型变量	表示：
存在条件：_FS_LOCK != 0




### FILINFO
成员	数据类型	说明
fsize	FSIZE_t 类型变量	表示：  /* File size */

fdate	WORD类型变量	表示：   /* Modified date */

ftime	WORD类型变量	表示：  /* Modified time */
fattrib	BYTE类型变量	AM_RDO	Read only
 	 	AM_HID	Hidden
 	 	AM_SYS	System
 	 	AM_DIR	Directory
 	 	AM_ARC	Archive
altname[13]	TCHAR类型数组	表示：/* Altenative file name */
存在条件：_USE_LFN != 0
fname[_MAX_LFN + 1]	BYTE类型数组	表示： /* Primary file name */
存在条件：_USE_LFN != 0
fname[13]	DWORD类型变量	表示：/* File name */
存在条件：_USE_LFN = 0


# 文件路径

## 创建一个路径

### DIR
结构类型，表示一个文件目录
成员包含一个文件目录的所有属性
成员	数据类型	说明
obj	_FDID 类型变量	表示： Object identifier (must be the 1st member to detect invalid object pointer)
dptr	DWORD类型变量	表示： /* Current read/write offset */

clust	DWORD类型变量	表示： /* Current cluster */

sect	DWORD类型变量	表示：/* Current sector */
dir	BYTE类型指针	表示：/* Pointer to the directory item in the win[] */

 fn[12]	BYTE类型数组	表示： /* SFN (in/out) {body[8],ext[3],status[1]} */
blk_ofs	DWORD类型变量	表示：/* Offset of current entry block being processed (0xFFFFFFFF:Invalid) */
存在条件：_USE_LFN != 0
 pat	TCHAR类型指针	表示： /* Pointer to the name matching pattern */
存在条件：_USE_FIND=1





f_mkdir	输入	TCHAR指针	输入一个字符串，表示要创建的路径的名称 ；如：res=f_mkdir( "sub1")
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	创建一个路径
 	 	使用条件	_FS_READONLY == 0 
 _FS_MINIMIZE == 0.

## 打开/关闭文件路径
f_opendir	输入	DIR指针	
 	 	TCHAR指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是要打开的路径所在的大路径的名称
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	接下来就是使用f_opendir函数打开指定的路径。
如果路径存在就使用f_readdir函数读取路径下内容，f_readdir函数可以读取路径下的文件或者文件夹，并保存信息到文件信息对象变量内。

使用f_opendir函数可以打开路径(这里不区分目录和路径概念，下同)，如果路径不存在则返回错误，使用f_closedir函数关闭已经打开的路径。新版的FatFs支持相对路径功能，使路径操作更加灵活。
f_opendir函数有两个形参
第一个参数为指向路径对象的指针
第二个参数为路径。

 	 	使用条件	.
	
f_closedir	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	

## 编辑路径
f_unlink	输入	TCHAR指针	输入一个固定格式的字符串，表示一个路径名称；具体见《路径名称path name》
表示的是移动的目标位置
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	
f_chdir
	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	
f_chdrive
	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	
f_getcwd
	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	
f_rename	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	f_rename函数是带有移动功能的重命名函数，它有两个形参，
第一个参数为源文件名称
第二个参数为目标名称。
目标名称可附带路径，如果路径与源文件路径不同见移动文件到目标路径下。
 	 	使用条件	
f_chmod
	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	
f_utime	输入	TCHAR指针	
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	
 	 	使用条件	




f_stat	f_stat函数用于获取文件的属性，有两个形参，
第一个参数为文件路径
第二个参数为返回指向文件信息结构体变量的指针。
文件信息结构体变量包含文件的大小、最后修改时间和日期、文件属性、短文件名以及长文件名等信息。 
f_closedir	
f_closedir函数只需要指向路径对象的指针一个形参。
f_readdir	f_readdir函数有两个形参
第一个参数为指向路径对象变量的指针
第二个参数为指向文件信息对象的指针。
f_readdir函数另外一个特性是自动读取下一个文件对象，即循序运行该函数可以读取该路径下的所有文件。
所以，在程序中，我们使用for循环让f_readdir函数读取所有文件，并在读取所有文件之后退出循环。

在f_readdir函数成功读取到一个对象时，我们还不清楚它是一个文件还是一个文件夹，此时我们就可以使用文件信息对象变量的文件属性来判断了，如果判断得出是个文件那我们就直接通过串口打印出来就好了。
f_findfirst	
f_findnext	






# 单个文件
## 创建/删除文件

### FIL

结构类型，表示一个文件，包含了文件很多基本属性，比如文件大小、路径、当前读写地址等等。
如果需要在同一时间打开多个文件进行读写，才需要定义多个FIL变量，不然一般定义一个FIL变量即可。 

成员	数据类型	说明
obj	_FDID 类型变量	表示： Object identifier (must be the 1st member to detect invalid object pointer) 

flag	BYTE类型变量	表示： /* File status flags */

err	BYTE类型变量	表示： /* Abort flag (error code) */
fptr	FSIZE_t类型变量	表示： /* File read/write pointer (Zeroed on file open) */

clust	DWORD类型变量	表示：/* Current cluster of fpter (invalid when fprt is 0) */

sect	DWORD类型变量	表示：/* Sector number appearing in buf[] (0:invalid) */

dir_sect	DWORD类型变量	表示：/* Sector number containing the directory entry */

存在条件：_FS_READONLY=0
dir_ptr	BYTE类型指针	表示： /* Pointer to the directory entry in the win[] */

存在条件：_FS_READONLY=0
cltbl	DWORD类型指针	表示： /* Pointer to the cluster link map table (nulled on open, set by application) */
cltbl=NULL：
如果要使用快速寻找功能那么需要设置_USE_FASTSEEK =1
buf[_MAX_SS]	BYTE类型数组	表示：/* File private data read/write window */



## 文件操作

### 打开/关闭文件
f_open	输入	FIL指针	FIL结构类型变量表示这里是将一个空白的传递给本参数
预先创建一个FIL结构类型变量
将地址传递给本参数；本函数执行完毕后，
 	 	TCHAR指针	输入一个字符串，表示一个文件的文件名：”文件名称.后缀名”  如： "message.txt"
表示本函数要操作的目标文件：
 	 	BYTE变量	文件访问模式选择
组合模式如：FA_CREATE_ALWAYS|FA_WRITE表示总是新建文件并对该文件进行写操作
_FS_READONLY == 1时，只支持FA_READ、  FA_OPEN_EXISTING 两种模式
 	 	 	FA_READ	读取文件内容
 	 	 	FA_WRITE	写文件	
 	 	 	FA_OPEN_EXISTING	打开文件，如果该文件不存在，函数退出
 	 	 	FA_CREATE_NEW	新建一个文件，如果该文件已存在，函数退出
 	 	 	FA_CREATE_ALWAYS	新建一个文件，如果该文件已存在，它会被覆盖
 	 	 	FA_OPEN_ALWAYS	打开文件，如果该文件不存在，就直接创建该文件(空白)，并打开
 	 	 	FA_OPEN_APPEND	同FA_OPEN_ALWAYS，但打开后，读/写指针在文件最后，
使用前面的6个操作模式打开文件，读/写指针在文件最前面，
 	输出	FRESULT变量	见《函数执行结果》
 	使用	使用限制	f_open打开的文件有2个属性：FIL类型指针 和 文件名
！！要无时无刻地保证：一个FIL类型指针对应一个单个文件！！所以：
不能使用同一个FIL类型指针的f_open函数打开两个不同文件名的文件(如果这么做，就出现了一个FIL类型指针对应2个单个文件的情况)；
在f_open之后使用的所有其他函数，都不再传递文件名，而是传递一个FIL类型指针，因为一个FIL类型指针对应一个单个文件，所以通过一个FIL类型指针就能找到对应的文件。
 	 	举例	res=f_open(&fil1,"message.txt",FA_READ);    //在操作文件前一定要先打开该文件
res=f_open(&fil2,"report.txt",FA_READ);  //打开另一个文件，因为使用了不同的FIL类型指针
res=f_read(&fil1,buffer,sizeof buffer,&fnum);  //操作&fil1对应的文件
res=f_write(&fil2,buffer,sizeof buffer,&fnum);  //操作&fil2对应的文件
res=f_write(&fil1,buffer,sizeof buffer,&fnum);  //继续操作&fil1对应的文件
res=f_truncate(&fil2);                   //继续操作&fil2对应的文件
res=f_close(&fil1);      //关闭&fil1对应的文件，然后才可以用&fil1打开另一个文件
res=f_sync(&fil2);      //还能继续操作&fil2对应的文件
res=f_close(&fil2);      //关闭&fil2对应的文件，然后才可以用&fil2打开另一个文件
res=f_open(&fil1,"novel.txt",FA_WRITE);     //用&fil1打开另一个文件"novel.txt"
res=f_open(&fil2,"fiction.txt",FA_WRITE);     //用&fil2打开另一个文件"fiction.txt"

f_close	输入	FIL类型指针	要操作的目标文件此时对应的FIL类型指针
 	输出	FRESULT变量	见《函数执行结果》
 	使用	使用f_close函数关闭某个文件，并停止对相应的FIL类型指针的使用，
这之后可以用f_open函数，传递该FIL指针，打开另一个文件
		
f_sync	输入	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	f_close包含f_sync ，f_sync拥有f_close函数的部分功能；可以说f_sync没有关闭完全
都是用于关闭某个文件，两个函数的区别是：
f_sync也用于关闭某个文件，但该文件被关闭后仍处于有效状态，接下来仍可以使用f_read、f_write等哈函数操作这个文件；
f_sync适用于需要长时间打开的文件，
in write mode, such as data logger. Performing f_sync function of periodic or immediataly after f_write function can minimize the risk of data loss due to a sudden blackout or an unintentional media removal. For more information, refer to application note.

 	 	使用条件	_FS_READONLY == 0

### 读文件
f_read	输入	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	 	void指针	void表示任意的数据类型；
根据将要读取的数据的数据类型，预先创建一个相应数据类型的数组；
数组名传递给本参数；本函数执行完毕后，会有一组数据赋给这个数组；
这个数组的值就是要读取的文件中的数据；
 	 	UINT变量	指定读取的字节数
 	 	UINT指针	预选创建一个32位无符号整形变量；
变量的地址传递给本参数；本函数执行完毕后，会有一个数值赋给这个变量；
这个变量的值就是本次读操作成功读取的字节个数。
读取完毕后，应该检查该数值是否就是用户指定读取的字节的个数；
如果该数值较小，说明指针已经到了文件最末；
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	使用f_read从目标文件读取数据
读取时，是从当前读/写指针在该文件中的位置开始
		
f_forward	输入	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	 	函数指针	预先定义一个函数：
函数输入一个BYTE指针和一个UINT变量，输出为UINT变量
函数名传递给本参数，本函数执行完毕后

 	 	UINT变量	指定读取的字节数
 	 	UINT指针	预选创建一个32位无符号整形变量
变量的地址传递给本参数；本函数执行完毕后，会有一个数值赋给这个变量；
该变量的值是本次读操作成功读取的字节数 
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	使用f_forward从目标文件读取数据；
读取时，是从当前读/写指针在该文件中的位置开始
与f_read不同的是，f_forward要求输入一个函数的指针而不是一个void的指针；

 	 	使用条件	_USE_FORWARD == 1.

f_gets	输入	TCHAR指针	TCHAR数据类型的数组可以用于储存一个字符串
预先创建一个TCHAR数据类型的数组，
数组名传递给本参数；本函数执行完毕后，会有一个字符串赋给这个数组；
这个字符串就是要读取的文件中的数据（除字符串最后一个字符以外），
字符串最后一个字符是’\0’，不属于读取的目标文件中的内容，而是函数自己添加的表示字符串的结尾。
 	 	 	返回的数组的值是NULL，表明没有数据，或读取错误
 	 	int变量	TCHAR类型数组的容量，数组的一个元素就是一个字符；
 	 	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	使用f_gets函数读取一个字符串
读取时，是从当前读/写指针在该文件中的位置开始，直到读取到一个’\n’字符
 	 	使用条件	_USE_STRFUNC = 1：按照文件中的内容正常读取
或_USE_STRFUNC = 2：将读取的内容中的'\r'去除，再放到TCHAR数组中

### 写文件
f_write	输入	FIL指针	要操作的目标文件此时对应的FIL指针
 	 	void指针	void表示可以是任意数据类型
根据要写入的数据的数据类型，预先创建一个相应数据类型的数组，给该数组赋值将要写入的数据；
数组名传递给本参数；函数会将该数组的内容写入到文件中。
 	 	UINT变量	指定写入的字节数
 	 	UINT指针	预选创建一个32位无符号整形变量
将变量的地址传递给本参数；本函数执行完毕后，会有一个数值赋给这个变量；
这个变量的值就是本次读操作成功写入的字节个数。
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	使用f_write向目标文件写入数据
写入时，是从当前读/写指针在该文件中的位置开始
 	 	使用条件	_FS_READONLY == 0

f_putc	输入	TCHAR变量	TCHAR数据类型的变量可以用于储存一个字符
预先创建一个TCHAR数据类型的变量，给该变量赋值将要写入的字符；
变量地址传递给本参数；函数会将该字符写入到文件中。
 	 	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	输出	INT变量	1：写入成功
-1：写入失败
 	使用	功能	使用f_putc写入一个字符
写入时，是从当前读/写指针在该文件中的位置开始
 	 	使用条件	_FS_READONLY == 0
_USE_STRFUNC = 1 ，按照TCHAR变量的内容正常写入
_USE_STRFUNC = 2，如果输入的TCHAR变量是的'\n' ，那么变为'\r'+'\n'，再写入到文件中

f_puts	输入	TCHAR指针	TCHAR数据类型的变量可以用于储存一个字符
预先创建一个TCHAR数据类型的变量，给该变量赋值将要写入的字符；
变量地址传递给本参数；函数会将该字符写入到文件中。
 	 	FIL指针	要操作的目标文件此时对应的FIL类型指针

 	输出	INT变量	返回成功写入的字符的数目
返回-1表明写入错误
 	使用	功能	使用f_putc写入一个字符串
写入时，是从当前读/写指针在该文件中的位置开始
 	 	使用条件	_FS_READONLY == 0
_USE_STRFUNC = 1 ，按照TCHAR数组的内容正常写入
_USE_STRFUNC = 2，将输入的TCHAR数组中的'\n' 变为'\r'+'\n'，再写入到文件中

f_printf	输入	FIL指针	要操作的目标文件此时对应的FIL类型指针
 	 	TCHAR指针	
 	 	TCHAR指针	
 	 	……	……
 	输出	INT变量	返回成功写入的字符的数目
返回-1表明写入错误
 	使用	功能	使用f_printf写入一个多个字符串
写入时，是从当前读/写指针在该文件中的位置开始
 	 	使用条件	_FS_READONLY == 0
_USE_STRFUNC = 1 ，按照TCHAR数组的内容正常写入
_USE_STRFUNC = 2，将输入的TCHAR数组中的'\n' 变为'\r'+'\n'，再写入到文件中



f_lseek	输入	FIL指针	要操作的目标文件此时对应的FIL指针
 	 	FSIZE_t变量	读/写指针的位置，变量的值表示距离文件最开始处相差的单位数
_FS_EXFAT=0：一个单位32bit
_FS_EXFAT=1：一个单位64bit

 	输出	FRESULT变量	见《函数执行结果》
 	使用		使用f_lseek函数移动文件中的读/写指针
除FA_OPEN_APPEND以外，使用其他操作模式打开文件时，读/写指针都是在最开始处
 	 		如果输入的FIL指针的cltbl成员，就可以 CLMT (cluster link map table)
 	 	使用条件	_FS_MINIMIZE <= 2. 


f_truncate	输入	FIL指针	要操作的目标文件此时对应的FIL指针
 	输出	FRESULT变量	见《函数执行结果》
 	使用	使用f_truncate函数去除所选文件当前读/写指针所在位置后面的所有内容
		
f_expand	输入	FIL指针	要操作的目标文件此时对应的FIL指针
 	 	FSIZE_t变量	
 	 	BYTE变量	0：至准备好分配区域，延时分配
1：直接分配
 	输出	FRESULT变量	见《函数执行结果》
 	使用	功能	使用f_expand函数

 	 	使用条件	_USE_EXPAND == 1 
 _FS_READONLY == 0



## 连接函数

配置设备物理编号
	FatFs支持多物理设备同时运行的，需要为各个设备配置唯一的物理编号，来作区分。
源文件有一些默认的物理编号宏定义，用户可以根据自己所使用的设备进行修改或添加

disk_status	FatFs支持同时挂载多个存储设备，通过定义为不同编号以区别。
SD卡一般定义为编号0，编号1预留给串行Flash芯片使用。
使用宏定义方式给出SD卡块大小，方便修改。
实际上，SD卡块大小一般都是设置为512字节的，不管是标准SD卡还是高容量SD卡。 
本函数要求返回存储设备的当前状态，对于SD卡一般返回SD卡插入状态，这里直接返回正常状态。 
disk_initialize	该函数用于初始化存储设备，一般包括相关GPIO初始化、外设环境初始化、中断配置等等。
对于SD卡，直接调用SD_Init函数实现对SD卡初始化，如果函数返回SD_OK说明SD卡正确插入，并且控制器可以与之正常通信。 
disk_read	disk_read函数用于从存储设备指定地址开始读取一定的数量的数据到指定存储区内。
对于SD卡，最重要是使用SD_ReadMultiBlocks函数读取多块数据到存储区。
这里需要注意的地方是SD卡数据操作是使用DMA传输的，并设置数据尺寸为32位大小，为实现数据正确传输，要求存储区是4字节对齐。
在某些情况下，FatFs提供的buff地址不是4字节对齐，这会导致DMA数据传输失败，所以为保证数据传输正确，可以先判断存储区地址是否是4字节对齐，如果存储区地址已经是4字节对齐，无需其他处理，直接使用SD_ReadMultiBlocks函数执行多块读取即可。
如果判断得到地址不是4字节对齐，则先申请一个4字节对齐的临时缓冲区，即局部数组变量scratch，通过定义为DWORD类型可以使得其自动4字节对齐，scratch所占的总存储空间也是一个块大小，这样把一个块数据读取到scratch内，然后把scratch存储器内容拷贝到buff地址空间上就可以了。 

SD_ReadMultiBlocks函数用于从SD卡内读取多个块数据，它有四个形参，分别为存储区地址指针、起始块地址、块大小以及块数量。为保证数据传输完整，还需要调用SD_WaitReadOperation函数和SD_GetStatus函数检测和保证传输完成。 
disk_write	disk_write函数用于向存储设备指定地址写入指定数量的数据。对于SD卡，执行过程与disk_read函数是非常相似，也必须先检测存储区地址是否是4字节对齐，如果是4字节对齐则直接调用SD_WriteMultiBlocks函数完成多块数据写入操作。如果不是4字节对齐，
申请一个4字节对齐的临时缓冲区，先把待写入的数据拷贝到该临时缓冲区内，然后才写入到SD卡。 
SD_WriteMultiBlocks函数是向SD卡写入多个块数据，它有四个形参，分别为存储区地址指针、起始块地址、块大小以及块数量，它与SD_ReadMultiBlocks函数执行相互过程。最后也是需要使用相关函数保存数据写入完整才退出disk_write函数。 
disk_ioctl	参数：pdrv设备物理编号，cmd控制指令，buff指令对应的数据指针。 
控制指令包括发出同步信号、获取扇区数目、获取扇区大小、获取擦除块数量等等指令，对于SD卡，为支持格式化功能，需要用到获取扇区数量(GET_SECTOR_COUNT)指令和获取块尺寸(GET_BLOCK_SIZE)。

另外，SD卡扇区大小为512字节，串行Flash芯片一般设置扇区大小为4096字节，所以需要用到获取扇区大小(GET_SECTOR_SIZE)指令。 