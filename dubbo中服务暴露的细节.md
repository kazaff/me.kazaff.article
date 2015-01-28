[前一篇]()文章只是分析了一下从xml到service的代码流程细节，从中我们发现了一些架构层面的设计，小弟我非常的在意。所以我们这次在原先的基础上再深挖一点，看看能否出油。

这篇文字里会出现大量本尊的瞎掰（其实每篇都挺能瞎掰的），希望大家多多提醒，打脸什么的我丫根本就不怕。

服务接口类型的Wrapper处理
---

在ServiceConfig.java中的doExportUrlsFor1Protocol方法中，我们看到，在得到最终URL之前，会执行下面的代码逻辑：

	String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
    if(methods.length == 0) {
        logger.warn("NO method found in service interface " + interfaceClass.getName());
        map.put("methods", Constants.ANY_VALUE);
    }
    else {
        map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
    }

那么这个`Wrapper.getWrapper()`的作用是什么呢？从代码层面来看，它按照dubbo自身的需求完成了类似java反射的工作，无非就是**根据给定的服务实现接口类型，按照dubbo的要求提供读取该类型的相关类型信息的方法**，有些绕口。这么做，可以让dubbo更自由的控制获取类型信息的相关操作，同时也一定程度的统一了调用方式。举个例子，wrapper后的对象可以在其他业务调时有效的屏蔽那些不希望被感知的原类型数据信息，对应设计模式的“[适配器模式](http://blog.chinaunix.net/uid-22283027-id-3488042.html)”。

除此之外，`Wrapper`还提供了对象缓存池的概念来提升性能：

	Wrapper ret = WRAPPER_MAP.get(c);
    if( ret == null )
    {
        ret = makeWrapper(c);
        WRAPPER_MAP.put(c,ret);
    }
    return ret;

至于还有没有其他更深层次的作用，期待您的补充。


暴露前的proxy处理
---

回顾之前提过的bean转service过程，我们当时提到了`url`，它作为不同层之间通信的keyword起到了重要的作用，但仅仅有key是不够的，**如何通过key找到实际提供服务的bean才是本质**。

dubbo的[Invoker](http://alibaba.github.io/dubbo-doc-static/RPC+Detail-zh.htm)模型是非常关键的概念，看下图：

![](http://pic.yupoo.com/kazaff/Eo4DsJLe/10mLOC.png)

图中可以很清楚的看到`url`，`ref`，`interface`，`ProxyFactory`，`Protocol`和`Invoker`之间的关系，代码上来看，就是下面这两行：

	......
	Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);

    Exporter<?> exporter = protocol.export(invoker);
	......


看似简单的两次调用之中，其实执行了非常多的逻辑。我们先来看一下这里`proxyFactory`对象是怎么拿到的（在`ServiceConfig`中声明了该静态属性）：

	private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

对，没错，扩展点加载规则。不过较为麻烦的是从`ProxyFactory`接口中只能看出其使用的默认扩展点为"javassist"，可代码中却指定使用的是自适应扩展点，看一下配置文件中定义了什么：

	stub=com.alibaba.dubbo.rpc.proxy.wrapper.StubProxyFactoryWrapper
	jdk=com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory
	javassist=com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory

依次查阅这三个类的源码，并没有发现自适应扩展点的痕迹，也就是说最终dubbo会动态创建一个自适应扩展点类，这里作为外行新手，需要吐槽一下，**dubbo中大量的动态类生产方式采用的是字符串拼接源码方式**，这给代码审阅带来了非常大的困难，我相信即便是项目开发人员很难一次就写正确所有的逻辑。

不过幸好，我们有聪明的[IDE](http://www.jetbrains.com/products.html)，在辅助工具的帮助下，我们可以还原出dubbo针对ProxyFactory所动态创建的自适应扩展点类的完整代码：

	package com.alibaba.dubbo.rpc;
	import com.alibaba.dubbo.common.extension.ExtensionLoader;

	public class ProxyFactory$Adpative implements com.alibaba.dubbo.rpc.ProxyFactory {
	
		public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
			if (arg0 == null) 
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
	
			if (arg0.getUrl() == null)
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
			
			com.alibaba.dubbo.common.URL url = arg0.getUrl();
			String extName = url.getParameter("proxy", "javassist");
			if(extName == null)
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
			
			com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
			
			return extension.getProxy(arg0);
		}
	
		public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws java.lang.Object {
			if (arg2 == null)
				throw new IllegalArgumentException("url == null");
	
			com.alibaba.dubbo.common.URL url = arg2;
			String extName = url.getParameter("proxy", "javassist");
			if(extName == null)
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
	
			com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
	
			return extension.getInvoker(arg0, arg1, arg2);
		}
	}

最终我们可以看出，dubbo会默认调用`com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory`作为proxyFactory的实际逻辑：

	 public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper类不能正确处理带$的类名
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, 
                                      Class<?>[] parameterTypes, 
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

哇塞，一下子就通透了。之所以命名为`JavassistProxyFactory`，就是因为它使用的是前面提到的`Wrapper`实例。逻辑很简单，直接完成了Invoker实例的创建，我们前面说了，Invoker是个很关键的概念，它的一个抽象定义如下：

	public abstract class AbstractProxyInvoker<T> implements Invoker<T> {
    
	    private final T proxy;
	    
	    private final Class<T> type;
	    
	    private final URL url;
	
	    public AbstractProxyInvoker(T proxy, Class<T> type, URL url){
	        if (proxy == null) {
	            throw new IllegalArgumentException("proxy == null");
	        }
	        if (type == null) {
	            throw new IllegalArgumentException("interface == null");
	        }
	        if (! type.isInstance(proxy)) {
	            throw new IllegalArgumentException(proxy.getClass().getName() + " not implement interface " + type);
	        }
	        this.proxy = proxy;
	        this.type = type;
	        this.url = url;
	    }
	
	    public Class<T> getInterface() {
	        return type;
	    }
	
	    public URL getUrl() {
	        return url;
	    }
	
	    public boolean isAvailable() {
	        return true;
	    }
	
	    public void destroy() {
	    }
	
	    public Result invoke(Invocation invocation) throws RpcException {
	        try {
	            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
	        } catch (InvocationTargetException e) {
	            return new RpcResult(e.getTargetException());
	        } catch (Throwable e) {
	            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
	        }
	    }
	    
	    protected abstract Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable;
	
	    @Override
	    public String toString() {
	        return getInterface() + " -> " + getUrl()==null?" ":getUrl().toString();
	    }
	}


也挺简单的，值得关注的是`invoke`方法，该方法是Invoker真正可以被其他对象调用的方法，逻辑也很简单，主要注意它的参数和返回值类型。

目前为止，我们已经完成了Invoker的转换，剩下的就是暴露服务的底层实现了。


服务暴露的底层实现
---

老规矩，为了更好的理解代码意图，我们利用IDE把dubbo动态创建的protocol自适应扩展点类的代码还原：

	package com.alibaba.dubbo.rpc;
	import com.alibaba.dubbo.common.extension.ExtensionLoader;
	
	public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {
		
		public int getDefaultPort() {
			throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
		}
		
		public void destroy() {
			throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
		}
		
		public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
			if (arg0 == null) 
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
			
			if (arg0.getUrl() == null) 
				throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
			
			com.alibaba.dubbo.common.URL url = arg0.getUrl();
			String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
			
			if(extName == null) 
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
			
			com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
			
			return extension.export(arg0);
		}
		
		public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
			if (arg1 == null)
				throw new IllegalArgumentException("url == null");
			
			com.alibaba.dubbo.common.URL url = arg1;
			String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
			
			if(extName == null) 
				throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
			
			com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
			
			return extension.refer(arg0, arg1);
		}
	}

