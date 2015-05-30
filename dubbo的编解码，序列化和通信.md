dubbo的调研已经快完结了（按照我自己拟定的计划），计划内剩下的内容就只有：

- 序列化
- 编解码
- 通信实现

打算写在一篇里，年前彻底搞定dubbo[x]的调研，过完年来了就要投入使用了，好紧张哇~~哟呵呵呵呵！其实前两块的内容并没有啥好讲的，毕竟咱目的是了解源码来辅佐如何使用，而非像当当网的团队那样做dubbo的升级开发。

按照源码的阅读习惯，我们按照上面列表的逆序来一个一个的分析。废话不多说，走着~


通信实现
---

我们主要基于dubbo推荐默认使用的通信框架：netty，来了解一下dubbo是如何完成两端通信的。我们直接从`DubboProtocol`类开始看起：

	export()  -->  openServer()  -->  createServer()
	   										|
											+-->  server = Exchangers.bind(url, requestHandler);  //创建服务

	

dubbo从要暴漏的服务的URL中取得相关的配置（host，port等）进行服务端server的创建，并且保证相同的配置（host+port）下只会开启一个server，这和netty提供的模型有关（NIO），这个我们后面再说。

我们先来继续看`Exchangers`的相关部分，

	......
	 public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");	//这里尝试配置了编解码的方式
        return getExchanger(url).bind(url, handler);
    }
	......
	public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);	//默认使用HeaderExchanger
        return getExchanger(type);
    }
	......
	public static Exchanger getExchanger(String type) {
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
    }
	......


可以看出，`Exchangers`只是根据URL的参数提供了策略模式。我们依然以dubbo默认的处理方式为主，接下来代码执行到`HeaderExchanger`类：

	public class HeaderExchanger implements Exchanger {
    
	    public static final String NAME = "header";
	
	    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
	        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
	    }
	
	    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
	        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
	    }	
	}

这些代码看起来非常的**设计模式**：

	return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
						|						  |					 |					|				  |
						V						  |					 |					|				  V
				1.提供统一的服务操作接口			  |					 |					|				利用装饰器模式，这个才是最靠近业务的逻辑（直接调用相关的invoker）
				2.创建心跳定时任务				  V					 |					|
										1.利于扩展点机制选择通信框架	 |					|
										2.格式化回调函数				 |					|
																	 V					V
															    消息的解码???	     处理dubbo的通信模型：单向，双向，异步等通信模型



要想理解现在的内容，就得先搞清楚[JAVA NIO channel概念](http://ifeve.com/channels/)，搞清楚netty的[NIO线程模型](http://www.infoq.com/cn/articles/netty-threading-model/#show-last-Point)。

了解了这两个基础知识点，那么我们就可以继续分析源码了，上面那一行代码中`Transporters.bind()`默认会调用`NettyTransporter`：

	public class NettyTransporter implements Transporter {

	    public static final String NAME = "netty";
	    
	    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
	        return new NettyServer(url, listener);
	    }
	
	    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
	        return new NettyClient(url, listener);
	    }
	}

接下来我们就真正进入到了netty的世界，我们先来看一下`NettyServer`的家谱：

![](http://pic.yupoo.com/kazaff/EqfbO3HE/NAsri.png)

要时刻记着，dubbo是一个非常灵活的框架，我们不仅可以使用netty作为底层通信组件，也可以仅靠url参数就可以改变底层通信的实现，这种架构设计彰显了开发人员对代码的驾驭能力。

`AbstractServer`抽象父类把创建server所需的公共逻辑抽离出来集中完成，而需要根据特定通信框架的逻辑则交给特定子类（NettyServer）利用重载（doOpen）完成，这样的代码结构在dubbo中随处可见。

	@Override
    protected void doOpen() throws Throwable {
        NettyHelper.setNettyLoggerFactory();
        ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
        ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
        ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
        bootstrap = new ServerBootstrap(channelFactory);
        
        final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
        channels = nettyHandler.getChannels();
        // https://issues.jboss.org/browse/NETTY-365
        // https://issues.jboss.org/browse/NETTY-379
        // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec() ,getUrl(), NettyServer.this);
                ChannelPipeline pipeline = Channels.pipeline();
                /*int idleTimeout = getIdleTimeout();
                if (idleTimeout > 10000) {
                    pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
                }*/
                pipeline.addLast("decoder", adapter.getDecoder());  //Upstream
                pipeline.addLast("encoder", adapter.getEncoder());  //Downstream
                pipeline.addLast("handler", nettyHandler);          //Upstream & Downstream
                return pipeline;
            }
        });
        // bind
        channel = bootstrap.bind(getBindAddress());
    }

如果是熟悉netty的童鞋，肯定早已习惯这个方法的写法，就是创建了netty的server嘛，不过需要注意的是，netty本身是基于事件的，留意一下上面的`NettyServer`的继承关系，其中`ChannelHandler`并不是netty的那个`ChannelHandler`，这就意味着要让前者转换成后者，才可以供netty使用，这也就是`NettyHandler`的意义，同样，类似这样的做法也可以在dubbo中找到多处。

同时也要注意，`NettyServer`和`NettyHandler`都有同一个用于记录**打开中**的channel的集合：

	private final Map<String, Channel> channels = new ConcurrentHashMap<String, Channel>(); // <ip:port, channel>，其中ip:port指的是调用端的ip和端口号

其中的`Channel`类型也并非netty的`Channel`，而是dubbo的`NettyChannel`，该类负责把netty的`Channel`，dubbo自身的`url`和`handler`映射起来，依赖这样的设计思想，就可以完全把业务和底层基础实现很好的隔离开来，灵活性大大提高，当然，复杂度也随之增加了，这是架构师需要权衡的一个哲学问题。

