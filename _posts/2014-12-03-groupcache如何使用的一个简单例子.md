---
layout: post
title: groupcache如何使用的一个简单例子
categories:
- 技术
tags:
- groupcache
---
>在网上查了挺多groupcache的相关信息，但是搜出来大部分都是copy，实际的例子也没有，所以就看了下源码，也在golang-nuts上问了，github上搜索了一些groupcache的使用例子，在此作个总结，目前对这个缓存库还仅处于了解状态

##大概介绍
其实关于groupcache的介绍网上非常的多，搜索出来清一色都是说的介绍，当然也有配图如何部署，但是文字与配图完全不在一个时空，图也是copy国外的一篇[博客](http://www.capotej.com/blog/2013/07/28/playing-with-groupcache/)。它不像其它的一些缓存数据库有个服务端，需要客户端去连接，文档中明确说明了它就是一个程序库，所以没有服务端与客户端这么一说，换句话说，它本没有服务端，又或者是人人都是服务端，食神的感觉油然而生。

##主要代码结构
它的代码结构也比较清晰，代码量也不是很大，很适合大家去阅读学习。主要分为了`consistenthash`**(提供一致性哈希算法的支持)**，`lru`**(提供了LRU方式清楚缓存的算法)**，`singleflight`**(保证了多次相同请求只去获取值一次，减少了资源消耗)**，还有一些源文件：`byteview.go` **提供类似于一个数据的容器**，`http.go`**提供不同地址间的缓存的沟通的实现**，`peers.go`**节点的定义**，`sinks.go`**感觉就是一个开辟空间给容器，并和容器交互的一个中间人**，`groupcache.go`**整个源码里的大当家，其它人都是为它服务的**。

##一个测试例子
在使用这个缓存的时候比较重要的几方面也是我之前犯错的几个地方

* 需要监听两个地方，一个是监听节点，一个是监听请求
* 在批量设置节点地址的时候需要在地址前面加上`http://`，因为一开始我没有加上去，所以缓存信息一直不能再节点之间交互
* 启动的节点地址要与设置的节点地址一致：数量和地址值。因为我每次少一个就无法在节点间交互。

*以上的一些信息可能也有不对的地方，只是我个人的一个测试后的结果。*

###代码部分###

{% highlight go %}

		package main

		import (
			"flag"
			"fmt"
			"github.com/golang/groupcache"
			"io/ioutil"
			"log"
			"net/http"
			"os"
			"strconv"
			"strings"
		)
		
		var (
			// peers_addrs = []string{"127.0.0.1:8001", "127.0.0.1:8002", "127.0.0.1:8003"}
			//rpc_addrs = []string{"127.0.0.1:9001", "127.0.0.1:9002", "127.0.0.1:9003"}
			index = flag.Int("index", 0, "peer index")
		)
		
		func main() {
			flag.Parse()
			peers_addrs := make([]string, 3)
			rpc_addrs := make([]string, 3)
			if len(os.Args) > 0 {
				for i := 1; i < 4; i++ {
					peers_addrs[i-1] = os.Args[i]
					rpcaddr := strings.Split(os.Args[i], ":")[1]
					port, _ := strconv.Atoi(rpcaddr)
					rpc_addrs[i-1] = ":" + strconv.Itoa(port+1000)
				}
			}
			if *index < 0 || *index >= len(peers_addrs) {
				fmt.Printf("peer_index %d not invalid\n", *index)
				os.Exit(1)
			}
			peers := groupcache.NewHTTPPool(addrToURL(peers_addrs[*index]))
			var stringcache = groupcache.NewGroup("SlowDBCache", 64<<20, groupcache.GetterFunc(
				func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
					result, err := ioutil.ReadFile(key)
					if err != nil {
						log.Fatal(err)
						return err
					}
					fmt.Printf("asking for %s from dbserver\n", key)
					dest.SetBytes([]byte(result))
					return nil
				}))
		
			peers.Set(addrsToURLs(peers_addrs)...)
		
			http.HandleFunc("/zk", func(rw http.ResponseWriter, r *http.Request) {
				log.Println(r.URL.Query().Get("key"))
				var data []byte
				k := r.URL.Query().Get("key")
				fmt.Printf("cli asked for %s from groupcache\n", k)
				stringcache.Get(nil, k, groupcache.AllocatingByteSliceSink(&data))
				rw.Write([]byte(data))
			})
			go http.ListenAndServe(rpc_addrs[*index], nil)
			rpcaddr := strings.Split(os.Args[1], ":")[1]
			log.Fatal(http.ListenAndServe(":"+rpcaddr, peers))
		}
		
		func addrToURL(addr string) string {
			return "http://" + addr
		}
		
		func addrsToURLs(addrs []string) []string {
			result := make([]string, 0)
			for _, addr := range addrs {
				result = append(result, addrToURL(addr))
			}
			return result
		}


{% endhighlight %}

执行方式：`./demo 127.0.0.1:8001 127.0.0.1:8002 127.0.0.1:8003`

在上面的命令中我们就启动了一个节点，并且设置节点地址个数为3个，这里由于我是默认的index为0，所以在启动其它节点的时候变换下地址的顺序，使第一个地址三次都不一样就好了。(抱歉，这里实现的不是很好),这样同样方法启动三个节点。

打开浏览器访问`127.0.0.1:9001?key=1.txt`，这里1.txt是需要获取数据的之际地方，类似于实际中的数据库，我这里直接用一个文件代替了。

运行结果：

* 当我访问`:9001`时结果
 ![](/image/2014blog/20141203_0.PNG)

 ![](/image/2014blog/20141203_1.PNG)

 ![](/image/2014blog/20141203_2.PNG)

在上面图中，我们像`:8001`这个节点请求数据，它没有缓存数据，然后会去其它节点中寻找这个`key`的数据，当都没有的时候就会去数据库中获取(这里我们是一个文件),所以会出现在`:8003`这个节点中获取数据库数据的操作。

* 然后我访问`:9002`
 ![](/image/2014blog/20141203_0.PNG)

 ![](/image/2014blog/20141203_3.PNG)

 ![](/image/2014blog/20141203_2.PNG)

根据上图看到，第一个地址为`:8002`这个节点直接从缓存里面取值，但是在请求之前这个节点并没有缓存数据，这个也同样是节点间的交互完成的。

我在本地开启了两个虚拟机，在同一个局域网中，测试也能得出相同的结果。这里结果就不再贴上了，当然运行的时候节点地址要做相应的变动，根据每个机子的局域网中的地址。
>这篇博客关于groupcache的介绍和源代码的说明部分比较少，主要就是贴出了一个测试的例子，因为我看到网上很少，并且压根就没有给出如何运行或者运行结果(不包括刚才提到的老外的一个博客，他写的还是很好的，我就是看了他的博客才着手自己编写的)。