---
layout: post
title: "Photon札记：可靠Udp、ChannelId、Queue Warning"
category: Programming
tags: [Photon]
---
{% include JB/setup %}

使用<a href="https://www.exitgames.com/" target="_blank">Photon Server</a>也有一段时间了。当时了解到Photon的时候简直就觉得是为我这类人量身定做的~

- Unity3D, C# 用户。
- 业余开发者(晚上，周末)，没很多的时间从零开始写一个服务器。
- 提供免费的学习licence.
- 基于C++，性能也经过大型游戏的检验。
- 完善的教程例子，文档以及技术支持。

这篇日志是使用Photon遇到的一些问题，记录下来方便以后查阅。
<!--break-->

<br />
## Tcp 还是 Udp?

Tcp 和 Udp是我最先遇到的一个选择，Photon都支持，这个选择并不难，并没有太多犹豫地选择了Udp, 原因是我要做的是注重实时性的网络游戏。正如<a href="http://gafferongames.com/networking-for-game-programmers/udp-vs-tcp/" target="_blank">Udp vs Tcp</a>一文所说，网游可以容忍某些普通包的丢弃，但等待包的重发带来的延迟却会使玩家抓狂~ Photon Team的Kaiserludi也说：

	TCP is only implemented as a fallback solution in Photon in case, that one can't use UDP for some reason. 

但是单纯的Udp又会引发另一个问题，可靠性。考虑以下两个场景：

<div id="case-one">
第一，玩家的位置同步，位置信息每0.1s更新一次到服务器端然后分发到其他客户端，其他客户端根据接收到的包来预测某玩家下0.1s的移动，并在地图上体现。在这里位置的更新包可以认为是当网络环境恶劣时可丢弃的包。如果中间丢弃了其中一个更新包，则客户端要根据上一次接受的数据包来预测某玩家下0.2s的移动，然后在0.3秒接收到一个更新的数据包进行同步。可以看出，这一次的丢包对用户体验影响是极小的。
</div>

第二，交易系统，整个交易系统在开始之后应该全程可靠，不能出现一个玩家付了钱，但却没收到物品的情况。这就需要Udp提供可靠性的保证。

所幸的是这些Photon都为用户考虑好了。相对于普通的Udp，Photon的可靠Udp会在发送端和接收端建立一个连接。一个Udp包含有一个序号和一个标志符表示是否为可靠传输。如果是，那么接收端必须响应这条命令。可靠的命令会以很短的间隔循环发送，直到收到响应为止，如果没收到则视为超时。Photon中的超时机制是客户端和服务端独立的。
服务器端的阀值在`PhotonServer.config`中设置：
`MinimumTimeout="5000"`
`MaximumTimeout="30000"`
客户端则是通过`PhotonPeer.DisconnectTimeout`属性来设置。

可靠Udp效率相比Tcp除了包头大小之外，没特别的优势，使用可靠Udp主要是为了灵活性，由用户去决定某个包重要与否。

<br />
## Channel Id

Photon中的Channel是一个优先级机制，它具有以下几个特点：

1. 只会在Udp中起作用，因为Tcp所有的命令都是可靠有序的传输，不存在优先级之说。
2. 默认的Channel数是`2`，用户可以通过`PhotonPeer.ChannelCount`属性修改其值。
3. Channel Id越小，优先级越高。默认所有命令都会在Channel 0中发送，除了Channel -1是Photon保留内部使用外，Channel 0具有最高优先级。
4. 同一Channel中的可靠命令以及不可靠命令都会各自有序的到达接收端，但可靠命令有可能比不可靠命令要早。
5. 高优先级的Channel中的命令可能会比低优先级Channel中的命令提前到达。
6. 每个Channel中有两组队列，一组是`incomingQueue`， 另一组`outgoingQueue`，每组分别有可靠的和不可靠的两种。

当客户端每次调用`PhotonPeer.Service`或者`PhotonPeer.SendOutgoingCommands`时，Photon内部是按照以下顺序来构造一个包的:

1. 先取出Acks(可靠命令的响应包)
2. "Ping"命令，只会当过去`1`秒内没有可靠命令的传输时才会有
3. 从最低数字的Channel开始:  
	3.1 取出Channel中的可靠命令  
	3.2 取出Channel中的不可靠命令
4. 直到到达一个包的容量

这是使得小点的Channel有更高优先级的原理。

Channel的概念令得Photon的Udp更为灵活，考虑以下场景：

玩家的移动命令是不可靠的`Unreliable`，优先级高的`ChannelId = 0`。聊天命令是可靠的`Reliable`，优先级低的`ChannelId = 1`。 这样的设置有效的保证了，当聊天命令产生丢包要重传时，对玩家的移动不会造成任何影响，同时也保证了玩家的聊天信息最终会被处理。

