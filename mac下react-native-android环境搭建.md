最近闲来无事，就决定摆弄摆弄react-native，其实之前尝试过，不过那个时候react-native还没有出android版，而ios版本的环境搭建只需要给xcode装上，基本上就算搭建完成了。而这次大意了，原以为android环境会一样简单明了的，随便看了几篇环境搭建的帖子就以为能轻松完成，结果一搞就搞了一天半啊～

首先吐槽的就是，很多博客里写的安装步骤其实都不是很具体，总是缺少一些细节，对我这种小白来说就会变成噩梦！虽然我这篇可能也无法保证事无巨细，但至少我碰见的坑我会叮嘱一下大家撒～

其实说了那么多，都怪自己太懒，按照官方提供的搭建流程，基本上就会规避非常多的坑，强烈推荐看官方手册：

- [步骤一](https://facebook.github.io/react-native/docs/getting-started.html#content)
- [步骤二](https://facebook.github.io/react-native/docs/android-setup.html)

下面主要来说说我自己碰见的坑吧，希望能帮助到你～

官方推荐（很多博客也推荐），使用brew来安装android-sdk，但我一开始是从另外一个博客上看到的，自己去找了一个mac版本的android sdk manger，按说应该和brew安装的一样，只不过位置不同而已～可是后面却碰到了死活无法找到android sdk的路径问题。单这一个问题就让我花了半天来排查，主要是无法搜索到有用的资料！

删除掉自己下载的那个manger，使用brew安装即可：

	brew install android-sdk
	
不过一定要按照顺序来，第一步肯定是要保证你的mac下已经安装了jdk，直接gg可以搜索到mac版的jdk安装文件的！brew貌似无法安装jdk～～

第二个坑呢，就是这个“ANDROID_HOME”环境变量的问题了，由于第一步我们是使用brew来安装android-sdk的，所以安装位置在“/usr/local/opt/android-sdk”，官方有叮嘱这一步，我的步骤是：

	cd ~
	touch .bash_profile
	vi .bash_profile

然后在打开的输入界面贴进去：

	export ANDROID_HOME=/usr/local/opt/android-sdk
	
保存退出，并关闭终端！！！

第三个坑，是一定要安装最新版本的watchman和flow：

	brew install watchman
	brew install flow
	
不然你执行“react-native run-android”后，开启的node server终端总是会报错，提示什么"root of null"！我由于之前就装过watchman，可能是版本问题，又让我花了tm的多半天！！

第四个坑，是当执行“react-native run-android”后，会下载大量的jar包，如果中间出现卡住了，可以直接ctrl+C，然后重新运行命令，要说的不是这个问题，当你下载完所有的包后，会开始部署代码，这个时候可能会碰到一个类似“xxx parent directory xxxx”无法创建的问题，应该是gradle无权限操作目录的问题，我尝试使用sudo来切换成root账号，结果会碰到由于切换了用户，之前设置的“ANDROID_HOME”环境变量未定义的问题，所以对于和我一样的小白来说，最简单的就是在你的项目文件上右键“简介”，弹出的窗口最下面调整一下操作权限，并记得继承到所有下级目录上！！！


第五个坑，还是绕回来说android-sdk，安装完后执行：

	android
	
该命令会打开一个界面，按照官方说的安装所有的推荐工具包，强烈推荐选择统一的版本，比方说23.0.1，不然会碰到对应版本找不到的报错！我贴一下我这边目前选择的包情况：

![](http://pic.yupoo.com/kazaff/F9aG4ZAQ/4wZMw.jpg)

![](http://pic.yupoo.com/kazaff/F9aG5Lb8/alWUY.jpg)

![](http://pic.yupoo.com/kazaff/F9aG6uNJ/ywiXw.jpg)

![](http://pic.yupoo.com/kazaff/F9aG7Zl0/hpstJ.jpg)
	
未必都有用，但是至少不保错了！


最后，对于跟我一样使用老mac的玩家，强烈推荐不要使用android-sdk自建模拟器，因为那真的实在是太tm的卡了！我足足等了30分钟，开启后还进到android系统后还是很卡～可能是我配置的不好吧，但这种方式的体验真的好差！

按照官方推荐的，我们乖乖的来使用Genymotion，选择个人开发者，注册账号即可免费下载，然后再根据要求下载个最新版本的VirtualBox即可！配置非常简单，启动很迅速，效果好极了！但很多功能都是收费版的，不过对于尝鲜的我来说，足够了！

这一趟下来，感觉好麻烦！还是react-native-ios爽一些，不过android还是不能错过的！！

一切就绪，只需要run-android了！

真机调试一样的简单，只需要插上数据线即可，当然，要记得把手机开启usb调试模式，我使用的是锤子坚果手机，连接后会弹出授权窗口，不确认会提示无法连接设备的哟～

最后你还可能碰见无法加载jsbundle的问题，只需要摇晃手机，在弹出菜单中选择 Dev Setting > Debug Server host for device，然后填入 Mac 的 IP 地址（ifconfig 命令可查看本机 IP）即可！

参考：[http://www.liaohuqiu.net/cn/posts/react-native-1/](http://www.liaohuqiu.net/cn/posts/react-native-1/)
