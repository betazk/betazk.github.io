---
layout: post
title: 关于select中的一个case-channel的阻塞问题
categories:
- 技术
tags:
- golang
---
>昨天在群里看到有人发了一段代码，是个关于select中case的阻塞问题，第一次见到，所以就自己运行后写点到目前为止的想法，不一定正确。

Code
=

{% highlight go %}

	package main

	import (
		"fmt"
		"time"
	)
	
	func main() {
		fmt.Println("start")
		c := make(chan struct{})
		//c := make(chan struct{}, 1)
		go slet(c)
		time.Sleep(time.Second * 3)
		c <- struct{}{}
		time.Sleep(time.Second * 3)
	}
	
	func slet(ch chan struct{}) {
		for {
			select {
			case ch <- <-ch:
				fmt.Println(len(ch))
			default:
				fmt.Println("default")
			}
		}
	}

{% endhighlight %}

代码部分稍微做了一下小修改，这段代码的运行结果是只打印了一次"default"。

上面比较有意思的在`select`中的`ch<-<-ch`，说实话我是第一次见到go程序中出现这个东西，也许这和我go程序写的少也有关系吧。总之在看到代码的时候我还是没想到只打印一次"`default`"，既然有了结果，那就根据结果自己做一个猜想。打印出的信息也是在3秒之后才开始打印出来的，所以当goroutine运行的时候，在main还在暂停3秒的时候一直处于阻塞的状态，这个阻塞就发生在select块中的`ch<-<-ch`这部分。这就很奇怪了，不是说select语句中当有某个case发生阻塞的时候会继续寻找下面的不发生阻塞的case吗，况且这里还有一个default存在，按理说就算所有的case发生阻塞那么也是会执行default的，而这里偏偏没有执行。

这里要注意到我们之前定义的channel类型的变量c是没有缓存的，意思就是说当执行c<-value的时候，要是没有<-c读取操作，那么c<-value是会一直阻塞着的，直到有读取操作的发生。那么当运行`slet`的时候，很显然main还在休眠并没有执行到c<-struct{}{}，也就是说`ch<-<-ch`这个操作就一直阻塞在后面一个`<-ch`中，也就是说这整个case语句压根就没有判断结果的出炉。

那么这里也出现了阻塞，为什么没有运行下面的default，我的理解是，`select中说的case阻塞，那么久会运行default操作`，这个阻塞是当case后面的操作完成后判断出是阻塞的，那么才会运行default操作，要是这个操作一直没有完成，那么它会等待这个操作完成来判断是否是阻塞的。而上面的代码中，由于第二个`<-ch`一直处于等待状态，导致这个整个操作也一直处于等待的状态。换句话说，第二个`<-ch`操作可以理解为一个死循环，当有一个break条件让它跳出的之后，它才会运行结束，而这个地方的break条件就是main中的`c<-struct{}{}`。

可能`ch<-<-ch`看起来不是很清楚，我们可以把后面一个`<-ch`看成是一个函数

{% highlight go %}

	func dowhile(ch chan struct{}) struct{} {
		return <-ch
	}

{% endhighlight %}

那么`ch<-<-ch`就变成了`ch<-dowhile(ch)`，这下问题可能就清晰多了。

我个人觉得，对上面的问题会有不同答案的原因还是对于阻塞的理解。**当select中所有的case阻塞的时候就会执行select块中的default语句**，对这个`阻塞`的含义的理解是比较重要的。

上面的就是我自己的一些个人理解，具体的select操作时如何实现的还是要看源代码来理解(我还未看过代码是如何实现的)，要是理解错误或者偏了还望大神及时指出。