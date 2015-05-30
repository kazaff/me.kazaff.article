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
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker); //完成真正的服务暴露逻辑：默认以netty创建server服务来处理远程调用，打算回头专门写一下dubbo使用netty的细节

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


到这里，主线轮廓已经勾勒出来了，我们接下来看一下dubbo和zookeeper之间在服务注册阶段的通信细节，要从上面这个方法中的下面三行下手：

	//registry provider
    final Registry registry = getRegistry(originInvoker);  //根据url参数获取对应的注册中心服务实例，这里就是ZookeeperRegistry

    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl); //向注册中心注册当前暴露的服务的URL

正如注释标明的，第一行会获取invoker中url指定的注册中心实例，我们的情况就是拿到`zookeeperRegistry`。第二行其实就是过滤掉url中的注册中心相关参数，以及过滤器，监控中心等参数，按照我们上面的例子，`registedProviderUrl`大概应该如下：

	dubbo://192.168.153.1:20880/com.alibaba.dubbo.demo.bid.BidService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.bid.BidService&methods=throwNPE,bid&optimizer=com.alibaba.dubbo.demo.SerializationOptimizerImpl&organization=dubbox&owner=programmer&pid=3872&serialization=kryo&side=provider&timestamp=1422241023451

我们主要看第三行，真正完成向zookeeper中注册的工作就是靠register方法完成的，先来看一下zookeeperRegistry的继承关系：

![](http://pic.yupoo.com/kazaff/Ep8RFf0S/7imCY.png)

真正声明register方法的是zookeeperRegistry的父类：FailbackRegistry，从名字就能直观的看出它的作用，主要就是负责注册中心失效重试逻辑的。我们不打算在这里展开说这个话题。好吧，我们继续看zookeeperRegistry的doRegister方法（FailbackRegistry的register方法会调用zookeeperRegistry的doRegister的方法）：

	protected void doRegister(URL url) {
        try {
        	zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));     //参见：http://alibaba.github.io/dubbo-doc-static/Zookeeper+Registry-zh.htm
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

到这里就已经可以告一段落了，需要叮嘱的是`toUrlPath`方法，它的作用就是把url格式化成最终存储在zookeeper中的数据格式，尤其要注意`category`参数，它表示注册类型，如下图：

![](http://pic.yupoo.com/kazaff/Ep8ZmnoV/RmZZL.jpg)

在我们的例子中，最终这次注册就会在对应serverInterface下的providers下创建一个url节点。




subscribe
---

我们再来看看服务消费方对所引用服务的订阅细节，与服务提供方大致一样（忽略集群逻辑），只不过到达`RegistryProtocol`后调用的是`refer`方法：

	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        //处理注册中心的协议，用url中registry参数的值作为真实的注册中心协议
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);   //拿到真正的注册中心实例，我们的例子中就是zookeeperRegistry

        if (RegistryService.class.equals(type)) {   //todo 不太理解，貌似是注册中心服务本身的暴露
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

真正完成订阅是在`doRefer`方法中：

	 private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);   //这个directory把同一个serviceInterface对应的多个invoker管理起来提供概念上的化多为单一，供路由、均衡算法等使用
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());

        //注册自己
        if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false)));
        }

        //订阅目标服务提供方
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
                Constants.PROVIDERS_CATEGORY 
                + "," + Constants.CONFIGURATORS_CATEGORY 
                + "," + Constants.ROUTERS_CATEGORY));

        return cluster.join(directory); //合并所有相同invoker
    }


可见代码和上面给的那个图很吻合，服务消费方不仅会订阅相关的服务，也会注册自身供其他层使用（服务治理）。特别要注意的是订阅时，同时订阅了三个分类类型：**providers，routers，configurators**。目前我们不打算说另外两种类型的意义（因为我也不清楚），后面分析道路由和集群的时候再来扯淡。

继续深挖dubbo中服务消费方订阅服务的细节，上面方法中最终把订阅细节委托给`RegistryDirectory.subscribe`方法，注意，这个方法接受的参数，此时的url已经把`category`设置为`providers，routers，configurators`：

	public void subscribe(URL url) {
        setConsumerUrl(url);
        registry.subscribe(url, this);
    }

这里`registry`就是zookeeperRegistry，这在`doRefer`方法可以看到明确的注入。然后和注册服务时一样，订阅会先由`FailbackRegistry`完成失效重试的处理，最终会交给`zookeeperRegistry.doSubscribe`方法。zookeeperRegistry实例拥有ZookeeperClient类型引用，该类型对象封装了和zookeeper通信的逻辑（默认是使用zkclient客户端），这里需要注意的一点，小爷我就被这里的一个数据结构卡住了一整天：

	private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();

