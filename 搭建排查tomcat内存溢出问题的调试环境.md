上个月赶工上线的门户网站，由于种种原因导致部署到线上服务器后每隔一段时间后就会导致tomcat内存溢出，今天我就要来直面这个棘手的问题。

<!--more-->

要解决的问题对我来说还是有点难度的，原因有二：

1. 代码不是我写的；
2. 我对java并不熟悉。

废话不多说，就由我这个小白依靠GG带领大家来启程吧！

凭借我多年的编程经验，我认为首先要找到趁手的工具，那么，问题就来了，挖掘机技术到底哪家强？……

好吧，GG一下，可以很容易查到很多用来监控jvm实时状态的工具，我们以jconsole为第一款尝试的工具吧。



jconsole
---

这里要说明的是，我们需要搭建的监控环境是在win桌面机上远程监控一台centos服务器。按照网上说的，搭建起这么一个环境没有多大难度，大家可以参考这里：[传送门](http://zhumeng8337797.blog.163.com/blog/static/100768914201282833448384/)。

如果你像我一样碰到了timeout提示，那多半就是centos防火墙拦截导致的，可以暂时关闭防火墙再尝试一下：

	/etc/init.d/iptables stop

好的，终于有了一个监控界面了，是不是感觉心里敞亮了不少呢？不过我感觉还是太笼统了，只能大概知道jvm的状况，而对于我们要排查代码导致的内存泄露问题似乎并没有帮到太大的忙~~

不过可以通过提供的一些信息来判断是否配置了比较合理的参数，比方说可以通过`GC时间`看出是否给tomcat分配了适当的内存大小，尽可能的设置一个合理的内存来减少gc的次数和耗时。

那我们接下来换哪个工具呢？



JProfiler
---

好吧，上大杀器！无需我多讲，我相信没有人会对我的选择有质疑吧？哇哈哈哈哈哈~~不过JProfiler是个商用产品，钱花到位才能享受生活，这一点儿错都没有。

大家可以参考下面这些链接，相信你们很快就能搭建成功：

[传送门1](http://www.flybi.net/article/101),[传送门2](http://sgq0085.iteye.com/blog/1947526),[传送门3](http://blog.csdn.net/attilax/article/details/17077857)

按照上面提供的信息，我相信你很顺利就能安装部署完毕jprofiler（毕竟是商用产品嘛，肯定做的足够简单），但如果你和我一样是第一次用这个玩意儿，肯定会被它的默认界面震出翔！

不要pia，先阅读一下这篇文章： [传送门](http://www.cnblogs.com/jayzee/p/3184087.html)。

这里要提到的一点是，可能是软件版本的原因，按照上面前辈说的方法我却死活查不到方法调用Tree，查了一下GG才发现是需要调整配置选项，如下图：

![查看调用轨迹](http://pic.yupoo.com/kazaff_v/E8qo6nio/W5piy.png)

我这属于暴力解决吧，毕竟我把能开启的选项都选中了，不过想得到的问题也就是速度慢点，对于远程连接方式来说带宽占用多一些而已吧~~



测试代码
---

找到了趁手的兵器，下一步就是要挖的坑了！哦，no，是一段会造成内存溢出的测试代码，我简单地修改了一下tomcat提供的例子中的`HelloWorldExample.java`：

	import java.io.IOException;
	import java.io.PrintWriter;
	import java.util.ResourceBundle;
	
	import javax.servlet.ServletException;
	import javax.servlet.http.HttpServlet;
	import javax.servlet.http.HttpServletRequest;
	import javax.servlet.http.HttpServletResponse;
	
	import java.util.ArrayList;
	
	public class HelloWorldExample extends HttpServlet {
	
	    private static final long serialVersionUID = 1L;
	
	    private static ArrayList list = new ArrayList();
	
	    @Override
	    public void doGet(HttpServletRequest request,
	                      HttpServletResponse response)
	        throws IOException, ServletException
	    {
	        ResourceBundle rb =
	            ResourceBundle.getBundle("LocalStrings",request.getLocale());
	        response.setContentType("text/html");
	        PrintWriter out = response.getWriter();
	
	        out.println("<html>");
	        out.println("<head>");
	        out.println("</head>");
	        out.println("<body bgcolor=\"white\">");
	        
		
		for(int i=0; i < 1000; i++){
			//Object o = new String("by kazaff, index is :" + i);
			HelloWorldExample.list.add(new kazaffBean());
			//o = null;	
		}	
	
		out.println("<div>kazaff in here!!!!!!!</div>");	
	
		out.println("</body>");
	        out.println("</html>");
	
	    }
	}
	
	class kazaffBean {
		String name= "";
	}

然后我们还要手动编译修改后的java代码，进入到`/usr/local/apache-tomcat-7.0.53/webapps/examples/WEB-INF/classes/HelloWorldExample.java`所在的文件夹中，执行下面的命令：

	 javac HelloWorldExample.java -cp /usr/local/apache-tomcat-7.0.53/lib/servlet-api.jar 

当然，你的tomcat路径和我的很可能不同，酌情修改即可~~

然后我们重启tomcat，对了，重启之前最好改一下分配给tomcat的内存上限，改小一些，有助于问题快速的暴露：

	-Xms20m -Xmx20m

重启吧，然后为了加速内存溢出，我们可以使用apache自带的`ab`做压力测试：

	ab -c 100 -n 10000 http://192.168.153.128:81/examples/jsp/

执行上面的这条命令后，基本上tomcat肯定就已经挂掉了，注意，我说的是tomcat挂掉了！也就是说你在终端中执行`jps`命令，不再会看到`Bootstrap`这个进程了！我之所以强调这一点，是因为我在测试中发现直接导致`jconsole`断开连接，并且再也无法建立连接。

而奇怪的是，`jprofiler`照常可以连接并获取到远程服务器上的jvm的监控数据，这种情况一度使我陷入深深的迷惘。因为我在tomcat的logs里死活找不到任何关于内存溢出或其他异常的记录，直到我的目光落在终端上：


	Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "http-bio-81-exec-13"
	……
	java.lang.OutOfMemoryError: GC overhead limit exceeded
	……
	java.lang.OutOfMemoryError: Java heap space
	……
	Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: Java heap space

这才是乖孩子嘛，就应该是这样的才对嘛~~不过还有几个点想不明白：

- 虽然`jps`已经查不到tomcat的进程，但是从jprofiler的线程监控中还是可以看到相关的线程，如下图：

![](http://pic.yupoo.com/kazaff_v/E8rChOsQ/FN83O.png)

- 既然tomcat都已经挂了，为什么`jprofiler`没有像`jconsole`那样连接断开呢？

我这种小白目前是搞不定这两个问题了，还是抛到社区给大牛们分析吧。我们继续往下走~



关于内存溢出
---

通过我上面列出的异常信息，已经是非常常见的了，对于一些java老鸟而言肯定是再熟悉不过的了！不过我还是找到了一篇排版不咋滴但是比较全面的文章： [传送门](http://yanguz123.iteye.com/blog/2017335)。

到此为止，就算做好了一切准备，可以去真正的项目上搞了！


