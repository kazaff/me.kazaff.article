前几天又一遍熟悉了一下常用的负载均衡软件，比方说Nginx，还挺顺利的。既然有了负载均衡，那就要求应用能具备良好的伸缩性，而要达到这一点，就要解决多个应用镜像之间的数据共享问题，说白了就是会话状态的共享！

其实**无会话状态**才是服务追求的目标，搞过RESTful的童鞋应该知道这一点！之前玩PHP的时候，要做到Session共享其实不难，只需要把[session存储到mysql中](http://blog.csdn.net/eflyq/article/details/12081281)，这事儿就搞定一多半了，剩下的就是设置session的作用域等参数，能够保证客户端请求时携带任意一台服务器为它创建的session_id即可！~那，在javaEE下又该怎么做呢？

先从基础知识讲起吧，看一下这些文章：[传送门1](http://blog.csdn.net/ghsau/article/details/13023425)，[传送门2](http://www.cnblogs.com/EvanLiu/p/3356925.html)。其实和PHP里定义的Session差不多，这也很正常，本来这个概念就不是由语言提出来的，而是由HTTP引出的，所以语言相关性不大。而且类比apache+php，其实在java web中，session也是交给tomcat这种容器来管理的，而servlet只是提供了相关的接口定义而已，想了解这其中的内部细节的童鞋可以看一下这篇[文章](http://gearever.iteye.com/blog/1546423)。

好了，到这里为止基本上已经算是熟悉java下的session了！接下来我们就可以直奔主题了，其实实作方式应该也和php的差不多，只不过需要写成tomcat的“插件”，具体细节其实已经有相关的扩展了，我这里直接找到一篇非常实战的文章供大家操作：[传送门](http://zhangqiaoqifgdqsn.iteye.com/blog/1975797)。

这篇文章就到这里吧，虽然感觉上并没有写什么，哇哈哈哈，毕竟作为新手能遇到的问题都应该被前辈大妞们解决了，我们只需要拼命的学习就行啦！



