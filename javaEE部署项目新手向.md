今天主要说的是关于java的web项目部署时候的琐碎。公司之前的几个javaEE项目我都没有参与编码，更别提部署了！不过接下来的项目我就要参与到所有环节中拉，所以趁着有时间，搞一下预备工作，都是很基础的东西，只是作为总结记录下来而已~~

以问题的方式来排版吧，这样便于阅读：




项目中配置文件里使用的`classpath`到底指向哪里？
---

可以通过下面这行代码打印出来`classpath`的绝对路径：

	System.out.println("当前classpath的绝对路径：" + Thread.currentThread().getContextClassLoader().getResource(""));

我用Idea13部署项目后可以看到终端中打印出：

	当前classpath的绝对路径：file:/D:/Program%20Files/apache-tomcat-7.0.53/webapps/springMVCDemo/WEB-INF/classes/

可以看到`classpath`指向的是部署后的`/WEB-INF/classes/`，而这个`classes`文件夹就是存放编译后的`.class`文件的地方。

而这个文件夹在部署之前是不存在的，那么IDE中配置文件里使用的`classpath:`又指向哪里呢？因为像Idea这样的吊炸天IDE都提供即时功能的，比方说我在`web.xml`中添加下面这个配置：

	...
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:mvc-dispatcher-servlet.xml</param-value>
	...

IDE是会帮我检查对应文件夹下是否存在指定文件的，那我们之前说过编译前`classes`文件夹是不存在的，那么IDE又是帮我去哪找的呢？看下图：

<a href="http://pic.yupoo.com/kazaff/E9bw5Lp6/8VcSP.png"><img src="http://pic.yupoo.com/kazaff/E9bw5Lp6/medish.jpg" width="640" height="426" border="0" /></a>

图中左上角中的箭头指向的那个位置就是IDE查找的位置，也就是说`java`文件夹对应着编译后的`classes`文件夹。

上图中的项目是基于springMVC模板的，那文件夹这种对应关系又是在哪里设置的呢？其实不同的IDE都有自己的一套项目文件组织结构，之前公司用的是Netbeans，也有人使用Eclipse的，默认情况下它们的项目结构都不太一样，但是不管你用什么IDE，向Tomcat中部署的时候，都要输出一致的，Tomcat理解的标准目录，在Idea的`Artifacts`窗口下可以定义编译部署的项目结构，上图中下方的箭头和红框标出来的就是我们为了测试而修改`mvc-dispatcher-servlet.xml`位置后需要对应做的修改，这样才能保证IDE部署项目时导出正确的部署结构。
	




如何在`Controller`中读取属性文件中的设置？
---

