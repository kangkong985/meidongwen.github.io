理想情况：无线设备 = 内部电路 + 天线 ，其中，内部电路和天线都是50Ω匹配电阻   
但如果内部电路或天线不满足50Ω电阻匹配，则需要加匹配网络进行纠正：  
单独取出内部电路或天线，测量链接点的阻抗，显然不在smith chart的中心  
按顺序增加以下电路：  
1. 并联电容或电阻  
2. 串联电容或电阻  
3. 并联电容或电阻  
然后引出新的链接点。

案例：  
lora-module产品在实验室测试发现二倍频超标，  
1. 天线采用螺旋弹簧天线  
2. 内部电路假设已做好50阻抗匹配  
3. 天线与内部电路之间有一个PI型匹配电路  
我们打算保留天线和内部电路不变，仅修改PI型匹配电路的参数，来减少二倍频超标  
即换算到simith chart时，470MHZ的点尽量靠近圆心，940MHZ的点处于圆图的边缘
使用罗德施瓦茨的ZNB网络分析仪  
断开内部电路和天线  
单独取出天线，测量阻抗ZL  
若是在电路板上测量。测量线的屏蔽线接电路板的射频地

我是使用ZV-Z135进行校准的，测量时机器接测量电缆，但是测量电缆没法直接接到电路板上的，所以我是先将测量电缆接一根我自己做的测量线，测量线另一端再焊到我的电路板的上，但是我校准时是只有测量电缆的，所以测出来的结果是不准的，即多了那根测量线，但是测量线又没办法接在测量电缆上一起校准，因为测量线的另一端是裸线需要焊接到电路板上的
解决办法：使用ZNB校准完毕后，连接那根测量线，在使用offset embeded，是ZNB内置的补偿功能，然后再将测量线接入电路板进行测量。

使用simith软件时，加入选择PI型匹配电路的元件时，选择的元件应该处于，变化速度缓慢平缓的区域。



Auto Length and tools