> 越是老的经久不衰的系统，越是会存在非常多的惊喜。

这话放在wordpress（以下简称wp）上更是贴切！三五年前在我告别了百度空间后，其实就已经开始使用wp当博客系统了。只可惜那时候的我能力还有限（其实就是懒），从没有好好研读过wp的架构和代码。

这段时间需要将公司的形象网站重构，经领导的推荐，最终决定要基于wp来做！那么总算有了解她的理由了！

简单的gg了一下，海量的教程，皮肤，插件。可见其生态环境还是依然不错，尽管最近几年不少轻量级博客系统的诞生，但似乎并没有对wp造成太大的影响，尤其是在针对一些比较复杂需求的企业站开发时，wp的强大就体现出来了。

作为新手，这篇文章会记录下我的学习过程，作为我个人的记忆存根，如果对你有帮助，那真是让我非常荣幸~

### wordpress的要点

随着我对wp的了解，目前感觉到它的核心要点有以下几点：

- [hook](http://www.wpdaxue.com/wordpress-hook.html)
- [模版层级](https://developer.wordpress.org/themes/basics/template-hierarchy/)
- [自定义查询](http://www.wpdaxue.com/create-custom-queries-in-wordpress.html)
- [Widgets](http://www.wpdaxue.com/wordpress-widgets-api.html)

### wordpress的小惊喜

- [shortcode](http://www.wpdaxue.com/wordpress-shortcode.html)
- [子主题](https://codex.wordpress.org/zh-cn:%E5%AD%90%E4%B8%BB%E9%A2%98)



### 子主题下覆盖原主题的shortcode

这个必须在子主题的`functions.php`中如下定义：

```php
add_action( 'after_setup_theme', 'my_ag_child_theme_setup' );

function my_ag_child_theme_setup() {
   remove_shortcode( 'divider' );
   add_shortcode( 'divider', 'my_ag_divider' );
}

function my_ag_divider( $atts, $content = null ) {
    extract(shortcode_atts(array(
    ), $atts));
	$out = '<div class="divider"><h2><span>'.do_shortcode($content).'</span></h2></div>';
    return $out;
}
```
上述方案参考[这里](https://wordpress.org/support/topic/override-themes-shortcodesphp#post-2877657)。


### 子主题下覆盖wordpress默认widget

看[这里](https://gist.github.com/paulruescher/2998060)。


### 彻底关闭post下面的comment

不管网上你搜到什么样的教程教你实现这个需求，最简单的还是安装“disable comments”插件。


### 创建page template

只需要在你的主题文件夹下建立模版文件，然后在模版文件顶部添加下面的注释即可：

```php
/*
Template Name: My Custom Template
*/
```


### 设置自定义url后导致403

首先你要保证你开启了apache的rewrite模块，然后在你配置的虚拟主机环境下设置下面两项：

```
Options Indexes FollowSymLinks
AllowOverride All
```

nginx下就很简单了，按照[这里](http://www.ccvita.com/336.html)教的做就可以了。


### Contact Form 7 下的邮件发送问题

目前我用的这套theme，需要激活Contact Form 7 插件，这个插件就是用来生成像“联系我们”的表单的，不同于wp的comments在于，可以使用自定义的语法来自定义你需要的表单项，并通过发送mail到指定邮箱来通知管理员的。

自定义表单项我没咋搞，这里主要说一下邮件发送的问题，花了我一天的时间，痛苦死了。默认该插件会使用wp自带的`wp_mail()`来完成邮件发送，但奇怪的时，wp后台并没有给出smtp配置的地方。从一些文档上看貌似是需要你的主机支持邮件功能的（具体我也没深究，总之就是我本地环境下不行）。按照网上的推荐，我安装了另一个插件：**Postman SMTP**。这样意味着可以将CF7的邮件发送托管给postman来做，不过奇怪的是我本地环境下cf7就是无法使用postman，看了不少文档也没有头绪。奇迹发生在我将项目部署在阿里云服务器以后，一切都正常了！阿西吧~


### 正在执行例行维护 请一分钟后回来

遇过你碰见这个问题，请参考[这里](http://blog.wpjam.com/m/maintenance-sucks-problem/)。


### 手把手

不是我夸，老外就是屌，早有牛人出了系列教程，教你如何不写一行代码完成wp建站：[教程](http://tyler.com/)


### 最后推荐的神器

朋友强烈推荐给我的一个基于wp的牛逼框架：[themosis](http://framework.themosis.com/)，据说用上它以后，你将改变宇宙的格局。
