很久很久以前，我就把ztree夸成一朵花了，现在依然对它点赞！

不过之前在ng中使用它时就碰到了一个小麻烦，直至昨天同事问我，我再次翻阅ztree的文档依然没有提供自定义ajax的header的功能，这让基于REST风格的api在使用起来很不方便。

当然，直接修改ztree的源码也十分简单，但这并不是一个追求完美的coder可以接受的！那我们来看看到底该怎么做来最优雅的解决这个问题！

其实，这就要从jquery上下点功夫（ztree依赖jquery1.4+），我们在api的手册上早已经看到这么一个[方法](http://www.jquery123.com/ajaxSend/)，所以我们只需要简单的在你需要使用ztree的页面，增加下面这段代码即可：

```javascript
$(document).ajaxSend(function(event, jqxhr, settings) {
		jqxhr.setRequestHeader("Auth","kazaff");
});
```

需要提醒的是，这样会全局修改所有基于jq的ajax请求，如果你页面上有其它依赖它的代码或第三方组件，那么可能会相互影响，不过，你可以按照官方的一个例子来做处理：

```javascript
$(document).ajaxSend(function(event, jqxhr, settings) {
  if ( settings.url == "ajax/test.html" ) {
    $( ".log" ).text( "Triggered ajaxSend handler." );
  }
});
```
好了，希望对你有帮助！
