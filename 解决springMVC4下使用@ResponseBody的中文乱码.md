由于现在的项目一般都追求前后端分离，依靠Ajax进行通信，这样有助于团队分工、项目维护和后期的平台移植，这就使得后端框架对视图层的功能要求越来越低~

今天要说的是基于SpringMVC开发web后端时，为了简单而直接在控制器方法中返回json字符串时碰到的中文乱码问题。算是非常基础的问题，大牛请绕道~

其实我自己一开始也没觉得能有多复杂，认为一搜索就能找到一大把解决方案，所以没有计划耗费多久时间，更没打算转成写一篇博文记录过程。可不曾想到，足足花了我2个半小时，今天看来又要加班了！其实确实在GG和百度中搜索到了大量的相关解决方案，晒晒捡捡也发现至少有不下七八种解决方案。可悲剧的是统统在我测试下无效。

对于JAVAEE，我真真儿的是新手，项目也没有给我太多时间来深究源码，只能快速的试错，总算把几个方案拼凑出来一个能用的了！下面我就简单的说一下我的解决方法吧。

首先，我们要知道，为毛`@ResponseBody`不支持中文：[传送门](http://tchen8.iteye.com/blog/993504)，这是我找到的写的最细的一篇文章了，尽管它并没有解决我的问题。

知道了原因，再来选择解决方案，一开始满心欢喜的找到一个最简单的方案：

	@RequestMapping(value = "/add", produces = {"application/json;charset=UTF-8"})

可是不管用撒，原因不明~~

好吧，那再试一个：

	response.setContentType("text/plain;charset=UTF-8");

也不行哇，这个其实只是设置了响应头中的字符集，但是`@ResponseBody`最终还是会把字符以“ISO-8859-1”的方式输出，可恶！

简单的方法木牛了，只能选择手动设置字符转换类了：

	<bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter" >  
		<property name="messageConverters">   
			<list>   
				<bean class = "org.springframework.http.converter.StringHttpMessageConverter">   
					<property name = "supportedMediaTypes">
						<list>
							<value>text/html;charset=UTF-8</value>   
						</list>   
					</property>   
				</bean>   
			</list>   
	   </property>  
	</bean>  

注意，需要把这段放在`xxx-servlet.xml`中`<context:component-scan base-package="xxxxx"/>`前面哦~

其实这样已经可以解决了，不过不完美，留一下这个时候的响应头，你会发现体积非常大（Accept-Charset会达到4K+），这是因为默认情况下`StringHttpMessageConverter.writeInternal()`会将所有可用字符集回写到响应头中，这会消耗非常大的带宽！浪费可耻！

一筹莫展了吧~幸好发现`StringHttpMessageConverter`提供的参数：`writeAcceptCharset`，所以最终的写法如下：

	<!-- 用于使用@ResponseBody后返回中文避免乱码 -->
    <bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter" >
        <property name="messageConverters">
            <list>
                <bean id="stringHttpMessageConverter" class="org.springframework.http.converter.StringHttpMessageConverter">
                    <property name="writeAcceptCharset" value="false" /><!-- 用于避免响应头过大 -->
                    <property name="supportedMediaTypes">
                        <list>
                            <value>text/html;charset=UTF-8</value>
                        </list>
                    </property>
                </bean>
            </list>
        </property>
    </bean>

哎，两个多小时就干了点儿这，感觉好尴尬啊！！
