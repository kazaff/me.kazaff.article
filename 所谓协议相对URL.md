之前在搞前端的时候，看到过很多国外的插件或library都好使用没有指定协议的url，查过原因，但忘记了～～这次又看到，决定记录下来。

<!-- more -->

	
	<script src="//cdnjscn.b0.upaiyun.com/libs/jquery/2.1.1/jquery.min.js"></script>
	
这种写法还有一个较为学术的名字：**协议相对URL（The Protocol-relative URL）**。

具体作用其实很简单，也就是根据页面使用的协议来适配。例如上面的那行代码，如果页面使用的是http，则会加载：

	http://cdnjscn.b0.upaiyun.com/libs/jquery/2.1.1/jquery.min.js
	
如果是https，则加载：

	https://cdnjscn.b0.upaiyun.com/libs/jquery/2.1.1/jquery.min.js
	
就是这么简单明了！！

参考：
[协议相对URL](http://www.fising.cn/2014/09/the-protocol-relative-url.shtml)