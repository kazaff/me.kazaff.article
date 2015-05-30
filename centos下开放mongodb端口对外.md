对于我这种菜鸟来说，这个问题网上的很多帖子都写得不足够清楚，哪怕遗漏了一点，也会导致我配置失败！

东拼西凑总算凑出正确的配制方法（至少我确实成功了！）

好吧，记录一下步骤：

	vi /etc/sysconfig/iptables

然后，添加下面一行：

		-A INPUT -p tcp -m tcp --dport 27017 -j ACCEPT

切记的是，不要把这行加载到末尾，应该加载在大概这个位置：

		-A INPUT -p tcp -m tcp --dport 27017 -j ACCEPT
		-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
		....
		-A INPUT -p icmp -j ACCEPT
		-A INPUT -i lo -j ACCEPT
		-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
		-A INPUT -j REJECT --reject-with icmp-host-prohibited
		-A FORWARD -j REJECT --reject-with icmp-host-prohibited

一开始就是因为添加在文件末尾，死活不通过的有没有！

最后：

	service iptables restart

你可以使用下面的命令来查看配置后的结果：

	iptables -L