这个问题网上有不少贴，可以看这里：[传送门](http://kanpiaoxue.iteye.com/blog/1989026)，已经是非常具体了。可以看出，`Controller`中靠注解直接把配置文件中的键值注入到类属性中，很是NB啊！

实际测试中还是需要按照第一个问题里说的那样调整一下`Artifacts`的部署结构，这里我发现在Idea13中，我把属性文件直接放在WEB-INF目录下，IDE中竟然显示找不到文件，但是部署后是正常的，就是显示红色的提示让我看着很不爽，所以才扔到`classes`下的~




nginx代理项目中的静态资源文件
---

公司前几个小项目都出现了不同程度的性能问题，其中比较早遇见的就是关于静态文件太多导致的响应卡顿的问题！

其实并不能一定断定就是Tomcat处理静态文件不利导致的，只不过看网上大家都吐槽它，还是果断放弃为好~~我们只需要搭建一个反向代理服务即可，这里第一选择肯定就是`Nginx`。

我在自己的开发机上直接尝试搭建这个测试环境，大概环境如下：

	win7 64bit
	jdk1.7
	tomcat7
	nginx1.7.6

测试项目部署完毕后的目录结构为：

	springMVCDemo
			|
			|--images	//这里就是我们的静态文件夹
			|
			|-WEB-INF		

	
好啦，现在直接修改Nginx的配置文件`/conf/nginx.conf`:

	 server {
        listen       80;
        server_name  localhost;
        root "D:/Program Files/apache-tomcat-7.0.53/webapps/springMVCDemo";

        location / {
            proxy_pass http://localhost:8080/springMVCDemo/;
        }

        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ {
			expires 30d;
		}

		location ~ .*\.(js|css)?$ {
			expires 1h;
		}

很简单吧，这样就搞定了。这里面有个小插曲我需要说一下：我是直接用IDE编译打包成一个`war`文件，然后拷贝到`tomcat`的`webapps`下，由于我的天真，我一直试图尝试在`Nginx`的配置中直接指向`war`文件内部的`images`目录~~哇哈哈哈，网上找了一大圈都没有发现，结果最后才发现一个问题：`tomcat`在启动之初，如果发现需要加载`war`项目，会直接把`war`文件解压出来。阿西吧~

最后，还要叮嘱一件事儿，那就是关于`nginx`的反向代理配置不当造成的安全隐患，具体细节可以看这里： [WooYun](http://drops.wooyun.org/papers/60)。






springMVC中静态文件的处理
---

虽然在上一个问题里我们已经把静态文件交给`Nginx`来处理了，但是我觉得那是生产环境下的配置~~在开发环境下还是要尽可能的直观一些，这样可以让配置较低的开发机更快一些，而且也可以让调试变得简单高效。

其实实现的方法网上说了[好多](http://www.cnblogs.com/fangqi/archive/2012/10/28/2743108.html)，但我还是偏向于使用Spring提供的方式：

	<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-3.1.xsd">

	    <context:component-scan base-package="me.kazaff.springmvc"/>
	
	    <mvc:annotation-driven />
	    <mvc:resources mapping="/images/**"  location="/images/" />
		
		....

	</beans>

这样就可以啦，注意在头部要加上`mvc`这个命名空间的定义，一共要修改两个位置哟：

- 在`beans`节点上增加：
	
	xmlns:mvc="http://www.springframework.org/schema/mvc

- 在`xsi:schemaLocation`属性上增加：

	http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-3.1.xsd
 
其实在测试中还遇到了一个问题，就是一旦加上`mvc:resources`定义后就会造成无法访问页面，错误是：
	
	警告: No mapping found for HTTP request with URI [/springMVCDemo/] in DispatcherServlet with name 'mvc-dispatcher'

所以一定要记得加上：
	
	<mvc:annotation-driven />

理由可以查看这里： [传送门](http://blog.csdn.net/jbgtwang/article/details/7359592)。







javaEE负载均衡环境搭建
---

接下来我们简单聊一下**高可用**和**可伸缩**的解决方案，我们可以在架构设计上做一些工作，从而使得我们的业务应用服务可以搬到集群中进行部署，至于要达到`可伸缩性`而需要解决的技术点我们暂且不表，毕竟这篇文章是新手向，我们主要聊一下集群环境下的负载均衡话题~~

其实负载均衡不仅仅能做到平摊负载，牛逼一些的负载均衡软件还可以负责自动容灾处理。还拿`Nginx`来说，非常简单，看[这里](http://blog.jobbole.com/24574/)和[那里](http://saiyaren.iteye.com/blog/1914865)。能感觉到还是非常的简单明了吧？这也是`Nginx`能脱颖而出的原因：简单粗暴！

但这个世界没有银弹，我们来看一下常用的负载均衡软件的优缺点：[传送门](http://www.csdn.net/article/2014-07-24/2820837)。

看前辈总结了那么多，心里应该有谱了吧？！上面那一篇文章最大的价值在于描述了一个项目从小到大所处的不同阶段应该做什么样的选择，哥甚是喜欢！

我还找到了一个碉堡的视频共大家观看：[传送门](http://edu.51cto.com/lesson/id-27846.html)。






如何在一台物理机运行多个tomcat进程？
---

最后这个问题是非常有意义的，不过你可能会很好奇谁会这么做？首先，部署在同一台物理机上，就谈不上什么**高可用**，毕竟如果这台机器宕机了，所有tomcat实例都会挂掉。而且由于**进程间切换**也会导致性能上的折损，那谁会这么做呢？

并非只有在开发环境或测试环境下为了模拟集群才会这么做，其实生产环境下这么做也是有目的的。

我们先聊聊操作系统和JDK，一般服务器现在都会装64bit的版本，这样可以使内存突破4G的上限。而这个时候可能你就会理所应当的安装64bit版的jdk。那么，问题就来了：**目前来说，64位的jdk版本整体性能不如32位版**！这个现状会随着时间飞逝而逐渐消失。

那么除了jdk本身的性能因素，还有其它问题么？

我们简单的说一下关于`JVM`内存管理的内容。在我们启动tomcat时是可以设置几个和内存相关的参数的，包括：堆大小（新生代），永久代，虚拟机栈，垃圾回收器等。这些参数都是jvm性能调优的关键，我们暂不深究，只是知道它们很重要即可！

刚才提到了，64位的系统和jdk版本可以使内存突破上限，这样我们的jvm相关设置就可以调的更大，理应拿到更好的性能，对吧？但其实不然，来看一下主要的问题点：

- 太大的堆栈会直接导致垃圾回收时间增加，出现明显的卡顿；
- 相同的程序下，64位比32位更吃内存（指针膨胀等）；
- 由于内存分配过大，导致调试工具（jmap等）无法使用（dump一次会产生非常大的文件，而且分析这个文件也变得不再可行），这就要求程序必须有足够高的稳定性，确保不会出现内存泄露

综上所述，目前我们可以选择64位的操作系统+32位的jdk作为搭配，但是这样的话服务器上大量的内存资源就浪费了，浪费是可耻的！

所以才需要我们在一台物理机上运行多个tomcat实例，这样可以达到最佳效果。而且由于我们的目的是为了换取更大的内存使用率，所以多个tomcat进程上的多个项目镜像不需要做特殊处理（会话粘性），直接可以用`nginx`做个`ip_hash`负载均衡即可。

好啦，扯了那么多，到底如何在同一台物理机上运行多个tomcat实例呢？[点我](http://www.importnew.com/12553.html)




最后
---

目前大概也就这些，够开发用了，我会继续努力的！

PS：做新人的感觉真TM爽，每天都那么充实~~

PS2：推荐一款加速网页显示的[大神器](https://github.com/jiacai2050/gooreplacer4chrome#install)，你们懂的。