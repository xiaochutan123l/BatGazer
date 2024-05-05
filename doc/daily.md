## Motivation:
最近在焊接LGA芯片的时候，发现怎么都焊不好，由于引脚都在焊盘底下，所以没发直接或间接测试特定引脚焊接情况。我知道像LGA、BGA这种芯片，没法通过肉眼判断引脚是否成功焊接，所以常用的方式是X扫描成像。中所周知这种设备体积巨大而且关键是价格昂贵，不是个人diy能享受的服务。
第一想法是diy一个低成本的小型x光扫描仪，但是考虑到这东西有辐射对身体有危害，万一在研究过程中没注意防护可能就会影响生命，所以果断放弃了这个想法。

在研究替代方案的时候，发现超声波也具有穿透性，而且比x光好的是可以可以无损检测。故将目标换成了diy小型低成本超声波扫描显微镜。

这个文件将记录每次的研究和实践的进展。

### 2024年5月3日

- 在网上搜寻已有方案，总结如下

超声波不仅可以测距，还可以穿透物体，根据物体内部不同材质反射情况不同可以判断出物体内部的情况。 
超声波频率越高探测精度越高，但是探测深度越低，由于我只需要探测芯片焊接情况所以探测深度很低
超声波频率越低穿透性越好，但是精度就越低，适合精度要求不高但是探测深度深的物体

应用
- B超
- 探伤
- 工业检测

扩展：像B超探测目标是动态的，涉及到多普勒效应，一般都是伪彩影。 B超需要实时显示，相当于拍视频，对数据传输要求比较高，但我的目标是检测静态芯片，相当于拍照，成像时间比较充裕

实现方式
单个探头无阵列
1. 传感器，超声波的发生器和接收器是一样的，都是换能片，原理是压电效应，通电使换能片高频震动发出超声波，接受的时候接受声波导致震动产生电压
2. 放大器，放大传感器数值
3. 高进度adc，接受传感器数值并转化为数字信号
3. 控制器，超声波频率，接受和处理传感器数据
4. 成像，将数据可视化

- 开源项目：
1. The Murgen open-source ultrasound imaging dev-kit： https://github.com/kelu124/murgen-dev-kit/tree/master
2. https://www.eet-china.com/mp/a31917.html 原始项目网页没找到  

### 2024年5月4日

今天从HY-SRF05模块上拆下来两个换能器，进行了简单的测试，在示波器上成功抓到了发射端和接收端的信号
- 发射端，用Arduino板连接HY-SRF05模块进行发射，程序设置的500ms发射一次，示波器直连模块上发射侧换能器的引脚，采集到多组40khz的连续8个方波
- 接收端：拆下的换能器并联100欧电阻，换能器近距离大概10厘米对准发射端，示波器连接电阻两端，采集到周期性的脉冲，振幅大概是噪音的两倍达到25mV左右，频率和发射端一致（500ms间隔），放大脉冲发现是8个40khz的连续脉冲，和理论一致。

### 2024年5月5日

阅读和研究Murgen开源方案
这是文档里写的work flow：
1. A pulse generator generates a high negative voltage pulse, which is as low as possible and to excite the transducer (or crystal), which will resonate at his central frequency. In our case, this is 3.5MHz, but that depends on the transducer being used.
他们用的3.5mhz频率，属于高频了，在我的项目里应该够用，我可以现在低频一点的发射器上测试，比如我拆下来的，最佳频率是40khz，之后肯定要换成高频的发射器。

2. A signal will come back through the echoes, the reverberation of the acoustic waves on "obstacles", exciting the transducer, having it generate an electric signal.
接收超声波回声（多个回声，对应不同位置的物体或表面的反射，这也是穿透之后成像的依据）

3. This will be prepared by a switch. Indeed, the signal needs to be separated from the excitation. Excitation being at -150V, the signal coming back will be in mV - and it needs to be isolated.
没太懂这里的switch具体值得什么，可能是说使用开关将接收到的回声信号与激发脉冲分离。总之这里不是一个具体步骤，但是提到了需要把有效信号分离出来，大概是说要从回声中分理处各个反射的信号。

4. The signal usually goes through a LNA - low noise amplifier - to be amplified
放大信号，没啥好说的，提到了低噪声这个关键词，之后可能会是一个重点。

5. then it's processed by a TGC to compensate for signal attenuation. Indeed, the deeper the echoes come back from, the more attenuated they are, and this needs to be corrected.
TGC 调整信号以补偿随着回声源深度增加而发生的衰减。更深层的回声较弱，TGC 修正这种信号强度的损失。
等比例地放大超声波信号,不会损失有效信息、

6. An AAF (Anti-Alias Filte) can be put in between, and in the case of the first Murgen release, we also include an enveloppe detection: indeed, echoes are signals that are coming from a couple of ultrasound periods reflected on interfaces, and what interests us is not (at first) the signal itself, rather the enveloppe of it.
抗混叠滤波器 (AAF)：可以使用AAF来平滑信号并去除可能干扰信号分析的高频成分。
包络检测：在Murgen项目的第一版中，还包括了包络检测。由于超声波回声本质上是几个超声波周期的反射，分析这些信号的包络而不是原始信号本身是关注的重点。
对超声波信号进行包络Envelop处理（通常涉及整流和低通滤波步骤），我们可以更清晰地看到回声信号的振幅变化。这种振幅信息是分析被扫描物体内部结构的关键。

7. The "cleaned" signal goes through an ADC to be digitalized
将干净的信号数字化，用的ADC。

8. this is output to buffers, in our case to protect the BBB GPIOs.
BBB- BeagleBone Black. 数字信号输出到buffer。


B超需要实时显示，相当于拍视频，对数据传输要求比较高，这也是Murgen项目的瓶颈所在。但我的目标是检测静态芯片，相当于拍照，成像时间比较充裕，我可以探测一个点完成后再移动到下一个点再探测，最后汇总传给上位机就行，我可以花一定时间等待一张图像。

Thoughs：
- 滤波应该可以带通，毕竟不管是发射的还是反射的超声波信号，应该都是同样频率的。
- 超声波经过不同的材质会不同程度地被反射回来，因为每种组织的声学阻抗不同。可以作为不同材质的判断依据