今天要聊一个可能被其他dubbo源码研究的童鞋容易忽略的话题：Filter和Listener。

我们先来看一下这两个概念的官方手册：

- [拦截器](http://alibaba.github.io/dubbo-doc-static/Filter+SPI-zh.htm)
- 监听器：[引用监听器](http://alibaba.github.io/dubbo-doc-static/InvokerListener+SPI-zh.htm)和[暴露监听器](http://alibaba.github.io/dubbo-doc-static/ExporterListener+SPI-zh.htm)


老实说，依赖之前的源码分析经验，导致我饶了很大的弯路，一直找不到`filter`和`listener`被使用的位置。看过前几篇文章的朋友应该也有这个疑惑，为什么按照url参数去匹配框架的执行流程，死活找不到dubbo注入拦截器和监听器的位置呢？

	ReferenceConfig -->  RegistryProtocol --> DubboProtocol  -->  invoker  -->  exporter

按照这个调用流程，没错啊，可每一个环节都没有使用`filter`和`listener`属性的痕迹，有点抓瞎了啊。要说用好IDE确实很重要啊，光靠脑子想真的很伤身，下面来看一下谜底。


先来回忆一下dubbo的SPI机制，根据接口类型，dubbo会去读取并解析对应的配置文件，从中拿到对应的扩展点实现，好，我们先来看一下`Protocol`接口对应的配置文件：

	registry=com.alibaba.dubbo.registry.integration.RegistryProtocol
	dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol			
	filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper			#注意这一行
	listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper		#注意这一行
	mock=com.alibaba.dubbo.rpc.support.MockProtocol
	injvm=com.alibaba.dubbo.rpc.protocol.injvm.InjvmProtocol
	rmi=com.alibaba.dubbo.rpc.protocol.rmi.RmiProtocol
	hessian=com.alibaba.dubbo.rpc.protocol.hessian.HessianProtocol
	com.alibaba.dubbo.rpc.protocol.http.HttpProtocol
	com.alibaba.dubbo.rpc.protocol.webservice.WebServiceProtocol
	thrift=com.alibaba.dubbo.rpc.protocol.thrift.ThriftProtocol
	memcached=com.alibaba.dubbo.rpc.protocol.memcached.MemcachedProtocol
	redis=com.alibaba.dubbo.rpc.protocol.redis.RedisProtocol
	rest=com.alibaba.dubbo.rpc.protocol.rest.RestProtocol


我们已经找到了`filter`和`listener`对应的扩展点了。接下来看一下它们是怎么一步一步的被注入到上面的流程里的。

在`ReferenceConfig`类中我们会引用和暴露对应的服务，我们以服务引用为场景来分析：

	get()  -->  init()  -->   createProxy()
									|
									+--->  invoker = refprotocol.refer(interfaceClass, urls.get(0));


注意上面提到的这一行代码，这里的`refprotocol`是引用的`Protocol$Adpative`，这个类是dubbo的SPI机制动态创建的自适应扩展点，我们在之前的文章中已经介绍过，看一下它的`refer`方法细节：

	public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
		if (arg1 == null)
			throw new IllegalArgumentException("url == null");
		
		com.alibaba.dubbo.common.URL url = arg1;
		String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
		
		if(extName == null) 
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
		
		//注意这一行，根据url的协议名称选择对应的扩展点实现
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		
		return extension.refer(arg0, arg1);
	}

乍一看，并没有感觉有什么蹊跷，不过在单步调试中就会出现"诡异"现象（由于该类是动态创建的，所以该方法并不会被单步到，所以为分析带来了一定的干扰），我们得再往回倒一下，之前在[dubbo中SPI的基础](http://blog.kazaff.me/2015/01/15/dubbo%E4%B8%ADSPI%E7%9A%84%E5%9F%BA%E7%A1%80--Cooma%E5%BE%AE%E5%AE%B9%E5%99%A8/)中曾经分析过`ExtensionLoader`的源码，但是当时由于了解的不够确实忽略了一些细节。

我们再来看一下它的执行流程：

	getExtension()  -->  createExtension()
								|
								+-->  	......
										Set<Class<?>> wrapperClasses = cachedWrapperClasses;
							            if (wrapperClasses != null && wrapperClasses.size() > 0) {
							                for (Class<?> wrapperClass : wrapperClasses) {  //装饰器模式
							                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
							                }
							            }
										......


一看到这行代码，就知道关键点在这里，这种写法刚好就是和常见的拦截器和监听器的实现方法吻合，而且事实证明也确实是在这个地方完成的注入，那么我们就需要看一下这个`cachedWrapperClasses`到到底存了什么？

我们最后看一下`ExtensionLoader.loadFile`方法，它是负责解析我们开头提到的那个SPI扩展点配置文件的，它会依次扫描配置文件的每一行，然后根据配置内容完成等号两边的键值对应关系，例如：

	test=com.alibaba.dubbo.rpc.filter.TestFilter

`loadFile`的任务就是把`test`和解析过以后的`TestFilter`类关系对应上，供以后的`getExtension`查找使用。注意看其中的这几行代码：

	......
	 clazz.getConstructor(type); //判断是否为wrapper实现
    Set<Class<?>> wrappers = cachedWrapperClasses;
    if (wrappers == null) {
        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
        wrappers = cachedWrapperClasses;
    }
    wrappers.add(clazz);
	......

这里就完成了`cachedWrapperClasses`的初始化，它根据查看配置文件中定义的扩展点实现是否包含一个带有当前类型的构造方法为条件，确定哪些是wrapper，这样我们就可以发现：

	filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
	listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper

这两行命中了。这也是之后在真正获取`protocol`扩展点时会动态注入的两个重要包装类，前者完成拦截器，后者完成监听器。



至于拦截器和监听器的使用方法，我实在不知道除了官方提到的内容以外还有什么好补充的了，那就写到这里吧~
