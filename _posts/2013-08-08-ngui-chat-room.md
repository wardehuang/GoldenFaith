---
layout: post
title: "NGUI Chat Room"
category: Practice
tags: [Unity3d, NGUI]
---
{% include JB/setup %}

NGUI是接触的最早的一个unity插件，在我刚开始学unity的时候，NGUI已经很成熟了貌似是2.2.2版（够2），可能也和我学unity比较晚有关系~ 这也导致了我对内置的GUI异常不熟悉=v= 这篇文章不是技术文章，更不是NGUI教程。一来NGUI的教程已经够多，再来NGUI本身不算难上手，最后文笔太差。。。

看完了官方的视频后，就跟着官方的教程一个场景一个场景的过~ 当过到聊天框那个的时候，心里暗叫了一声（卧槽。。。） 在学NGUI之初，就已经定好一个目标用作检验自己的成果，想的是一个游戏的聊天框，因此在看到自带的聊天框那渣功能的时候不免吐一下槽~~~
<!--break-->

预想中的聊天框是具有以下功能的：  
1. 表情  
2. 支持将背包中的道具输入到聊天框展示  
3. 将玩家输入到聊天框  
4. 将坐标输入到聊天框  
5. 点击道具/玩家/坐标的超链接能得到相应的回馈  
6. Ctrl-C/V (这个没实现出来啦= =) <a href="http://www.tasharen.com/forum/index.php?topic=27.0" target="_blank">作者也没办法啦>,<</a>  
7. 多频道  
8. 滚屏/锁屏/清屏  

可以看出都是些很基础的功能。 高清无码预览图

<img src="/assets/custom/images/posts/NGUIChatRoomScreenshot.jpg" />

首先是`表情`。 NGUI2.05开始就支持表情，<a href="http://www.youtube.com/watch?v=JbqfK3mU140">猛击</a>。 看完视频后，作为一个敲惯代码的农民表示比较难以接受，基本都是手动的活。 我理想中的实现是，只要表情sprite的名字是有规律的，写代码循环生成就可以。 某个更新后，这个过程变得方便了点，直接在Inspector中就可以添加表情，但还没方便到可以满足我这种懒人。

道具展示。 就是把道具贴到聊天框，其他玩家可以查看。

玩家信息。 就是点击玩家名字，可以私聊啊啥啥的。

坐标信息。 就是点击了能寻路~ 上面三个都是一个超链接的功能而已。

多频道。 就是相应的消息只发到相应的频道只有相应的玩家能看到~

很早前就已经实现了，现在决定把它放上来，因此作了点改动，现在它是个聊天室，支持20个人在线， 服务器搭建在<a href="http://cloud.exitgames.com/" >Photon Cloud</a>上。 动态字体支持后，马上就改了（恨透了bitmap字体，特别是中文）

WebPlayer链接：<a href="http://edwardwong.github.io/Asset/ChatRoom.html">NGUI Chat Room</a>

注意： 
等待"登录"按钮出现后，随意输入你的大名就可登录了~ 如果"登录"按钮没出现，请F5，有时服务器会抽风- =!!

<img src="/assets/custom/images/posts/JoinedLobby.jpg" />

<br />