身为一个刚满三十岁的程序猿（还差2个月），我婶婶的体会到技术的无止境，这是一件多么有挑战的工作啊~

这几天花时间看了一下ReactJS，感觉确实是个不错的思想，之前在公司内部推广AngularJS，虽然基本上已经在团队内部普及了MVVM的理念，但**组件化**还是做得一塌糊涂啊~

初学ReactJS，最大的感触就是组件化的那叫一个彻底~以我目前的了解，尽管无法说出其AngularJS指令的功能差异，但至少在理念上，ReactJS更接近趋势一些~

老实讲，原先认为可能要花很久来学习ReactJS，但没想到粗略的看了两天文档，就对其有了一个比较全面的了解，相比Angular确实有更好的学习曲线~

其实原本的目的是为了`React-Native`，但现在对React本身就产生了非常大的兴趣，入门教程强烈推荐：[阮一峰的分享](http://www.ruanyifeng.com/blog/2015/03/react.html)。相信你看完以后，基本上就对React有了一个较为完整的了解，再花点时间啃一下[官方文档](http://reactjs.cn/react/docs/getting-started.html)，基本上就可以看懂简单的项目了……

为什么说**简单的项目**呢？因为很少有实际项目是仅使用基础React就完成全部的，原因是因为：

> React仅仅是UI

其实这也是我一开始的疑惑，相比Angular提供整套的路由，控制器，作用域等概念，React简直太轻了，甚至可以说是缺乏必要的功能（当然，这都是个人观点~）。

正因如此，Facebook又进一步推出了一个思想：[Flux](http://zhuanlan.zhihu.com/FrontendMagazine/19900243#show-last-Point)。其宣扬**单向数据流**，让我这个当时震撼于“双向绑定”的童鞋又一次被撼动（小人物就是这样，永远只能追赶大牛）~

观望了一下社区，发现大家又各自选择自己中意的Flux具体实现（Flux更合适被当做一种思想，而非实现），这里有一个[项目](https://github.com/voronianski/flux-comparison)，用来对比各种具体实现的差异。简直太宏伟了，感觉一辈子也看不完的资料啊，最后我选择的是：[Refluxjs](https://github.com/spoike/refluxjs)。这个看起来还是非常亲切的，相比Facebook提供的原版Flux，要“简单”很多~

现在，基本上可以看懂中大型项目了，但貌似还少了点啥，路由，没错，就是每个传统web项目非常重要的一环，不过不要怕，也已经有非常强大的扩展来帮我们完成：[React-router](http://rackt.github.io/react-router/)。

总结下来，感觉要比angular轻量很多（至少在概念上），下一个小项目中我感觉自己会尝试一下它~

PS：貌似Facebook在推一个新的React框架（应该算框架吧）：[Relay](http://segmentfault.com/a/1190000002570887)，强大的一笔~