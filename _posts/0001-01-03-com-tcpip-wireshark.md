ץ����������
ether host b8:f8:83:f1:7c:15       //ץȡmac��ַ��
ether src host 00:88:ca:86:f8:0d   //ץȡmac��ַ��
ether dst host 00:88:ca:86:f8:0d   //ץȡmac��ַ��

host 192.168.1.1                   //ץȡIP��ַ
src host 192.168.1.1               //ץȡIP��ַ
dst host 192.168.1.1               //ץȡIP��ַ

port 80                            //ץȡ�˿�
!port 80                           //ץȡ�˿�
dst port 80                        //ץȡ�˿�
src port 80                        //ץȡ�˿�

src host 192.168.1.1 && dst port 80  ������


��ʾ��������
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