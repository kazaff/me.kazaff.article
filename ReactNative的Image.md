今天就聊一个问题，React Native的Image读取本地图片的方法。（甚少见我如此不罗嗦了吧？那是因为真的没时间啊……）

按照官方[文档](http://facebook.github.io/react-native/docs/image.html#content)中的描述，RCT（React Native）中考虑到诸多原因，提供了一种快速获取本地图片资源的方法：`require('image!你的图片名称')`。

看上去很简单，但对于我这种移动端开发小白来说，简单的有点简陋了，首先，我们要从如何把本地图片导入到项目说起，在xcode的文件导航栏，找到你的项目文件夹，点击其中的“Images.xcassets”，然后你就会看到右边工作区界面，然后直接把你想导入的本地图片直接拖入工作区即可，具体步骤可参见[这里](http://www.cocoachina.com/ios/20141210/10587.html)，虽然该帖子的xcode版本不是最新的，但步骤没啥问题。

这里注意的是，我们只能导入**png**，这可能并不是xcode要求的，但是RCT只会去读取png图片～～此外，我在导入的时候，xcode总是提醒我**权限不够**，你只需要修改“Images.xcassets”文件的权限即可，如下图：

![](http://pic.yupoo.com/kazaff/EFjwGAwJ/UVRDA.png)

好了，接下来你就可以使用官方提供的代码实例测试了：

	return (
    	<View>
      		<Image
        		style={styles.icon}
        		source={require('image!myIcon')}
      		/>
    	</View>
    );
    
你会发现，图片竟然还是没有显示，这是为什么呢？告诉你，你还需要重新编译项目哦，简单的依靠RCT提供的动态刷新是无法加载你刚新增的本地资源的～

ok，按照我的描述，你应该就不会遇到下面这个头疼的报错了：

![](http://i.stack.imgur.com/SvCdg.png)