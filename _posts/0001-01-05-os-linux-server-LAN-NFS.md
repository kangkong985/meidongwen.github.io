# 配置
所需要的软件

以 CentOS 6.x 为例的话，要设定好 NFS 服务器我们必须要有两个软件才行，分别是：

- RPC 主程序：rpcbind

  就如同刚刚提的到，我们的 NFS 其实可以被视为一个 RPC 服务，而要启动任何一个 RPC 服务之前，我们都需要做好port 的对应 (mapping) 的工作才行，这个工作其实就是『 rpcbind 』这个服务所负责的！也就是说，在启动任何一个 RPC 服务之前，我们都需要启动 rpcbind 才行！ (在 CentOS 5.x 以前这个软件称为 portmap，在 CentOS 6.x 之后才称为 rpcbind 的！)

  

- NFS 主程序：nfs-utils

  就是提供 rpc.nfsd 及 rpc.mountd 这两个 NFS daemons 与其他相关 documents 与说明文件、执行文件等的软件！这个就是NFS 服务所需要的主要软件啦！一定要有喔！

好了，知道我们需要这两个软件之后，现在干嘛？赶快去你的系统先用 RPM 看一下有没有这两个软件啦！没有的话赶快用 RPM 或 yum 去安装喔！不然就玩不下去了！

