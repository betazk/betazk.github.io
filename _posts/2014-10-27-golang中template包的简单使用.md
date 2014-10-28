---
layout: post
title: golang中html/template包的简单实用
categories:
- 技术
tags:
- golang 
---

>今天看了一段golang的代码，其中有个文本输出是使用了html/template，由于之前老是遗忘这个包是如何使用的，这次查看了下，就记录下简单的使用方法。

##使用环境
看这个包的名字，我的感觉就是它是在html中使用的或者是服务器跟前端交互中展现、渲染页面使用的。
在查看了[官网](http://golang.org/pkg/html/template/)后看到一句话说是：

`It provides the same interface as package text/template and should be used instead of text/template whenever the output is HTML`

它提供了与`text/template`一样的接口，也就是说`text/template`这个包能做的那么它也能做，可以作为`html`页面的渲染展示，也可以作为普通文本的输出使用。
##使用方式
通过一个小代码来解释一下吧：

{{% highlight go %}}
	
	var (
		httpTemplate = `
		{{range .}}
	{{.AppName | trim }} will come
		{{end}}
		`
	)
	
	type TestData struct {
		AppName template.HTML
	}
	
	func main() {
		datas := make([]TestData, 0)
		datas = append(datas, TestData{AppName: "zk"})
		datas = append(datas, TestData{AppName: "zkp"})
		datas = append(datas, TestData{AppName: "zkc"})
		t := template.New("top")
		t.Funcs(template.FuncMap{"trim": func(s template.HTML) template.HTML {
			return template.HTML(strings.Repeat(string(s), 4))
		}})
		template.Must(t.Parse(httpTemplate))
		err := t.Execute(os.Stdout, datas)
		if err != nil {
			panic(err)
		}
	}

{{% endhightlight %}}

上面这段代码只有一个很简单的输出内容，就是输出`TestData`这个结构体的AppName这个字段。这里的`httpTemplate`字符串是一个模板渲染的语法。这个稍后解释，先看看模板的使用方法，首先通过`new`这个方法创建一个模板实例`t`，类型为`*template`，这个`t`有很多的方法，下面看看它其中的三个比较重要的方法，第一个就是`Parse`方法,它解析了需要进行渲染的字符串，我个人的理解是它起到了一个类似于编译的作用，编译了模板语法写成了语句，这里就是`httpTemplate`这个字符串语句。编译好了之后当然就是执行了，`Execute`方法就是执行这个编译过后的语句，方法有两个参数，一个是需要把最终结果输出到哪里，还有一个是语句参数的值。之后就会在参数一的地方输出执行后的结果了。`template.Must`这个方法是保证我们在编译语句的时候出错了就立刻引发panic异常。而`t.Funcs`这个方法在看过语句之后在来解释，现在可以完全忽略它。

##模板语法
现在看看模板语句，也就是`httpTemplate`这个字符串，双大括号是区分模板代码(里面的变量由Execute执行时传入的)和HTML或者正常文本的分隔符(可通过template包提供的Delims方法修改)，括号里面可以显示输出的数据，或者是控制语句，比如if判断或者range循环等。跟在range后面的必须是一个array，slice或者map类型变量，这点跟golang的循环中range后面的类型一样，但是在语句中只是一个点“`.`”并看不出来是什么类型，那这个点代表了什么？这里就要和上面的程序代码结合起来了，我们在Execute的时候传进去第二个参数datas，很明显datas这个变量是个slice类型的，忽略“`| trim`”，我们可以把这个语句翻译成

{{% highlight go %}}

	for _,v:=range datas{
		fmt.Println(v.AppName+"will come")	
	}

{{% endhightlight %}}

`.AppName`不需要解释也知道是代表什么了吧，就是TestData这个结构体里的变量。那么输出结果就很明显了。

现在回到之前被忽略了的地方，遭受冷落心里总归是不爽的，那么现在我们就用极大的热情看看它们吧。
首先看看语句里面被忽略的`| trim`，要是使用过linux或者类unix的操作系统的人肯定第一反应就是匿名管道，这里我们也可以把它认为就是个匿名管道，那么管道的左边是数据的输入，右边是操作左边流入的数据的程序。左边的.AppName不用解释了，它将把值往`|`中流入，那么操作这个数据流的程序找来找去只看到了`trim`这个东西了，操作数据流的方法就是它了，我们程序中的方法无外乎两种，一种就是包自带的，还有就是自己实现的。那么包中没有trim这个方法，那就要写个方法实现它了。如何实现，我们就要看代码中之前被我们忽略的地方了，哎，它都等不及要表现一下了。没做就是
	
	t.Funcs(template.FuncMap{"trim": func(s template.HTML) template.HTML {
        return template.HTML(strings.Repeat(string(s), 4))
    }})

`t.Funcs`方法有个FuncMap类型的参数，这个具体可以查看[文档](http://golang.org/pkg/text/template/#Template.Funcs),FuncMap是个map类型`type FuncMap map[string]interface{}`,顾名思义，这个方法的功能就是用来函数映射的，将一个字符串映射到一个函数中去，函数的输入参数就是管道左边的数据流变量值，函数的输出参数就是对数据流处理过后的结果。那么`{{.AppName | trim }}`我们可以把它看作是:

{{% highlight go %}}

	func trim(s template.HTML) template.HTML{
		return template.HTML(strings.Repeat(string(s),4))
	}
	fmt.Println(trim(AppName))
{{% endhightlight %}}

执行的结果：
![执行结果](/image/20141027.PNG)