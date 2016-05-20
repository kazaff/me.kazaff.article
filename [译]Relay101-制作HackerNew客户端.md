原文地址：[https://medium.com/@clayallsopp/relay-101-building-a-hacker-news-client-bb8b2bdc76e6](https://medium.com/@clayallsopp/relay-101-building-a-hacker-news-client-bb8b2bdc76e6)

[React](https://facebook.github.io/react/)让我们可以使用javascrip创建用户界面组件；[Relay](https://facebook.github.io/relay/)则可以让我们很容易的打通react组件和远程服务器的数据通信。为了实现这个目标Relay需要一个条件--它假设客户端和服务器端必须满足某种要求，这可能增加了使用它的门槛，但对于一些项目来说这是非常值得的！

没有Relay的日子，你需要自己实现数据的下载，传输，缓存。像Flux和Redux这类工具帮你避免了在这过程中的一些bug，但当大量数据来往于应用与服务器之间时还是有非常多的人为性错误存在的可能。Relay会减少非常多的通用样板代码，让客户端开发人员可以更简洁安全的获取他们想要的数据。

按照之前Rails的十五分钟搭建blog的传统，我们这里将快速使用Relay搭建一个HAcker News客户端。教程假设你了解Node，NPM和react，没有其他要求了～

###Getting GraphQL

目前Relay需要你的服务器提供[GraphQL](https://facebook.github.io/graphql/)接口。GraphQL非常优雅，但除非你是facebook的工程师，否则你可能并没有这么一个服务接口。

为了避免我们自己搭建GRaphQL服务接口，我们这里将会使用[GraphQLHub](http://www.graphqlhub.com/)。GraphQLHub是一个发展迅速的GraphQL现有API翻译的仓库，和HackerNews和Reddit很像，我会一直维护它:)

本篇指南会带领你快速了解GraphQL基础语法，所以你并不需要提前学习它的内容。如果你对GraphQL很感兴趣，你可以阅读[Your First GraphQL Server](https://medium.com/@clayallsopp/your-first-graphql-server-3c766ab4f0a2)（译：我已经翻译过了，可以在本博客查找）。

###Setting Up The Project

在2015年，有大量的工具帮助你创建浏览器端javascript应用。我们选择使用[Webpack](https://webpack.github.io/)来打包我们的代码供浏览器解析，并使用[Babel](https://babeljs.io/)来编译我们的React和Relay代码。这些工具都是Relay小组推荐使用的，但你也可以选择使用其它的。

我们不会关注太多Webpack和Babel的内容，毕竟我们的关注点在Relay。

让我们创建一个新的项目：

	$ mkdir relay-101 && cd ./relay-101
	$ npm init
	# you can hit Enter a bunch of times
	
这将会在你的文件夹中创建package.json文件，然后我们安装一些包：

	$ npm install webpack@1.12.2 webpack-dev-server@1.11.0 babel-core@5.8.25 babel-loader@5.3.2 --save

Webpack会查找"webpack.config.js"文件，所以我们需要在当前目录下创建它：

	$ touch webpack.config.js
	
并把下面的内容复制粘贴进去：

	var path = require('path');

	module.exports = {
  		entry: path.resolve(__dirname, 'index.js'),
  		module: {
    		loaders: [
      			{
        			test: /\.js$/,
        			loader: 'babel',
        			query: {stage: 0}
      			}
    		]
  		},
  		output: {filename: 'index.bundle.js', path: './'}
	};
	
你可能注意到了我们这里提到了一个index.js文件。目前我们创建一个简单的实现：

	$ echo 'alert("hello relay!");' > index.js
	
所有客户端的代码都会放在这个文件里，所以建议你在你的编辑器中保持打开它。

我们还需要在package.json文件中加入“start”入口设定：

	{
  		// ...
  		"scripts": {
    		"start": "./node_modules/.bin/webpack-dev-server",
    		"test": "echo \"Error: no test specified\" && exit 1"
  		},
	}
	
这么做是为了允许我们直接使用“npm start”来运行我们之前安装的webpack development server的。我们可以在我们的终端中瞄一眼：

	$ npm start
	> relay-101-test@1.0.0 start ~/relay-101
	> webpack-dev-server

	http://localhost:8080/webpack-dev-server/
	webpack result is served from /
	
浏览器中打开[http://localhost:8080/webpack-dev-server](http://localhost:8080/webpack-dev-server)，你会看到文件列表！这很好，但我们需要的是我们客户端的html页面。回到我们的项目文件夹，创建一个index.html并输入下面的内容：

	$ touch index.html

	# paste this inside of index.html
	<html>
	<head></head>
	<body>
  		<div id='container'>
  		</div>
  		<script src="/index.bundle.js" charset="utf-8"></script>
	</body>
	</html>
	
刷新我们的开发服务器，将会看到一个期望的弹窗：

![](https://cdn-images-1.medium.com/max/800/1*z-akwOi3BMk-NBVGT7pmag.png)

现在我们就可以开始做一些好玩的事儿了。

###Building A Static Component

我们的小app将会模仿[Hacker News](https://news.ycombinator.com/)界面。开始之前，我们需要安装React和React-DOM包：

	$ npm install react@0.14.0-rc1 react-dom@0.14.0-rc1 --save
	
注意我们这里指定了明确的版本号（如果将来你发现接口变更，请通知我）。回到index.js，删除我们原先的alert代码，开始实现我们的Item组件：

	// inside index.js

	let React    = require('react');
	let ReactDOM = require('react-dom');

	class Item extends React.Component {
  		render() {
    		let item = this.props.store.item;

    		return (
      			<div>
        			<h1><a href={item.url}>{item.title}</a></h1>
        			<h2>{item.score} - {item.by.id}</h2>
        			<hr />
      			</div>
    		);
  		}
	};

注意，我们所有的数据都来自于“store”属性--稍后我们会解释～

让我们先在屏幕上伪造一些item：

	// at the bottom of index.js

	let mountNode = document.getElementById('container');
	let item = {
  		id  : '1337',
  		url : 'http://google.com',
  		title : 'Google',
  		score : 100,
  		by : { id : 'clay '}
	};
	let store = { item };
	let rootComponent = <Item store={store} />;
	ReactDOM.render(rootComponent, mountNode);
	
刷新我们的开发服务器，你将会看到：

![](https://cdn-images-1.medium.com/max/800/1*S4ZZYS8uOoXxm6LgJhKt8w.png)

###Data From The Server

是时候来加点Relay了。我们将会从GraphQLHub上根据ID获取一些itme数据来代替我们的静态数据。我们先来安装一些Relay包：

	$ npm install react-relay@0.3.2 babel-relay-plugin@0.2.5 sync-request@2.0.1 graphql@0.4.4 --save
	
为什么我们安装了这么多东西而不仅仅是react-relay？好吧，目前Relay要求我们做的确实有点多--特别是，我们需要“babel-relay-plugin”来结合Babel。这个plugin会去GraphQLHub获取更多的Relay的配置。

为了连接plugin，我们需要修改一下webpack.config.js中“query”选项：

	module.exports = {
  		// ...
  		module: {
    		loaders: [
      			{
        			// ...,
        			// note that this is different!
        			query: {stage: 0, plugins: ['./babelRelayPlugin']}
      			}
    		]
  		}
  		// ...
	};
	
这么做将会告诉Babel去加载一个叫babelRelayPlugin.js的文件。创建这个文件并放入下面的内容：

	$ touch babelRelayPlugin.js

	// inside that file
	var babelRelayPlugin   = require('babel-relay-plugin');
	var introspectionQuery = require('graphql/utilities').introspectionQuery;
	var request            = require('sync-request');

	var graphqlHubUrl = 'http://www.GraphQLHub.com/graphql';
	var response = request('GET', graphqlHubUrl, {
  		qs: {
    		query: introspectionQuery
  		}
	});

	var schema = JSON.parse(response.body.toString('utf-8'));

	module.exports = babelRelayPlugin(schema.data, {
  		abortOnError: true,
	});

Cool--现在杀掉你的“npm start”进程并重启它。现在每次你重新打包你的app，它都会去GraphQLHub服务器（使用GraphQL优雅的自解释API）询问和预处理我们的Relay代码。

回到index.js，是时候导入Relay啦：

	let React    = require('react');
	let ReactDOM = require('react-dom');
	let Relay    = require('react-relay');
	
接下来该做甚？我们将把我们的Item组件用[higher-order](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750#fd19)包装一下。这个新的组件将被Relay管理，这就是黑科技的地方：

	class Item extends React.Component {
  		...
	}
	Item = Relay.createContainer(Item, {
  		fragments: {
    		store: () => Relay.QL`
      			fragment on HackerNewsAPI {
        			item(id: 8863) {
          				title,
          				score,
          				url
          				by {
            				id
          				}
        			}
      			}
    		`,
  		},
	});
	
哒啦，搞定。翻译成大白话，上面的写法解释为：

1. 嘿 Relay，我将把我的item组件作为原件放到新的组件容器中。
 
2. 对于组件的“store”属性，我需要数据已经用GraphQL片段描述了。
 
3. 我知道这些数据在http://graphqlhub.com/playground/hn的“HackerNewsAPI”对象中。

注意这里我们只是描述了一个GraphQL片段（片段的概念很类似于查询中的别名），并不是最终获取所有数据的完整query。这是Relay的一个优点--组件只需要声明它需要的数据，而不用管如何获取数据。

有些时候我们确实需要一个完整的GraphQL query，这就是Relay Routes的主场了。Relay.Route跟浏览器里的history或URLs半毛钱关系都木有--它是用来创建引导我们数据获取请求的“root query”的。

嗖，让我们来搞一个Relay Route。在我们的Itme定义下面加入：

	Item = // ...;

	class HackerNewsRoute extends Relay.Route {
  		static routeName = 'HackerNewsRoute';
  		static queries = {
    		store: ((Component) => {
      			// 这里Component就是我们的Item组件
      			return Relay.QL`
      				query root {
        				hn { ${Component.getFragment('store')} },
      				}
    			`}),
  		};
	}
	
现在我们的GraphQL补全了root query。Relay允许我们使用ES6的字符串解析特性注入我们的片段，这就完成了组件分享（不同于复制）它的数据需求给父组件的过程。

是时候展示点数据到屏幕上了！修改我们之前的代码如下：

	class HackerNewsRoute {
 		// ...
	}

	Relay.injectNetworkLayer(
  		new Relay.DefaultNetworkLayer('http://www.GraphQLHub.com/graphql')
	);

	let mountNode = document.getElementById('container');
	let rootComponent = <Relay.RootContainer
  		Component={Item}
  		route={new HackerNewsRoute()} />;
	ReactDOM.render(rootComponent, mountNode);

这里Relay.RootContainer是顶级组件，它构建了一个组件层级的query。我们还做了一些网络配置，最终渲染新的组件到DOM。你会在浏览器中看到下面的景象：

![](https://cdn-images-1.medium.com/max/800/1*GOT3SozMFQCo3RcSiTMmcA.png)

###A List Of Components

我们下面开始做一个类似Hacker News首页的页面。相比硬编码一个特定的item，我们需要展示一个item列表。用Relay的话说，这需要我们创建一个新的list组件，其中嵌套多个独立的item组件（每个都请求自己的数据）。

回到代码，我们开始创建我们的TopItems组件：

	class TopItems extends React.Component {
  		render() {
    		let items = this.props.store.topStories.map(
      			(store, idx) => <Item store={store} key={idx} />
    		);
    		return <div>
      			{ items }
    		</div>;
  		}
	}
	
我们就不再想刚才那样先“创建伪造数据”了，而是直接动真格的，用Relay封装TopItems：

	TopItems = Relay.createContainer(TopItems, {
  		fragments: {
    		store: () => Relay.QL`
      			fragment on HackerNewsAPI {
        			topStories { ${Item.getFragment('store')} },
      			}
    		`,
  		},
	});
	
相比之前的单独item，现在我们需要请求“topStories”。针对每个item，GraphQL会根据声明的片段请求对应的数据，所以我们将只请求我们需要的数据。

但是稍等--目前我们的item片段定义了一个特定的id（[#8863](https://news.ycombinator.com/item?id=8863)）。我们需要修改我们的query：

	Item = Relay.createContainer(Item, {
  		fragments: {
    		store: () => Relay.QL`
      			fragment on HackerNewsItem {
        			id
        			title,
        			score,
        			url
        			by {
          				id
        			}
      			}
    		`,
  		},
	});
	
由于我们不再请求一个特定的item片段，所以我们需要修改在render函数中存取prop的方式：

	class Item extends React.Component {
  		render() {
    		let item = this.props.store;
    		// ...
  		}
	}
	
最后，我们需要更新Relay RootContainer来使用我们的TopItems组件：

	let rootComponent = <Relay.RootContainer
  		Component={TopItems}
  		route={new HackerNewsRoute()} />;
  		
瞧：

![](https://cdn-images-1.medium.com/max/800/1*5r4OXLb20RzSJppF6zZVqg.png)


###Variables in Queries

现在我们对创建一个Relay app有了最基础的了解，但我希望展现Relay另一个特性：variables。

在多数挨派派里，查询并不是一成不变的，我们经常需要请求不同的数据。Relay允许我们在GraphQL query中注入变量来达到这个目的。在我们这个小挨派派里，我们将添加一个开关来切换我们需要的数据类型（按照热度或时间等排序）。

开始之前，我们需要修改我们的TopItems的query：

	TopItems = Relay.createContainer(TopItems, {
  		initialVariables: {
    		storyType: "top"
  		},
  		fragments: {
    		store: () => Relay.QL`
      			fragment on HackerNewsAPI {
        			stories(storyType: $storyType) { 
        				${Item.getFragment('store')} 
        			},
      			}
    		`,
  		},
	});
	
“$storyType”代表一个GraphQL变量（这并不是ES6的字符串解析语法）。我们通过initialVariables配置给它设置了一个初始默认值“top”。

这只是Relay层面我们需要做的简单修改。我们并不需要对具体的组件做任何渲染或数据获取的修改--完全解耦了。

现在我们需要编辑我们的TopItems组件来使用开关类型。更新render函数如下：

	class TopItems extends React.Component {
  		render() {
    		let items = this.props.store.stories.map(
      			(store, idx) => <Item store={store} key={idx} />
    		);
    		let variables = this.props.relay.variables;

    		// To reduce the perceived lag
    		// There are less crude ways of doing this, but this works for now
    		let currentStoryType = (this.state && this.state.storyType) || variables.storyType;

    		return <div>
      			<select onChange={this._onChange.bind(this)} value={currentStoryType}>
        			<option value="top">Top</option>
        			<option value="new">New</option>
        			<option value="ask">Ask HN</option>
        			<option value="show">Show HN</option>
      			</select>
      			{ items }
    		</div>;
  		}

  	// to be continued

这里有些新的知识点！我们使用了“Relay”prop，它包含一些特定的属性。任何使用Relay封装的组件都会被注入这个prop--如果我们想对我们的TopItems组件进行单元测试，我们需要自己注入一个伪造的对象。

除此Relay变量之外，其它都是普通的React--我们创建一个下拉菜单，并给它一个默认值，并监听它的修改。当改变下拉菜单的选项，我们会告诉Relay去使用新的变量值，如下：

	class TopItems extends React.Component {
  		render() {
    		// ...
  		}

  		_onChange(ev) {
    		let storyType = ev.target.value;
    		this.setState({ storyType });
    		this.props.relay.setVariables({
      			storyType
    		});
  		}
	}
	
一切都很简答--Relay会察觉query的一部分发生了变化，并且根据需要重新去获取数据。为了简单，我们设置了组件的内部状态，

刷新你的浏览器并且切换不同的下拉选项。你会发现，Relay并不会重复获取已经加载了的数据类型。

![](https://cdn-images-1.medium.com/max/800/1*eTObnmhvdB4lFI7CPU3GQQ.png)

###Relay 102

SO，以上就是关于Relay的简单介绍。我们还没有涉及到mutations（用来完成更新服务端数据）和在获取数据时显示加载提醒。Relay非常的灵活，但代价是需要比我们这里提到的更多
的配置。

Relay可能并不是适合所有的挨派派和团队，但它在解决某些常见问题上确实令人非常激动。

这篇文章中的源码放在[Github](https://github.com/clayallsopp/relay-101)，关注我[@clayallsopp](http://twitter.com/clayallsopp)和[@GraphQLHub](http://twitter.com/GraphQLHub)来获取更多的信息。


###译者注

看了不少相关资料，个人感觉，目前Relay+GraphQL相比react确实还有待社区的考量，并且它们确实还太新，官方也强调可能会出现比较大的变动。所以，目前用在实际项目里的风险确实较大，还是继续观望吧～纯属个人看法！