抓包过滤器：
ether host b8:f8:83:f1:7c:15       //抓取mac地址：
ether src host 00:88:ca:86:f8:0d   //抓取mac地址：
ether dst host 00:88:ca:86:f8:0d   //抓取mac地址：

host 192.168.1.1                   //抓取IP地址
src host 192.168.1.1               //抓取IP地址
dst host 192.168.1.1               //抓取IP地址

port 80                            //抓取端口
!port 80                           //抓取端口
dst port 80                        //抓取端口
src port 80                        //抓取端口

src host 192.168.1.1 && dst port 80  ，复合


显示过滤器：
arp
tcp
not http
dns and udp
http or ftp


ip.addr==192.168.1.1
ip.src==192.168.0.0
ip.dst==192.168.0.0
ip.src==192.168.0.0 and ip.dst==192.168.1.1
ip.addr==192.168.1.1 and ip.addr==192.168.0.0

tcp.port==80
udp.port==4000
tcp.srcport==80
tcp.dstport==80
tcp.flags.syn==1