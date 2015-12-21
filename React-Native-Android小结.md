最近尝试使用react-native android为我们的一个创业项目写了一个demo，项目放在[github](https://github.com/kazaff/ZhuiYuanDemo)了，有兴趣的朋友可以看看，下面给出一些效果：

<embed src="http://player.youku.com/player.php/sid/XMTQxOTg2Nzk2MA==/v.swf" allowFullScreen="true" quality="high" width="480" height="400" align="middle" allowScriptAccess="always" type="application/x-shockwave-flash"></embed>


###关于height
	
很多时候，这都是一个不折不扣的大坑，比方说当你Image加载一个网络图片时，如果你不给该Image设置width和height，那你将什么都看不到。这还不算，当你使用ListView时如果该组件同时存在一个兄弟元素，那么此时ListView必须设置height，否则你会发现它不再响应用户的滚动操作。。。

###关于Fetch

当你试图提交附件表单数据的时候，请一定要使用FormData对象将数据包裹，这应该算不上什么太古怪的事儿～

如果你只是普通的json提交，就按照官方例子来做：

	fetch(API_LOGIN_URL, {
        method: 'POST',
        headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          account: this.state.account,
          password: this.state.password,
        })
      })
    .....

不然fetch会报错说你使用了不支持的body属性值类型～

如果你和我一样需要实现类似用户头像修改的功能，你可能需要下面两个类库：

[react-native-image-picker](https://github.com/marcshilling/react-native-image-picker)：用于头像图片获取

[react-native-fileupload](https://github.com/PhilippKrone/react-native-fileupload)：用于图片文件表单上传，这是由于你使用image-picker获取到的图片只是本地url，你需要这个uploader来组装成原始的file上传表单～～

顺便说一下，第三方依赖更新较快，代码也可能有bug，各种bug，所以你最好主动的去github上和项目的维护人员沟通，一起想办法解决问题～～


###关于async/await

至少在我目前的环境（默认安装react-native0.16.0自带的babel）是没有默认开启该特性的，不过貌似是可以手动开启的，有兴趣可以看[这里](https://medium.com/the-exponent-log/react-native-meets-async-functions-3e6f81111173#.ekvqrsala)。

###关于第三方类库

个人感觉，目前很多关注度高的第三方库还是值得信赖的，不过面向android平台的库功能还不足够强大，这可能由于fb刚开源不久的缘故，但是关注度高的库的作者响应也一般都很快，这一点很值得赞许～

但是，老实讲，如果你要开发的app包含大量特殊的业务，或者效果，这个时候你就只能自己去实现FB暂时没有提供的原生控件了，这就要求你同时要熟悉android原生开发，目前这个阶段，我觉得你是无法避免这种尴尬局面的。

###响应屏幕尺寸的变化

FB提供了Dimensions组件可以用来获取屏幕的当前尺寸：

	var deviceHeight = Dimensions.get('window').height;
	var deviceWidth = Dimensions.get('window').width;
	
不过，由于设备可能旋转，所以如果你的app支持设备旋转的话，你就需要找到一种方法来实时获取当前设备尺寸，官方推荐的方法是每次render都要调用这个方法来刷新设备尺寸，**而目前我还没有解决，主要是不知道应该如何获取屏幕旋转事件～～**

###style小知识

当你打算在某个控件上套用设置好的某个样式，但需要单独为其设置一个额外特殊值时，你可以这样：

	<View style={[styles.demo, {backgroundColor: "blue"}]}></View>

我也不懂，为何用数组类型的，但就是能这么写哟～

###布局

布局和web比起来其实不算复杂，但你务必要掌握Flex布局，强烈推荐看[这里](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html#show-last-Point)。

###TextInput

这里主要说的是键盘问题，当用户点击这个控件后，系统会弹出键盘，这个键盘会占据屏幕控件的，所以，你可以将整个表单View放在ScollView中！这样用户就可以在屏幕上进行滚动了，但注意，滚动的是整个表单View，而不是滚动的TextInput内容（当你设置为支持多行输入时），最后还有就是别忘了给ScollView添加keyboardDismissMode属性，如下：
	<ScrollView keyboardDismissMode="on-drag">

该属性会在滚动触发后自动关闭键盘～～

***至于怎么滚动TextInput内部数据，目前没找到方法。***

默认android系统的TextInput会有一个丑陋的下边框，style是无法去掉它的，不过你可以使用underlineColorAndroid属性将其设置为与背景色一样，来达到隐藏效果，这个时候你在外部包一个View来定义样式即可～～