dubbo封装netty就介绍到这里，我们的分析并没有深入到netty太多，因为小弟我对netty的了解也是非常的皮毛，为了避免误人子弟，所以更多的细节就留给高手来分享吧。





编解码
---

socket通信中有一个很好玩儿的部分，就是定义消息头，作用非常重大，例如解决粘包问题。dubbo借助netty这样的第三方框架来完成底层通信，这样一部分工作就委托出去了，不过还是有一些工作是需要dubbo好好规划的，我们来看一张官方提供的消息头格式：

![](http://pic.yupoo.com/kazaff/EqdnuPxV/oqq2m.jpg)

只有搞清楚了消息头结构设计，才能完成消息体的编码解码，才能交给底层通信框架去收发。上图中我们其实只需要关注Dubbo部分，其部分意义已经在[这篇文章](http://blog.kazaff.me/2014/09/20/dubbo%E5%8D%8F%E8%AE%AE%E4%B8%8B%E7%9A%84%E5%8D%95%E4%B8%80%E9%95%BF%E8%BF%9E%E6%8E%A5%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91%E5%A6%82%E4%BD%95%E5%8D%8F%E5%90%8C%E5%B7%A5%E4%BD%9C/)中阐述过了，我们这里只关注代码实现，再来看一下`NettyServer`类：


	@Override
    protected void doOpen() throws Throwable {
    	......
				NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec() ,getUrl(), NettyServer.this);
                ChannelPipeline pipeline = Channels.pipeline();
                pipeline.addLast("decoder", adapter.getDecoder());  //Upstream
                pipeline.addLast("encoder", adapter.getEncoder());  //Downstream
                pipeline.addLast("handler", nettyHandler);          //Upstream & Downstream
                return pipeline;
		......
    }

注意看我在每一行后面加的注释，参见这一篇[关于netty的流处理顺序](http://www.cnblogs.com/montya/archive/2012/12/26/2834279.html)的文章，我们就可以理解dubbo的编码解码是如何配置的。下面接着看一下`getCodec()`方法：

	 protected static Codec2 getChannelCodec(URL url) {
        String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");		//这里的codecName值为dubbo
        if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) {
            return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
        } else {
            //应该是向下兼容 或者 阿里内部才会执行的代码
            return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class)
                                               .getExtension(codecName));
        }
    }


这里又一次尝试根据url中的codec参数来确定最终使用的编解码类，不过我们可以在`DubboProtocol`类的定义中看到，其实这个参数已经被硬编码了：

	//这里强行设置编码方式，有点硬啊
    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);

注意这里`Version.isCompatibleVersion()`会去查找是否存在"com/taobao/remoting/impl/ConnectionRequest.class"，但我们知道，这是taobao内部的实现。

根据参数，我们看一下对应的配置文件：

	transport=com.alibaba.dubbo.remoting.transport.codec.TransportCodec
	telnet=com.alibaba.dubbo.remoting.telnet.codec.TelnetCodec
	exchange=com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec
	dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec		#使用的是这个
	thrift=com.alibaba.dubbo.rpc.protocol.thrift.ThriftCodec

再看回来，`NettyCodecAdapter`完成了把netty和dubbo隔离的任务，使在后面进行编码解码时使用的channel不再是特定的通信框架提供的，而是dubbo提供的抽象实现。

再往下深挖，就会看到dubbo是如何处理数据包的拆装，由于过于琐碎，我决定暂时不继续下去了，日后如果在使用时出现问题，会单独拿出来讲讲。





序列化
---

dubbo本身支持多种序列化方式，当当的duubox也在序列化方面做了新的工作，PRC中要解决跨进程通信的一个首要问题就是对象的系列化问题，业界各大佬公司和开源组织也都开源了很多优秀的项目，而要了解所有的序列化库是需要花大量时间的，我们依旧只关注dubbo是如何在代码层面触发序列化工作的。只有序列化算法本身，还是交给大家去对应官网进行深度学习吧。

序列化是在向对端发送数据前的重要工作，事实上我是在`DubboCodec`类中发现序列化工作的入口的：

	protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        ......
        Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);   //获取对应的序列化库
		......
		decodeEventData(channel, deserialize(s, channel.getUrl(), is));
		......
		
	}

	private ObjectInput deserialize(Serialization serialization, URL url, InputStream is)
            throws IOException {
        return serialization.deserialize(url, is);
    }


	//该方法继承自ExchangeCodec父类
	protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
		Serialization serialization = getSerialization(channel);
		......
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
		......
	}


而从dubbo的配置文件中看，dubbo[x]支持的序列化方式包括：

	dubbo=com.alibaba.dubbo.common.serialize.support.dubbo.DubboSerialization
	hessian2=com.alibaba.dubbo.common.serialize.support.hessian.Hessian2Serialization
	java=com.alibaba.dubbo.common.serialize.support.java.JavaSerialization
	compactedjava=com.alibaba.dubbo.common.serialize.support.java.CompactedJavaSerialization
	json=com.alibaba.dubbo.common.serialize.support.json.JsonSerialization
	fastjson=com.alibaba.dubbo.common.serialize.support.json.FastJsonSerialization
	nativejava=com.alibaba.dubbo.common.serialize.support.nativejava.NativeJavaSerialization
	kryo=com.alibaba.dubbo.common.serialize.support.kryo.KryoSerialization
	fst=com.alibaba.dubbo.common.serialize.support.fst.FstSerialization
	jackson=com.alibaba.dubbo.common.serialize.support.json.JacksonSerialization



---
好吧，到此为止，我们就算了解dubbo啦，如果有什么遗漏的地方，可以留言提醒小弟，一起学习进步。