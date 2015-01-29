对于分布式服务架构，解决服务的发现问题，引入了注册中心中间件，从而很好的解决了服务双方（消费方和提供方）的直接依赖问题。这种解耦的意义是非凡的，不仅在程序运行时保证了灵活性，在开发阶段也使得快速迭代成为了可能，甚至在运维层面也提供了非常好的自由度。

夸了这么多，但要实现一个完美的注册中心系统却不是一件那么容易的事儿，你必须时刻注意关注它的可用性（包括**稳定，实时和高效**），这一点在任何一款分布式系统中都是件很复杂的事儿。当然这篇文章并不是打算摆平这么个庞然大物，我们只是从dubbo和zookeeper之间的关系来了解一下在dubbo架构中注册中心的相关知识：

![](http://pic.yupoo.com/kazaff/EogBsej0/gvnAo.jpg)

上图是官方给出的一张描述服务提供方、服务消费方和注册中心的关系图，其实dubbo提供多种注册中心实现，不过常用的就是zookeeper，我们也就拿它来当例子来分析。从图中可见，**消费方远程调用服务方是不通过注册中心的**，这有效的降低了注册中心的负载，也不会存在明显的单点瓶颈（尽管可以搭建注册中心的集群，但每次调用都走注册中心的话肯定对性能产生较大的伤害）。

官方提供的规则是：

- 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小；
- 注册中心，服务提供者，服务消费者三者之间均为长连接；
- 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者；
- 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者；
- 注册中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表；
- 注册中心是可选的，服务消费者可以直连服务提供者；
- 注册中心对等集群，任意一台宕掉后，将自动切换到另一台。

好啦，更多的理论我就不转载了，官方已经描述的非常详细了，我们按照老套路，从代码级别看一下dubbo到底是怎样实现的。



register
---

我们需要承接之前的[文章](http://blog.kazaff.me/2015/01/26/dubbo%E5%A6%82%E4%BD%95%E4%B8%80%E6%AD%A5%E4%B8%80%E6%AD%A5%E6%8B%BF%E5%88%B0bean/)里的例子，从拿到需要暴露成服务的url开始：

	registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo%3A%2F%2F192.168.153.1%3A20880%2Fcom.alibaba.dubbo.demo.bid.BidService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.bid.BidService%26methods%3DthrowNPE%2Cbid%26optimizer%3Dcom.alibaba.dubbo.demo.SerializationOptimizerImpl%26organization%3Ddubbox%26owner%3Dprogrammer%26pid%3D3872%26serialization%3Dkryo%26side%3Dprovider%26timestamp%3D1422241023451&organization=dubbox&owner=programmer&pid=3872&registry=zookeeper&timestamp=1422240274186

以这个url为基准暴露服务的话，dubbo会首先会根据指定协议（`registry`）拿到对应的protocol（`RegistryProtocol`），这部分是怎么做到的呢？还是之前通过IDE拿到的dubbo动态创建的protocol自适应扩展点，我们重点看`export`方法：


	package com.alibaba.dubbo.rpc;
	import com.alibaba.dubbo.common.extension.ExtensionLoader;
	
	public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
		
		......
		
		public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
			if (arg0 == null) 
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
			
			if (arg0.getUrl() == null) 
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
			
			com.alibaba.dubbo.common.URL url = arg0.getUrl();
			String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );	//注意这句，根据我们的例子，extName=registry
			
			if(extName == null) 
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
			
			com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);	//根据扩展点加载规则，最终拿到RegistryProtocol实例。
			
			return extension.export(arg0);
		}
		
		......
	}


	
我们需要注意`RegistryProtocol`的私有属性：

	private Protocol protocol;
    
    public void setProtocol(Protocol protocol) {
        this.protocol = protocol;   //由SPI机制为其赋予一个protocol的自适应扩展点（动态创建的）
    }

这个属性真正被赋值的地方是在SPI机制中为扩展点注入的阶段（`injectExtension`方法）：

	private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);	//注意这里，我们的例子中，这个object会是SPI动态创建的自适应扩展点实例：Protocol$Adpative
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }

