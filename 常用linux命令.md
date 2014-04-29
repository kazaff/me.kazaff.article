今天公司新配的服务器到位了，领导嘚瑟的对我说：“kz啊，去，写个死循环测测咱服务器的性能！”。我无语了，呵呵~~不过对服务器的基本情况做个了解还是有必要的~

我对linux命令并不是非常熟悉，常用的几个还行，其他的就得靠搜索了~趁机会把比较重要的命令记录下来，以后省的到处找了！

# 1. 查看操作系统位数
	uname -a
返回的信息中若包含x86_64，则表明是64位，若显示i386，i686则表明是32位的。

# 2. 查看CentOS版本
	
	cat /etc/issue
或
	cat /etc/redhat-release

# 3. 查看系统分区
	df -lh

# 4. 查看内存
	free -m
单位为M

# 5. 查看cpu型号
	grep "model name" /proc/cpuinfo

# 6. 查看核心数
	grep "core id" /proc/cpuinfo | sort -u | wc -l

# 7. 查看线程数
	grep "processor" /proc/cpuinfo | sort -u | wc -l

# 8. 查看时区
	cat /etc/sysconfig/clock

# 9. 查看主机名
	cat /etc/sysconfig/network

# 10. 修改主机时间
	date -s 04/28/2014
	date -s 12:40:23
	date //查看	

# 11. 安装lnamp
我选择安装wdlinux提供的[lnamp集成环境](http://www.wdlinux.cn/bbs/thread-6292-1-1.html)。
