两台服务器之间需要进行文件的主从同步，常用的方案是rsync+inotify，网上可以看到一大把相关的[资料](http://dl528888.blog.51cto.com/2382721/771533)。

可我在按照前辈说的方法配置时却遇到了一些问题，首先是因为自己的懒惰，直接从网页上把shell脚本拷贝到win系统下的编辑器中，然后再放入cecntos中，结果发现执行报错，是那种很诡异的提示：

	syntax error: unexpected end of file

只需要在centos中用vi手动key一遍脚本即可，这个错误应该是有win->linux时编码不同导致的。

接下来脚本执行成功了，可每次同步时都会报错，提示无法连接从机地址，这是由于防火墙没有放行对应端口导致的，简单的方法就是关闭防火墙：

	service iptables stop

最后我又发现，除了mv命令以外其他命令都可以触发同步，唯独mv不行，查了一下发现是上面前辈提供的脚本并没有监控mv对应的事件导致的，详情可见这篇[博文](http://www.lvtao.net/tool/inotify.html)。

知道原因了就很简答了，把最终脚本改成：

	#!/bin/bash
	host=192.168.76.135
	src=/www/       
	des=web
	user=kazaff
	/usr/local/inotify/bin/inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%f%e' -e  modify,delete,create,attrib,moved_to $src | while read files
	do
	/usr/local/rsync/bin/rsync -vzrtopg --delete --progress --password-file=/usr/local/rsync/rsync.passwd $src $user@$host::$des >>dev/null 2>>/tmp/rsync.log  
	done