有点乱，回到`RegistryProtocol`类，我们知道，在服务暴露阶段，会调用它的`export`方法，在这个方法里会完成服务的注册逻辑：

	public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //export invoker
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker); //完成真正的服务暴露逻辑

        //registry provider
        final Registry registry = getRegistry(originInvoker);  //根据url参数获取对应的注册中心服务实例，这里就是ZookeeperRegistry

        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
        registry.register(registedProviderUrl); //向注册中心注册当前暴露的服务的URL

        // 订阅override数据
        // FIXME 提供者订阅时，会影响同一JVM既暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        //保证每次export都返回一个新的exporter实例
        return new Exporter<T>() {
            public Invoker<T> getInvoker() {
                return exporter.getInvoker();
            }
            public void unexport() {
            	try {
            		exporter.unexport();
            	} catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
                try {
                	registry.unregister(registedProviderUrl);
                } catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
                try {
                	overrideListeners.remove(overrideSubscribeUrl);
                	registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
                } catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
            }
        };
	}

dubbo默认会使用zkclient与zookeeper服务进行通信，关于服务的注册就说到这里吧。





subscribe
---

我们再来看看服务消费方对所引用服务的订阅细节，与服务提供方大致一样（忽略集群逻辑），只不过到达`RegistryProtocol`后调用的是`refer`方法：

	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        //处理注册中心的协议，用url中registry参数的值作为真实的注册中心协议
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);   //拿到真正的注册中心实例，我们的例子中就是zookeeper

        if (RegistryService.class.equals(type)) {   //todo 不太理解，貌似是注册注册中心服务本身的暴露
        	return proxyFactory.getInvoker((T) registry, type, url);
        }

        //分组聚合处理，http://alibaba.github.io/dubbo-doc-static/Merge+By+Group-zh.htm
        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
        String group = qs.get(Constants.GROUP_KEY);
        if (group != null && group.length() > 0 ) {
            if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                    || "*".equals( group ) ) {
                return doRefer( getMergeableCluster(), registry, type, url );
            }
        }

        return doRefer(cluster, registry, type, url);
    }

真正完成订阅是在`doRefer`方法中，由于代码非常直观（不包含集群逻辑）就不在赋值粘贴了。

从dubbo源码中可以看出，架构师和开发人员对面向对象和设计模式的理解非常的深刻，合理的运用继承和组合，打造了非常灵活的一套系统，保证概念统一的前提下展现了非常强大的多态性，感叹！


notify
---

最后看一下注册推送细节，在订阅时你会注意到，订阅真正操作的是用`RegistryDirectory`类型封装过的对象，这个类型实现了一个接口`NotifyListener`，该接口用于描述支持推送通知逻辑：
	
	public interface NotifyListener {
	
	    /**
	     * 当收到服务变更通知时触发。
	     * 
	     * 通知需处理契约：<br>
	     * 1. 总是以服务接口和数据类型为维度全量通知，即不会通知一个服务的同类型的部分数据，用户不需要对比上一次通知结果。<br>
	     * 2. 订阅时的第一次通知，必须是一个服务的所有类型数据的全量通知。<br>
	     * 3. 中途变更时，允许不同类型的数据分开通知，比如：providers, consumers, routers, overrides，允许只通知其中一种类型，但该类型的数据必须是全量的，不是增量的。<br>
	     * 4. 如果一种类型的数据为空，需通知一个empty协议并带category参数的标识性URL数据。<br>
	     * 5. 通知者(即注册中心实现)需保证通知的顺序，比如：单线程推送，队列串行化，带版本对比。<br>
	     * 
	     * @param urls 已注册信息列表，总不为空，含义同{@link com.alibaba.dubbo.registry.RegistryService#lookup(URL)}的返回值。
	     */
	    void notify(List<URL> urls);
	}

更多细节，将在分析dubbo的Router、Filter和cluster时再细说，因为这几个部分在概念上比较靠近。



好了， 先到这里，再见。