今天要搞一搞的，是把logstash和kafka整合起来，由于我们使用的是logstash1.5.0+版本，此版本下官方已经提拱了plugin用来整合kafka，这篇文章的目的就是简单的搭建这么一个环境。

kafka的安装
---

kafka是基于scala实现的，scala是一种jvm语言，也就是说你得先装jdk，我就不从jdk开始介绍如何安装了，我们直接开始安装kafka，其实这玩意儿官方提供了编译好的版本，你只需要下载，解压，运行即可~

当然，如果想搭建集群，还是需要了解一下kafka的配置的，这不是我们关注的重点，so，我们就简单地先跑起来它吧：

1. [下载](http://kafka.apache.org/downloads.html)
2. 解压，我们解压在`/usr/local`下
3. 运行，为了测试方便，我们需要一共开启四个终端窗口：

首先，我们要运行zookeeper，kafka自带了zookeeper，所以我们无需下载，只需要在/usr/local/kafka目录下执行：

	bin/zookeeper-server-start.sh config/zookeeper.properties

然后，再开启一个终端，执行：

	bin/kafka-server-start.sh config/server.properties

这样，我们的kafka就已经运行起来了，不过还不是集群环境，只有一个borker哟~但是，我们测试足够了。

再然后，开启一个新的终端，执行：

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

这样我们就创建了一个用于测试的topic，接下来继续执行：

	bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

该命令执行完毕后会阻塞终端，你可以随便输入一些数据，每一行都相当于一个消息，会发送给kafka。

最后，再再开启一个新终端，执行：

	bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning

你会看到你之前输入的那些消息都会显示在终端中，这就完成了kafka的测试环境搭建。

**值得注意的是**，上面消息生产者的命令，需要参数`broker-list`，也就是说我们的kafka生产者必须自己知道所有的kafka broker的位置，而其它命令则只需要填写zookeeper的位置即可，我不清楚这样做的用意是什么，我只是隐约感到有些问题（**如果broker出现扩容，如何更新应用代码中的borker信息？？**），但这不是我们本次的关注的重点。




logstash-->kafka
---

我们现在想要干的，是从logstash的Shipper中收集到的数据发送给kafka，所以我们需要安装[logstash-output-kafka](https://github.com/logstash-plugins/logstash-output-kafka)插件。

但是由于未知原因，我试图安装插件时却碰到了报错：

	[root@kazaff logstash-1.5.0]# bin/plugin install logstash-output-kafka
	Validating logstash-output-kafka
	Plugin logstash-output-kafka does not exist
	ERROR: Installation aborted, verification failed for logstash-output-kafka 

不像是被墙了的味道，因为提醒的是不存在，而不是网络连接超时。本来还想搭建一个翻墙环境，后来执行了一下这个命令：

	bin/plugin list  

竟然发现kafka插件已经预装好了，我也是醉了。OK，我们可以继续了，接下来就是配置一下logstash：

	input {
		stdin{}
	}
	
	
	output {
		kafka {
			broker_list => "localhost:9092"
			topic_id => "test"
			compression_codec => "snappy"
		}
	}

不多做解释了，在终端运行logstash后，就可以直接输入“helloworld”测试一下了，如果没有问题的话，你将会在之前的kafka消费者终端中看到输出：

	{"message":"hello world!","@version":"1","@timestamp":"2015-06-11T10:01:21.183Z","host":"kazaff"}

就是这么简单啦~

	
参考
---

[Kafka快速入门](http://colobu.com/2014/08/06/kafka-quickstart/#show-last-Point)

[Logstash入门教程 - 启动命令行参数及插件安装](http://corejava2008.iteye.com/blog/2215545)

[logstash-input-file以及logstash-output-kafka插件性能测试](http://bigbo.github.io/pages/2015/03/26/logstash_performance/)