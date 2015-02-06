如果说单单只完成远程调用的话，dubbo还算不上是一个合格的SOA服务架构，而它之所以那么碉堡，是因为它还提供了服务治理的功能，今天就让我们来研究一下关于服务治理，dubbo都做了什么。

听起来服务治理挺高大上的，但其实做的都是一些非常琐碎的事儿，了解了dubbo的做法，你就会发觉其实一切并没有想的那么复杂。远程调用要解决的最本质的问题是通信，通信就好像人和人之间的互动，有效的沟通建立在双方彼此了解的基础上（我们团队在沟通上就有死穴），同样道理，服务提供方和消费方之间要相互了解对方的基本情况，才能做到更好的完成远程调用。这里面就要提到dubbo的做法：**URL**。

前几篇中大量提到dubbo的分层之间是依靠什么纽带工作的：invoker，没错，比invoker更low的就是URL，这是dubbo带给我的另一个非常重要的经验。才疏学浅，并不知道dubbo是借鉴的哪里，但影响了全世界的WEB就是依赖URL机制建立了互联网帝国的！

依赖URL机制，dubbo不仅打通了通信两端，而且还靠URL机制完成了服务治理的任务。我们可以先看一下这些内容：

- [路由规则](http://alibaba.github.io/dubbo-doc-static/Router+Rule-zh.htm)
- [配置规则](http://alibaba.github.io/dubbo-doc-static/Configurator+Rule-zh.htm)
- [服务降级](http://alibaba.github.io/dubbo-doc-static/Service+Degradation-zh.htm)
- [负载均衡](http://alibaba.github.io/dubbo-doc-static/Load+Balance-zh.htm)

其实dubbo的路由和集群是在服务暴露，服务发现，服务引用中透明完成的，暴露给其他层的是同一个接口类型：Invoker。[dubbo官方](http://alibaba.github.io/dubbo-doc-static/Fault+Tolerance-zh.htm)提供了一张巨清晰无比的图：

![](http://pic.yupoo.com/kazaff/EopVm33y/Rzbmx.jpg)

这张图是站在服务消费方的视角来看的（**dubbo的服务治理都是针对服务消费方的**），当业务逻辑中需要调用一个服务时，你真正调用的其实是dubbo创建的一个proxy，该proxy会把调用转化成调用指定的invoker（cluster封装过的）。而在这一系列的委托调用的过程里就完成了服务治理的逻辑，最终完成调用。



集群
---

当相同服务由多个提供方同时提供时，消费方就需要有个选择的步骤，就好比你去电商平台买一本书，你自然会看一下哪儿买的最便宜。同样，消费方也需要根据需求选择到底使用哪个提供方的服务，而集群的主要作用就是从**容错**的维度来帮我们选择合适的服务提供方。

我们需要从`Protocol`接口的部分定义开始：

	
	/**
     * 引用远程服务：<br>
     * 1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。<br>
     * 2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。<br>
     * 3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。<br>
     * 
     * @param <T> 服务的类型
     * @param type 服务的类型
     * @param url 远程服务的URL地址
     * @return invoker 服务的本地代理
     * @throws RpcException 当连接服务提供方失败时抛出
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

注意这个方法的返回值，根据我们这一系列文章一直使用的场景（有注册中心），看一下`RegistryProtocol.doRefer`方法的最后一行：

	return cluster.join(directory);

之前的文章提到过这个`directory`，它在后面我们会再次提到，这里你只需要知道它不是我们需要的invoker类型，那么这个`cluster`对象又是什么呢？根据dubbo的SPI机制，我们知道，这里的cluster是动态创建的自适应扩展点：

	package com.alibaba.dubbo.rpc.cluster;
	import com.alibaba.dubbo.common.extension.ExtensionLoader;
	
	public class Cluster$Adpative implements com.alibaba.dubbo.rpc.cluster.Cluster {
		
		public com.alibaba.dubbo.rpc.Invoker join(com.alibaba.dubbo.rpc.cluster.Directory arg0) throws com.alibaba.dubbo.rpc.cluster.Directory {
			
			if (arg0 == null)
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument == null");
		
			if (arg0.getUrl() == null)
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.cluster.Directory argument getUrl() == null");
			
			com.alibaba.dubbo.common.URL url = arg0.getUrl();
			String extName = url.getParameter("cluster", "failover");	//默认使用failover实现
		
			if(extName == null)
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.cluster.Cluster) name from url(" + url.toString() + ") use keys([cluster])");
		
			com.alibaba.dubbo.rpc.cluster.Cluster extension = (com.alibaba.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
		
			return extension.join(arg0);
		}
	}

我们再来看一下默认使用的`FailoverCluster`定义：

	/**
	 * 失败转移，当出现失败，重试其它服务器，通常用于读操作，但重试会带来更长延迟。 
	 * 
	 * <a href="http://en.wikipedia.org/wiki/Failover">Failover</a>
	 * 
	 * @author william.liangf
	 */
	public class FailoverCluster implements Cluster {
	
	    public final static String NAME = "failover";
	
	    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	        return new FailoverClusterInvoker<T>(directory);
	    }
	}


看到了吗，这就是一开始图上的所表明的，**cluster把存有多个invoker的directory对象封装成了单个的invoker**。我们在来看一下`FailoverClusterInvoker`类的UML图：

![](http://pic.yupoo.com/kazaff/EpluFFDn/KIoh9.png)

根据官方文档的说明，dubbo提供了多种集群容错方案供我们直接使用，至于各种集群容错模式算法可以交给大家自己阅读源码来消化了，后面只会以`FailoverClusterInvoker`为基准来讨论。




路由和配置
---

如果说集群帮我们以容错的维度来完成选择，那么路由和配置是在更细颗粒度的层面做的选择，具体有多细，可以从官方文档和dubbo-admin管理后台来了解，如下多图：

![](http://pic.yupoo.com/kazaff/EplGoQvX/HGn7S.png)

![](http://pic.yupoo.com/kazaff/EplGpS9R/s9XJ2.png)

![](http://pic.yupoo.com/kazaff/EplGqBu7/wFdrA.png)

![](http://pic.yupoo.com/kazaff/EplH3FBD/n5KTo.png)

![](http://pic.yupoo.com/kazaff/EplGCTY9/BtehS.png)

总之很细吧，这么多配置参数最终都会交给谁来管理呢？


我们需要从`Directory`接口出发，你应该想到了该接口的一个实现类：

![](http://pic.yupoo.com/kazaff/Epl1KPlv/RI5VI.png)

没错，就是这个`RegistryDirectory`，它在服务引用时被创建，用于充当url与多invoer的代理（或者叫目录类更合适），从源码可以看出，当服务引用时，对应该服务的目录类实例会负责向注册中心（zookeeper）订阅该服务，第一次订阅会同步拿到当前服务节点的详细信息（也就是所有提供服务的提供方信息，包括：地址，配置，路由等），然后该目录实例会根据这些信息来为后续的服务调用提供支撑。

根据描述我们可以锁定代码位置，`RegistryDirectory.notify`：

	......
	// configurators 更新缓存的服务提供方动态配置规则
    if (configuratorUrls != null && configuratorUrls.size() >0 ){
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers  更新缓存的路由配置规则
    if (routerUrls != null && routerUrls.size() >0 ){
        List<Router> routers = toRouters(routerUrls);
        if(routers != null){ // null - do nothing
            setRouters(routers);
        }
    }
	......

这些配置在什么时候发挥作用呢？往下看~

前面说到当调用invoker时，其实调用的是集群模块封装过的代理invoker，那么以我们的场景为例，最终会被调用的是`FailoverClusterInvoker.invoke`：

	public Result invoke(final Invocation invocation) throws RpcException {

        checkWheatherDestoried();

        LoadBalance loadbalance;

		//这里就是路由，配置等发挥作用地方，返回所有合法的invoker供集群做下一步的筛选        
        List<Invoker<T>> invokers = list(invocation);

        if (invokers != null && invokers.size() > 0) {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
        } else {
            loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
        }
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }

再来看一下这个`list`方法的定义：

	protected  List<Invoker<T>> list(Invocation invocation) throws RpcException {
    	List<Invoker<T>> invokers = directory.list(invocation);
    	return invokers;
    }

很直接的把选择合法invoker的工作交给了我们的目录类实例，再来看一下directory是怎么list的：

	public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed){
            throw new RpcException("Directory already destroyed .url: "+ getUrl());
        }

        //根据请求服务的相关参数（方法名等）返回对应的invoker列表
        List<Invoker<T>> invokers = doList(invocation);

        List<Router> localRouters = this.routers; // local reference
        if (localRouters != null && localRouters.size() > 0) {
            for (Router router: localRouters){
                try {
                    //是否在每次调用时执行路由规则，否则只在提供者地址列表变更时预先执行并缓存结果，调用时直接从缓存中获取路由结果。
                    //如果用了参数路由，必须设为true，需要注意设置会影响调用的性能，可不填，缺省为flase。
                    if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                        invokers = router.route(invokers, getConsumerUrl(), invocation);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
                }
            }
        }

        return invokers;
    }

到这里我们就已经把路由和配置的相关流程介绍完了，至于路由和配置的具体参数是如何发挥效果的，这个大家可以结合文档提供的实例直接阅读源码即可。




负载均衡
---

到了负载均衡环节，维度就成了**性能**，这个词你可以从gg里搜索大量的相关文献，我就不在这里卖弄了。把焦点拉回到`FailoverClusterInvoker.invoke`方法：

	......
	if (invokers != null && invokers.size() > 0) {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    } else {	
		//todo 如果invokers为空，还有必要往下走么？
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }

    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    return doInvoke(invocation, invokers, loadbalance);
	......


可以看到，这里就创建了要使用的负载均衡算法，我们接下来看一下到底是怎么使用这个`loadbalance`对象的，一路跟踪到`AbstractClusterInvoker.doselect`方法：

	......
	Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
	......

（其实本人并不喜欢这样截取部分代码展示，因为会让读者很窘迫，不过请相信我，这里这么做可以很好的排除干扰。）可见，最终是依靠负载均衡这最后一道关卡我们总算拿到了要调用的invoker。我们依然不去过多在意算法细节，到目前为止，负载均衡的流程也介绍完了。



---

其实dubbo服务治理相关的内容还有很多，官方文档也提供了详细的[说明](http://alibaba.github.io/dubbo-doc-static/User+Guide-zh.htm#UserGuide-zh-%E7%A4%BA%E4%BE%8B)，希望大家都能成为dubbo大牛，bye~