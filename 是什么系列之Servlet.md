公司已经开始大面积转型j2ee，正在使用的是整套java web的技术解决方案：Tomcat+springMVC+jdbc。作为一个从php转型过来的程序猿，还是需要好好消化一下java web开发的一些基础概念的。实不相瞒，我就一直闹不懂什么是Servlet，总是无法正确的理解它的作用及位置~

刚好今天有时间，就找了篇不错的技术贴好好科普一下，下面记录一些相关重点。

首先，先看一下我们要解决的疑惑：

1. 以Tomcat为例了解Servlet容器是如何工作的？
2. 一个Web工程在Servlet容器中是如何启动的？
3. Servlet容器如何解析你在web.xml中定义的Servlet？
4. 用户的请求是如何被分配给指定的Servlet的？
5. Servlet容器如何管理Servlet生命周期？

先从Servlet容器聊起
---
Servlet和Servlet容器，从名字上来猜，就想水和器皿，之所以独立出两个东西，完全是为了满足工业化生产，也就是说是为了解耦，通过标准接口来让这两个东西协作从而完成实际需求。

目前成熟的Servlet容器产品很多，像Jetty，Tomcat都是java web开发人员耳熟能详的。我们下面就以Tomcat为例来讲一下Servlet容器如何管理Servlet的。

