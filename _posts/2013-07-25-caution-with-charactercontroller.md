---
layout: post
title: "慎用Character Controller"
category: Programming
tags: [Unity3d]
---
{% include JB/setup %}

CharacterController(角色控制器)是Unity3d的基础组件，顾名思义用来模拟角色的行为。在unity3d中，模拟角色行为的方式不止一个，一般来说按照个人的需求
选择。如果你设计的是一个RPG游戏，你期望你的角色能够行走，战斗，与场景交互等等。那么我建议使用CharacterController，它不会受到物理力的作用，而且
对斜坡，楼梯有比较好的支持不需要额外的代码。特别是当你要处理跳跃，飞行，反重力等其他需求的时候，它更易于操纵。如果你的角色不是人类，例如飞船，球
等，而且需要受到物理引擎力的作用可以选择RigidBody + Transalte来模拟。如果你需要寻路，则专业版内置的`Nav Mesh Agent`也是一个选择。
<!--break-->

CharacterController是好用且易用的，但不能滥用，因为其性能低下。在<a href="http://edwardwong.github.io/Asset/CCvsTRvsRVO.html" target="_blank">这样一个极其简单的场景</a> 中，300个移动中的CC会让帧率明显的下降，对于一般复杂的游戏场
景，100个或更少的CC就会让游戏fps低得无法接受。因此，对于大量的单位，CC明显不适用。其实CC应该只用在RPG游戏中玩家本身控制的角色，而这个角色在每一个客户端应该不超过一个，例如CS中自身角色，WOW中
的自身角色。以下几种场景则应该选择其他替代方式模拟角色：

RTS游戏中，一个玩家控制几十个单位，当几个玩家狭路相逢时，界面上很容易就超过上百个移动的单位。  
TD游戏中，怪不断的刷出，往终点跑，数量也很容易过百。  
MMORPG， 其他的玩家，NPC, 怪物加起来，数目也会越来越多根据在线玩家数目（暂不考虑LOD, Interest Management 等其他的优化)。

<br />
## Transalte

`Transform.Translate`是u3d中最简单的移动api，性能极优。如果需要物理则可以添加`RigidBody`组件。如果只需要相对简单的物理那么可以直接搭配`Raycast`使用。<a href="http://docs.unity3d.com/Documentation/ScriptReference/Transform.Translate.html" target="_blank">Transform.Translate</a>

<br />
## RVOController

我买了<a href="http://arongranberg.com/astar/" target="_blank">A\* Pathfinding Pro</a>, 用在AI服务器中寻路模块，正苦恼着怎么能提高大量AI同时寻路时的性能问题，发现A\*提供了`RVO Character Controller`，作为u3d CC的替代品。性能虽不如
`Transform.Translate`，但比CC好很多，在官方的例子中可模拟5000个Agent. 更重要的是它支持Reciprocal Velocity Obstacles，简单来说就是Agent在
移动时会相互避让对方，如下图所示：

<img src="/assets/custom/images/posts/RVO2.gif" />

这对于模拟AI特别有用, 因此把`RVOController`也作为一个选择项。

<br />
## Comparison

要知道确切的性能差别，最好的方法就是对各个选择进行profile. 测试程序就是上面提到的<a href="http://edwardwong.github.io/Asset/CCvsTRvsRVO.html" target="_blank">简单的场景</a>, 调整下选项就能对比出fps的明显不同。

Profile脚本: <a href="http://wiki.unity3d.com/index.php?title=Profiler" target="_blank">Profiler</a> 或者 <a href="http://arongranberg.com/astar/docs/class_astar_profiler.php" target="_blank">AstartProfiler</a>  
平台: Windows x86 Standalone, 连接<a href="http://docs.unity3d.com/Documentation/Manual/Profiler.html" target="_blank">Unity Profiler</a>.  
机器配置:   
  型号	Dell XPS l502x  
  Cpu 	i7-2630QM  
  Ram 	16G  
  Video 	Nvidia GT540M 2G  
	
对于CC, 主要消耗来自`CharacterController.Move`, 因此profile代码很简单：

