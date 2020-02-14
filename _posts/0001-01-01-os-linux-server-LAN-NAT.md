Network Address Translation：网络地址转换  
路由器的延伸服务器

内部LAN主机的IP分享器
DMZ：非军事化隔离区

数据包通过iptables传送到后端主机的表格与链的流程：  
1. 经过NAT TABLE的PREROUTING链
2. 经由路由判断确定这个数据吧是否要进入本机，若不进入本机则下一步
3. 经过filter table 的FORWARD链
4. 通过NAT table 的POSTROUTING链，最后传送出去

重点在与修改IP
POSTROUTING链：修改srcIP ->SNAT
PREROUTING链：修改dstIP  ->DNAT


###SNAT - IP分享
假设有客户端，192.168.1.100 连接到某网站，他的数据包报头的变化：
 - 数据发送：
  1. 客户端发出的数据包报头中，来源会是192.168.1.100，然后传送到NAT这台主机
  2. NAT的接口192.168.1.2收到数据，分析报头数据，因为报头数据显示的并不是linux本机，所以开始路由分析，然后将此数据传送到Internet的public IP
  3. 由于private IP和public IP 不能互通，所以linux主机通过iptables的NAT table内的POSTROUTING链将数据包的报头的来源伪装成linxu的public IP
并且将两个不同的来源：192.168.1.100即public IP 的数据包对应写入暂存内存当中，然后将此数据包传送出去

 - 数据接收
  1. 主机收到数据包时，将响应数据传送给public IP的主机
  2. 当Linux NAT 服务器收到来自Internet的响应数据包后，分析该数据包的序号，并比对刚刚记录到内存当中的数据，
  由于发现该数据包为后端主机，即192.168.1.100，然后发现目标已经不是本机(public IP)，所以开始通过路由分析数据包流向
  3. 数据包会传送到192.168.1.2，这个内部接口，然后再传送到最终目标192.168.1.100机器上去
  
可见，所有内部LAN主机都可以通过NAT服务器连接出去，而大家在Intenet上面看到的时同一个IP(NAT服务器的public IP)
所以如果内部LAN主机没有连上不明网站，那么内部主机其实就具有一定程度的安全性，因为Internet上的其他主机没有办法主动攻击LAN内的PC
基本上，NAT服务器就是路由器，但由于它会修改IP报头数据，因此与一般的路由器有区别
IP分享器一定有一个privateIP和一个publicIP，让LAN内的privateIP可以通过IP分享器的publicIP传送出去
而路由器通常连边都是public IP或同时为private IP  


在 Linux 的 NAT 服务器服务当中，最常见的就是IP 分享器功能了。 
这个 IP 分享器的功能其实就是 SNAT！作用就只是在 iptables 内的 NAT 表格当中，那个路由后的 POSTROUTING 链进行 IP 的伪装就是了。
另外， 你也必须要了解，你的 NAT 服务器必须要有一个 public IP 接口，以及一个内部 LAN 连接的 private IP 界面才行。
底下的范例中，鸟哥的假设是这样的：
外部接口使用 eth0 ，这个接口具有 public IP 喔；
内部接口使用 eth1 ，假设这个 IP 为 192.168.100.254 ；

记住！当你利用前面几章谈到的数据来设定你的网络参数后，务必要进行路由的检测， 
因为在 NAT 服务器的设定方面，最容易出错的地方就是路由了！
尤其是在拨接产生 ppp0 这个对外接口的环境下， 这个问题最严重。
反正你要记得：『如果你的 public IP 取得的方式是拨接或 cable modem 时，
你的配置文件 /etc/sysconfig/network, ifcfg-eth0, ifcfg-eth1 等档案，
千万不要设定 GATEWAY 啦！』否则就会出现两个 default gateway ，反而会造成问题。

如果你刚刚已经下载了 iptables.rule ，那么该档案内已经含有 NAT 的脚本了！ 
你可以看到该档案的第二部份关于 NAT 服务器的部分，应该有看到底下这几行：

	iptables -A INPUT -i $INIF -j ACCEPT
	# 这一行为非必要的，主要的目的是让内网 LAN 能够完全的使用 NAT 服务器资源。
	# 其中 $INIF 在本例中为 eth1 接口
	echo "1" > /proc/sys/net/ipv4/ip_forward
	# 上头这一行则是在让你的 Linux 具有 router 的能力
	iptables -t nat -A POSTROUTING -s $innet -o $EXTIF -j MASQUERADE
	# 这一行最关键！就是加入 nat table 封包伪装！本例中 $innet 是 192.168.100.0/24
	# 而 $EXTIF 则是对外界面，本例中为 eth0 

重点在那个『 MASQUERADE 』！这个设定值就是『 IP 伪装成为封包出去 (-o) 的那块装置上的 IP 』！
以上面的例子来说，就是 $EXTIF ，也就是 eth0 啦！
所以封包来源只要来自 $innet (也就是内部 LAN 的其他主机) ，只要该封包可透过 eth0 传送出去， 
那就会自动的修改 IP 的来源表头成为 eth0 的 public IP 啦！就这么简单！ 
你只要将 iptables.rule 下载后，并设定好你的内、外网络接口， 执行 iptables.rule 后，
你的 Linux 就拥有主机防火墙以及 NAT 服务器的功能了！


###DNAT
SNAT用于内部LAN连接到Internet，DNAT用在为内部主机架设可以让Internet访问的服务器，类似DMZ内的服务器
如上图所示，假设我的内部主机 192.168.1.210 启动了 WWW 服务，这个服务的 port 开启在 port 80 ， 
那么 Internet 上面的主机 (61.xx.xx.xx) 要如何连接到我的内部服务器呢？
当然啦， 还是得要透过 Linux NAT 服务器嘛！所以这部 Internet 上面的机器必须要连接到我们的 NAT 的 public IP 才行。

1. 外部主机想要连接到目的端的 WWW 服务，则必须要连接到我们的 NAT 服务器上头；
2. 我们的 NAT 服务器已经设定好要分析出 port 80 的封包，所以当 NAT 服务器接到这个封包后， 会将目标 IP 由 public IP 改成 192.168.1.210 ，且将该封包相关信息记录下来，等待内部服务器的响应；
3. 上述的封包在经过路由后，来到 private 接口处，然后透过内部的 LAN 传送到 192.168.1.210 上头！
4. 192.186.1.210 会响应数据给 61.xx.xx.xx ，这个回应当然会传送到 192.168.1.2 上头去；
5. 过路由判断后，来到 NAT Postrouting 的链，然后透过刚刚第二步骤的记录，将来源 IP 由 192.168.1.210 改为 public IP 后，就可以传送出去了！

其实整个步骤几乎就等于 SNAT 的反向传送！这就是 DNAT 