题:请问我的主机是以 RPM 为套件管理的 Linux distribution ，例如 Red Hat, CentOS 与  SuSE 等版本，那么我要如何知道我的主机里面是否已经安装了 rpcbind 与 nfs 相关的软件呢？ 

 答：简单的使用『 rpm -qa | grep nfs 』与『 rpm -qa | grep rpcbind 』即可知道啦！如果没有安装的话， 在 CentOS 内可以使用『 [yum](http://linux.vbird.org/linux_server/0210network-secure.php#update) install nfs-utils 』来安装！ 







### 主要配置文件：/etc/exports
   这个档案就是 NFS 的主要配置文件了！不过，系统并没有默认值，所以这个档案『 		不一定会存在』，你可能必须要使用 vim 		主动的建立起这个档案喔！我们等一下要谈的设定也仅只是这个档案而已吶！

  

### NFS 文件系统维护指令：/usr/sbin/exportfs
  这个是维护 NFS 分享资源的指令，我们可以利用这个指令重新分享 /etc/exports  		变更的目录资源、将 NFS Server 分享的目录卸除或重新分享等等，这个指令是 NFS  		系统里面相当重要的一个喔！至于指令的用法我们在底下会介绍。
在开始 NFS 服务器的设定之前，你必须要了解的是，NFS 会直接使用到核心功能，所以你的核心必须要有支持 NFS 才行。万一如果你的核心版本小于 2.2 版，或者重新自行编译过核心的话，那么就得要很注意啦！因为你可能会忘记选择 NFS 的核心支持啊！

还好，我们 CentOS 或者是其他版本的 Linux ，预设核心通常是支持 NFS 功能的，所以你只要确认你的核心版本是目前新的 2.6.x 版，并且使用你的 distribution 所提供的核心，那应该就不会有问题啦！
Tips:
上面会提醒您这个问题的原因是，以前鸟哥都很喜欢自行编译一个特别的核心，但是某次编译核心时，却忘记加上了 NFS 的核心功能，结果 NFS server 无论如何也搞不起来～最后才想到原来俺的核心是非正规的.
至于 NFS 服务器的架设实在很简单，你只要编辑好主要配置文件 /etc/exports 之后，先启动 rpcbind (若已经启动了，就不要重新启动)，然后再启动 nfs ，你的 NFS 就成功了！ 不过这样的设定能否对客户端生效？那就得要考虑你权限方面的设定能力了。废话少说，我们就直接来看看那个 /etc/exports 应该如何设定吧！某些 distributions 并不会主动提供 /etc/exports 档案，所以请你自行手动建立它吧。
```bash
[root@www ~]# vim /etc/exports
/tmp         192.168.100.0/24(ro)   localhost(rw)   *.ev.ncku.edu.tw(ro,sync)
[分享目录]   [第一部主机(权限)]     [可用主机名]    [可用通配符]
```
你看看，这个配置文件有够简单吧！每一行最前面是要分享出来的目录，注意喔！是以目录为单位啊！ 然后这个目录可以依照不同的权限分享给不同的主机，像鸟哥上面的例子说明是： 要将 /tmp 分别分享给三个不同的主机或网域的意思。记得主机后面以小括号 () 设计权限参数， 若权限参数不止一个时，则以逗号 (,) 分开。且主机名与小括号是连在一起的喔！在这个档案内也可以利用 # 来批注呢。

至于主机名的设定主要有几个方式：

    可以使用完整的 IP 或者是网域，例如 192.168.100.10 或 192.168.100.0/24 ，或 192.168.100.0/255.255.255.0 都可以接受！
    
    也可以使用主机名，但这个主机名必须要在 /etc/hosts 内，或可使用 DNS 找到该名称才行啊！反正重点是可找到 IP 就是了。如果是主机名的话，那么他可以支持通配符，例如 * 或 ? 均可接受。

至于权限方面 (就是小括号内的参数) 常见的参数则有：
rw/ro：该目录分享的权限是可擦写 (read-write) 或只读 (read-only)，但最终能不能读写，还是与文件系统的 rwx 及身份有关。
sync/async：sync 代表数据会同步写入到内存与硬盘中，async 则代表数据会先暂存于内存当中，而非直接写入硬盘！
no_root_squash/root_squash：客户端使用 NFS 文件系统的账号若为 root 时，系统该如何判断这个账号的身份？预设的情况下，客户端 root 的身份会由 root_squash 的设定压缩成 nfsnobody， 如此对服务器的系统会较有保障。但如果你想要开放客户端使用 root 身份来操作服务器的文件系统，那么这里就得要开 no_root_squash 才行！
all_squash：不论登入 NFS 的使用者身份为何， 他的身份都会被压缩成为匿名用户，通常也就是 nobody(nfsnobody) 啦！
anonuid/anongid：anon 意指 anonymous (匿名者) 前面关于 *_squash 提到的匿名用户的 UID 设定值，通常为 nobody(nfsnobody)，但是你可以自行设定这个 UID 的值！当然，这个 UID 必需要存在于你的 /etc/passwd 当中！ anonuid 指的是 UID 而 anongid 则是群组的 GID 啰。

这是几个比较常见的权限参数，如果你有兴趣玩其他的参数时，请自行 man exports 可以发现很多有趣的数据。 接下来我们利用上述的几个参数来实际思考一下几个有趣的小习题：

例题一：让 root 保有 root 的权限
我想将 /tmp 分享出去给大家使用，由于这个目录本来就是大家都可以读写的，因此想让所有的人都可以存取。此外，我要让 root 写入的档案还是具有 root 的权限，那如何设计配置文件？
答：
```bash
[root@www ~]# vim /etc/exports
# 任何人都可以用我的 /tmp ，用通配符来处理主机名，重点在 no_root_squash
/tmp  *(rw,no_root_squash)
```
主机名可以使用通配符，上头表示无论来自哪里都可以使用我的 /tmp 这个目录。 再次提醒，『 *(rw,no_root_squash) 』这一串设定值中间是没有空格符的喔！而 /tmp 与 *(rw,no_root_squash) 则是有空格符来隔开的！特别注意到那个 no_root_squash 的功能！在这个例子中，如果你是客户端，而且你是以 root 的身份登入你的 Linux 主机，那么当你 mount 上我这部主机的 /tmp 之后，你在该 mount 的目录当中，将具有『root 的权限！』 

例题二：同一目录针对不同范围开放不同权限
我要将一个公共的目录 /home/public 公开出去，但是只有限定我的局域网络 192.168.100.0/24 这个网域且加入 vbirdgroup (第一章的例题建立的群组) 的用户才能够读写，其他来源则只能读取。
答：
```bash
[root@www ~]# mkdir /home/public
[root@www ~]# setfacl -m g:vbirdgroup:rwx /home/public
[root@www ~]# vim /etc/exports
/tmp          *(rw,no_root_squash)
/home/public  192.168.100.0/24(rw)    *(ro)
# 继续累加在后面，注意，我有将主机与网域分为两段 (用空白隔开) 喔！
```

上面的例子说的是，当我的 IP 是在 192.168.100.0/24 这个网段的时候，那么当我在 Client 端挂载了 Server 端的 /home/public 后，针对这个被我挂载的目录我就具有可以读写的权限～ 至于如果我不是在这个网段之内，那么这个目录的数据我就仅能读取而已，亦即为只读的属性啦！

需要注意的是，通配符仅能用在主机名的分辨上面，IP 或网段就只能用 192.168.100.0/24 的状况， 不可以使用 192.168.100.* 喔！ 


例题三：仅给某个单一主机使用的目录设定
我要将一个私人的目录 /home/test 开放给 192.168.100.10 这个 Client 端的机器来使用时，该如何设定？ 假设使用者的身份是 dmtsai 才具有完整的权限时。
答：
```bash
[root@www ~]# mkdir /home/test
[root@www ~]# setfacl -m u:dmtsai:rwx /home/test
[root@www ~]# vim /etc/exports
/tmp          *(rw,no_root_squash)
/home/public  192.168.100.0/24(rw)    *(ro)
/home/test    192.168.100.10(rw)
# 只要设定 IP 正确即可！
```
这样就设定完成了！而且，只有 192.168.100.10 这部机器才能对 /home/test 这个目录进行存取喔！ 

例题四：开放匿名登录的情况
我要让 *.centos.vbird 网域的主机，登入我的 NFS 主机时，可以存取 /home/linux ，但是他们存数据的时候，我希望他们的 UID 与 GID 都变成 45 这个身份的使用者，假设我 NFS 服务器上的 UID 45 与 GID 45 的用户/组名为 nfsanon。
答：
```bash
[root@www ~]# groupadd -g 45 nfsanon
[root@www ~]# useradd -u 45 -g nfsanon nfsanon
[root@www ~]# mkdir /home/linux
[root@www ~]# setfacl -m u:nfsanon:rwx /home/linux
[root@www ~]# vim /etc/exports
/tmp          *(rw,no_root_squash)
/home/public  192.168.100.0/24(rw)    *(ro)
/home/test    192.168.100.10(rw)
/home/linux   *.centos.vbird(rw,all_squash,anonuid=45,anongid=45)
# 如果要开放匿名，那么重点是 all_squash，并且要配合 anonuid 喔！
```
特别注意到那个 all_squash 与 anonuid, anongid 的功能！如此一来，当 clientlinux.centos.vbird 登入这部 NFS 主机，并且在 /home/linux 写入档案时，该档案的所有人与所有群组，就会变成 /etc/passwd 里面对应的 UID 为 45 的那个身份的使用者了！ 


上面四个案例的权限如果依照13.1.4 存取设定权限来思考的话， 那么权限会是什么情况呢？让我们来检查一下：
- 客户端与服务器端具有相同的 UID 与账号：
假设我在 192.168.100.10 登入这部 NFS (IP 假设为 192.168.100.254) 服务器，并且我在 192.168.100.10 的账号为 dmtsai 这个身份，同时，在这部 NFS 上面也有 dmtsai 这个账号， 并具有相同的 UID ，果真如此的话，那么：
由于 192.168.100.254 这部 NFS 服务器的 /tmp 权限为 -rwxrwxrwt ，所以我 (dmtsai 在 192.168.100.10 上面) 在 /tmp 底下具有存取的权限，并且写入的档案所有人为 dmtsai ；
在 /home/public 当中，由于我有读写的权限，所以如果在 /home/public 这个目录的权限对于 dmtsai 有开放写入的话，那么我就可以读写，并且我写入的档案所有人是 dmtsai 。但是万一 /home/public 对于 dmtsai 这个使用者并没有开放可以写入的权限时， 那么我还是没有办法写入档案喔！这点请特别留意！
在 /home/test 当中，我的权限与 /home/public 相同的状态！还需要 NFS 服务器的 /home/test 对于 dmtsai 有开放权限；
在 /home/linux 当中就比较麻烦！因为不论你是何种 user ，你的身份一定会被变成 UID=45 这个账号！所以，这个目录就必需要针对 UID = 45 的那个账号名称，修改他的权限才行！

- 客户端与服务器端的账号并未相同时：
假如我在 192.168.100.10 的身份为 vbird (uid 为 600)，但是 192.168.100.254 这部 NFS 主机却没有 uid=600 的账号时，情况会变成怎样呢？
我在 /tmp 底下还是可以写入，只是该档案的权限会保持为 UID=600 ，因此服务器端看起来就会怪怪的， 因为找不到 UID=600 这个账号的显示，故档案拥有者会填上 600 呦！
我在 /home/public 里面是否可以写入，还需要视 /home/public 的权限而定，不过，由于没有加上 all_squash 的参数， 因此在该目录下会保留客户端的使用者 UID，同上一点所示。
/home/test 的观点与 /home/public 相同！
/home/linux 底下，我的身份就被变成 UID = 45 那个使用者就是了！

- 当客户端的身份为 root 时：
假如我在 192.168.100.10 的身份为 root 呢？ root 这个账号每个系统都会有呀！权限变成怎样呢？
我在 /tmp 里面可以写入，并且由于 no_root_squash 的参数，改变了预设的 root_squash 设定值，所以在 /tmp 写入的档案所有人为 root 喔！
我在 /home/public 底下的身份还是被压缩成为 nobody 了！因为默认属性里面都具有 root_squash 呢！所以，如果 /home/public 有针对 nobody 开放写入权限时，那么我就可以写入，但是档案所有人变成 nobody 就是了！
/home/test 与 /home/public 相同；
/home/linux 的情况中，我 root 的身份也被压缩成为 UID = 45 的那个使用者了！

这样的权限讲解之后，你可以了解了吗？这里是最重要的地方，如果这一关通过了，底下的咚咚就没有问题啦！ ^_^！ 在你将本文读完后，最好还是回到13.1.4 NFS 的档案访问权限好好的瞧一瞧， 才能解决 NFS 的问题喔！




### 分享资源的登录档：/var/lib/nfs/*tab
   在 NFS 服务器的登录文件都放置到 /var/lib/nfs/ 目录里面，在该目录下有两个比较重要的登录档，一个是 etab ，主要记录了 NFS 所分享出来的目录的完整权限设定值；另一个 xtab  则记录曾经链接到此 NFS 服务器的相关客户端数据。



### 客户端查询服务器分享资源的指令：/usr/sbin/showmount
   这是另一个重要的 NFS 指令。exportfs 是用在 NFS Server 端，而 showmount 则主要用在 Client 端。这个 showmount 可以用来察看 NFS 分享出来的目录资源喔！


# 启动
配置文件搞定后，当然要开始来启动才行啊！而前面我们也提到过，NFS 的启动还需要 rpcbind 的协助才行啊！ 所以赶紧来启动吧！
```bash
[root@www ~]# /etc/init.d/rpcbind start
# 如果 rpcbind 本来就已经在执行了，那就不需要启动啊！

[root@www ~]# /etc/init.d/nfs start
# 有时候某些 distributions 可能会出现如下的警告讯息：
exportfs: /etc/exports [3]: No 'sync' or 'async' option specified 
for export "192.168.100.10:/home/test".
  Assuming default behaviour ('sync').
# 上面的警告讯息仅是在告知因为我们没有指定 sync 或 async 的参数，
# 则 NFS 将默认会使用 sync 的信息而已。你可以不理他，也可以加入 /etc/exports。

[root@www ~]# /etc/init.d/nfslock start
[root@www ~]# chkconfig rpcbind on
[root@www ~]# chkconfig nfs on
[root@www ~]# chkconfig nfslock on
```
那个 rpcbind 根本就不需要设定！只要直接启动它就可以啦！启动之后，会出现一个 port 111 的 sunrpc 的服务，那就是 rpcbind 啦！至于 nfs 则会启动至少两个以上的 daemon 出现！然后就开始在监听 Client 端的需求啦！你必须要很注意屏幕上面的输出信息， 因为如果配置文件写错的话，屏幕上会显示出错误的地方喔！

此外，如果你想要增加一些 NFS 服务器的数据一致性功能时，可能需要用到 rpc.lockd 及 rpc.statd 等 RPC 服务， 那么或许你可以增加一个服务，那就是 nfslock 啰！启动之后，请赶快到 /var/log/messages 里面看看有没有被正确的启动呢？
```bash
[root@www ~]# tail /var/log/messages
Jul 27 17:10:39 www kernel: Installing knfsd (copyright (C) 1996 okir@monad.swb.de).
Jul 27 17:10:54 www kernel: NFSD: Using /var/lib/nfs/v4recovery as the NFSv4 state 
recovery directory
Jul 27 17:10:54 www kernel: NFSD: starting 90-second grace period
Jul 27 17:11:32 www rpc.statd[3689]: Version 1.2.2 starting
```
在确认启动没有问题之后，接下来我们来瞧一瞧那么 NFS 到底开了哪些埠口？
```bash
[root@www ~]# netstat -tulnp| grep -E '(rpc|nfs)'
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address  Foreign Address  State   PID/Program name
tcp        0      0 0.0.0.0:875    0.0.0.0:*        LISTEN  3631/rpc.rquotad
tcp        0      0 0.0.0.0:111    0.0.0.0:*        LISTEN  3601/rpcbind
tcp        0      0 0.0.0.0:48470  0.0.0.0:*        LISTEN  3647/rpc.mountd
tcp        0      0 0.0.0.0:59967  0.0.0.0:*        LISTEN  3689/rpc.statd
tcp        0      0 0.0.0.0:2049   0.0.0.0:*        LISTEN  -
udp        0      0 0.0.0.0:875    0.0.0.0:*                3631/rpc.rquotad
udp        0      0 0.0.0.0:111    0.0.0.0:*                3601/rpcbind
udp        0      0 0.0.0.0:897    0.0.0.0:*                3689/rpc.statd
udp        0      0 0.0.0.0:46611  0.0.0.0:*                3647/rpc.mountd
udp        0      0 0.0.0.0:808    0.0.0.0:*                3601/rpcbind
udp        0      0 0.0.0.0:46011  0.0.0.0:*                3689/rpc.statd
```
注意看到上面喔！总共产生了好多的 port 喔！真是可怕！不过主要的埠口是：
rpcbind 启动的 port 在 111 ，同时启动在 UDP 与 TCP；
nfs 本身的服务启动在 port 2049 上头！
其他 rpc.* 服务启动的 port 则是随机产生的，因此需向 port 111 注册。

好了，那我怎么知道每个 RPC 服务的注册状况？没关系，你可以使用 rpcinfo 来观察的。
```bash
[root@www ~]# rpcinfo -p [IP|hostname]
[root@www ~]# rpcinfo -t|-u  IP|hostname 程序名称
选项与参数：
-p ：针对某 IP (未写则预设为本机) 显示出所有的 port 与 porgram 的信息；
-t ：针对某主机的某支程序检查其 TCP 封包所在的软件版本；
-u ：针对某主机的某支程序检查其 UDP 封包所在的软件版本；

# 1. 显示出目前这部主机的 RPC 状态
[root@www ~]# rpcinfo -p localhost
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100011    1   udp    875  rquotad
    100011    2   udp    875  rquotad
    100011    1   tcp    875  rquotad
    100011    2   tcp    875  rquotad
    100003    2   tcp   2049  nfs
....(底下省略)....
# 程序代号 NFS版本 封包类型 埠口  服务名称

# 2. 针对 nfs 这个程序检查其相关的软件版本信息 (仅察看 TCP 封包)
[root@www ~]# rpcinfo -t localhost nfs
program 100003 version 2 ready and waiting
program 100003 version 3 ready and waiting
program 100003 version 4 ready and waiting
# 可发现提供 nfs 的版本共有三种，分别是 2, 3, 4 版呦！

仔细瞧瞧，上面出现的信息当中除了程序名称与埠口的对应可以与 netstat -tlunp 输出的结果作比对之外，还需要注意到 NFS 的版本支持！新的 NFS 版本传输速度较快，由上表看起来，我们的 NFS 至少支持到第 4 版，应该还算合理啦！ ^_^！ 如果你的 rpcinfo 无法输出，那就表示注册的数据有问题啦！可能需要重新启动 rpcbind 与 nfs 喔！
```


# 观察


在你的 NFS 服务器设定妥当之后，我们可以在 server 端先自我测试一下是否可以联机喔！就是利用 showmount 这个指令来查阅！
```bash
[root@www ~]# showmount [-ae] [hostname|IP]
选项与参数：
-a ：显示目前主机与客户端的 NFS 联机分享的状态；
-e ：显示某部主机的 /etc/exports 所分享的目录数据。

# 1. 请显示出刚刚我们所设定好的相关 exports 分享目录信息
[root@www ~]# showmount -e localhost
Export list for localhost:
/tmp         *
/home/linux  *.centos.vbird
/home/test   192.168.100.10
/home/public (everyone)
```
很简单吧！所以，当你要扫瞄某一部主机他提供的 NFS 分享的目录时，就使用 showmount -e IP (或hostname) 即可！非常的方便吧！这也是 NFS client 端最常用的指令喔！ 另外， NFS 关于目录权限设定的数据非常之多！在 /etc/exports 只是比较特别的权限参数而已，还有很多预设参数呢！ 这些预设参数在哪？我们可以检查一下 /var/lib/nfs/etab 就知道了！
```bash
[root@www ~]# tail /var/lib/nfs/etab
/home/public    192.168.100.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,
no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)
# 上面是同一行，可以看出除了 rw, sync, root_squash 等等，
# 其实还有 anonuid 及 anongid 等等的设定！
```
上面仅仅是一个小范例，透过分析 anonuid=65534 对比 /etc/passwd 后，会发现 CentOS 出现的是 nfsnobody 啦！这个账号在不同的版本都可能会不一样的！另外，如果有其他客户端挂载了你的 NFS 文件系统时，那么该客户端与文件系统信息就会被记录到 /var/lib/nfs/xtab 里头去的！

另外，如果你想要重新处理 /etc/exports 档案，当重新设定完 /etc/exports 后需不需要重新启动 nfs ？ 不需要啦！如果重新启动 nfs 的话，要得再向 RPC 注册！很麻烦～这个时候我们可以透过 exportfs 这个指令来帮忙喔！
```bash
[root@www ~]# exportfs [-aruv]
选项与参数：
-a ：全部挂载(或卸除) /etc/exports 档案内的设定
-r ：重新挂载 /etc/exports 里面的设定，此外，亦同步更新 /etc/exports
     及 /var/lib/nfs/xtab 的内容！
-u ：卸除某一目录
-v ：在 export 的时候，将分享的目录显示到屏幕上！

# 1. 重新挂载一次 /etc/exports 的设定
[root@www ~]# exportfs -arv
exporting 192.168.100.10:/home/test
exporting 192.168.100.0/24:/home/public
exporting *.centos.vbird:/home/linux
exporting *:/home/public
exporting *:/tmp

# 2. 将已经分享的 NFS 目录资源，通通都卸除
[root@www ~]# exportfs -auv
# 这时如果你再使用 showmount -e localhost 就会看不到任何资源了！
```
要熟悉一下这个指令的用法喔！这样一来，就可以直接重新 exportfs 我们的记录在 /etc/exports 的目录数据啰！但是要特别留意，如果你仅有处理配置文件，但并没有相对应的目录 (/home/public 等目录) 可以提供使用啊！ 那可能会出现一些警告讯息喔！所以记得要建立分享的目录才对！

# 安全性
安全性

在 NFS 的安全性上面，有些地方是你必须要知道的喔！底下我们分别来谈一谈：

- 防火墙的设定问题与解决方案：

一般来说， NFS 的服务仅会对内部网域开放，不会对因特网开放的。然而，如果你有特殊需求的话， 那么也可能会跨不同网域就是了。但是，NFS 的防火墙特别难搞，为什么呢？因为除了固定的 port 111, 2049 之外， 还有很多不固定的埠口是由 rpc.mountd, rpc.rquotad 等服务所开启的，所以，你的 iptables 就很难设定规则！ 那怎办？难道整个防火墙机制都要取消才可以？

为了解决这个问题， CentOS 6.x 有提供一个固定特定 NFS 服务的埠口配置文件，那就是 /etc/sysconfig/nfs 啦！ 你在这个档案里面就能够指定特定的埠口，这样每次启动 nfs 时，相关服务启动的埠口就会固定，如此一来， 我们就能够设定正确的防火墙啰！这个配置文件内容很多，绝大部分的数据你都不要去更改，只要改跟 PORT 这个关键词有关的数据即可。 那么需要更改的 rpc 服务有哪些呢？主要有 mountd, rquotad, nlockmgr 这三个，所以你应该要这样改：
```bash
[root@www ~]# vim /etc/sysconfig/nfs
RQUOTAD_PORT=1001   <==约在 13 行左右
LOCKD_TCPPORT=30001 <==约在 21 行左右
LOCKD_UDPPORT=30001 <==约在 23 行左右
MOUNTD_PORT=1002    <==约在 41 行左右
# 记得设定值最左边的批注服务要拿掉之外，埠口的值你也可以自行决定。

[root@www ~]# /etc/init.d/nfs restart
[root@www ~]# rpcinfo -p | grep -E '(rquota|mount|nlock)'
    100011    2   udp   1001  rquotad
    100011    2   tcp   1001  rquotad
    100021    4   udp  30001  nlockmgr
    100021    4   tcp  30001  nlockmgr
    100005    3   udp   1002  mountd
    100005    3   tcp   1002  mountd
# 上述的输出数据已经被鸟哥汇整过了，没用到的埠口先挪掉了啦！
```
很可怕吧！如果想要开放 NFS 给别的网域的朋友使用，又不想要让对方拥有其他服务的登入功能， 那你的防火墙就得要开放上述的十个埠口啦！有够麻烦的～假设你想要开放 120.114.140.0/24 这个网域的人能够使用你这部服务器的 NFS 的资源，且假设你已经使用第九章提供的防火墙脚本， 那么你还得要这样做才能够针对该网域放行喔：
```bash
[root@www ~]# vim /usr/local/virus/iptables/iptables.allow
iptables -A INPUT -i $EXTIF -p tcp -s 120.114.140.0/24 -m multiport \
         --dport 111,2049,1001,1002,30001 -j ACCEPT
iptables -A INPUT -i $EXTIF -p udp -s 120.114.140.0/24 -m multiport \
         --dport 111,2049,1001,1002,30001 -j ACCEPT

[root@www ~]# /usr/local/virus/iptables/iptables.rule
# 总是要重新执行这样防火墙规则才会顺利的生效啊！别忘记！别忘记！
```

- 使用 /etc/exports 设定更安全的权限：

这就牵涉到你的逻辑思考了！怎么设定都没有关系，但是在『便利』与『安全』之间，要找到你的平衡点吶！善用 root_squash 及 all_squash 等功能，再利用 anonuid 等等的设定来规范登入你主机的用户身份！应该还是有办法提供一个较为安全的 NFS 服务器的！

另外，当然啦，你的 NFS 服务器的文件系统之权限设定也需要很留意！ 不要随便设定成为 -rwxrwxrwx ，这样会造成你的系统『很大的困扰』的啊！

- 更安全的 partition 规划：

如果你的工作环境中，具有多部的 Linux 主机，并且预计彼此分享出目录时，那么在安装 Linux 的时候，最好就可以规划出一块 partition 作为预留之用。因为『 NFS 可以针对目录来分享』，因此，你可以将预留的 partition 挂载在任何一个挂载点，再将该挂载点 (就是目录啦！)由 /etc/exports 的设定中分享出去，那么整个工作环境中的其他 Linux 主机就可以使用该 NFS 服务器的那块预留的 partition 了！所以，在主机的规划上面，主要需要留意的只有 partition 而已。此外，由于分享的 partition 可能较容易被入侵，最好可以针对该 partition 设定比较严格的参数在 /etc/fstab 当中喔！

此外，如果你的分割做的不够好，举例来说，很多人都喜欢使用懒人分割法，亦即整个系统中只有一个根目录的 partition 而已。这样做会有什么问题呢？假设你分享的是 /home 这个给一般用户的目录好了，有些用户觉得这个 NFS 的磁盘太好用了， 结果使用者就将他的一大堆暂存数据通通塞进这个 NFS 磁盘中。想一想，如果整个根目录就因为这个 /home 被塞爆了， 那么你的系统将会造成无法读写的困扰。因此，一个良好的分割规划，或者是利用磁盘配额来限制还是很重要的工作。

- NFS 服务器关机前的注意事项：

需要注意的是，由于 NFS 使用的这个 RPC 服务，当客户端连上服务器时，那么你的服务器想要关机， 那可就会成为『不可能的任务』！如果你的服务器上面还有客户端在联机，那么你要关机， 可能得要等到数个钟头才能够正常的关机成功！嗄！真的假的！不相信吗？不然你自个儿试试看！^_^！

所以啰，建议你的 NFS Server 想要关机之前，能先『关掉 rpcbind 与 nfs 』这两个东西！ 如果无法正确的将这两个 daemons 关掉，那么先以 netstat -utlp 找出 PID ，然后以 kill 将他关掉先！这样才有办法正常的关机成功喔！这个请特别特别的注意呢！

当然啦，你也可以利用 showmount -a localhost 来查出来那个客户端还在联机？ 或者是查阅 /var/lib/nfs/rmtab 或 xtab 等档案来检查亦可。找到这些客户端后， 可以直接 call 他们啊！让他们能够帮帮忙先！ ^_^

事实上，客户端以 NFS 联机到服务器端时，如果他们可以下达一些比较不那么『硬』的挂载参数时， 就能够减少这方面的问题喔！相关的安全性可以参考下一小节的 客户端可处理的挂载参数与开机挂载。