{% highlight c# %}
    private void CharacterControllerMove()
    {
        AstarProfiler.StartProfile("CharacterControllerMove");
        _characterController.Move(_dir * Time.deltaTime);
        AstarProfiler.EndProfile("CharacterControllerMove");
    }
{% endhighlight %}

`Transform.Translate`：

{% highlight c# %}
    private void TranslateMove()
    {
        AstarProfiler.StartProfile("TranslateMove");
        transform.Translate(_dir * Time.deltaTime, Space.World);
        AstarProfiler.EndProfile("TranslateMove");
    }
{% endhighlight %}

`RVOController`的profile相对复杂  
`RVOController`的消耗，主要在于`RVOSimulator.Updated + RVOController.Updated * units`, `RVOSimulator`在场景里只会存在一个，因此一帧只会
有一次调用，无论有多少个Agent。`RVOSimulator`的作用是每帧计算所有Agent的位置，因为它是多线程的，也就是说它的计算基本上不会影响游戏的帧率。因此，对于`RVO`的profile就有两个地方。

{% highlight c# %}
    /** Update the rvo simulation */
	void Update () 
	{
        AstarProfiler.StartProfile(string.Format("RvoSimulatorUpdated_{0}", CurrentUnit));
		
		/** Update the rvo simulation */
		
        AstarProfiler.EndProfile(string.Format("RvoSimulatorUpdated_{0}", CurrentUnit));
	}
{% endhighlight %}

其中`CurrentUnit`系指测试Agent的个数。

{% highlight c# %}
    /** Update the rvo controller */
	public void Update () 
	{
        AstarProfiler.StartProfile("RvoControllerUpdate");
		
		/** Update the rvo controller */
		
        AstarProfiler.EndProfile("RvoControllerUpdate");
	}
{% endhighlight %}

时间结果如下：下表表示的是每个调用的消耗，总时间，调用次数以及平均消耗。注意的是，`RvoSimulatorUpdated`每帧只会调用一次，而`CharacterControllerMove`, `TranslateMove`, `RvoControllerUpdate`每帧调用`unit`次。


<table class="table table-condensed table-striped table-bordered">
	  <thead>
		<tr>
		  <th>Name</th>
		  <th>Total Time (ms)</th>
		  <th>Total Calls</th>
		  <th>Avg/Call (ms)</th>
		</tr>
	  </thead>
	  <tbody>
		<tr>
		  <td>CharacterControllerMove</td>
		  <td>11974.7</td>
		  <td>130695</td>
		  <td>0.092</td>
		</tr>
		<tr>
		  <td>TranslateMove</td>
		  <td>1070.1</td>
		  <td>437356</td>
		  <td>0.002</td>
		</tr>
		<tr>
		  <td>RvoControllerUpdate</td>
		  <td>2276.1</td>
		  <td>378167</td>
		  <td>0.006</td>
		</tr>
		<tr>
		  <td>RvoSimulatorUpdated_1</td>
		  <td>52.0</td>
		  <td>769</td>
		  <td>0.068</td>
		</tr>
		<tr>
		  <td>RvoSimulatorUpdated_50</td>
		  <td>80.0</td>
		  <td>879</td>
		  <td>0.091</td>
		</tr>
		<tr>
		  <td>RvoSimulatorUpdated_100</td>
		  <td>62.0</td>
		  <td>735</td>
		  <td>0.084</td>
		</tr>
		<tr>
		  <td>RvoSimulatorUpdated_300</td>
		  <td>126.0</td>
		  <td>790</td>
		  <td>0.160</td>
		</tr>
		<tr>
		  <td>RvoSimulatorUpdated_1000</td>
		  <td>194.0</td>
		  <td>591</td>
		  <td>0.328</td>
		</tr>
	  </tbody>
</table>

<div id="container"></div>
<script type="text/javascript">
$(function () {
        $('#container').highcharts({
    
            chart: {
                type: 'column',
				zoomType: 'xy'
            },
    
            title: {
                text: 'CC vs Translate vs RVO'
            },
    
            xAxis: {
                categories: ['1 unit', '50 units', '100 units', '300 units', '1000 units']
            },
    
            yAxis: {
                allowDecimals: false,
                min: 0,
                title: {
                    text: 'Cost (ms)'
                }
            },
    
            tooltip: {
                formatter: function() {
                    return '<b>'+ this.x +'</b><br/>'+
                        this.series.name +': '+ this.y +'<br/>'+
                        'Total: '+ this.point.stackTotal;
                }
            },
    
            plotOptions: {
                column: {
                    stacking: 'normal'
                }
            },
    
            series: [{
                name: 'RvoSimulatorUpdated',
                data: [0.084, 0.063, 0.098, 0.147, 0.385],
                stack: 'Rvo'
            }, {
                name: 'RvoControllerUpdate',
                data: [0.006, 0.3, 0.6, 1.8, 6],
                stack: 'Rvo'
            }, {
                name: 'Translate',
                data: [0.003, 0.15, 0.3, 0.9, 3],
                stack: 'Translate'
            }, {
                name: 'CharacterController',
                data: [0.092, 4.6, 9.2, 27.6, 92],
                stack: 'CharacterController'
            }]
        });
    });
</script>

<br />
## Conclusion

1. 一个客户端中CC的数量应该尽量少，大多数情况下1个。
2. 大量角色的模拟，使用Translate + Rigidbody.

<br />







