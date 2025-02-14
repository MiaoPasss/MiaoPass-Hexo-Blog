---
title: TCP/IP 五层网络模型
date: 2025-02-12 17:26:08
categories: 编程笔记
tags:
- 网络模型
---

五层体系结构包括：应用层、运输层、网络层、数据链路层和物理层。 
五层协议只是OSI和TCP/IP的综合，实际应用还是TCP/IP的四层结构。为了方便可以把下两层称为网络接口层。

---

# TCP/IP网络模型各层协议功能

#### 1. 应用层
应用层是网络通信的最高层，它定义了应用程序和网络之间的接口。在这一层，用户可以直接与应用程序进行交互。常见的应用层协议有HTTP、FTP、SMTP等。

- **Proxy** 
A proxy server (forward proxy) takes requests from a client and forwards them to the internet, a reverse proxy takes requests from the internet and forwards them to a server.

#### 2. 传输层
传输层负责在源主机和目标主机之间建立数据传输通道。它提供了可靠的数据传输服务，确保数据的正确传输顺序和可靠性。TCP协议就是传输层协议的一种，它提供了可靠的、面向连接的数据传输服务。

#### 3. 网络层
网络层负责在网络上寻址和路由数据包。它定义了数据在网络中的传输路径，使得数据可以从源主机传输到目标主机。常见的网络层协议有IP协议。

#### 4. 数据链路层
数据链路层协议负责将网络层传输的数据分组封装成帧，传输到物理层，并通过物理介质进行传输。它负责数据的分段和重新组装，以及物理介质的访问控制。常见的数据链路层协议有以太网协议。

- **以太网帧**
![](ethernet-frame-format.png)

在以太网帧中，*Destination Address(目的地址)*放在最前面。接收方收到一个以太网帧后，最先处理*Destination Address*字段。如果发现该帧不是发给自己的，后面的字段以及数据就不需要处理了。

什么是帧间距（IFG）

网络设备和组件在接收一个帧之后，需要一段短暂的时间来恢复并为接收下一帧.做准备互联网帧间隙共20字节，包括：
- 以太网最小帧间隙 12Byte
- 数据链路层帧 前导码 7Byte，用于时钟同步 This is a sequence of alternate 0s and 1s that denotes the beginning of the frame and enables bit synchronization between the sender and receiver.
- 帧开始标识 1Byte （标识帧的开始）

#### 5. 物理层
物理层是网络通信的最底层，它负责在物理介质上传输比特流。它定义了物理连接的特性，如电压、频率等。常见的物理层介质有光纤、双绞线等。

---

#### 各层间数据传递

* 不同的协议层对数据包有不同的称谓，在传输层叫做段(segment)，在⽹络层叫做数据报 (datagram)，在链路层叫做帧(frame)。
* 应⽤层数据通过协议栈发到⽹络上时，每层协议都要加上⼀个数据⾸部(header)，称为封装 (Encapsulation)。
* ⾸部信息中包含了⼀些类似于⾸部有多⻓，载荷(payload)有多⻓，上层协议是什么等信息。
* 数据封装成帧后发到传输介质上，到达⽬的主机后每层协议再剥掉相应的⾸部，根据⾸部中的 "上层协议字段" 将数据交给对应的上层协议处理。

![图片2](NetworkPacket.png "Dissecting a Network Packet")

---

# Credits

详解TCP/IP五层网络模型：https://blog.csdn.net/2201_75437633/article/details/137373813

---