---
layout: post
title: "Autofit WebPlayer Template"
description: ""
category: Programming
tags: [Unity3d]
---
{% include JB/setup %}

Unity可以编译出网页版，个人觉得这是非常方便的，无论你做出个啥东西，都可以直接部署在自己的vps上，或放到github，dropbox等地方。你的朋友们就可以很方便的看到你的成果~ 内置支持的HTML模板有三个，略显不足，至少我想要的自适应模板没有。笔记本分辨率为1920x1080, 外置显示器为2560x1440。每次打开WebPlayer的时候总是有一大片空白，但又不能把在编译前把分辨率设死，毕竟不是所有用户的分辨率都那么高~

翻翻文档，发现Unity是支持自定义模板的<a href="http://docs.unity3d.com/Documentation/Manual/UsingWebPlayertemplates.html" target="_blank">猛击</a>。于是我就决定自己动手，丰衣足食了~
<!--break-->

首先，定好需求。  
1. 自适应多分辨率  
2. 可以设置最低分辨率（目的是不让用户把浏览器resize得过小，导致UI都挤在一起）  
3. 支持多浏览器  

需求简单且明确，先看<a href="http://edwardwong.github.io/Asset/ChatRoom.html" target="_blank">最终效果</a>。 在这个例子中我定义了最小分辨率为1024x768. 测试浏览器为： Chrome 28, FF 23, IE 10.

第一步， 下载<a href="http://edwardwong.github.io/Asset/WebPlayerTemplates.zip">模板</a>.

下载回来的是个zip包，解压后的目录结构如下：

	└─WebPlayerTemplates
    └─AutoFit
            index.html
            thumbnail.png

第二步， 把整个解压出来的`WebPlayerTemplates`文件夹copy到你的项目`Asset`文件夹下。  
<img src="/assets/custom/images/posts/AutofitWebPlayerTemplate01.jpg" />

然后打开Player Setting就可以看到，新增的模板~  
<img src="/assets/custom/images/posts/AutofitWebPlayerTemplate02.jpg" />

要注意的是： `Default Screen Width` 和 `Default Screen Height` 并不代表它们字面上的意思，而是设置一个最小分辨率。 设置值为1024x768意味着，当用户的分辨率小于1024x768，WebPlayer的大小会保持1024x768，且会出现Scrollbar。 如下图所示：  
<img src="/assets/custom/images/posts/AutofitWebPlayerTemplate03.jpg" />

当然，如果你的代码写的非常优秀，根本没有最低分辨率的限制，可以将`Default Screen Width` 和 `Default Screen Height` 设置为0x0，代表着无最小分辨率。

最后， 选着这个模板编译你自己的项目，就可以看到结果。
