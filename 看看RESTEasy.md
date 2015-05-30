由于dubbox的这个[问题](https://github.com/dangdangdotcom/dubbox/issues/30)，我决定先了解一下这个框架。为了快速认识RESTEasy，我决定翻译一篇看似还不错的[文章](http://www.mastertheboss.com/jboss-frameworks/resteasy/resteasy-tutorial)。下面就开始，不过我英文能力确实不如我的中文能力，请大家见谅。

---

RESTEasy是一个实现了JAX-RS规范的轻量实现，该规范是针对基于http协议的RESTful Web Service而提供标准的JAVA API定义。这篇指南我们来手把手演示如何使用Resteasy来创建一些简单的RESTful Web Service。

#安装RESTEasy
---
根据你所使用的不同版本的JBoss，安装RESTEasy需要不同的步骤。

##JBoss 6/7

在此版本下安装RESTEasy你不需要下载任何包，RESTEasy类已经内嵌在server中。应用服务器会自动发现导出成RESTful的服务资源。

更多关于RESTEasy和JBoss 7的例子可以看[这里](http://www.mastertheboss.com/resteasy/restful-web-services-on-jboss-as-7)。

你唯一需要做的只是在`web.xml`中插入正确的命名空间：

	<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
 
	</web-app>

##旧JBoss版本

如果你工作在旧的JBoss版本下，安装RESTEasy你只需要做下面几步：

###1.下载RESTEasy

你需要先从[这里](http://sourceforge.net/projects/resteasy/files/Resteasy%20JAX-RS/)下载最新的RESTEasy稳定版。

###2.把RESTEasy类加入你的web应用

![](http://www.mastertheboss.com/images/stories/ws/resteasy-libs.PNG)

###3.定义listenser和bootstrap

在`web.xml`中添加下面的配置：

	<?xml version="1.0"?>
	<!DOCTYPE web-app PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
	"http://java.sun.com/dtd/web-app_2_3.dtd">
	<web-app>
	    <display-name>RestEasy sample Web Application</display-name>
	 
	    <listener>
	        <listener-class>
	            org.jboss.resteasy.plugins.server.servlet.ResteasyBootstrap
	        </listener-class>
	    </listener>
	 
	    <servlet>
	        <servlet-name>Resteasy</servlet-name>
	        <servlet-class>
	            org.jboss.resteasy.plugins.server.servlet.HttpServletDispatcher
	        </servlet-class>
	        <init-param>
	            <param-name>javax.ws.rs.Application</param-name>
	            <param-value>sample.HelloWorldApplication</param-value>
	        </init-param>
	    </servlet>
	 
	    <servlet-mapping>
	        <servlet-name>Resteasy</servlet-name>
	        <url-pattern>/*</url-pattern>
	    </servlet-mapping>
	 
	</web-app>

**org.jboss.resteasy.plugins.server.servlet.ResteasyBootstrap**类是一个**ServletContextListener**，它负责配置实例化 ResteasyProviderFactory 和 Registry。

然后，你还需要设置 **javax.ws.rs.core.Application** 参数，赋值一个被用于提供给你的应用去作为所有JAX-RS的根资源和提供者的入口单例实现（工厂模式）。不过在JBoss 6-M4或更高版本的环境下可以省略这个设置，因为在你部署应用时应用服务器已经帮你自动处理过了。

最终你还需要指定一个url模式，用于RESTEasy来识别对应服务，如上面的配置例子，`/*`表示所有资源都会被RESTEasy servlet匹配。

###4.创建一个单例类

现在，我们来创建一个单例类，该类就是前面提到的 **javax.ws.rs.core.Application** 所需要的实现：

	package sample;
	 
	import java.util.HashSet;
	import java.util.Set;
	import javax.ws.rs.core.Application;
	 
	import sample.HelloWorld;
	 
	public class HelloWorldApplication extends Application
	{
	    private Set<Object> singletons = new HashSet();
	    private Set<Class<?>> empty = new HashSet();
	 
	    public HelloWorldApplication() {
	        // 把你的rest资源添加到这个属性中
	        this.singletons.add(new HelloWorld());
	    }
	 
	    public Set<Class<?>> getClasses()
	    {
	        return this.empty;
	    }
	 
	    public Set<Object> getSingletons()
	    {
	        return this.singletons;
	    }
	}

在启动时上面声明的这个类会被用于初始化你想提供的RESTful服务，在这个例子中，我们加了一个 HelloWorld 服务。

####资源的自动发现

如果你更希望让RESTEasy自动去发现你的资源，而不是像刚才那样手动添加一个单例bean来完成这个目的，你可以在`web.xml`中添加下面的配置：

	<context-param>
	  <param-name>resteasy.scan</param-name>
	  <param-value>true</param-value>
	</context-param>
	<context-param>
	  <param-name>resteasy.servlet.mapping.prefix</param-name>
	  <param-value>/</param-value>
	</context-param>


#创建你的第一个RESTful服务

基于REST风格的架构，一切都是资源。 资源可以通过http提供的通用接口（POST,GET,PUT,DELETE）来操作。所有资源都应该支持这些通用接口，并且资源都应该有一个全局唯一的ID（通常指的就是URL）。

在我们的第一个例子里，我们创建一个RESTful服务，每当它被以get方式访问请求时，会返回一个字符串：

	package sample;
 
	import javax.ws.rs.*;
	 
	@Path("tutorial")
	public class HelloWorld
	{
	    @GET
	    @Path("helloworld")
	    public String helloworld() {
	        return "Hello World!";
	    }
	}

在类上添加的那个**@Path**注解的作用是指定了该资源的统一URL前缀，而方法上的**@Path**注解则是定义了更具体的某一个资源操作。（译者：原文中的“action”违背了RESTful规范，资源url应该以名词为主）

在这个例子中，如果你想调用我们的helloworld服务，你需要访问下面这个URL（我们假设应用打包为：RestEasy.war）:

![](http://www.mastertheboss.com/images/stories/ws/resteasy-path.PNG)

访问后在你的浏览器会显示“Hello World!”字符串。现在我们添加一个参数：

	@GET
	@Path("helloname/{name}")
	public String hello(@PathParam("name") final String name) {
	  return "Hello " +name;
	}

现在，我们定义了一个“helloname”服务，它接受一个**{name}**参数。服务会从请求url中获取对应参数并返回，如下：

http://localhost:8080/RestEasy/tutorial/helloname/francesco

将返回：

Hello Francesco



#使你的RESTEasy服务返回XML

根据JAX-RS规范要求，RESTEasy支持多个JAXB注解，并提供JAXB Providers。

RESTEasy会根据资源的参数类型或返回类型来选择不同的provider。当方法返回的对象的类声明加上了JAXB注解（@XmlRootEntity或@XmlType）时，RESTEasy会选择注解指定的JAXB Provider。

举个例子，下面将返回xml：

	@GET
	@Path("item")
	@Produces({"application/xml"})
	public Item getItem() {
		Item item = new Item("computer",2500);
		return item;
	} 

现在把这个方法加到你的 HelloWorld 服务中，然后调用这个服务（http://localhost:8080/RestEasy/tutorial/item），将返回下面这个xml：

	<item>
	   <description>computer</description>
	   <price>2500</price>
	</item>

如果你需要返回一个数组，你可以简单的这么写：

	@GET
	@Path("itemArray")
	@Produces({"application/xml"})
	public Item[]  getItem() {
	  Item item[] = new Item[2];
	  item[0] = new Item("computer",2500);
	  item[1] = new Item("chair",100);
	 
	  return item;
	}

调用后将会返回：

	<collection>
	   <item>
	     <description>computer</description>
	     <price>2500</price>
	   </item>
	   <item>
	     <description>computer</description>
	     <price>2500</price>
	   </item>
	</collection>

有时候，你需要使用标准的java集合，这时候你就不得不提供一个包装类：

	@GET
	@Path("itemList")
	@Produces({"application/xml"})
	 public ItemList  getCollItems() {
	  ArrayList list = new ArrayList();
	  Item item1 = new Item("computer",2500);
	  Item item2 = new Item("chair",100);
	  Item item3 = new Item("table",200);
	 
	  list.add(item1);
	  list.add(item2);
	  list.add(item3);
	 
	  return new ItemList(list);
	 }

我们来看一下 **ItemList** 包装类的定义：

	package sample;
 
	import javax.xml.bind.annotation.XmlRootElement;
	import javax.xml.bind.annotation.XmlElement;
	import java.util.List;
	 
	 
	@XmlRootElement(name="listing")
	public class ItemList
	{
	    private List<Item> items;
	 
	    public ItemList(){}
	 
	    public ItemList(List<Item> items)
	    {
	        this.items = items;
	    }
	 
	    @XmlElement(name="items")
	    public List<Item> getItems()
	    {
	        return items;
	    }
	}

返回的xml如下：

	<listing>
 
	  <items>
	    <description>computer</description>
	    <price>2500</price>
	  </items>
	
	  <items>
	    <description>chair</description>
	    <price>100</price>
	  </items>
	 
	  <items>
	    <description>table</description>
	    <price>200</price>
	  </items>
	 
	</listing>


#使用JSON providers

根据[文档](http://docs.jboss.org/resteasy/docs/2.0.0.GA/userguide/html/Built_in_JAXB_providers.html)，RESTEasy支持多个不同的providers去创建xml，而JSON是一个非常有用的返回集合的结构，它支持lists，sets，arrays，如下：

	@GET
	@Path("items")
	@Produces("application/json")
	@Mapped
	public ItemList  getJSONItems() {
	  ArrayList list = new ArrayList();
	  Item item1 = new Item("computer",2500);
	  Item item2 = new Item("chair",100);
	  Item item3 = new Item("table",200);
	 
	  list.add(item1);
	  list.add(item2);
	  list.add(item3);
	 
	  return new ItemList(list);
	}

返回的数据将依照下面的格式：

	{xtypo_code}
	{"listing":{"items":[{"description":{"$":"computer"},"price":{"$":"2500"}},{"description":{"$":"chair"},"price":{"$":"100"}},{"description":{"$":"table"},"price":{"$":"200"}}]}}    
	{/xtypo_code}

你可以使用下面的ajax客户端来调用这个json服务：

	<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
	"http://www.w3.org/TR/html4/loose.dtd">
	<html>
    	<head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
            <script type="text/javascript">
				function createXHR() {
				    var request = false;
				    try {
				        request = new ActiveXObject('Msxml2.XMLHTTP');
				    } catch (err2) {
				        try {
				            request = new ActiveXObject('Microsoft.XMLHTTP');
				        } catch (err3) {
				            try {
				                request = new XMLHttpRequest();
				            } catch (err1) {
				                request = false;
				            }
				        }
				    }
				    return request;
				}
				 
				function loadJSON(fname) {
				    var xhr = createXHR();
				    xhr.open("GET", fname, true);
				    xhr.onreadystatechange = function() {
				        if (xhr.readyState == 4) {
				            if (xhr.status != 404) {
				                var data = eval("(" + xhr.responseText + ")");
				                document.getElementById("zone").innerHTML = "<h2>Items:</h2>";
				                for (i = 0; i < 3; i++) {
				                    document.getElementById("zone").innerHTML += data.listing.items[i].description + ', price <i>' + data.listing.items[i].price + "</i><br/>";
				                }
				            } else {
				                document.getElementById("zone").innerHTML = fname + " not found";
				            }
				        }
				    }
				    xhr.send(null);
				}
            </script>
			<title>Ajax Get JSON Demo</title>
		</head>

	    <body bgcolor="#FFFFFF">
	        <p>
	            <font size="+3">Ajax JSON/JAXB Demo</font>
	        </p>
	        <hr>
	            <FORM name="ajax" method="POST" action="">
	                <p>
	                    <INPUT type="BUTTON" value=" Click to load the JSON file "
	                        ONCLICK="loadJSON('resteasy/tutorial/items')">
	                </p>
	 
	            </FORM>
	 
	            <div id="zone"></div>
	 
	    </body>
	</html>       

注意上面javascript部分，它负责从json中拿到数据并放到“zone”DIV中。

	document.getElementById("zone").innerHTML += data.listing.items[i].description + ', price <i>' + data.listing.items[i].price + "</i><br/>";

可以阅读这个指南的第二部分：[RESTEasy web parameters handling](http://www.mastertheboss.com/resteasy/resteasy-tutorial-part-two-web-parameters)。