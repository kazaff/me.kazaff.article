这两天发生了一件尴尬的事儿，倾听我娓娓道来：我在开发机上（Win7）上写了一段java代码，使用`jodconverter`类调用`OpenOffice`来将office格式的文件转换成pdf。写完以后第一件事儿就是去虚拟机中的CentOS中测试兼容性，一切比想象中的还要顺利！

事儿如果就这么完了，那就没有这篇日志了，昨天交给同事往线上服务器上部署，可他说有异常，提示无法连接openoffice服务。我就说嘛，事儿不可能这么顺利的！

网上查了一下原因，苦逼了：[OpenOffice安装声明](http://www.openoffice.org/dev_docs/source/sys_reqs_aoo41.html)。

> X-Server with 1024 x 768 pixel or higher resolution with at least 256 colors (16.7 million colors recommended)

图形界面
---

沃特华克，还需要X-Server啊，线上服务器上是没有装图形界面的，难道需要在为系统安装一套图形界面么？这勾起了我大学时代的残酷回忆，曾经给Ubuntu安装图形界面搞崩系统N多次！我怎么可能有胆子直接给线上服务器上搞这个动作呢？

可是不搞又不行，领导在后面催得要死要活，只能另想办法了！最坏的打算就是给服务器重装系统，所以拜托运维同学先把服务器上的系统进行了备份，然后找机房负责人协商装系统的事儿，不过在此期间，我还是在本地虚拟机装了一套测试环境来尝试解决这个问题。

[安装jdk](http://hermosa-young.iteye.com/blog/1798026)和[openoffice](http://www.if-not-true-then-false.com/2010/install-openoffice-org-on-fedora-centos-red-hat-rhel/)的流程我就不写了，我们直接从安装`Xvfb`开始。

	yum install  xorg-x11-server-Xvfb

挺顺利的，一次性搞定，不过还是建议先执行`yum update`一下。这个东西我理解的就是相当于给系统装了一套虚拟的图形界面，这样就满足了openoffice的要求。

安装完后，执行：

	Xvfb :1 -screen 0 1024x768x256 &
	soffice -headless -accept="socket,host=127.0.0.1,port=8100;urp;" -nofirstartwizard -display :1 &

执行完上面第一条命令后，系统提示了一个报错：

	......
	Initializing built-in extension GLX
	The XKEYBOARD keymap compiler (xkbcomp) reports:
	> Internal error:   Could not resolve keysym XF86AudioMicMute
	Errors from xkbcomp are not fatal to the X server

不过提示的好像不是致命错误，所以暂时没有理会它。

其实上面第二条命令也可以忽略，如果你用的是`jodconverter3.0+`的话！

现在我们已经可以成功的启动openoffice服务了，我写的那个服务也成功的进行了pdf的生成。

中文字体
---

可别高兴太早，打开生成的pdf一看，妈蛋，无字经书么？你拿我当唐僧么？这个问题其实也很好理解，系统里没有中文嘛，好说好说，把`C:\Windows\Fonts`下所有的字体文件都打包传到服务器上，放在你安装的openoffice的文件夹中，我的是这里：

	/opt/openoffice4/share/fonts/truetype/

不过需要注意的是，放字体的时候最好是先让刚才开启的openoffice服务停止，如果忘记进程id了，可以执行：

	netstat -lnp | grep 8100
	netstat -lnp | grep 2002 //这个是jodconverter3.0+默认使用的端口号

直接kill掉就行了。


ok，完美了，终于可以不用重装系统就搞定问题了！！！

	
	