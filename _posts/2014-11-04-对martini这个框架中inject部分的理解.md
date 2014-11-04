---
layout: post
title: 对martini这个框架中inject部分的理解
categories:
- 技术
tags:
- golang 
- martini
- inject
---
>martini是一个golang开发的简单的web开发框架，代码量非常小，很多其它工作像是模板的渲染等也都以中间件的形式提供，这样保持了整个框架的简洁。在阅读了之后，也做个笔记。

Inject使用例子
=
martini的整个框架的后台动力都是来自inject这个包，它其实是独立于框架的一个功能包，里面主要是运用了反射的技术来完成所需要的功能。我自己在学习或者接触一段新代码或者知识点的时候都会先直接找**sample**运行看跑的结果，所以下面先看一段代码及执行结果<**代码1**>：

{% highlight go %}

	package main

	import (
		"fmt"
		"github.com/codegangsta/inject"
		"log"
		"reflect"
	)
	
	func do(i int) int {
		fmt.Println(i)
		return 23
	}
	
	func main() {
		in := inject.New()
		in.Map(12)
		val, err := in.Invoke(do)
		if err != nil {
			log.Fatal("调用出错:" + err.Error())
			return
		}
		for _, v := range val {
			if v.Kind() == reflect.Int {
				fmt.Println(v.Int())
			}
		}
	}

输出结果为:  `12
            23`

{% endhighlight %}

下面根据上面的代码来一步步分析下inject的执行过程与原理。

 代码结构
=
有一个inject结构

{% highlight go %}

	type injector struct {
		values map[reflect.Type]reflect.Value
		parent Injector
	}
{% endhighlight %}

`values`是用来存储每一个`<type-value>`键值对的，每一个类型只对应于一个值，因为要是有两个值的类型相同的话，那么后面一个值将会把前面的一个值覆盖掉。`parent`这个字段存储了此节点的父节点。

`代码1`中，`in := inject.New()`就是创建了一个inject结构，并返回。代码就是

{% highlight go %}

	// New returns a new Injector.
	func New() Injector {
		return &injector{
			values: make(map[reflect.Type]reflect.Value),
		}
	}
{% endhighlight %}
下面是`in.Map(12)`这条语句，顾名思义就是将`12`这个数值进行映射，在inject中唯一可以存储映射的就只有`values`了，那么它当然就应该存储到values里面。代码如下：

{% highlight go %}

	// Maps the concrete value of val to its dynamic type using reflect.TypeOf,
	// It returns the TypeMapper registered in.
	func (i *injector) Map(val interface{}) TypeMapper {
		i.values[reflect.TypeOf(val)] = reflect.ValueOf(val)
		return i
	}
{% endhighlight %}

返回的这个`TypeMapper`是一个接口

{% highlight go %}

	// TypeMapper represents an interface for mapping interface{} values based on type.
	type TypeMapper interface {
		// Maps the interface{} value based on its immediate type from reflect.TypeOf.
		Map(interface{}) TypeMapper
		// Maps the interface{} value based on the pointer of an Interface provided.
		// This is really only useful for mapping a value as an interface, as interfaces
		// cannot at this time be referenced directly without a pointer.
		MapTo(interface{}, interface{}) TypeMapper
		// Provides a possibility to directly insert a mapping based on type and value.
		// This makes it possible to directly map type arguments not possible to instantiate
		// with reflect like unidirectional channels.
		Set(reflect.Type, reflect.Value) TypeMapper
		// Returns the Value that is mapped to the current type. Returns a zeroed Value if
		// the Type has not been mapped.
		Get(reflect.Type) reflect.Value
	}
{% endhighlight %}

下面我们会看到其实`inject`这个结构实现了`TypeMapper`这个接口。所以返回值我们也可以理解为inject这个结构本身。是否需要这个返回值根据情况而定，大多数时候都不需要(但是不管你要不要，它都给你返回了^_^)。

