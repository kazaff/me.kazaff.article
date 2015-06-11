本周的工作计划是调研并测试ELK，那什么是ELK呢？简而言之就是开源的主流的日志收集与分析系统。

ELK是三个工具的总称：

- E: ElasticSearch
- L: Logstash
- K: Kibana

我这里主要想强调的，是它们三个组合起来以后，提供强大的**开箱即用**的日志收集和检索功能，这对于创业公司和小团队来说，简直就是完美~

**可能对我来讲，最不理想的就是它基于ruby语言，这么高逼格的语言我不会啊……**

但是也不要太乐观，社区和大牛们表示，想用好这三个工具也不是一件很简单的事儿，我们本次的目的，就是搞清楚这三者之间的关系和它们各自的作用，并最终搭建一个可测试环境，好吧，废话不多说，开始吧。

最为强迫症的我，一向是尽可能用最新版本，也就是说，我们本次尝试搭建的ELK环境，软件版本分别为：

- [ElasticSearch-1.5.2](https://www.elastic.co/downloads/elasticsearch)
- [Logstash-1.5.0](https://www.elastic.co/downloads/logstash)
- [Kibana-4.0.2](https://www.elastic.co/downloads/kibana)

安装
---

之前提过“开箱即用”，绝对不是吹牛逼的，安装流程就是：

	下载 --> 解压 --> 简单配置 --> 运行 --> 看效果

完全不需要编译安装，真是醉了~我们从logstash开始，下载上面给的版本后，解压并拷贝到`/usr/local`下：

	$ cd /usr/local/logstash
	$ mkdir conf logs

我们创建了两个文件夹，分别用来放配置文件和日志。

接下来我们先简单的配置一下logstash，在conf下创建一个central.conf，内容如下：

	input {
		stdin {}
	}

	output {
		stdout {}
		elasticsearch {
			cluster => "elasticsearch"
			codec => "json"
			protocol => "http"
		}   
	}

保存，运行下面这个命令启动：

	$ ./bin/logstash agent --verbose --config ./conf/central.conf --log ./logs/stdout.log

接下来安装elasticsearch，一样简单，解压到`/usr/local/`下，然后直接启动：

	$ ./bin/elasticsearch -d

最后来配置kibana，解压到`/usr/local/`下，然后直接启动：

	$ ./bin/kibana

我们使用的是kibana默认的配置，现在可以直接在浏览器中访问： [http://127.0.0.1:5601](http://127.0.0.1:5601)。

怎么测试我们安装的是否成功呢？很简单，在刚才我们执行logstash的终端中，直接输入字符串：hello world，然后刷新kibana页面，你将会看到对应的日志信息~~

知道什么叫开箱即用了么？

搭建好了，不等于我们就可以直接使用在产品环境下了，毕竟这个例子貌似没有卵用...下面我们来聊一下理论知识。


架构
---

![](http://nkcoder.github.io/images/post/ELKR-log-platform.jpg)

说明：

- 多个独立的agent(Shipper)负责收集不同来源的数据，一个中心agent(Indexer)负责汇总和分析数据，在中心agent前的Broker(使用redis实现)作为缓冲区，中心agent后的ElasticSearch用于存储和搜索数据，前端的Kibana提供丰富的图表展示。
- Shipper表示日志收集，使用LogStash收集各种来源的日志数据，可以是系统日志、文件、redis、mq等等；
- Broker作为远程agent与中心agent之间的缓冲区，使用redis实现，一是可以提高系统的性能，二是可以提高系统的可靠性，当中心agent提取数据失败时，数据保存在redis中，而不至于丢失；
- 中心agent也是LogStash，从Broker中提取数据，可以执行相关的分析和处理(Filter)；
- ElasticSearch用于存储最终的数据，并提供搜索功能；
- Kibana提供一个简单、丰富的web界面，数据来自于ElasticSearch，支持各种查询、统计和展示；

上面这个图是参考前辈的（下面的参考列表中有具体链接），需要知道的是上面的这种搭配并非是唯一的，例如你可以不使用redis而是用kafka来做消息队列，你也可以让Shipper直接从你希望的日志源中拉取日志数据，如你喜欢你也可以让数据存到Elasticsearch后再存一份到hdfs中，反正大牛们说logstash的插件非常的多，我是信了~~

接下来，我们就试试用logstash来获取一些常用的web server的access log，例如：nginx，tomcat等。由于测试环境都是在同一台机器上本地完成，所有就不需要劳烦消息队列中间件了。


Nginx-->Logstash
---

根据上图所示，我们其实就是让logstash去监听nginx的日志文件，首先我们得先修改一下上面的那个logstash的配置文件：

	input {
		file {
			type => "nginx_access"
			path => ["/usr/local/nginx/logs/*.log"]
			exclude => ["*.gz","error.log","nginx.pid"]
			sincedb_path => "/dev/null"
			codec => "json"	
		}
	}
	
	output {
		stdout {}
		elasticsearch {
			cluster => "elasticsearch"
			codec => "json"
			protocol => "http"
		}
	}

ok，接下来我们就需要安装nginx了，这部分我就不多讲了，安装好nginx后在启动之前，我们需要先修改一下nginx的配置文件（`nginx.conf`）:
	
	......
	log_format   json  '{"@timestamp":"$time_iso8601",'
				'"@source":"$server_addr",'
				'"@nginx_fields":{'
					'"client":"$remote_addr",'
					'"size":$body_bytes_sent,'
					'"responsetime":"$request_time",'
					'"upstreamtime":"$upstream_response_time",'
					'"upstreamaddr":"$upstream_addr",'
					'"request_method":"$request_method",'
					'"domain":"$host",'
					'"url":"$uri",'
					'"http_user_agent":"$http_user_agent",'
					'"status":$status,'
					'"x_forwarded_for":"$http_x_forwarded_for"'
				'}'
			'}';

    access_log  logs/access.log  json;
	......

大功告成了，启动nginx后，浏览器访问nginx下的页面，你会看到kibana中的对应更新。


Tomcat-->Logstash
---

tomcat的日志比nginx要复杂一些，打开对应的日志文件（`catalina.out`）,你会发现类似下面这样的日志结构：

	Jun 12, 2014 11:17:34 AM org.apache.catalina.core.AprLifecycleListener init
	INFO: The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
	Jun 12, 2014 11:17:34 AM org.apache.coyote.AbstractProtocol init
	INFO: Initializing ProtocolHandler ["http-bio-8080"]
	Jun 12, 2014 11:17:34 AM org.apache.coyote.AbstractProtocol init
	SEVERE: Failed to initialize end point associated with ProtocolHandler ["http-bio-8080"]
	java.net.BindException: Address already in use <null>:8080
		at org.apache.tomcat.util.net.JIoEndpoint.bind(JIoEndpoint.java:411)
		at org.apache.tomcat.util.net.AbstractEndpoint.init(AbstractEndpoint.java:640)
		at org.apache.coyote.AbstractProtocol.init(AbstractProtocol.java:434)
		at org.apache.coyote.http11.AbstractHttp11JsseProtocol.init(AbstractHttp11JsseProtocol.java:119)
		at org.apache.catalina.connector.Connector.initInternal(Connector.java:978)
		at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
		at org.apache.catalina.core.StandardService.initInternal(StandardService.java:559)
		at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
		at org.apache.catalina.core.StandardServer.initInternal(StandardServer.java:813)
		at org.apache.catalina.util.LifecycleBase.init(LifecycleBase.java:102)
		at org.apache.catalina.startup.Catalina.load(Catalina.java:638)
		at org.apache.catalina.startup.Catalina.load(Catalina.java:663)
		at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
		at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
		at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
		at java.lang.reflect.Method.invoke(Method.java:606)
		at org.apache.catalina.startup.Bootstrap.load(Bootstrap.java:280)
		at org.apache.catalina.startup.Bootstrap.main(Bootstrap.java:454)
	Caused by: java.net.BindException: Address already in use
		at java.net.PlainSocketImpl.socketBind(Native Method)
		at java.net.AbstractPlainSocketImpl.bind(AbstractPlainSocketImpl.java:376)
		at java.net.ServerSocket.bind(ServerSocket.java:376)
		at java.net.ServerSocket.<init>(ServerSocket.java:237)
		at java.net.ServerSocket.<init>(ServerSocket.java:181)
		at org.apache.tomcat.util.net.DefaultServerSocketFactory.createSocket(DefaultServerSocketFactory.java:49)
		at org.apache.tomcat.util.net.JIoEndpoint.bind(JIoEndpoint.java:398)
		... 17 more

无法直视啊简直，不过还好，logstash提供了强大的插件来帮助我们解析各种各样的日志输出结构，分析一下可以得出，日志结构是：时间戳，后面跟着类名，再后面跟着日志信息，这样，我们就可以根据这种结构来写过滤规则：
	
	COMMONAPACHELOG %{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)
	COMBINEDAPACHELOG %{COMMONAPACHELOG} %{QS:referrer} %{QS:agent}

	CATALINA_DATESTAMP %{MONTH} %{MONTHDAY}, 20%{YEAR} %{HOUR}:?%{MINUTE}(?::?%{SECOND}) (?:AM|PM)
	CATALINALOG %{CATALINA_DATESTAMP:timestamp} %{JAVACLASS:class} %{JAVALOGMESSAGE:logmessage}

我们把[这些规则](https://gist.github.com/LanyonM/8390458#file-grok-patterns)存储在一个文件（`/logstash安装路径/patterns/grok-patterns`）中，接下来我们要改写logstash的配置文件了：

	input {
		file {
			type => "tomcat_access"
			path => ["/usr/local/tomcat/logs/catalina.out"]
			exclude => ["*.log","*.txt"]
			sincedb_path => "/dev/null"
			start_position => "beginning"
		}
		file {
			type => "apache_access"
			path => ["/usr/local/tomcat/logs/*.txt"]
			exclude => ["*.log"]
			sincedb_path => "/dev/null"
			start_position => "beginning"
		}
	}
	
	filter {
		if [type] == "tomcat_access" {
			multiline {
				patterns_dir => "/usr/local/logstash/patterns"
				pattern => "(^%{CATALINA_DATESTAMP})"
			        negate => true
			        what => "previous"
			}
			if "_grokparsefailure" in [tags] {
				drop { }
			}
			grok {
			      	patterns_dir => "/usr/local/logstash/patterns"
			      	match => [ "message", "%{CATALINALOG}" ]
			}
			date {
			      	match => [ "timestamp", "yyyy-MM-dd HH:mm:ss,SSS Z", "MMM dd, yyyy HH:mm:ss a" ]
			}
		}
		if [type] == "apache" {
			grok {
				patterns_dir => "/usr/local/logstash/patterns"
		      		match => { "message" => "%{COMBINEDAPACHELOG}" }
		    	}
		    	date {
		      		match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
		    	}
		}
	}
	
	output {
		stdout {}
		elasticsearch {
			cluster => "elasticsearch"
			protocol => "http"
			embedded => true
		}
	}

然后你就可以重启一下logstash看结果啦。注意这里，我配置的是`start_position => "beginning"`，字面意思就是从日志文件头部开始解析，这样可以把一些原始日志给收集过来，但是可能会造成一些重复日志~~


Log4j-->Logstash
---

其实通过上面两个场景，你大概也知道是个什么尿性了，对，就是指定好要监控的日志文件（input），然后解析对应的格式（filter），然后导入到对应的存储中（output），而且整个过程是管道化的，前面提到了，由于咱们的测试环境是单机本地化，所以并没有使用消息队列，否则你会感受到更明显的管道化风格。

把log4j产生的日志导入到logstash要做的事儿依然如此，不过官方提供了更简单的方式：[log4j-jsonevent-layout](https://github.com/logstash/log4j-jsonevent-layout)，这玩意儿的作用相当于我们在nginx中干的事儿，直接将log4j的日志格式定义成json的，有助于性能提升~

剩下的事儿就是老样子了，只需要改一下我们的logstash配置文件即可：

	......
	file {
		type => "log4j"
		path => ["/usr/local/tomcat/logs/logTest.log"]
		sincedb_path => "/dev/null"
		start_position => "beginning"
	}
	......


高可用
---

有经验的童鞋不难看出，上图中存在单点故障，但，其实是可以通过把对应单点集群化部署来增加这套架构的可用性和处理性能，具体可参考[ElasticSearch+LogStash+Kibana+Redis日志服务的高可用方案](http://nkcoder.github.io/blog/20141106/elkr-log-platform-deploy-ha/)。

总结
---

上面的场景和简单介绍都是非常入门级的，放在线上场景中肯定太粗糙了，肯定还需要考量更多的因素，例如数据量，日志大小，通信方式等，不过我相信到现在大家应该对ELK有了一个基本的认知。


参考
---

[Kibana 中文指南](http://kibana.logstash.es/content/)

[使用ElasticSearch+LogStash+Kibana+Redis搭建日志管理服务](http://nkcoder.github.io/blog/20141031/elkr-log-platform-deploy/)

[使用Logstash收集Nginx日志](http://john88wang.blog.51cto.com/2165294/1632190)

[Logstash Multiline Tomcat and Apache Log Parsing](http://blog.lanyonm.org/articles/2014/01/12/logstash-multiline-tomcat-log-parsing.html)

[Logstash and Log4j](http://drumcoder.co.uk/blog/2014/nov/11/logstash-and-log4j/)