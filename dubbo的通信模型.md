上接[dubbo的编解码，序列化和通信](http://blog.kazaff.me/2015/02/01/dubbo%E7%9A%84%E7%BC%96%E8%A7%A3%E7%A0%81%EF%BC%8C%E5%BA%8F%E5%88%97%E5%8C%96%E5%92%8C%E9%80%9A%E4%BF%A1/)。

在上面那一篇文章中，大致的介绍了一下dubbo的通信机制。不过感觉说的有点乱，总结了下面一张图：

![](http://pic.yupoo.com/kazaff/EwVU804K/78m8v.png)

图中的从“外到内”对应着“从dubbo底层到应用的业务逻辑层”，我把在这个过程中起到关键作用的类都标注了出来，**注意：**这里还是基于dubbo的默认协议dubbo，默认通信框架netty，以及默认的序列化方式dubbocodec。

这里我想讨论的是，注意看图中的“AllChannelHandler”类的职责之一：

> 把对应事件（connected、disconnected、received、caught）的执行业务分配给线程池中可使用线程

dubbo处理handler所使用的线程并非来自netty提供的I/O Work线程，而是dubbo自身来维护的一个java原生线程池，源码见**com.alibaba.dubbo.remoting.transport.dispatcher.WrappedChannelHandler**。why？

但从[netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model/#show-last-Point)的分析中，可以认为netty提供的那些nio工作线程主要被用于消息链路的读取、解码、编码和发送。而dubbo把业务逻辑的执行放在自身维护的线程池中是否就是为了贯彻netty的这一原则呢？

从上面给的链接中可以注意到下面这段话：

> Netty是个异步高性能的NIO框架，它并不是个业务运行容器，因此它不需要也不应该提供业务容器和业务线程。合理的设计模式是Netty只负责提供和管理NIO线程，其它的业务层线程模型由用户自己集成，Netty不应该提供此类功能，只要将分层划分清楚，就会更有利于用户集成和扩展。

正如文中所说，dubbo这么做有利于分离通信层，方便的替换掉netty。至于是否还有更高深的理由，我就不清楚了，希望大牛赐教。

另外dubbo在进行数据的读取和解析上做了很多工作，体现出了开发人员的功底，由于我在这部分没有经验，勉强看得懂就已经不错了，谈不上分析，就不瞎掰了。

