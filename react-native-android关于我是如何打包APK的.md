app写好了，最后一步多半应该是拿出来装逼喽～哇哈哈哈哈

但你总不能指望每次都USB连上你的开发机，然后run-android一下吧，虽然我承认看着命令行中刷刷刷的执行命令有种黑客帝国的范儿，但让所有人都连接的开发机，感觉也是怪怪的，更别提你还指望所有设备都在一个局域网下才能更新js文件。。。。

好吧，你还是应该考虑把你的项目打包成apk，然后传给任何你想装逼给他看的人！哇哈哈哈哈～

时至今日，2015-12-21 晚上九点四十分，官方版本应该是0.16.0，我按照[官方教程](http://react-native.cn/docs/signed-apk-android.html#content)进行打包，还是没能一次性就成功！但这并不能阻挡我试图装逼的心！

好吧，解决问题之旅开始了！

官方文档中，在打包之初是让你先生成了一个签名文件，原因是，如果你打包未签名的APK，在非root的设备上是不允许安装的，所以，你懂的！官方已经给出了非常具体的签名步骤，我这里就不重复了！

好的，签名也弄好，执行`gradlew assembleRelease`命令坐等完成吧～

但是，怎么可能让你如此轻易就达到目的？毫不意外的，我碰到了报错：

```
* What went wrong:
Execution failed for task ':app:packageRelease'.
> Unable to compute hash of /Users/kazaff/Documents/React-Native/ZhuiYuan/android/app/build/intermediates/classes-proguard/release/classes.jar
```

不过GG了一下，看来碰见这个错误的人不少，按照[stackoverflow](http://stackoverflow.com/questions/31643339/errorexecution-failed-for-task-apppackagerelease-unable-to-compute-hash)给出的终结方案：***在proguard-rules.pro文件末尾增加：**

```
-dontwarn java.nio.file.Files
-dontwarn java.nio.file.Path
-dontwarn java.nio.file.OpenOption
-dontwarn org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement

-keep class com.google.android.gms.** { *; }
-dontwarn com.google.android.gms.**
-dontwarn butterknife.**
```

还是被我机智的搞定了！

注意，再次执行`gradlew assembleRelease`之前，请先执行`gradlew clean`，清除之前打包的一些临时文件，不然你可能还是会悲剧～

好了，其实说了这么多，我只是想贱贱的贴一个下载连接：

![](http://pic.yupoo.com/kazaff/Fc35Cqee/swrie.jpg)

[http://pan.baidu.com/s/1pKhY6wj](http://pan.baidu.com/s/1pKhY6wj)

上图是[ZhuiYuanDemo](https://github.com/kazaff/ZhuiYuanDemo)APK的文件二维码，供大家把玩～

