说来不怕你笑话，搞了好些年的php，可还是会在配置环境的时候出现各种个样的逗比情况。
<!-- more -->
今天用自己的笔记本调试一个php网站，需要搞一个vhost。配置好后发现提示我403错误：

> You don’t have permission to access on this server.
	

按照个人的条件反射，第一件事儿就是把指定的web目录设置为777权限。可是发现还是不行，查了一下网上的解决方案，竟然发现很多人都说是：

	<Directory />
		Options Indexes FollowSymLinks MultiViews
        AllowOverride None
        Order deny,allow
        Allow from all	#这里是重点
    </Directory>
    
但是对我来说，肯定没用～～这个时候我就有点抓瞎了，后来总算知道问题了，**原来不仅仅要对当前web目录设置正确的权限，还要把其父目录设置正确的权限**。

参考：
[Mac配置Apache的Vhosts权限问题](http://www.orzcc.com/2011/07/641629.html)