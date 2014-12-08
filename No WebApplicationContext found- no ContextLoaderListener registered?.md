> No WebApplicationContext found: no ContextLoaderListener registered?

上面个异常信息可把我给坑苦了，整整一下午啊，外加一晚上。其实吧，问题不在于人家这个报错，你瞅瞅，多体贴，甚至都加上了人性化的提醒，相对其他异常信息来说，这已经很友好了不是么？

先来简单说一下问题场景吧：**我的目的很直白，就是想在Servlet的Filter中使用Spring的Ioc注入指定的Bean**。

怎么样，直白的都要哭了！

对于我这种新手，确实一开始是不知道Filter中是无法使用spring的依赖注入的，原因很简单，因为Filter并不受spring管理，自然无法使用Ioc了！不过，我又不能直接“new”想用的bean，因为最终我的bean会使用依赖注入相关的资源！

要说这种需求一点儿都不偏门，gg上一搜一大把：[苦逼版](http://zy116494718.iteye.com/blog/1918131)，[二逼版](http://blog.csdn.net/geloin/article/details/7441937)。

苦逼版其实也挺好的，简单明了，不过对我这种强迫症患者来说，总感觉难受，可能是由于手动调用`getBean()`的缘故吧～我也不清楚，反正难受！

着重来说说这个二逼版，`DelegatingFilterProxy`这个类就是spring专门针对我们的问题场景量身定做的一个现成的实现版本，其实TA内部会帮我们调用苦逼版中的相关代码，有兴趣的同学强烈推荐看看[源码](http://grepcode.com/file/repository.springsource.com/org.springframework/org.springframework.web/3.2.2/org/springframework/web/filter/DelegatingFilterProxy.java#DelegatingFilterProxy.doFilter%28%29)。

我就是由于白天浮躁的环境让我一不小心上了鬼子的当！我其实是一个温文尔雅的人，真真儿的。但我现在杀人的心都有！如果你点开了我上面提供的二逼版链接，那你留意一下TA的这句话：

>  (1) contextAttribute，使用委派Bean的范围，其值必须从org.springframework.context.ApplicationContext.WebApplicationContext中取得，默认值是session；

卧槽，写的真尼玛专业啊，我都信了！原因是因为我的场景恰恰是需要设置注入进来的Bean的生命周期为request的，我就理所应当的认为TA说的木牛错，我还傻不拉唧的花了一下午的时间把"singleton"、“session”、“request”等使了个遍！

片头那个错误信息折磨了我几个小时，作者你造么？你能别吓唬乱写么？

哥这性子就是倔，回到家游戏都没玩，电影也没看，美剧缓冲着也不管了，心想劳资这么正儿八经的一个需求，怎么可能Spring就不能满足呢？

我又写了个简单的测试专门来排查问题，然后又去看源码！可算让我知道原因了！就是上面的那句扯犊子的话给我坑的！

`contextAttribute`属性是在调用`WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName)`的时候当第二个参数用的，追到`WebApplicationContextUtils`类去查源码，你就知道世界满满的恶意了！

哥为了真理，网吧包宿的钱都准备好了，作者你造么？

那么这个属性到底做甚的？看[这里](http://grepcode.com/file/repository.springsource.com/org.springframework/org.springframework.web/3.2.2/org/springframework/web/context/support/WebApplicationContextUtils.java#WebApplicationContextUtils.getWebApplicationContext%28javax.servlet.ServletContext%2Cjava.lang.String%29)：

> attrName the name of the ServletContext attribute to look for

就这吧，我还能说点啥，妈蛋，哥再也不相信你(http://my.csdn.net/geloin)了。


