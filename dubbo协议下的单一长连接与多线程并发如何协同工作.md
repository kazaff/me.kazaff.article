上班的路上突然就冒出了这么个问题：既然在[dubbo中描述](http://alibaba.github.io/dubbo-doc-static/Dubbo+Protocol-zh.htm)消费者和提供者之间采用的是单一长连接，那么如果消费者端是高并发多线程模型的web应用，单一长连接如何解决多线程并发请求问题呢？

其实如果不太了解socket或者多线程编程的相关知识，不太容易理解这个问题。传统的最简单的RPC方式，应该是为每次远程调用请求创建一个对应的线程，我们先不说这种方式的缺点。至少优点很明显，就是简单。简单体现在哪儿？

> 通信双方一对一（相比NIO来说）。

通俗点来说，socket通信的双方发送和接受数据不会被其它（线程）干扰，这种干扰不同于数数据包的“粘包问题”。其实说白了就相当于**电话线路**的场景：

> 试想一下如果多个人同时对着同一个话筒大喊，对方接受到的声音就会是重叠且杂乱的。

对于单一的socket通道来说，如果发送方多线程的话，不加控制就会导致通道中的数据乱七八糟，接收端无法区分数据的单位，也就无法正确的处理请求。

乍一看，似乎dubbo协议所说的单一长连接与客户端多线程并发请求之间，是水火不容的。但其实稍加设计，就可以让它们和谐相处。

socket中的粘包问题是怎么解决的？用的最多的其实是定义一个定长的数据包头，其中包含了完整数据包的长度，以此来完成服务器端拆包工作。

那么解决多线程使用单一长连接并发请求时包干扰的方法也有点雷同，就是给包头中添加一个标识id，服务器端响应请求时也要携带这个id，供客户端多线程领取对应的响应数据提供线索。

其实如果不考虑性能的话，dubbo完全也可以为每个客户端线程创建一个对应的服务器端线程，但这是海量高并发场景所不能接受的~~

那么脑补一张图：

![](http://pic.yupoo.com/kazaff_v/E38Ivr2W/jp9OS.png)

下面咱们试图从代码中找到痕迹。

一路追踪，我们来到这个类：`com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.java`，先来看看其中的`request`方法，大概在第101行左右：


 	public ResponseFuture request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion("2.0.0");
        req.setTwoWay(true);
        req.setData(request);

		//这个future就是前面我们提到的：客户端并发请求线程阻塞的对象
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try{
            channel.send(req);  //非阻塞调用
        }catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }

注意这个方法返回的`ResponseFuture`对象，当前处理客户端请求的线程在经过一系列调用后，会拿到`ResponseFuture`对象，最终该线程会阻塞在这个对象的下面这个方法调用上，如下：


	public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        if (! isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                while (! isDone()) {	//无限连
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            if (! isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        return returnFromResponse();
    }

上面我已经看到请求线程已经阻塞，那么又是如何被唤醒的呢？再看一下`com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.java`，其实所有实现了`ChannelHandler`接口的类都被设计为装饰器模式，所以你可以看到类似这样的代码：

	 protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
        return new MultiMessageHandler(
                new HeartbeatHandler(
                        ExtensionLoader.getExtensionLoader(Dispather.class).getAdaptiveExtension().dispath(handler, url)
                ));
    }

现在来仔细看一下`HeaderExchangeHandler`类的定义，先看一下它定义的`received`方法，下面是代码片段：

	public void received(Channel channel, Object message) throws RemotingException {
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            if (message instanceof Request) {
              .....
            } else if (message instanceof Response) {   
				//这里就是作为消费者的dubbo客户端在接收到响应后，触发通知对应等待线程的起点
                handleResponse(channel, (Response) message);
            } else if (message instanceof String) {
               .....
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }

我们主要看中间的那个条件分支，它是用来处理响应消息的，也就是说当dubbo客户端接收到来自服务端的响应后会执行到这个分支，它简单的调用了`handleResponse`方法，我们追过去看看：

	static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {  //排除心跳类型的响应
            DefaultFuture.received(channel, response);
        }
    }

熟悉的身影：`DefaultFuture`，它是实现了我们上面说的`ResponseFuture`接口类型，实际上细心的童鞋应该可以看到，上面`request`方法中其实实例化的就是这个`DefaultFutrue`对象：

	DefaultFuture future = new DefaultFuture(channel, req, timeout);

那么我们可以继续来看一下`DefaultFuture.received`方法的实现细节：

	public static void received(Channel channel, Response response) {
        try {
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                future.doReceived(response);
            } else {
                logger.warn("The timeout response finally returned at " 
                            + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date())) 
                            + ", response " + response 
                            + (channel == null ? "" : ", channel: " + channel.getLocalAddress() 
                                + " -> " + channel.getRemoteAddress()));
            }
        } finally {
            CHANNELS.remove(response.getId());
        }
    }

留一下我们之前提到的**id**的作用，这里可以看到它已经开始发挥作用了。通过`id`，`DefaultFuture.FUTURES`可以拿到具体的那个`DefaultFuture`对象，它就是上面我们提到的，阻塞请求线程的那个对象。好，找到目标后，调用它的`doReceived`方法，这就是标准的java多线程编程知识了：

	private void doReceived(Response res) {
        lock.lock();
        try {
            response = res;
            if (done != null) {
                done.signal();
            }
        } finally {
            lock.unlock();
        }
        if (callback != null) {
            invokeCallback(callback);
        }
    }


这样我们就可以证实上图中左边的绿色箭头所标注的两点。

---

接下来我们再来看看右边绿色箭头提到的两点是如何实现的？其实dubbo在NIO的实现上默认依赖的是netty，也就是说真正在长连接两端发包和接包的苦力是netty。由于哥们我对netty不是很熟悉，所以暂时我们就直接把netty当做黑箱，只需要知道它可以很好的完成NIO通信即可。


参考：

[Dubbo基本原理机制](http://blog.csdn.net/paul_wei2008/article/details/19355681)

[ALIBABA DUBBO框架同步调用原理分析](http://www.blogjava.net/xiaomage234/archive/2014/05/09/413465.html)

