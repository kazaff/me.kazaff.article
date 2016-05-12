这次来细聊一下vue的组件间通信，这部分内容官方有花了不少的[篇幅](http://vuejs.org.cn/guide/components.html#父子组件通信)介绍，但却没有一个清晰的例子供参考，真是不知道这算不算偷懒。不过还好热心人不少，这里有一个[配套的例子](http://www.imys.net/20160503/vue-data-interaction.html)很屌。我个人推荐 [**vuex**](http://vuex.vuejs.org/zh-cn/intro.html)来管理通信，不过这也意味着你要引入更多的依赖。

从资料中可以看到，目前只用vue来实现兄弟组件间的通信，还是很麻烦的。不过，一定要搞清楚的是，兄弟间通信的需求场景是否合理，确保你确实需要兄弟组件间通信的前提下，我们继续往下聊。

这里要提到一个理论概念，之前在翻译react相关的文章中看到过：**smart component** 和 **dummy component**。简单的解释就是说，你的组件到底是否需要 **保存** 外部的状态，注意这里是指的保存，而不是感知。如果一个组件只是需要感知一个外部状态的话，起始使用`props`来做是非常合适的。

不过，如果你的组件确实需要独自保存一份来自外部的数据状态，那它就不再是一个dummy component，至少它的可复用性要大打折扣。所以除非你有足够的理由来这么做，不然还是尽可能保持组件的“愚蠢”吧！

关于组件间通信的内容，上面给的第二个链接已经讲的很透彻了，我不准备在重复了。本篇我们主要来讨论一下新手容易碰到的灵异事件（我就是新手）。

### 灵异事件之一：子组件无法捕获事件

我在项目中并没有引入vuex，所以我使用的是原始的父组件代理事件的方式来实现兄弟组件间通信的功能。不过在测试时发现子组件死活无法接受到父组件转发的事件，真是日了狗了！

通过debug，我发现父组件在`$broadcast()`内部，并没有检查到目标事件监听的子组件，我才恍然大悟，确实页面上连该子组件的dom都不曾出现。

经过排查，原来是因为：**vue不能很好的识别单独html元素标签**，如下：

```html
<div id="toolbar" class="container">
	<a-bar @event-route="eventRoute" />
	<b-detail @event-route="eventRoute" />
</div>
```

改成：

```html
<div id="toolbar" class="container">
	<a-bar @event-route="eventRoute"></a-bar>
	<b-detail @event-route="eventRoute"></b-bar>
</div>
```

一切就正常了，真是见鬼了，谁可以来解释一下为何如此奇葩？！

### 灵异事件之二：vue-strap的按钮组控件

其实严格意义上这个并非vue组件间通信的问题，但却让我足足摆弄了3个小时。直到现在，我也只能归结于是自己的乱用导致的这个问题。

问题描述：

当首次切换选中按钮时，会触发两次`change`或`click`事件，但再次切换就一切正常了！

前提是，你像我一样任性的如下使用：

```html
<radio-group :value.sync="radioValue" type="primary" :change="log()">
  <radio value="left">Left</radio>
  <radio value="middle" checked>Middle</radio>
  <radio value="right">Right</radio>
</radio-group>
```

vue-strap对开发者来说可以理解为一个黑盒子，而官方例子又非常的不具体，导致我们很容易用错。这里我可能就犯了一个思维错误，至少vue-strap的哲学不推荐我这么干（我猜的）。

到底哪里有问题呢？我想当然的根据使用jquery的思维来使用radio-group组件，导致我使用`onchange`事件来处理变更事件。

但注意到`:value.sync`的绑定方式，我们这里应该使用双向绑定的思维来实现这个需求：

```javascript
ready: function(){
	this.$watch('radioValue', function(newVal, oldVal){
		this.$dispatch("event-route", {eventName:"change", data: newVal})});
	});
}
```

这里我们在对应组件的生命周期方法中绑定了`$watch`回调，这样每当`radioValue`变更时就会捕捉到，我们并没有直接在`radio-group`组件上来做这件事儿，一切灵异事件都不见了（包括初始化问题）。之所以会想到这么做，是因为在排查bug时，我们不应该对黑盒子做任何假设，一切逻辑都尽可能由我们自己来完成，以最大程度的排除干扰。

感悟：只用使用匹配的思维来使用一个组件或库，才能真正的提升效率，否则可能事倍功半。