下面回到`Map`这个方法，它使用了`reflect`这个反射包，这样就可以将需要映射的`12`这个值存储到了values中了，关于`reflect`这个包的用法可以查看[官方的文档](http://golang.org/pkg/reflect/)。

既然函数的参数已经映射完成了，那么久可以执行调用操作了。`in.Invoke(do)`就是调用`do`方法，所以Map这个方法其实是为了将所要调用的函数的参数事先存储到`in`这个实例中服务的，可能看到这里大家就会想，上面`TypeMapper`接口中不是还有一个MapTo吗，长的跟`Map`这么像是干嘛的，嗯，当然不是耍帅用的，最开始的时候，介绍inject这个结构的时候说，values是个map类型的，每一个类型只能对应一个唯一的值，那么下面问题来了：我们函数的参数类型不可能都不一样吧。do函数只有一个参数，可我们要是再加一个同样是int类型的参数呢，岂不是要把之前的参数值给覆盖掉了，为了解决这个问题引入了MapTo这个方法，完成的功能和Map是一样的，只是多了一个参数。**go语言**中我们可以基于一个已有类型建立一个新类型，`type myInt interface{}`,那么`myInt`就是基于`interface{}`的自建的类型了，interface{}这个就相当于**python**或者 **java**中的`Object`，**C/C++**中的`void*`。既然不能两个都叫int，那么我就换一个名字改叫myInt好了，这下就不会冲突了。同样，要是还有一个int型的参数，那就再定义一个myInt2等等，总之不跟你一个样就好了。
例如：本来是`do(i,j int)`,那么现在就变成了`do(i int,j myInt)`这个样子了。

回到Invoke这个方法，看看这个方法是怎么执行的：

{% highlight go %}

	// Invoke attempts to call the interface{} provided as a function,
	// providing dependencies for function arguments based on Type.
	// Returns a slice of reflect.Value representing the returned values of the function.
	// Returns an error if the injection fails.
	// It panics if f is not a function
	func (inj *injector) Invoke(f interface{}) ([]reflect.Value, error) {
		t := reflect.TypeOf(f)
	
		var in = make([]reflect.Value, t.NumIn()) //Panic if t is not kind of Func
		for i := 0; i < t.NumIn(); i++ {
			argType := t.In(i)
			val := inj.Get(argType)
			if !val.IsValid() {
				return nil, fmt.Errorf("Value not found for type %v", argType)
			}
	
			in[i] = val
		}
	
		return reflect.ValueOf(f).Call(in), nil
	}

{% endhighlight %}

`var in = make([]reflect.Value, t.NumIn())`这条语句创建了一个in变量，这个变量的作用就是用来存储被调用的函数的参数值的。`t.NumIn()`这个方法当t的类型不是函数的时候会直接panic，正常返回的是f这个函数的参数的个数，可以参看[官方的文档](http://golang.org/pkg/reflect/)。

既然是调用，那么肯定是需要参数的，而参数之前又已经存储到了values中，所以现在只需要到values中把参数取出来就OK了。由于values是一个`<type-value>`键值对，所以想要获取值首先得知道这个值的类型，`t.In(i)`就是获取参数的类型，i是一个索引，表示第几个参数(从0开始)，在获取类型之后，就可以到values中取值了，代码中取值使用了Get这个方法，下面是Get方法的代码，也很简单

{ % highlight go % }

	func (i *injector) Get(t reflect.Type) reflect.Value {
		val := i.values[t]
	
		if val.IsValid() {
			return val
		}
	
		// no concrete types found, try to find implementors
		// if t is an interface
		if t.Kind() == reflect.Interface {
			for k, v := range i.values {
				if k.Implements(t) {
					val = v
					break
				}
			}
		}
	
		// Still no type found, try to look it up on the parent
		if !val.IsValid() && i.parent != nil {
			val = i.parent.Get(t)
		}
	
		return val
	
	}

{%  endhighlight %}

首先根据类型t从values中取值，取值当然有取失败的，所以接下去就判断一个这个值是否是可用的，`isValid()`，要是可用的就直接返回这个值，否则就说明values中不存在这个类型t对应的值，那接下去Get会查看类型t是否实现了values中某接口类型，如果确实这样，那么就返回这个接口类型所对应的值。如果还找不到，那么Get还会尽最后的努力请求老爸，看看他那里是否有这个类型或者相关接口(代码也拼爹啊?)。

假设一切顺利，那么现在`in`这个变量已经存储了被调用函数的所有参数值了，万事俱备，只欠`Call`，那就Call，`Call`方法的参数就是存储有被调用函数的参数切片in，返回值为被调用函数f的返回值，这里需要注意的是虽然它的返回值是f的返回值，但是它是一个[]reflect.Value切片,使用的时候自己根据相关方法进行类型转换。

除了上面提到的方法之外，inject源码中还有一个很帅气的方法，**Apply**

{% highlight go %}
   
	package main

	import (
		"fmt"
		"github.com/codegangsta/inject"
	)
	
	type MyApp struct {
		Name string `inject`
		Age  int    `inject`
	}
	
	func main() {
		app := MyApp{}
		in := inject.New()
		in.Map("zhengk")
		in.Map(25)
		in.Apply(&app)
		fmt.Println(app.Name)
		fmt.Println(app.Age)
	}

{% endhighlight %}

运行结果为: `zhengk
           25`

还有一个就是为拼爹使用的找爹函数`SetParent`,功能：给当前inject实例设置一个父亲，在Get方法里面也见过它的使用场景。给一个简单的例子吧

{% highlight go %}

	package main
	
	import (
		"fmt"
		"github.com/codegangsta/inject"
	)
	
	func do(i int, j myInt, name string) {
		fmt.Println(i, j, name)
	}
	
	type myInt interface{}
	
	func main() {
		inP := inject.New()
		inP.Map("zhengk")
		inChild := inject.New()
		inChild.Map(25)
		inChild.MapTo(12, (*myInt)(nil))
		inChild.SetParent(inP)
		inChild.Invoke(do)
	}
{% endhighlight %}

运行结果为:`25 12 zhengk`

>inject就像是martini的引擎，带着它狂奔。。