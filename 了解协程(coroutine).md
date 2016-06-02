这个技术名词并不是第一次听到，但如果你和我一样并未深究过其细节，那可以先来看看大家是怎么评价协程的：

[openstack中的协程](http://niusmallnan.github.io/_build/html/_templates/openstack/coroutine_usage.html)

[知乎贴1](https://www.zhihu.com/question/20511233)

[知乎贴2](https://www.zhihu.com/question/32218874/answer/55469714)

[协程与事件循环](http://www.ituring.com.cn/article/207808)

[Java里的协程](https://blog.maxleap.cn/archives/816)

除此之外，还有不少文章在对比线程和协程的区别：

> 线程使用了隐式切换+抢占机制，而协程则是显式+协作式；

（隐式指的是靠OS来中断并切换任务，显式则表明交给程序员来做，这就意味着协程需要开发人员更多的精力来设计和开发。而且，资源非抢占会更容易导致 **饥饿任务**。）

> 线程可以实现多核并行，而协程则并非真正的并行。

其实仔细想想，前面提到的大牛们的解释，协程不过也就是提供给编程人员来自己调度执行流程的一种接口，在我看来，这与之前的`goto`语句比起来并没有太多差别！（其实不仅要解决执行流跳转问题，还需要保持跳转时的数据栈）

Nodejs中大量的异步函数都集中在IO上，而在IO阻塞时，将计算机资源释放，将控制权转让给其它可执行语句非常好理解。而对于非io密集型呢？是否也需要进行调度？我们举一个非IO的阻塞例子：**一个线程由于锁，或信号量而阻塞**。**其实，只要是存在协作的任务，就需要调度，不管是何种类型的**。以这种视角来理解协程，就容易接受很多！

既然线程才是真正意义上可以利用多核来并行的，那么协程这样的东西的价值是什么？问题在于，如果我们的场景里存在大量的并发逻辑，且这些逻辑可能会经常的自阻塞，那么就会造成os级别的线程上下文切换，即便是在创建的线程总数没有造成资源瓶颈的情况下也会造成性能折损。在这种场景下想要获得性能优化，协程就会发挥作用。如果我们将阻塞任务调度交给开发人员来管理，由于他比OS更了解场景的特殊性，那理应做出更佳的调度结果，相对的，我们的os的调度规则是面向通用场景的，无法照顾所有特殊的情况。


我们来看下面两个协程库：

[node-fibers](https://github.com/laverdet/node-fibers)

[quasar](https://github.com/puniverse/quasar)

### node-fibers

听名字就知道该库是nodejs系的，node一直是我比较关注的一个生态环境，所以我们就从它开始吧。

从官方例子来看，node-fibers很好的解决node的[回调地狱问题](https://bjouhier.wordpress.com/2012/03/11/fibers-and-threads-in-node-js-what-for/#show-last-Point)，不过这在`Promise`和`Generator`已主流的今天，似乎不再有价值了（纯属我个人观点）。

### quasar

前面说过协程和线程的差别，那么我们想想，在java中常见的处理并发业务时的做法应该是多线程吧，为了避免频繁的创建线程，一般还会配合一种线程池使用。

但可以创建的线程数量是受操作系统和硬件限制的，而且超过了最佳值后性能还会受到影响。那如何可以打破这个瓶颈呢？

quasar带来了轻量级线程概念，其实也就是协程的意思。它提供了M:N(M>N)模型来实现M个协程映射到N个系统级别线程，这样就可以让java拥有类似nodejs那样的性能。可以通过阅读[这篇博文](http://blog.paralleluniverse.co/2014/02/06/fibers-threads-strands/#show-last-Point)来了解一下quasar的实现细节。

其中，官方给出了一条法则：

> Code that runs in short bursts and blocks often (e.g. code that receives and responds to messages) is better suited to run in a fiber, while code performs a lot of computations and blocks infrequently better belongs in a plain thread.

意思是：

- fiber适合那些阻塞频繁而间歇性执行任务的场景；
- thread适合那些大量计算性任务而偶尔阻塞的场景。

这和上面的分析也是吻合的，只有阻塞频繁的业务才应该交给应用自己来处理。

quasar默认借助java提供的ForkJoinPool来做调度器的，使用java agent来对相关代码进行字节码增强，内部使用特定的异常机制来触发资源让步，并使用应用级别的栈容器来管理fiber的内部状态和代码指针的。

面对web领域，我们可以直接使用[`comsat`](http://www.paralleluniverse.co/comsat/)提供的高级框架层来写我们的逻辑，貌似很容易上手。它提供了actor model的编程模型，就是针对高性能应用场景的，有兴趣的朋友可以耍耍。

### fiber概念

其实不难，用于解决异步执行中的回调模式带来的各种问题（不易理解，不易阅读等），各种语言天生或第三方扩展出来fiber，也叫纤程，其实就是协程的意思。


### 神作

偶然间看到一篇[译文](http://www.zcfy.cc/article/365#show-last-Point)，感觉简直屌爆了！感谢那么牛逼的原作者，也感激同样牛逼的译者！

文章中有一段话很深刻：

>实现了一个虚拟机，将所有的代码转成状态机，然后通过抛异常来保存所有函数调用栈的状态。这样做意味着每一个函数都被转为一个非常大的 switch 语句，每一个表达式（语句）都对应成一个单独的 case 语句，这样做了之后，我就能够在代码的任意处跳转。

这似乎就是协程的秘密吧（只能算是一种实现方式~）。