目前我们主要关注`export`方法，可以看出这个自适应扩展点的逻辑也很简单，从url中找出适配的协议参数，并获取指定协议的扩展点实现，并调用其`export`方法，一气呵成。

我们主要分析默认协议`dubbo`，所以接下来要分析的是`DubboProtocol.java`：

	public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl(); //拿到url，可见url的重要性
        
        // export service.
        String key = serviceKey(url);   //根据url中的设置拿到能够唯一标识该exporter的key
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);    //其实export只是简单的在invoker上封装了一层，提供了更“语义”的接口
        exporterMap.put(key, exporter);
        
        //export an stub service for dispaching event
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY,Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice){
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0 ){
                if (logger.isWarnEnabled()){
                    logger.warn(new IllegalStateException("consumer [" +url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        openServer(url);    //根据url中定义的相关参数（协议，host，port等）创建服务，默认使用的是netty

        // modified by lishen
        optimizeSerialization(url);

        return exporter;
    }

其实`exporter`从代码层面来看只是对invoker封装了一层调用接口而已，并没有做其他什么转化操作，当然可以利用这一层封装来完成一些自定义逻辑，例如`DubboExporter`只是做了一层缓存处理。

到目前为止，我们可以知道，每个serviceConfig实例会根据配置中定义的注册中心和协议最终得到多个exporter实例。当有调用过来时，dubbo会通过请求消息中的相关信息来确定调用的exporter，并最终调用其封装的invoker的invoke方法完成业务逻辑。

更多的细节，推荐看一下之前发的这篇文章：[dubbo协议下的单一长连接与多线程并发如何协同工作](http://blog.kazaff.me/2014/09/20/dubbo%E5%8D%8F%E8%AE%AE%E4%B8%8B%E7%9A%84%E5%8D%95%E4%B8%80%E9%95%BF%E8%BF%9E%E6%8E%A5%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91%E5%A6%82%E4%BD%95%E5%8D%8F%E5%90%8C%E5%B7%A5%E4%BD%9C/)

不打算继续挖下去了，因为打算另起一篇专门聊dubbo中使用netty的文章。那就先瞎扯到这里吧，希望大牛能对上面的错误之处能够无情的给予打击。