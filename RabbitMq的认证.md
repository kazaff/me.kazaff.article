一旦要把模块上线，那么安全就成了一个关注点。这在互联网领域更是焦点话题！

我在本地开启RabbitMQ Server后，用 *localhost* 去连接其默认vhost（“/”）,代码一切正常，无需提供用户名密码。

但是一旦把localhost换成真实ip，RabbitMQ就会立刻提示你无权限操作（或者你尝试用localhost去连接非默认vhost也会报错）！

那么我们只需要为RabbitMQ创建用户即可：
	
	rabbitmqctl add_user kazaff 123456

如果需要创建新的vhos：

	rabbitmqctl add_vhost foo

然后还要把新创建的账户绑定到指定的vhost上：
	
	rabbitmqctl set_permissions -p foo kazaff ".*" ".*" ".*"

后面3个".*"分别代表kazaff用户拥有对foo虚拟机的配置，写，读等权限。举个例子：

1. ".*"：表示匹配任何Exchange和queue；
2. " "：表示不匹配任何Exchange和queue；
3. "kazaff-*"：表示匹配任何以kazaff-开头的Exchange和queue。

其他相关的命令如下：

	rabbitmqctl list_permissions -p vhost  //查看vhost上用户权限列表
	
	rabbitmqctl list_user_permissions user  //查看user用户在所有vhost上的配置权限列表

	rabbitmqctl clear_permissions -p vhost user  //删除vhost上user的权限


