昨天加班在搞一个关于session的问题，有兴趣的童鞋可以去[这里](http://segmentfault.com/q/1010000002406950)观望一下。

总之除了吃惊外就是感觉手脚束缚！现在只能彻底放弃使用默认的`session`机制了，不知道这算是好事儿还是坏事儿。

现在的思路是这样的：

1. 在Servlet过滤器中把`ServletRequest`对象替换为自己的`Wrapper`实例，在其中实现`getSession()`方法，返回我们自定义的`HttpSession`对象
2. 我们实现一个继承了`HttpSession`接口的自定义会话类，用于第一步的返回，该自定义会话类提供接受`sessionId`参数获取会话数据的方法
3. 通过拿在`request`对象中获取的`sessionId`去`redis`中取得对应的登录用户的信息：`sessionUser`对象；
4. 在每次请求处理完后（过滤器最后一行代码）让自定义会话类把会话数据回写入`redis`，由于要持久化对象到`redis`中，所以要选择一个高效能的序列化与反序列化实现。

当然也可以再进一步优化，例如说最后一步，检查如果`MySession`对象没有被修改则不需要回写等。

说到现在，貌似和这篇文章的主题没有半毛钱关系的说~~表着急嘛，这不是要开始说了嘛！

上面的第三步提到，我们自定义的这个会话对象的生命周期应该为：**每次请求**。如果采用`spring`默认提供的`singleton`的话就乱套了，你懂的！

在和同事讨论上面的自定义会话机制的实现时，他叮嘱我说：

> `@Scope("request")`作用域使用时要加上额外的参数（proxyMode）

不过并没有给我说清楚到底为啥。本着打破砂锅问到底的神经，我在GG和百度上搜索了一下，不知道是不是因为关键字写的不合理，总之并没有找到有用的中文资料，大多都是相互转载，而且讲的都是理论，并且用的也配置文件方式而非注解。

无奈只得翻墙查了一下，找到了一篇很好的[文章](http://whyjava.wordpress.com/2010/10/30/spring-scoped-proxy-beans-an-alternative-to-method-injection/)，而且早在2010年就已经总结出来了，为啥国内社区的就没有呢？真的是我搜的方式不对么？

不扯淡了，文章中说：

> Method Injection is useful in scenarios where you need to inject a smaller scope bean in a larger scope bean.  For example, you have to inject a prototype bean inside an singleton bean , on each method invocation of Singleton bean. Just defining your bean prototype, does not create new instance each time a singleton bean is called because container creates a singleton bean only once, and thus only sets a prototype bean once. 

大概意思是，当你把声明为相对范围小的作用域（例如：prototy）对象注入到相对范围大的作用域（例如：singleton）对象时，由于外层对象只会初始化一次，所以会导致内部注入的对象也只会被初始化一次。

道理十分的简单，就是因为这个原因，所以你有意把作用域定义为`prototy`的类可能并不是按照你认为的方式运作。

解决这个问题的方法有多种，比方说你可以使用**Method Injection**，而不是直接注入在类属性中，不过这就要求程序员在使用时需要特别对待这些类。

现在就是***proxyMode**发挥作用的时候：

	@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS, value = "prototype")

这么写，就可以保证该类会按照你大脑中的方式使用了：**每次使用都会重新初始化一个对象**。

源码就不去深究了。8~