<br />
## 奇怪的QueueOutgoingAcksWarning

最近，练习项目的客户端总会无端端的报出`QueueOutgoingAcksWarning`这个warning，但也是偶尔在Editor模式下会出现，编译出来后基本没事，也就没什么在意。但作为一个完美主义者，这始终是一条刺(It just killing me...)
不看不知道，一看吓一跳。这虽然是个warning，一般情况下不会影响游戏的运行，除非用户自己的代码对`UnExceptedStatusCode`作了特殊处理，因为这个warning会调用`OnStatusChanged`方法把`StatusCode`置为`StatusCode.QueueOutgoingAcksWarning`。但是，一旦出现了这个问题，则意味着你应该对你的游戏设计好好的反省一下。

这个warning的意思是：发送队列里已经堆积了太多的对可靠命令的响应。原因有两个，一个可能是发送端发送了太多了可靠命令，接收端来不及响应。另一个是接收端调用`PhotonPeer.Service`或者`PhotonPeer.SendOutgoingCommands`不够频繁。
而我的原因是第一个，场景如下：(半年前傻傻分不清想出来的做法，没有参考很多前人的经验，居然还觉得还好，到今天真心觉得是团shi啊=v=)

还是玩家在地图上同步。直接传坐标，接收端接收到之后没作任何预测插值，直接跳到接收到的坐标。这样做，首要问题就是移动不连续，于是靠提高发送的频率来获得更精准的移动(从10 updates/s到50 updateds/s). 问题似乎解决了，但其实是更严重问题的开始。过度频繁的更新，一个会加重服务器端的压力， 其次会耗尽客户端的资源。于是就有了以上的[实现](#case-one)。也算拨乱反正(值得鄙视的错误一一)

这个问题不能小看，如果频繁出现这样的warning，那么超时就不远了~ 类似的问题还有：

- **QueueOutgoingReliableWarning**  
  发送队列中可靠命令堆积过多。

- **QueueOutgoingUnreliableWarning**  
  发送队列中不可靠命令堆积过多。

- **QueueOutgoingAcksWarning**  
  发送队列里对可靠命令的响应堆积过多。

- **QueueIncomingReliableWarning**  
  接收队列中可靠命令堆积过多。

- **QueueIncomingUnreliableWarning**  
  接收队列中不可靠命令堆积过多。

所有以上的warning出现的阀值都是`100`。

在Photon的Unity3D SDK中，提供了一个脚本<a href="http://doc.exitgames.com/photon-cloud/PhotonStatsGui/" target="_blank">PhotonStatsGui</a>可以让用户很好的观察当前队列的状态。

<img src="/assets/custom/images/posts/PhotonStatsGui.jpg" />

- **Out|In|Sum**  
  包括了operation， response以及event的计数，总数和平均值。平均值的大小可以一定程度的反应队列的大小。

- **TotalPacketBytes**  
  共发送(接收)包的大小。 `TotalPacketBytes = TotalCommandBytes + TotalPacketCount * PackageHeaderSize`

- **TotalCommandBytes**  
  共发送(接收)命令的大小。包括`ReliableCommandBytes`可靠命令大小， `UnreliableCommandBytes`不可靠命令的大小， `FragmentCommandBytes`分片命令的大小，`ControlCommandBytes`Photon控制命令(Connect, Disconnect, etc.)的大小。

- **TotalPacketCount**  
  共发送(接收)包的数目。

- **TotalCommandsInPackets**  
  共发送(接收)命令的数目。一个包最少可以包含一个命令的分片，最多可以包含n个命令，一个包的最大容量为`1200byte`.

- **Longest delta between sent disp**  
  这是纯客户端的性能参数，表示两次连续调用的`PhotonPeer.Service`，`PhotonPeer.DispatchIncomingCommands` 或者`PhotonPeer.SendOutgoingCommands` 之间的最长间隔。 客户端调用这些方法的频繁度是由程序员自己控制的，假设都设为100ms。那么这两个值在100-150ms之间是正常的，如果这个值变得很高甚至超过一秒，那就应该找出到底是什么原因导致这些调用产生延迟。

- **Longest time for**  
  表示的是客户端反序列化event或者operation所需的最长时间，以及对应event code和operation code. 

<br />
## Conclusion

1. 可以使用Udp的时候坚决使用Udp，除非没得选(Flash, Websocket)
2. 合理利用Channel来安排命令的优先级。
3. 注意生产和消费的平衡，客户端每发起一次请求都有可能会对其他客户端产生一次事件，当玩家越来越多，客户端接收到的事件也会剧增，最坏的情况是: <a href="http://doc.exitgames.com/photon-server/PerformanceTips/" target="_blank">One side produces so much data that it breaks the receiving end.</a>

<br />
