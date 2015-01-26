redis已经成为我们项目中不可或缺的一部分，充当了各种业务数据的持久化：用户会话，部分数据关系，缓存等。那么，保证redis的高可用高扩展性自然成了一个不可忽视的目标。

值得庆幸的是业界已经有大量的[成功案例](http://jolestar.com/redis-ha/)用来驾驭redis，我们只需要拿来主义即可。今天要聊的就是一款普世的redis中间代理：[twemproxy](http://zhangxiong0301.iteye.com/blog/2157757)。相关内容可以在网上找到很多，这里肯定没必要重复了，我就先来说一下本人安装twemproxy时遇到的问题吧。

先安装git：

	yum install git

这个没啥好说的，接下来我们来下载twemproxy源文件：

	cd /usr/local
	git clone https://github.com/twitter/twemproxy.git
	cd twemproxy

第一次编译：

	autoreconf -fvi

结果提示autoconf版本太低，我们只能[更新autoconf](http://zhaohe162.blog.163.com/blog/static/3821679720147276238862/)。

第二次编译，依然不顺利，报错如下：

	autoreconf: Entering directory `.'
	autoreconf: configure.ac: not using Gettext
	autoreconf: running: aclocal --force -I m4
	Can't exec xxxxxxx //忘记了，不过不重要

查了一下，原因是由于我们手动安装的autoconf，导致其他两个必要的依赖包缺失，**依照顺序**安装下面两个包：

1. automake-1.12.tar.gz： ftp://ftp.gnu.org/gnu/automake/automake-1.12.tar.xz
2. [libtool-2.2.4.tar.gz](http://ftp.gnu.org/gnu/libtool/libtool-2.2.4.tar.gz)

一切就绪了，可以顺利安装了。


最后需要**提醒**你看一下twemproxy[不支持的命令集合](https://github.com/twitter/twemproxy/blob/master/notes/redis.md)，把自己项目中与redis相关的逻辑都过滤一遍，避免执行失败。