![](http://pic.yupoo.com/kazaff/DSjSZM95/KxYyT.jpg)

从上图可以看出，Tomcat的容器分为四个等级，真正管理Servlet的是Context容器，可以看到，一个Context对应一个web工程，这在Tomcat配置文件里也可以很容易的发现这一点：

	 <Context path="/projectOne " docBase="D:\projects\projectOne" reloadable="true" />

Servlet容器的启动过程
---
我们已经知道，一个Web项目对应一个Context容器，也就是Servlet运行时的Servlet容器，添加一个Web应用时将会创建一个StandardContext容器，并且给这个Context容器设置别要的参数（url，path等）。

下面看一张时序图，来分析一下Tomcat的启动过程：

![](http://pic.yupoo.com/kazaff/DSjXuOA2/6kRPD.jpg)

上图描述的主要是类之间的时序关系，我们主要关注StandardContext容器的启动过程。

当Context容器初始化状态设为init时，添加在Context容器上的Listener将会被调用，我们主要看一下ContextConfig做了什么，这个类会负责整个Web应用的配置文件的解析工作：

1. 创建用于解析xml配置文件的contextDigester对象
2. 读取默认context.xml配置文件，如果存在则解析它
3. 读取默认Host配置文件，如果存在则解析它
4. 读取默认Context自身的配置文件，如果存在则解析它
5. 设置Context的DocBase

ContextConfig的init方法完成后，Context容器会执行startInternal方法，主要做的是：

1. 创建读取资源文件的对象
2. 创建ClassLoader对象
3. 设置应用的工作目录
4. 启动相关的辅助类，如：logger，realm，resources等
5. 修改启动状态，通知感兴趣的观察者（web应用的配置）
6. 子容器的初始化
7. 获取ServletContext并设置必要的参数
8. 初始化“load on startup”的Servlet

Web应用的初始化工作实在ContextConfig的configureStart方法中实现的，应用的初始化主要是要解析web.xml文件，这个文件描述了一个web应用的关键信息，也是一个web应用的入口。

Tomcat首先会创建globalWebXml对象，接着是hostWebXml对象，再然后是对应应用的WebXml对象。Tomcat会将ContextConfig从web.xml文件中解析出的各项属性保存在WebXml对象中。再然后会将WebXml对象中的属性设置到Context容器中，这里面包括创建的Servlet对象，filter，listener等等，这个工作在WebXml的configureContext方法中完成，在该方法中会将Servlet包装成Context容器中的StandardWrapper，为什么要将Servlet包装成StandardWrapper而不直接创建Servlet对象呢？这是因为StandardWrapper是Tomcat容器的一部分，它具有容器的特征，而Servlet是一个独立的Web开发标准，不应该强耦合在Tomcat中。

除了将Servlet包装成StandardWrapper并作为子容器添加到Context中，其它的所有web.xml属性都被解析到Context中，所以说Context容器才是真正运行Servlet的Servlet容器。一个Web应用对应一个Context容器，容器的配置属性由应用的web.xml指定，这样我们就能理解web.xml到底起到什么作用了。

创建Servlet实例
---
我们已经完成了Servlet的解析工作，并且被包装成StandardWrapper添加在Context容器中，但它仍然不能工作，因为我们还没有实例化它。

如果Servlet的load-on-startup配置项大于0，那么在Context容器启动的时候就会实例化它，之前提到在解析配置文件时会读取在Tomcat目录下的Conf文件夹下的Web.xml文件的默认配置项，用来创建globalWebXml，其定义了两个Servlet，分别是：org.apache.catalina.servlets.DefaultServlet 和 org.apache.jasper.servlet.JspServlet 它们的 load-on-startup 分别是 1 和 3，也就是当Tomcat启动时这两个Servlet就会被实例化。

创建Servlet实例的方法是从Wrapper.loadServlet开始的，该方法要完成的就是获取servletClass，然后把它交给InstanceManager去创建一个基于servletClass.class的对象，并且如果这个Servlet配置了jsp-file，那么这个servletClass就是conf/web.xml中定义的org.apache.jasper.servlet.JspServlet了！

初始化Servlet
---
初始化Servlet这个工作是在StandardWrapper的initServlet方法中完成的，这个方法很简单，就是调用Servlet的init方法，同时把包装了StandarWrapper对象的StandardWrappFacade作为ServletConfig传递给Servlet，原因待会在说~

如果该Servlet关联的是一个jsp文件，那么在前面初始化的就是JspServlet，接下去会模拟一次简单的请求，请求调用这个jsp文件，以便编译这个jsp文件为class，并初始化这个class。

这样Servlet对象就初始化完成了，事实上Servlet从被 web.xml 中解析到完成初始化，这个过程非常复杂，中间有很多过程，包括各种容器状态的转化引起的监听事件的触发、各种访问权限的控制和一些不可预料的错误发生的判断行为等等。我们这里只抓了一些关键环节进行阐述，试图让大家有个总体脉络。

下面看一下时序图：

![](http://pic.yupoo.com/kazaff/DSl5s0re/X35p4.jpg)

Servlet体系结构
---

![](http://pic.yupoo.com/kazaff/DSladGCE/xeHEC.jpg)

上图可以看出，Servlet规范就是基于这几个类运转的，与Servlet主动关联的是三个类：ServletConfig，ServletRequest和ServletResponse。这三个类都是通过容器传递给Servlet的，其中ServletConfig是在Servlet初始化时就传递给Servlet的（前面我们提到了）。而后面这两个是在请求到达时调用Servlet时传参过来的。

那么，ServletConfig和ServletContex对Servlet有何作用呢？从ServletConfig的接口中声明的方法不难看出，这些方法都是为了获取这个Servlet的一些配置属性，而这些配置属性可能在Servlet运行时会被用到。

而ServletContext的作用呢？这里提到了Servlet的运行模式，Servlet的运行模式是一个典型的“握手型的交互式”运行模式。所谓“握手型的交互式”就是两个模块为了交换数据通常都会准备一个交易场景，这个场景一直跟随个这个交易过程直到这个交易完成为止。这个交易场景的初始化是根据这次交易对象指定的参数来定制的，这些指定参数通常就会是一个配置类。所以对号入座，交易场景就由ServletContext来描述，而定制的参数集合就由ServletConfig来描述。而ServletRequest和ServletResponse就是要交互的具体对象了，它们通常都是作为运输工具来传递交互结果。

下图是ServletConfig和ServletContex在Tomcat容器中的类关系图：

![](http://pic.yupoo.com/kazaff/DSliYtlh/9JjIN.jpg)

上图可以看出StandardWrapper和StandardWrapperFacade都实现了ServletConfig接口，而StandardWrapperFacade是 StandardWrapper门面类。所以传给Servlet的是StandardWrapperFacade对象，这个类能够保证从StandardWrapper中拿到 ServletConfig所规定的数据，而又不把ServletConfig不关心的数据暴露给Servlet。

同样ServletContext也与ServletConfig有类似的结构，Servlet中能拿到的ServletContext的实际对象也是 ApplicationContextFacade对象。ApplicationContextFacade同样保证ServletContext只能从容器中拿到它该拿的数据，它们都起到对数据的封装作用，它们使用的都是门面设计模式。

通过ServletContext可以拿到Context容器中一些必要信息，比如应用的工作路径，容器支持的Servlet最小版本等。

Tomcat一接受到请求首先将会创建org.apache.coyote.Request和org.apache.coyote.Response，这两个类是Tomcat内部使用的描述一次请求和响应的信息类，它们是一个轻量级的类，作用就是在服务器接收到请求后，经过简单解析将这个请求快速的分配给后续线程去处理，所以它们的对象很小，很容易被JVM回收。

接下去当交给一个用户线程去处理这个请求时又创建org.apache.catalina.connector.Request和org.apache.catalina.connector.Response对象。这两个对象一直穿越整个Servlet容器直到要传给Servlet，传给Servlet的是Request和Response的门面类RequestFacade和RequestFacade，这里使用门面模式与前面一样都是基于同样的目的——封装容器中的数据。一次请求对应的Request和Response的类转化如下图所示：

![](http://pic.yupoo.com/kazaff_v/DSr8j5Tq/ri7Eh.jpg)

Servlet如何工作
---
我们已经清楚了Servlet是如何被加载的，Servlet是如何被初始化的，以及Servlet的体系结构，现在的问题是它如何被调用？

当用户从浏览器向服务器发起一个请求，通常包含如下信息：**http://hostname:port/contextpath/servetpath**，hostname和port是用来与服务器建立tcp连接的，而后面的URL才是用来选择服务器中哪个子容器服务用户的请求。

根据URL来映射正确的Servlet容器的工作有专门的一个类来完成：org.apache.tomcat.util.http.mapper，这个类保存了Tomcat的Container容器中的所有子容器的信息，当org.apache.catalina.connector.Request类在进入Container容器之前，mapper将会根据这次请求的hostname和contextpath将host和context容器设置到Request的mappingData属性中。所以当Request进入Container容器之前就已经确定了它要访问哪个子容器了！看下图：

![](http://pic.yupoo.com/kazaff_v/DSrk9Yhb/3hHbN.jpg)

上图描述了一次Request请求是如何达到最终的Wrapper容器的，我们现正知道了请求是如何达到正确的Wrapper容器，但是请求到达最终的Servlet还要完成一些步骤，必须要执行Filter链，以及要通知你在web.xml中定义的listener。

接下去就要执行Servlet的service方法了，通常情况下，我们自己定义的servlet并不是直接去实现javax.servlet.servlet接口，而是去继承更简单的HttpServlet类或者GenericServlet类，我们可以有选择的覆盖相应方法去实现我们要完成的工作。

Servlet的确已经能够帮我们完成所有的工作了，但是现在的web应用很少有直接将交互全部页面都用servlet来实现，而是采用更加高效的MVC框架来实现。这些MVC框架基本的原理都是将所有的请求都映射到一个Servlet，然后去实现service方法，这个方法也就是MVC框架的入口。

当Servlet从Servlet容器中移除时，也就表明该Servlet的生命周期结束了，这时Servlet的destroy方法将被调用，做一些扫尾工作。

Session与Cookie
---

![](http://pic.yupoo.com/kazaff_v/DSrB5KEx/XhS9F.jpg)

从上图可以看到Session工作的时序图，有了Session ID服务器端就可以创建HttpSession对象了，第一次触发是通过request.getSession()方法，如果当前的Session ID还没有对应的HttpSession对象那么就创建一个新的，并将这个对象加到 org.apache.catalina.Manager的sessions容器中保存，Manager类将管理所有Session的生命周期，Session过期将被回收，服务器关闭，Session将被序列化到磁盘等。只要这个HttpSession对象存在，用户就可以根据Session ID来获取到这个对象，也就达到了状态的保持。


引用[原文](http://www.ibm.com/developerworks/cn/java/j-lo-servlet/)

