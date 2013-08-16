---
layout: post
title: "Coroutine"
category: Programming
tags: [Unity3d]
---
{% include JB/setup %}

打从完成<a href="/practice/2013/08/09/space-shooter" target="_blank">第一个u3d游戏</a>开始，我就一直很喜欢使用Coroutine。原因有两点，第一是它很灵活，我可以决定在何时暂停，何时继续，这等于是一种可控的异步。第二是它性能优越，基本没有overhead。

如果要用一句话来囊括什么是Coroutine, 那么这句话再适合不过了：  
     Coroutine是一个特殊的方法，它可以暂停执行直到yield语句完成。  
<!--break-->
这句话有几个关键点：  

1. 特殊的方法 - coroutine是一个方法，但是在unity里面，它的特殊表现在:  
	1. 在c#里它要求必须以IEnumerator作为返回值。 JS则无此要求。  
	2. 它必须使用StartCoroutine来调用，而不能直接调用。  
2. 它可以暂停 - 这个暂停，不是暂停整个程序，而是暂停这个方法的执行然后马上返回(yield return), 直到yield return语句后面声明的条件满足时，才会继续执行之后的语句。这样，在coroutine暂停的时候是不会阻塞其他功能的运行，异步。  
3. yield语句 - yield语句指明了，应该在什么时候去唤醒所在的coroutine。  

Coroutine的使用不难，官网的教程也很容易上手。在这篇文章中主要cover两点：Event Coroutine 以及Coroutine的性能。

<br />
## Event Coroutine

官网中有这样一句话`Any event handler can also be a coroutine. Note that you can't use yield from within Update or FixedUpdate, but you can use StartCoroutine to start a function that can.` 在这里`Any event handler`的含义不是很明确，到底是指unity内置的<a href="http://wiki.unity3d.com/index.php?title=Event_Execution_Order" target="_blank">Event Handler</a>还是指用户自定义的Event Handler?

通过研究，两者都是可以使用Coroutine的，但两者也各自有自己的限制。  

内置的Event中，就目前测试而言：Update， FixedUpdate， Awake and LateUpdate 还有OnGUI, 都是不可以作为Coroutine的。这几个也未必完整，要在以后的实践当中慢慢补全。在<a href="http://wiki.unity3d.com/index.php?title=Event_Execution_Order" target="_blank">Event Handler</a>这里列出的大部分，例如OnCollisionEnter / OnCollisionStay / OnCollisionExit/ OnTriggerEnter / OnTriggerStay / OnTriggerExit等都是可以作为一个Coroutine调用。由于这些内置的Event都是有Unity去调用的，我们只需要改变这些方法的返回值就可以，而不需要显式调用StartCoroutine。  
普通Event:  
{% highlight c# %}
    void OnTriggerEnter(Collider col)
    {
        Debug.Log("Collider triggered");
    }
{% endhighlight %}

作为Coroutine的Event：  
{% highlight c# %}
    IEnumerator OnTriggerEnter(Collider col)
    {
        Debug.Log("Collider triggered");
        yield return 0;
    }
{% endhighlight %}


而用户自定义的Event Coroutine，只有最后一个才会被执行。  
{% highlight c# %}
    delegate IEnumerator BookSelectEventHandler();
    event BookSelectEventHandler BookSelect;

    void Start()
    {
        BookSelect += BookSelect1;
        BookSelect += BookSelect2;
        OnBookSelect();
    }

    void OnBookSelect()
    {
        if (BookSelect != null)
            StartCoroutine(BookSelect());
    }

    IEnumerator BookSelect1()
    {
        Debug.Log("BookSelect1");
        yield break;
    }

    IEnumerator BookSelect2()
    {
        Debug.Log("BookSelect2");
        yield break;
    }
{% endhighlight %}

这种情况下就只有BookSelect2会被执行。

<br />
## Coroutine的性能


Coroutine可以用来模拟跨越多个frame的行为，可以由用户自定义Coroutine的唤醒时间，完全可以由用户自己模拟Update，模拟循环执行的逻辑。  
考虑以下场景： 每秒钟执行一次的逻辑。 很容易想到至少有3种常用的方法去实现：  
<a href="http://docs.unity3d.com/Documentation/ScriptReference/MonoBehaviour.InvokeRepeating.html" target="_blank">InvokeRepeating</a>  
<a href="http://docs.unity3d.com/Documentation/ScriptReference/Coroutine.html" target="_blank">Coroutine</a>  
<a href="http://docs.unity3d.com/Documentation/ScriptReference/MonoBehaviour.Update.html" target="_blank">Update</a>  

官网上有说明Coroutine的性能是优越的，但怀着怀疑的精神我还是做了以下的测试：  
场景中一共有5000个GameObject,然后分别挂上以下脚本。

### InvokeRepeating

{% highlight c# %}
    private int i = 0;

	void Start () 
    {
        InvokeRepeating("Test", .01f, 1.0f);
	}
	
    void Test()
    {
        ++i;
    }
{% endhighlight %}

测试结果为： 每帧大约0.3-0.5ms， 每隔一段时间有一次峰值大约6-7ms.  
Profiler的截图如下：  
<img src="/assets/custom/images/posts/InvokeRepeatingProfile.jpg" />  

<br />
### Coroutine

{% highlight c# %}
    IEnumerator Start()
    {
        var i = 0;
        while (true)
        {
            ++i;
            yield return new WaitForSeconds(1.0f);
        }
    }
{% endhighlight %}

测试结果为： 每帧大约0.3-0.5ms， 每隔一段时间有一次峰值大约1.5-1.7ms.  
Profiler的截图如下：  
<img src="/assets/custom/images/posts/CoroutineProfile.jpg" />  

<br />
### Update

{% highlight c# %}
    private float _timer = 0f;
    private int i = 0;

    void Update()
    {
        if (!(Time.time >= _timer)) return;
        _timer = Time.time + 1.0f;
        ++i;
    }
{% endhighlight %}

测试结果为： 每帧大约2.5-3ms， 每隔一段时间有一次峰值大约3.5-4ms.  
Profiler的截图如下：  
<img src="/assets/custom/images/posts/UpdateProfile.jpg" />  


由结果可以很直观的看出，`InvokeRepeating` 和 `Coroutine` 的平均性能几乎一样，但`Coroutine`更稳定。另一方面，`InvokeRepeating`的限制也比较大，例如你不能为它的方法指定参数。 而相比之下`Update`的
性能就相去甚远，这和它使用了`SendMessage`类似的机制有很大关系。


以上。就是我为何钟爱`Coroutine`的原因，灵活，高效~
<br />


















