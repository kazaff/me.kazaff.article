前前后后已经写过好多个语言与redis进行结合了，有php，c++，java，今天总算要用nodejs来操作redis了，不知为何还有点儿小兴奋~~

虽然限于nodejs的异步编程模型的影响，直观上和其它语言非常不同，但基本上花一些时间都可以很快的适应，操作redis的方法依然如此，与操作mysql的api没什么两样~

我还是准备着重介绍关于redis的认证，事务和批处理相关的内容，毕竟redis的数据结构本身非常的直观，并没有什么好多讲的~

哦，忘记说了，我使用的库是：[redis](https://github.com/mranney/node_redis)，从官网上可以了解很全面信息，这里面有个小插曲，我的桌面OS是win7 64bit，但是死活安装不好vs环境，导致无法成功编译hiredis，所以只能使用redis提供的javascript接口，这在[性能](https://github.com/mranney/node_redis#performance)上会有所降低，不过目前这并不是我的关注点。

javascript是基于事件驱动的，nodejs理所当然的也拥有事件驱动引擎，所以先看一下这个redis库提供了哪些重要的事件：

ready
---
与redis服务创建连接成功后，会触发“ready”事件，这表明服务器端已经准备好接受命令了，但这并不表明必须在该事件发生后才能使用client执行命令，那是因为在“ready”事件触发之前，用client执行的命令会存放在队列中，等到就绪后会根据顺序依次调用命令。

end
---
当连接关闭后，会触发“end”事件。

drain
---
当连接缓冲区可写后，会触发“drain”事件，这有助于你控制发送频率。



---
下面来介绍一些重要的接口：

redis.createClient(port, host, options)
---
默认port是6379，默认的host当然是127.0.0.1，options为一个对象，可以包含下列的属性：

* parser： redis协议的解析器，默认是hiredis，如果像我一样悲剧的装不上，那该项会被设置为javascript；
* no_ready_check： 默认为false，当连接建立后，服务器端可能还处在从磁盘加载数据的loading阶段，这个时候服务器端是不会响应任何命令的，这个时候node_redis会发送INFO命令来检查服务器状态，一旦INFO命令收到响应了，则表明服务器端已经可以提供服务了，此时node_redis会触发“ready”事件，这就是为什么会有“connect”和“ready”事件之分，如果你关闭了这个功能，我觉得会出现一些貌似灵异的问题；
* enable_offline_queue： 默认为true，前面提到过，当连接ready之前，client会把收到的命令放入队列中等待执行，如果关闭该项，所有ready前的调用都将会立刻得到一个error callback；
* retry_max_delay： 默认为null，默认情况下client连接失败后会重试，每次重试的时间间隔会翻倍，直到永远，而设置这个值会增加一个阀值，单位为毫秒；
* connect_timeout: 默认为false，默认情况下客户端将会一直尝试连接，设置该参数可以限制尝试连接的总时间，单位为毫秒；
* max_attempts: 默认为null，可以设置该参数来限制尝试的总次数；
* auth_pass： 默认为null，该参数用来认证身份。

client.auth(password, callback)
---
这个接口用来认证身份的，如果你要连接的redis服务器是需要认证身份的，那么你必须确保这个方法是创建连接后第一个被调用的。需要注意的是，为了让使重连足够的简单，client.auth()保存了密码，用于每次重连后的认证，而回调只有会执行一次哦~~

另外你需要确保的是，千万别自作聪明的把client.auth()放在“ready”或“connect”事件的回调中，你将会得到一行神秘的报错：

	Error: Ready check failed: ERR operation not permitted

只需要在redis.createClient()代码后直接调用clietn.auth()即可，具体原因我没有深究~

client.end()
---
这个方法会强行关闭连接，并不会等待所有的响应。如果不想如此暴力，推荐使用client.quit()，该方法会在收到所有响应后发送QUIT命令。

client.unref()
---
这个方法作用于底层socket连接，可以在程序没有其他任务后自动退出，可以类比nodejs自己的unref()。这个方法目前是个试验特性，只支持一部分redis协议。

client.multi([commands])
---
multi命令可以理解为打包，它会把命令都存放在队列里，直到调用exec方法，redis服务器端会一次性原子性的执行所有发来的命令，node_redis接口最终会返回一个Multi对象。

Multi.exec(callback)
---
client.multi()会返回一个Multi对象，它包含所有的命令，直到调用Multi.exec()。我们来主要说一下这个方法的回调函数中的两个参数：

* err: null或者Array，没有出错当然就会返回null，如果出错则返回命令队列链中对应的发生错误的命令的错误信息，数组中最后一个元素exec()方法自身的错误信息，**这里我要说的是，可以看出这种方式所谓的原子性主要是指的命令链中的命令一定会保证一起在服务器端执行，而不是指的像关系型数据库那样的回滚功能**；
* results： null或者Array，返回命令链中每个命令的返回信息。


	var redis  = require("./index"),
        client = redis.createClient(), set_size = 20;

    client.sadd("bigset", "a member");
    client.sadd("bigset", "another member");

    while (set_size > 0) {
        client.sadd("bigset", "member " + set_size);
        set_size -= 1;
    }

    // multi chain with an individual callback
    client.multi()
        .scard("bigset")
        .smembers("bigset")
        .keys("*", function (err, replies) {
            // NOTE: code in this callback is NOT atomic
            // this only happens after the the .exec call finishes.
            client.mget(replies, redis.print);
        })
        .dbsize()
        .exec(function (err, replies) {
            console.log("MULTI got " + replies.length + " replies");
            replies.forEach(function (reply, index) {
                console.log("Reply " + index + ": " + reply.toString());
            });
        });