一开始很不理解，为何要在url和NotifyListener之间再搞一个ChildListener接口出来，后来反复查看zkclient的文档说明和dubbo注册中心的设计，才悟出来点门道。这个**ChildListener接口用于把zkclient的事件（IZkChildListener）转换到registry事件（NotifyListener）**。这么做的深意不是特别的理解，可能是因为我并没有太多zookeeper的使用经验导致的，这里的做法**可以更好的把zkclient的api和dubbo真身的注册中心逻辑分离开**，毕竟dubbo除了zkclient以外还可以选择curator。从dubbo源码中可以看出，架构师和开发人员对面向对象和设计模式的理解非常的深刻，合理的运用继承和组合，打造了非常灵活的一套系统，保证概念统一的前提下展现了非常强大的多态性，感叹！

这样走一圈下来，关于服务订阅的大致流程就描述清楚了，部分问题需要留到未来再解决了。



notify
---

最后看一下注册推送细节，在订阅时你会注意到，订阅真正操作的是用`RegistryDirectory`类型封装过的对象，这个类型实现了一个接口`NotifyListener`（前面我们已经提到这个接口了），该接口用于描述支持推送通知逻辑：
	
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

前面提到了ChildListener接口，dubbo靠它把zkclient的事件转换成自己的事件类型，如果从代码上来看确实有点绕，事件的流程我手绘了一下：

[![](http://pic.yupoo.com/kazaff/Epc4RElK/medish.jpg)](http://pic.yupoo.com/kazaff/Epc4RElK/TJLi1.png)

我们主要看一下RegistryDirectory的notify方法：

	public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            //允许不同类型的数据分开通知，比如：providers, consumers, routers, overrides，允许只通知其中一种类型，但该类型的数据必须是全量的，不是增量的。
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category) 
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category) 
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators 更新缓存的服务提供方配置规则
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

        // 合并override参数
        List<Configurator> localConfigurators = this.configurators; // local reference
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && localConfigurators.size() > 0) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }

        // providers
        refreshInvoker(invokerUrls);
    }

dubbo提供了强大的服务治理功能，所以这里在每次消费方接受到注册中心的通知后，大概会做下面这些事儿：

- [更新服务提供方配置规则](http://alibaba.github.io/dubbo-doc-static/Configurator+Rule-zh.htm)
- [更新路由规则](http://alibaba.github.io/dubbo-doc-static/Router+Rule-zh.htm)
- 重建invoker实例

前两件事儿我们放在分析路由，过滤器，集群的时候再讲，我们这里主要看dubbo如何“重建invoker实例”，也就是最后一行代码调用的方法`refreshInvoker`：

	private void refreshInvoker(List<URL> invokerUrls){
        if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
                && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) { //如果传入的参数只包含一个empty://协议的url，表明禁用当前服务
            this.forbidden = true; // 禁止访问
            this.methodInvokerMap = null; // 置空列表
            destroyAllInvokers(); // 关闭所有Invoker
        } else {
            this.forbidden = false; // 允许访问
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference

            if (invokerUrls.size() == 0 && this.cachedInvokerUrls != null){ //如果传入的invokerUrl列表是空，则表示只是下发的override规则或route规则，需要重新交叉对比，决定是否需要重新引用
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<URL>();
                this.cachedInvokerUrls.addAll(invokerUrls);//缓存invokerUrls列表，便于交叉对比
            }

            if (invokerUrls.size() ==0 ){
            	return;
            }

            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls) ;// 将URL列表转成Invoker列表
            Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // 换方法名映射Invoker列表

            // state change
            //如果计算错误，则不进行处理.
            if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0 ){
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :"+invokerUrls.size() + ", invoker.size :0. urls :"+invokerUrls.toString()));
                return ;
            }

            this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
            this.urlInvokerMap = newUrlInvokerMap;

            try{
                destroyUnusedInvokers(oldUrlInvokerMap,newUrlInvokerMap); // 关闭未使用的Invoker
            }catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }


好吧，到这里我们已经完成了服务通知的业务逻辑，有兴趣的童鞋可以深究一下`toInvokers`方法，它又会走一遍**url->invoker**的逻辑（服务引用）。


那么，就先到这里吧，再会~


