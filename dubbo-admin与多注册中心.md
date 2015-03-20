dubbo-admin是否支持同时关联多个注册中心统一管理不同注册中心上注册的服务？

答案是：**不支持**！


这里指的多注册中心，并不是zookeeper集群，两者的差别看[这里](http://alibaba.github.io/dubbo-doc-static/Multi+Registry-zh.htm)和[这里](http://alibaba.github.io/dubbo-doc-static/Zookeeper+Registry-zh.htm)。

[dubbo-admin](http://alibaba.github.io/dubbo-doc-static/Operation+Tutorial-zh.htm)提供了运维界面，辅助我们更好的管理和维护服务之间的依赖等信息。刚认识dubbo时确实被这么逆天的功能给震惊了（现在也是~）。

那么该怎么办呢？目前想到的就只是为每一个注册中心部署一套admin管理后台。确实听起来有些土鳖，但应该是最省事儿的了吧~

当然，也可以改造一下现有的dubbo-admin逻辑，只不过以现在我的道行还做不到啊~坐等高手！

最后还要叮嘱的是，如果你的注册中心是以集群方式部署的，那么你在你的服务配置注册中心时**一定要以官方提供的集群配置写法来部署，否则会提示zkClient连接超时**：

	...
	nested exception is org.I0Itec.zkclient.exception.ZkTimeoutException: Unable to connect to zookeeper server within timeout: 5000
	...

期初我还偷懒只写了一个zookeeper leader的地址，结果死活连不上（dubbo-admin也连不上！），一直以为是防火墙的问题！后来老老实实的按照官方推荐的写法：

	 <dubbo:registry protocol="zookeeper" address="192.168.76.138:2182,192.168.76.138:2181,192.168.76.138:2183"/>

一切就正常了！希望大家引以为戒~~
