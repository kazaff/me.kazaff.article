最近沉浸在SOA相关的调研，当然是驱动于公司项目。慢慢的越发感觉架构师的魔力，同时也让我有了新的目标！

为了解决公司现有项目之间的数据通信性能低下且混乱的局面，感觉是要引入服务治理的概念了，这里关注点并不是分布式系统之间请求的流量限制或资源沙盒等方面，更多的是在服务发现，软负载均衡，高性能的服务调用，失效容错，服务监控等方面的高可用性方面。

通过GG搜索，发现了Dubbo，感觉自己现在的能力，无论如何也不可能在短时间内重新造一个比Dubbo还好的轮子，so，那就从掌握Dubbo开始吧~~看了几天Dubbo的文档，由衷的佩服国人大牛们，在此向他们致敬！

Dubbo的[文档](http://alibaba.github.io/dubbo-doc-static/Home-zh.htm)已经写的非常全面了，这里就不重复了~~我们从编译Dubbo提供的实例源码开始吧。

首先要说的是，似乎编译源码从来都不是让人可以开心的起来的一件事儿，不论再怎么成熟的项目，总可能会在编译时碰到问题，这还不算，更可恨的时，每个系统编译同一套代码的报错绝对不会完全一致，这就让很多搜索出来的编译步骤文章变得毫无意义。

为了不使这篇文章也变成废纸，我这里简单的描述一下我在编译中碰到的问题，但主要还是把前辈们写过的相关文章链接整理出来，供你快速的查阅~以免去到处查找所消耗的时间……

下载源码
---
你肯定会按照Dubbo文档提供的下载地址试图下载编译好的DEMO压缩包，但肯定是无法下载的，貌似code.alibabatech.com这个域名没有对外开放。

不过没有关系，开发组已经把Dubbo相关的源码移交在[Github](https://github.com/alibaba/dubbo/tree/master)上了，如果有一天github被墙了，那就悲催了~

如果没有github账号也没有关系，可以直接从[https://github.com/alibaba/dubbo/archive/master.zip](https://github.com/alibaba/dubbo/archive/master.zip)下载到最新的源码，目前最新版本为2.5.4-SNAPSHOT。但官方推荐的是2.4.9，再看一下github上的tag，发现似乎目前使用2.4.10这个版本比较保险~~

这就要求你必须在系统上安装git，并执行：
	
	git clone https://github.com/alibaba/dubbo.git dubbo
	git checkout dubbo-2.4.10

这样就把源码切换到2.4.10版本了~

开始编译
---
从gihub上README提供的步骤，似乎很复杂，不过其实呢，我在编译2.5.4版本的时候却是一路绿灯，直接Build Success！但是从其他人的帖子上来看，能这么顺利似乎是开发组已经把相关包发到Maven中央仓库了~省去了很多不必要的步骤~~但如果你像我一样打算使用2.4.10的话，那这些问题还是要解决的~~呵呵，直面困难，才是男银！

如果系统中没有安装maven2，请先安装：

	sudo apt-get install maven2

可以看出，我目前是基于Ubuntu的，不过Centos上步骤也是一致的，只不过要用`yum`命令而已~

有了`mvn`命令，我们可以看这个[帖子](http://jianshu.io/p/0dde591f21d0)，写得比较具体~~按照步骤，需要你切换到dubbo源码文件夹，执行：

	mvn clean install -Dmaven.test.skip

如果要在编译的时候忽略测试用例，可以这么写：

	mvn clean install -Dmaven.test.skip=true


接下来你就可能遇见很多问题了，这在上面给的帖子里应该有命中一些，以我的亲测经验，我碰到了`opensesame`本地安装的问题，还有就是修改`fastjson`的版本问题，这都在帖子里提供了解决方案。初次之外似乎一切顺利，这难道就是我人品的魅力？！

好吧，如果你还遇到了新的问题，可以尝试一下[这个贴](https://github.com/alibaba/dubbo/issues/22#issuecomment-42511830)~~应该可以助你成功编译！

除此之外，你可能还会遇到关于maven的一些问题，比方说`~/.m2`文件不存在，找不到maven的settings.xml文件位置等，都可以参考[这里](http://my.oschina.net/sherwayne/blog/108673)和[这里](http://my.oschina.net/hongdengyan/blog/150472)。


如果还不行，你可以先在Dubbo的github上留言，希望开发组或其他大牛们能给你解答~~
