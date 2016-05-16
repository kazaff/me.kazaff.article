我承认，这篇文章题目起的很不将就，不过相信我，已经尽力了。

最近一直在为项目写demo，算是帮同事提前采坑吧，不过也告一段落了。最后来记录两个小细节~

### VueStrap和bootstrap的冲突

测试使用的版本是：

- vue-strap 1.0.7
- bootstrap 3.3.6

按说vue-strap只是封装了一下bootstrap，可实际使用时，竟然发现VueStrap.radioGroup和bootstrap.[min].js文件有冲突，导致前者失效。

由于我的场景是想使用bootstrap的tooltip功能，所以我的解决方案是：不要加载bootstrap.[min].js，而是直接加载其独立的`./node_modules/bootstrap/js/tooltip.js`即可快速的越过该冲突。

### 组件的ready回调中$broadcast事件给子组件无效

这个问题要比第一个更有代表意义。vue组件化后，我们就有了很多父子关系的组件，在此场景下，如果你在父组件的ready回调中试图向其子组件广播事件，你可能会发现一切什么都未发生，原因也很简单，因为此时可能子组件还未渲染，更谈不上绑定事件了！

解决方案也很简单，就是使用vue提供的`$nextTick`方法，将广播事件的逻辑放在下一个时钟触发，即可以保证子组件响应指定事件。

### 预告

好了，vuejs相关的主题就暂时告一段落，接下来我可能会写一些wordpress二次开发的内容，因为项目要用~~
