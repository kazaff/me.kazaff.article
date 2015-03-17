我们今天来看一下dubbox多出来的那个“x”都包含什么，当然一定会存在遗落，毕竟我是从一个第三方使用者的角度来总结的。之前也写了几篇关于dubbo的文章，虽然都加了`dubbox`的tag，但这一篇才是真正的只与dubbox相关的哟~

先从业务应用的角度来看，其实dangdang给dubbo嫁接的rest协议是基于RESTEasy的，并且增加了序列化的方式，还有额外的servlet容器。前两个新增是开发者需要非常熟悉的，毕竟是要常打交道的。servlet容器更多的是和运维人员有交集。

RESTEasy不熟悉的童鞋（和我一样），不妨先看一些相关的技术文档，我这里粗糙的翻译了一篇，[尽请笑纳](https://github.com/kazaff/me.kazaff.article/blob/master/%E7%9C%8B%E7%9C%8BRESTEasy.md)。随着你对RESTEasy的熟悉，你会发现dubbox中很多的新增特性都是RESTEasy带来的，遵循JAX-RS规范。当然dangdang的大牛们也做了很多工作，后面我们一点一点分析。

这里还要再次说一个关于RESTEasy的小[问题](https://github.com/dangdangdotcom/dubbox/issues/30)，这也是最终有这篇文章的原因。

序列化部分，dangdang为dubbox新增了fst和kryo两种方式，使用方式和dubbo原有的保持一致，这一部分几乎是对业务透明的，之所以说是几乎，是因为不少大牛提到序列化这部分还是存在不少坑，这就需要我们在开发时进行比较全面的测试，由于我们目前还没正式投入使用，所以暂时就说这么多。

关于新增的servlet容器，目前知道的就是tomcat-embed，另外如果使用REST协议，还可以使用Tjws容器，不过RESEasy官方推荐这个玩意儿还是建议在测试环境中使用。当然，我们也可以选择使用`server="servlet"`来使用外部的servlet容器，例如和其他应用使用同一个tomcat。但是文档上也叮嘱，即便是打算使用外部tomcat，也**尽可能不要和其他应用混合部署，而应该用单独的实例**。

其实上面说的这些，在dubbox提供的[指南](http://dangdangdotcom.github.io/dubbox/rest.html#show-last-Point)中都有详细的说明，多说只是重复。



源码解析
---

我们接下来看一下新增的`RestProtocol`的具体实现。基于之前的分析结果，我们可以直接定位到核心的代码块（doExport）：

	protected <T> Runnable doExport(T impl, Class<T> type, URL url) throws RpcException {
        String addr = url.getIp() + ":" + url.getPort();

        //这么处理其实是为了配合dubbo的延迟暴露机制，延迟暴露的原理是创建新线程，所以ServiceImplHolder靠ThreadLocal来记录对应的类信息，而不是靠公共变量，避免竞争问题
        Class implClass = ServiceImplHolder.getInstance().popServiceImpl().getClass();

        //根据url中的地址创建server容器，相同地址的服务使用相同的容器
        RestServer server = servers.get(addr);
        if (server == null) {
            server = serverFactory.createServer(url.getParameter(Constants.SERVER_KEY, "jetty"));   //默认使用jetty，当当推荐使用tomcat
            server.start(url);
            servers.put(addr, server);
        }

        //查看implClass是否包含jax-rs注解
        final Class resourceDef = GetRestful.getRootResourceClass(implClass) != null ? implClass : type;
        server.deploy(resourceDef, impl, getContextPath(url));  //在server中部署指定服务

        final RestServer s = server;    //注意这样的写法，java要求线程中只能使用外部final变量
        return new Runnable() {
            public void run() {
                // TODO due to dubbo's current architecture,
                // it will be called from registry protocol in the shutdown process and won't appear in logs
                s.undeploy(resourceDef);
            }
        };
    }

该方法是继承自dubbo原有的`AbstractProxyProtocol`类：

![](http://pic.yupoo.com/kazaff/EvwMF0Rm/Br6Hp.png)

按照dubbo的逻辑，注意该方法最终返回的是一个可异步执行的callback类，完成的逻辑也很简单，就是从对应的server中把指定资源服务给“卸载”掉。

我们再来关注一下关于`server`的逻辑，这部分其实很简单，根据你的配置直接创建对应的server：

	public class RestServerFactory {

	    private HttpBinder httpBinder;  //提供http服务的容器
	
		//该方法为SPI注入时调用，会注入一个httpBinder自适应扩展实例
	    public void setHttpBinder(HttpBinder httpBinder) {
	        this.httpBinder = httpBinder;
	    }
	
	    public RestServer createServer(String name) {
	        // TODO move names to Constants
	        if ("servlet".equalsIgnoreCase(name) || "jetty".equalsIgnoreCase(name) || "tomcat".equalsIgnoreCase(name)) {
	            return new DubboHttpServer(httpBinder);
		//        } else if ("tjws".equalsIgnoreCase(name)) {
		//            return new TjwsServer();
	        } else if ("netty".equalsIgnoreCase(name)) {
	            return new NettyServer();
	        } else if ("sunhttp".equalsIgnoreCase(name)) {
	            return new SunHttpServer();
	        } else {
	            throw new IllegalArgumentException("Unrecognized server name: " + name);
	        }
    	}
	}

`DubboHttpServer`类主要贡酒是完成了RESTEasy和servlet容器的绑定，上面代码的`httpBinder`代表的就是你要使用的容器，例如tomcat。这部分逻辑你需要结合RESTEasy的初始化步骤来理解，我就不多说了。

tomcat-embed的使用方式，也非常的直观，结合官方提供的例子即可，也不多说了。


总之，这篇文章还是很水的，不管怎么说，这就是我这几天来的工作内容。更多内容，敬请期待~~


