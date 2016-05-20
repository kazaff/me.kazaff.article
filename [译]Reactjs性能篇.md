英文有限，技术一般，海涵海涵，由于不是翻译出身，所以存在大量的瞎胡乱翻译的情况，信不过我的，请看原文～～

原文地址：[https://facebook.github.io/react/docs/advanced-performance.html](https://facebook.github.io/react/docs/advanced-performance.html)

---

###性能优化

每当开发者选择将react用在真实项目中时都会先问一个问题：使用react是否会让项目速度更快，更灵活，更容易维护。此外每次状态数据发生改变时都会进行重新渲染界面的处理做法会不会造成性能瓶颈？而在react内部则是通过使用一些精妙的技巧来最小化每次造成ui更新的昂贵的dom操作从而保证性能的。


####避免直接作用于DOM
react实现了一层虚拟dom，它用来映射浏览器的原生dom树。通过这一层虚拟的dom，可以让react避免直接操作dom，因为直接操作浏览器dom的速度要远低于操作javascript对象。每当组件的属性或者状态发生改变时，react会在内存中构造一个新的虚拟dom与原先老的进行对比，用来判断是否需要更新浏览器的dom树，这样就尽可能的优化了渲染dom的性能损耗。

在此之上，react提供了组件生命周期函数，`shouldComponentUpdate`，组件在决定重新渲染（虚拟dom比对完毕生成最终的dom后）之前会调用该函数，该函数将是否重新渲染的权限交给了开发者，该函数默认直接返回`true`，表示默认直接出发dom更新：
	
	shouldComponentUpdate: function(nextProps, nextState) {
  		return true;
	}

值得注意的是，react会非常频繁的调用该函数，所以如果你打算自己实现该函数的逻辑，请尽可能保证性能。

比方说，你有一个拥有多个帖子的聊天应用，如果此时只有一个发生了变化，如果你如下实现了`shouldComponentUpdate`，react会根据情况避免重新渲染那些没有发生变化的帖子：

	shouldComponentUpdate: function(nextProps, nextState) {
  		// TODO: return whether or not current chat thread is different to former one.
  		// 根据实际情况判断当前帖子的状态是否和之前不同
	}
	
总之，react尽可能的避免了昂贵的dom操作，并且允许开发者干涉该行为。


####shouldComponentUpdate实战
这里举个包含子元素的组件例子，如下图：

![](https://facebook.github.io/react/img/docs/should-component-update.png)

图中每个圆点表示一个dom节点，当某个dom节点的`shouldComponentUpdate`返回`false`时（例如c2），react就无需为其更新dom，注意，react甚至根本不会去调用c4和c5节点的`shouldComponentUpdate`函数哦～

图中c1和c3的`shouldComponentUpdate`返回了`true`，因此react会检查检查其它们包含的直接子节点。最有趣的是c8节点，虽然调用它的`shouldComponentUpdate`方法返回的是`true`，但react检查后发现其dom结构并未发生改变，所以react最终是不会重新渲染其浏览器dom的。

上图的情况下，react最终只会重新渲染c6，原因你应该懂的。

那么我们应该如何实现`shouldComponentUpdate`函数呢？假设你有一个只包含字符串的组件，如下：

	React.createClass({
  		propTypes: {
    		value: React.PropTypes.string.isRequired
  		},

  		render: function() {
    		return <div>{this.props.value}</div>;
  		}
	});
	
我们可以简单的直接实现`shouldComponentUpdate`如下：

	shouldComponentUpdate: function(nextProps, nextState) {
  		return this.props.value !== nextProps.value;
	}

目前为止一切都很顺利，处理基础类型的属性和状态是很简单的，我们可以直接使用js语言提供的`===`比对来实现一个mix并注入到所有组件中，事实上，react自身已经提供了一个类似的：[PureRenderMixin](https://facebook.github.io/react/docs/pure-render-mixin.html)。

但是如果你的组件所拥有的属性或状态不是基础类型呢，而是复合类型呢？比方说是一个js对象，`{foo: 'bar'}`：

	React.createClass({
  		propTypes: {
    		value: React.PropTypes.object.isRequired
  		},

  		render: function() {
    		return <div>{this.props.value.foo}</div>;
  		}
	});
	
这种情况下我们刚才实现的那种`shouldComponentUpdate`就歇菜了：

	// 假设 this.props.value 是 { foo: 'bar' }
	// 假设 nextProps.value 是 { foo: 'bar' },
	// 但是nextProps和this.props对应的引用不相同
	this.props.value !== nextProps.value; // true
	
要想修复这个问题，简单粗暴的方法是我们直接比对`foo`的值，如下：

	shouldComponentUpdate: function(nextProps, nextState) {
  		return this.props.value.foo !== nextProps.value.foo;
	}
	
我们当然可以通过深比对来确定属性或状态是否确实发生了改变，但是这种深比对是非常昂贵的，还记得我们刚出说过`shouldComponentUpdate`函数的调用非常频繁么？更何况我们为每个model去单独实现一个匹配的深比对逻辑，对于开发人员来说也是非常痛苦的。最重要的是，如果我们不是很小心的处理对象引用关系的话，还会带来灾难。例如下面这个组件：

	React.createClass({
  		getInitialState: function() {
    		return { value: { foo: 'bar' } };
  		},

  		onClick: function() {
    		var value = this.state.value;
    		value.foo += 'bar'; // ANTI-PATTERN!
    		this.setState({ value: value });
  		},

  		render: function() {
    		return (
      			<div>
        			<InnerComponent value={this.state.value} />
        			<a onClick={this.onClick}>Click me</a>
      			</div>
    		);
  		}
	});
	
起初，InnerComponent组件进行渲染，它得到的value属性为`{foo: 'bar'}`。当用户点击链接后，父组件的状态将会更新为`{ value: { foo: 'barbar' } }`，触发了InnerComponent组件的重新渲染，因为它得到了一个新的属性：`{ foo: 'barbar' }`。

看上去一切都挺好的，其实问题在于，父组件和子组件供用了同一个对象的引用，当用户触发click事件时，InnerComponent的prop将会发生改变，因此它的`shouldComponentUpdate`函数将会被调用，而此时如果按照我们目前的`shouldComponentUpdate`比对逻辑的话，`this.props.value.foo`和`nextProps.value.foo`是相等的，因为事实上，它们同时引用同一个对象哦～所以，我们将会看到，InnerComponent的ui并没有更新。哎～，不信的话，我贴出完整代码：

	<!DOCTYPE html>
	<html>
		<head>
			<meta charset="utf-8">
			<title>demo</title>
			<!--引入React库-->
			<script src="lib/react.min.js"></script>
			<!--引入JSX转换库-->
			<script src="lib/JSXTransformer.js"></script>
			<!--组件样式-->
		</head>
		<body>
			<!--定义容器-->
			<div id="content"></div>

			<!--声明脚本类型为JSX-->
			<script type="text/jsx">

				var InnerComponent = React.createClass({
        			shouldComponentUpdate: function(nextProps, nextState) {
  						return this.props.value.foo !== nextProps.value.foo;
					},
  					render: function() {
    					return (
      						<div>
        						{this.props.value.foo}
      						</div>
    					);
  					}
				});

				var OutComponent = React.createClass({
  					getInitialState: function() {
    					return { value: { foo: 'bar' } };
  					},

  					onClick: function() {
    					var value = this.state.value;
    					value.foo += 'bar'; // ANTI-PATTERN!
    					this.setState({ value: value });
  					},

  					render: function() {
    					return (
      						<div>
        						<InnerComponent value={this.state.value} />
        						<a onClick={this.onClick}>Click me</a>
      						</div>
    					);
  					}
				});

				React.render(<OutComponent />, document.querySelector("#content"));
		
			</script>
		</body>
	</html>



####Immutable-js的救赎
[Immutable-js](https://github.com/facebook/immutable-js)是一个javascript集合库，作者是Lee Byron，该项目最近从fb开源。它提供了不可变集合类型：

+ 不可变性：一旦创建，这个集合不允许再更改。
+ 延续性：新集合可以衍生自一个已经创建过的集合，并作一些改动，此时源集合不会受到任何影响。
+ 结构共享：如果新集合衍生自一个老集合，那么新集合中与老集合相同的部份将会共享同一块内存，这样做的好处是节省内存开销，并能在创建新集合对象时减少内存拷贝的性能损耗。

不可变性使得监控状态变化变得可行，每次状态发生变化，将总是返回一个新的对象，我们只需要检查改变前后的对象引用是否相同即可。举个例子：

	var x = { foo: "bar" };
	var y = x;
	y.foo = "baz";
	x === y; // true
	
尽管变量y被修改，但由于y和x的引用相同，最后的比对仍然返回true。如果上面的代码用immutable-js来实现：

	var SomeRecord = Immutable.Record({ foo: null });
	var x = new SomeRecord({ foo: 'bar'  });
	var y = x.set('foo', 'baz');
	x === y; // false
	
看到了么，是不是很爽？

另外一种监控变量的方法是设置一个标识位，但这需要开发者编写额外的代码，哎，反正活着就是麻烦。

总之，不可变集合结构提供了你一个廉价且简单的监控对象改变的方法，你可以放在`shouldComponentUpdate`中。因此，如果你的model属性和状态是基于immutable-js来实现的，那么你就可以直接使用官方提供的`PureRenderMixin`哟～

####Immutable-js和Flux
如果你恰巧使用[Flux](https://facebook.github.io/flux/)，并且你又基于immutable-js来实现你的store，那你先看一下相关的[api](https://facebook.github.io/immutable-js/docs/#/)吧。

让我们来用个模拟应用演示一下使用一种可行的方案。首先，我们需要定义一下用到的model：

	var User = Immutable.Record({
  		id: undefined,
  		name: undefined,
  		email: undefined
	});

	var Message = Immutable.Record({
  		timestamp: new Date(),
  		sender: undefined,
  		text: ''
	});
	
每个Record接受一个对象，分别定义了字段和默认值。我们的messages store需要使用的是list结构：

	this.users = Immutable.List();
	this.messages = Immutable.List();
	
很简单吧，接着，每当store接受到一个新的message时，我们只需要创建一个新的record并把它加入到列表中即可：

	this.messages = this.messages.push(new Message({
  		timestamp: payload.timestamp,
  		sender: payload.sender,
  		text: payload.text
	});
	
对比我们刚刚讲过的immutable-js特性，在react的组件中我们只需要直接注入`PureRenderMixin`即可高枕无忧。

