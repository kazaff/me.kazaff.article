之前写过在window下管理nodejs版本的文章，也写过一个mac下的，今天再补充一个centos下的，套装完成！

我这人儿性子急，不喜欢超过三步的工作，所以我毅然决然选择使用nvm来快速安装nodejs，那到底有多快呢？

我的服务器系统是centOS6.5的，所以第一步我只需要：

	yum -y install git

如果你的centos版本不够新，那么安装git之前，你可能还要做一些[额外工作](http://m114.org/installing-git-on-centos/)。

第二步：

	git clone https://github.com/creationix/nvm.git ~/.nvm

最后：

	source ~/.nvm/nvm.sh

这样就把nvm安装完毕了！那么剩下的安装nodejs环境就太简单了，只需要下面一行命令：

	nvm install 0.10.26

搞定！

不仅如此，以后升级node版本，切换不同版本，尽情的使用nvm吧。

nvm的常用命令，看[这里](https://github.com/creationix/nvm)。