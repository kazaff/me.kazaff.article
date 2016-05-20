昨天看了一个技术分享，讲[react生态圈](http://www.infoq.com/cn/presentations/explore-react-ecosystem)的，很不错，尽管纯技术干货没太多，但贵在拓宽知识面，讲师也很有激情，推荐看之～

在react生态圈里，越来越多的富有创意和激情的东西呈现在我们眼前，对于前端工程师来说真的是求之不得的时代啊～

react之前也玩过了，和它相关的一些类库（例如router，redux）我们也都陆续介绍过，甚至包含react-natvie也有所涉及，那么今天要解惑的，就是[GraphQL](https://facebook.github.io/graphql/)。

官方文档冗长且晦涩，我们不妨延续以往的学习经验，先gg几篇不错的博文来共大家品尝，那么和GraphQL相关的姿势哪里找呢？请看[这里](https://github.com/chentsulin/awesome-graphql)，这里包含了各种语言的实现版本，当然也包含了入门文章的推荐，有兴趣的童鞋不妨长期关注该项目～

由于我也是第一天开始了解GraphQL，并不比大家知道的多多少，所以打算翻译两篇感觉很适合入门的实战文章来让大家过过瘾，这两篇文章虽然不是出自于同一人所写，但内容上关联性却很大，个人感觉合并在一起刚刚好，所以在此斗胆合并在一起翻译，原文地址：

[Your First GraphQL Server](https://medium.com/@clayallsopp/your-first-graphql-server-3c766ab4f0a2#.pkab58j87)

[Writing a Basic API with GraphQL](http://davidandsuzi.com/writing-a-basic-api-with-graphql/)

如果你按照我提供的轨迹，看过前面提到的那个关于“react生态圈”的技术分享，那你应该就已经知道GraphQL是为了解决什么问题而产生的，如果你没看，也没事儿，再开始翻译之前我还是会简单的阐述一下GraphQL的背景。

###GraphQL对比SQL

首先，官方说了，GraphQL是一个查询语言，而且目前还未完成，意味着未来可能会有更多更大的变动。

单这并不妨碍世界上那么多前端工程师的折腾之路！查询语言？那就是说和sql类似喽？你看名字里都有“QL”啊～

确实如此，确实是用来声明要查询的数据的，但要解决的问题却完全不一样，伴随我们多年的sql主要解决的是如何在数据库基础上提供简单易用且功能强大的沟通语言，使得我们人类可以轻而易举的从海量数据中获取到我们想要知道的数据片段。

GraphQL产生的背景却完全不同，在facebook内部，大量不同的app和系统共同使用着许许多多的服务api，随着业务的变化和发展，不同app对相同资源的不同使用方法最终导致需要维护的服务api数量爆炸式的增长，非常不利于维护（我们主要在restful场景上思考）。而另一方面，创建一个大而全的通用性接口又非常不利于移动端使用（流量损耗），而且后端数据的无意义聚合也对整个系统带来了很大的资源浪费。

在这样的背景下，fb工程师坐不住了，于是乎GraphQL的概念就诞生了～最了解客户端需要什么数据的只有客户端自己，所以如果给客户端提供一种机制，让其表述自己所需的数据结构，这岂不是最合理的么？

###GraphQL对比Rest

目前，最热的前后端通信方案应该是Restful，基于http的轻量级api，前端通过ajax请求服务端提供的rest api完成数据获取。

我们再往前一步，假设你的项目前端已经组件化了，一个业务肯定是需要多个组件结合来完成的，每个组件都各自管理自己的内部状态和所需数据，那么，目前的做法是，一旦前端路由匹配了对应的业务页面，那么自然会加载相关的组件实现，同时，你还需要调用rest api来获取组件所需数据。

不同的页面，组件的组合肯定也略有不同，不同的组件组合后，所需的数据自然也不会完全一致。这里你可能会说，既然以组件为单位复用，那rest api针对组件颗粒度提供一对一的服务即可，话是没错，但实际操作起来就不work了，试想前端狗被产品狗每天要求加这加那，前端狗就会不得不去求后端狗协同开发，这样最后就剩下死逼了！而且前面也说了，这样做创造出来的大量的api会变得无法维护～

那么，是时候考虑采用Rest以外的解决方案了！（尽管我认为，GraphQL并非是来取代Rest的，但为了我们的描述简单，这里就直接这么写了！）后端根据GraphQL机制提供一个具有强大功能的接口，用以满足前端数据的个性化需求，既保证了多样性，又控制了接口数量，完美～

###GraphQL到底是什么

> A GraphQL query is a string interpreted by a server that returns data in a specified format. 

这端描述如果不了解它的背景，确实很容易和sql混淆～但，现在，你应该知道GraphQL到底是个什么鬼了吧～

好的，下面我们就来开始GraphQL in Action!


Your First GraphQL Server
---

今天我们将要实现一个[GraphQL](http://facebook.github.io/graphql/)小服务，我不没有打算让你放弃一切转而拥抱GraphQL，但如果你对这玩意儿很好奇，并想知道它是如何实现的，那么就往下读～

###Setup an HTTP Server

我们需要一个服务来接受GraphQL查询，GraphQL文档虽然并没有规定其一定要基于HTTP协议，但既然目前[GraphQL参考实现](https://github.com/graphql/graphql-js)是基于JavaScript的，那么我们就基于[Express](http://expressjs.com/)来快速打造一个http服务器完成需求吧～

	$ mkdir graphql-intro && cd ./graphql-intro
	$ npm install express --save
	$ npm install babel --save
	$ touch ./server.js
	$ touch ./index.js
	
我们创建了项目文件夹（graphql-intro），并且安装了Express和[Babel](https://babeljs.io/)作为依赖。Babel并不是GraphQL所必须的，但它可以让我们使用[ES2015特性](https://babeljs.io/docs/learn-es2015/)，从而是我们可以书写更精彩的js。（译：我瞎掰的～）

最后，让我们写点代码：

	// index.js
	// by requiring `babel/register`, all of our successive `require`s will be Babel'd
	require('babel/register');
	require('./server.js');

	// server.js
	import express from 'express';

	let app  = express();
	let PORT = 3000;

	app.post('/graphql', (req, res) => {
	  res.send('Hello!');
	});
	
	let server = app.listen(PORT, function () {
	  let host = server.address().address;
	  let port = server.address().port;
	
	  console.log('GraphQL listening at http://%s:%s', host, port);
	});
	
现在运行我们的服务：

	$ node index.js
	GraphQL listening at http://0.0.0.0:3000
	
来测试一下我们的代码：

	$ curl -XPOST http://localhost:3000/graphql
	Hello!
	
我们选择使用`/graphql`作为入口，并使用HTTP POST方法请求，这些都不是影硬性要求--GraphQL并没有限制你如何与GraphQL服务端通信。

###Create a GraphQL Schema

现在咱们有了一个服务，是时候来加点GraphQL啦。具体改怎么做呢？

让我们回想一下GraphQL请求的模样（译：我们目前不太关心GraphQL文档的细节，只需要跟着作者的步骤即可）：

	query getHighScore { score }
	
目前，我们的GraphQL客户端需要请求`getHighScore`字段中的`score`字段，字段是用来告诉GraphQL服务返回哪些数据的，字段也可以拥有参数，如下：

	query getHighScores(limit: 10) { score }
	
它还能做更多的事儿，但让我们先往下看。

我们的GraphQL服务需要进行配置才能响应上面那样的请求--这种配置被成为schema。

构建一个schema其实和构建restful路由规则是很相似的。我们的schema需要描述哪些字段是服务器需要响应的，同时也需要包含这些响应对象的类型。类型信息对GraphQL来说是非常重要的，客户端可以放心的假设服务端会返回所指定的字段类型（或者一个error）。

如你所想，schema声明可以非常的复杂。但对于我们这个简单的GraphQL服务来讲，我们需要一个简单的字段：`Count`。

回到我们的终端：

	$ npm install graphql --save
	$ npm install body-parser --save
	$ touch ./schema.js
	
就是这么靠谱，对么？[graphql](https://www.npmjs.com/package/graphql)模块包含了GraphQL的技术实现，可以允许我们来组合我们的schema和处理graphql请求。而[body-parser](https://www.npmjs.com/package/body-parser)则是一个简单的Express中间件，用来让我们获取GraphQL请求体哒～

是时候来声明我们的schema：

	//schema.js
	import {
  		GraphQLObjectType,
  		GraphQLSchema,
  		GraphQLInt
	} from 'graphql/lib/type';

	let count = 0;

	let schema = new GraphQLSchema({
  		query: new GraphQLObjectType({
    		name: 'RootQueryType',
    		fields: {
    			count: {
        			type: GraphQLInt,
        			resolve: function() {
          				return count;
        			}
      			}
    		}
  		})
	});

	export default schema;
	
这里我们所做的就是创建了一个`GraphQLSchema`实例，它提供了一个配置。后面GRaphQL的其他部件会使用我们这个schema实例。通常情况下我们喜欢将schema的创建放在单独的文件中。

简单的解释，我们创建的schema的含义是：

> 我们的顶级查询对象需要返回一个`RootQueryType`对象，它包含一个类型为整型的`count`字段。

你可以猜到还有很多类似的内置基础类型（strings，lists等），当然你也可以创建自定义的特殊类型。

###Connect the Schema

目前我们意淫的schema并没有毛用，除非我们针对它进行查询。让我们把这个schema挂载到我们的http服务上：

	import express from 'express';
	import schema from './schema';
	// new dependencies
	import { graphql } from 'graphql';
	import bodyParser from 'body-parser';

	let app  = express();
	let PORT = 3000;

	// parse POST body as text
	app.use(bodyParser.text({ type: 'application/graphql' }));

	app.post('/graphql', (req, res) => {
  		// execute GraphQL!
  		graphql(schema, req.body)
  		.then((result) => {
    		res.send(JSON.stringify(result, null, 2));
  		});
	});

	let server = app.listen(PORT, function () {
  		var host = server.address().address;
  		var port = server.address().port;

  		console.log('GraphQL listening at http://%s:%s', host, port);
	});
	
现在任何POST请求`/graphql`都将会执行我们的GRaphQL schema。我们为每个请求强制设了一个“content type”（译：'application/graphql'），这并不是GraphQL规定的，但这么做是一个好的选择，特别是当我们在现有代码中加入GraphQL功能时。

执行下面的命令：

	$ node ./index.js // restart your server
	// in another shell
	$ curl -XPOST -H "Content-Type:application/graphql"  -d 'query 	RootQueryType { count }' http://localhost:3000/graphql
	{
  		"data": {
    		"count": 0
  		}
	}
	
完美！GraphQL允许我们省略掉`query RootQueryType`前缀，如下：

	$ curl -XPOST -H 'Content-Type:application/graphql'  -d 	'{ count }' http://localhost:3000/graphql
	{
  		"data": {
    		"count": 0
  		}
	}
	
现在我们已经完成了一个GraphQL例子，我们来花点时间讨论一下introspection。（译：应该翻译成“自解释”吧？）

###Introspect the server

有趣的是：你可以写一个GraphQL查询来请求GraphQL服务获取它的fields。

听起来很疯狂？看一下这个：

	$ curl -XPOST -H 'Content-Type:application/graphql'  -d '{__schema { queryType { name, fields { name, description} }}}' http://localhost:3000/graphql
	{
  		"data": {
    		"__schema": {
      			"queryType": {
        			"name": "RootQueryType",
        			"fields": [
          				{
            				"name": "count",
            				"description": null
          				}
        			]
      			}
    		}
  		}
	}
	
格式化一下我们上面发送的查询语句：

	{
  		__schema {
    		queryType {
      			name, 
      			fields {
        			name,
        			description
      			}
    		}
  		}
	}

通常，每个GraphQL根字段自动包含一个`__Schema`[字段](http://facebook.github.io/graphql/#sec-Schema-Introspection)，其包含用来查询的描述自身meta信息的字段--`queryType`。

更可爱的是，你可以定义一些很有意义的元信息，例如`description`，`isDeprecated`，和`deprecationReason`。facebook宣成他们的工具可以很好的利用这些元信息来提升开发者的体验～

为了让我们的服务更容易使用，我们这里添加了`description`字段：

	let schema = new GraphQLSchema({
  		query: new GraphQLObjectType({
    		name: 'RootQueryType',
    		fields: {
      			count: {
        			type: GraphQLInt,
        			// add the description
        			description: 'The count!',
        			resolve: function() {
          				return count;
        			}
      			}
    		}
  		})
	});
	
重启服务后看一下新的元数据展示：

	$ curl -XPOST -H 'Content-Type:application/graphql'  -d '{__schema { queryType { name, fields { name, description} }}}' http://localhost:3000/graphql
	{
  		"data": {
    		"__schema": {
      			"queryType": {
        			"name": "RootQueryType",
        			"fields": [
          				{
            				"name": "count",
            				"description": "The count!"
          				}
        			]
      			}
   			 }
  		}
	}
	

我们几乎已经完成了我们的GraphQL旅程，接下来我将展示mutations。

###Add a Mutation

如果你只对只读数据接口感兴趣，你就不用读下去了。但大多数应用，我们都需要去更改我们的数据。GraphQL把这种操作称为mutations。

Mutations仅仅是一个字段，所以语法和query字段都差不多，Mutations字段必须返回一个类型值--目的是如果你更改了数据，你必须提供更改后的值。

我们该如何为我们的schema增加mutations？和`query`非常相似，我们定义一个顶级键`mutation`：

	let schema = new GraphQLSchema({
  		query: ...
  		mutation: // todo
	)}
	
除此之外，还有啥不特别的？我们需要在`count`字段函数中更新我们的计数器或者做一些GraphQL根本不需要感知的其他变更操作。

mutation和query有一个非常重要的不同点，mutation是有执行顺序的，但是query没有这方面的保证（事实上，GraphQL推荐并行处理那些不存在依赖的查询）。GraphQL说明书给了下面的mutation例子来描述执行顺序：

	{
  		first: changeTheNumber(newNumber: 1) {
    		theNumber
  		},
  		second: changeTheNumber(newNumber: 3) {
    		theNumber
  		},
  		third: changeTheNumber(newNumber: 2) {
    		theNumber
  		}
	}

最终，请求处理完毕后，`theNumber`字段的值应该是`2`。

让我们新增一个简单的mutation来更新我们的计数器并返回新值：

	let schema = new GraphQLSchema({
  		query: ...
  		mutation: new GraphQLObjectType({
    		name: 'RootMutationType',
    		fields: {
      			updateCount: {
        			type: GraphQLInt,
        			description: 'Updates the count',
        			resolve: function() {
          				count += 1;
          				return count;
        			}
      			}
    		}
  		})
	});
	
重启我们的服务，试一下：

	$ curl -XPOST -H 'Content-Type:application/graphql' -d 'mutation 	RootMutationType { updateCount }' http://localhost:3000/graphql
	{
  		"data": {
    		"updateCount": 1
  		}
	}
	
看--数据已经更改了。你可以重新查询一下：

	$ curl -XPOST -H 'Content-Type:application/graphql' -d '{ count }' http://localhost:3000/graphql
	{
  		"data": {
    		"count": 1
  		}
	}

你可以多执行几次。

为了严格遵守GraphQ实现，我们应该提供一个更语意化的值名（例如`CountValue`），这样会让mutation和query返回值都更加有意义。

###Wrapping Up

本文章教你如何使用facebook提供的GraphQL javascript版实现来完成服务的。但我并没有涉及过多的强大话题--fields with arguments, resolving promises, fragments, directives等等。GraphQL说明书中介绍了很多特别屌的特性。其实还有很多不同的服务端GraphQL实现和schema API。你可以使用像java这样的强类型语言来实现GraphQL服务。

本篇是基于我的GraphQL的48小时体验而来--如果有什么遗漏或错误，别忘了让我知道。你可以在这里看到源码（每次提交都对应流程中的每一步）：

[https://github.com/clayallsopp/graphql-intro](https://github.com/clayallsopp/graphql-intro)

十分感谢RisingStack的关于GraphQL的[文章和例子](http://blog.risingstack.com/graphql-overview-getting-started-with-graphql-and-nodejs/)。


---

Writing a Basic API with GraphQL
---

这篇文章假设你了解GraphQL，并试图将你的后端实现转换成GraphQL，内容覆盖了同步/异步，query/mutation。

源码：[https://github.com/davidchang/graphql-pokedex-api](https://github.com/davidchang/graphql-pokedex-api)

基本的Pokedex客户端实现来自于我上周的[Redux Post](http://davidandsuzi.com/writing-a-basic-api-with-graphql/davidandsuzi.com/writing-a-basic-app-in-redux/)，让我们使用GraphQL来实现一个Pokedex后端API。我们需要两个query方法（查询口袋妖怪列表和指定用户所拥有的口袋妖怪）和两个mutation方法（创建用户和用户捕获口袋妖怪）。

口袋妖怪列表数据存储在内存中，而用户和其拥有的口袋妖怪数据则将存储再MongoDB。

###Starting

继续前面提到的[文章](https://medium.com/@clayallsopp/your-first-graphql-server-3c766ab4f0a2)（其实就是上一篇），我使用babel在node上创建了一个Express服务端，它包含了一个`/graphql`路由。我的server.js入口文件如下：

	import express from 'express';
	import schema from './schema';
	import { graphql } from 'graphql';
	import bodyParser from 'body-parser';

	let app  = express();

	// parse POST body as text
	app.use(bodyParser.text({ type: 'application/graphql' }));

	app.post('/graphql', (req, res) => {
  		// execute GraphQL!
  		graphql(schema, req.body)
    		.then(result => res.send(result));
		});

	let server = app.listen(
		3000,
  		() => console.log(`GraphQL running on port ${server.address().port}`)
	);
	
###Synchronous Query
####{ pokemon { name } }

让我们从Pokemon列表开始。我们的列表数据来自[这里](https://gist.github.com/MathewReiss/20a58ad5c1bc9a6bc23b#file-phone-js)，每个Pokemon对象都包含name，type，stage和species属性。我们需要定义一个新的GraphQLObjectType来描述这种对象。定义如下：

	import {
		GraphQLObjectType,
		GraphQLInt,
		GraphQLString,
	} from 'graphql';

	let PokemonType = new GraphQLObjectType({
  		name: 'Pokemon',
  		description: 'A Pokemon',
  		fields: () => ({
    		name: {
      			type: GraphQLString,
      			description: 'The name of the Pokemon.',
    		},
    		type: {
      			type: GraphQLString,
      			description: 'The type of the Pokemon.',
    		},
    		stage: {
      			type: GraphQLInt,
      			description: 'The level of the Pokemon.',
    		},
    		species: {
      			type: GraphQLString,
      			description: 'The species of the Pokemon.',
    		}
  		})
	});

其中name，type和species都是字符串类型（type和species也可能是枚举类型），stage是整型。

为了使用GraphQL来请求Pokemon，我们需要在GraphQLSchema中定义一个根query。一个标准的空的schema看起来是这样的：

	import { GraphQLSchema } from 'graphql';

	let schema = new GraphQLSchema({
  		query: new GraphQLObjectType({
    		name: 'RootQueryType',
    		fields: {
      			// root queries go here!
    		}
  		})
	});

	export default schema;

为了定义我们的新的根查询，我们需要在fields中添加一个键值对对象，key为查询的名字，值为定义查询如何工作的一个对象。如下所示：

	pokemon: {
  		type: new GraphQLList(PokemonType),
  		resolve: () => Pokemon // here, Pokemon is an in-memory array
	}

`type`指定了GraphQL的返回类型 - 一个PokemonType对象类型的列表。`resolve`告诉GraphQL如何获取所需的数据。这里的数据仅仅是前面提到的pokemontype类型的js内存数组。

鼓掌，我们的GraphQL API现在已经有了一个root query。我们可以发送一个携带查询内容的post请求来验证这部分代码（这么做可避免因使用GET而带来的编码问题）。（GraphQL并不关心如何获取这个查询--这完全有实现者来解决。）

	curl -XPOST -H 'Content-Type:application/graphql'  -d '{ pokemon { name } }' http://localhost:3000/graphql
	
如果我们只想获取type和species，而不要name，我们可以这么做：

	curl -XPOST -H 'Content-Type:application/graphql'  -d '{ pokemon { type, species } }' http://localhost:3000/graphql
	
###Asynchronous Query with an Argument
####{ user(name: “david”) { name, caught } }

接下来，我们需要提供一种查询，用来根据用户名来获取对应的pokemon。让我们再次从定义user类型开始：

	let UserType = new GraphQLObjectType({
  		name: 'User',
  		description: 'A User',
  		fields: () => ({
    		name: {
      			type: GraphQLString,
      			description: 'The name of the User.',
    		},
    		caught: {
      			type: new GraphQLList(GraphQLString),
      			description: 'The Pokemon that have been caught by the User.',
    		},
    		created: {
      			type: GraphQLInt,
      			description: 'The creation timestamp of the User.'
    		}
  		})
	});
	
user包含一个字符串类型的name，字符串数组类型的表示所拥有pokemon的caught和一个整型的时间戳。

回到我们的GraphQLSchema，我们将添加一个user根查询，并期望得到一个UserType类型的返回。我们需要可以指定用户名的参数-我们利用`args`来实现：

	user: {
  		type: UserType,
  		args: {
    		name: {
      			description: 'The name of the user',
      			type: new GraphQLNonNull(GraphQLString)
    		}
  		},
  		resolve: (root, {name}) => {

  		}
	}
	
我们在resolve函数中接受name参数。在这个例子中，我想使用MongoDB，所以我们的resolve函数将需要去查询MongoDB数据库并获取对应的用户对象，不过GraphQL（包括读者你）都不需要关心这个实现细节-唯一需要关心的是resolve函数将返回一个promise。（promise是一个包含then函数的对象。）所以，resolve函数可以如下定义：

	resolve: (root, {name}) => {
  		return someLogicReturningAPromise();
	}
	
[更屌的是如果你使用Babel，你就可以使用async/await特性（但我并没有使用，事实上，我使用的是q，而不是原生的Promises）]。

为了验证这个新增的root query，我们可以执行：

	curl -XPOST -H 'Content-Type:application/graphql'  -d '{ user(name: “david”) { name, caught, created } }' http://localhost:3000/graphql
	
###Mutations

mutations通常会返回修改后的新的modified数据对象（不同于只读的query）。对于我们的Pokedex API，我们将实现一个创建user和为指定user添加pokemon到服务。为了实现这个目的，我们需要在GraphQLSchema中添加mutation定义，和我们添加query类似：

	let schema = new GraphQLSchema({
  		query: new GraphQLObjectType({
    		name: 'RootQueryType',
    		fields: {
      			// root queries went here
    		}
 	 	}),
  		mutation: new GraphQLObjectType({
    		name: 'Mutation',
    		fields: {
      			// mutations go here!
    		}
  		})
	});
	
mutation配置和query很类似-它包含`type`，`args`和`resolve`。

我们第一个mutation是添加用户：

	upsertUser: {
  		type: UserType,
  		args: {
    		name: {
      			description: 'The name of the user',
      			type: new GraphQLNonNull(GraphQLString)
    		}
  		},
  		resolve: (obj, {name}) => {
    		return someLogicReturningAPromise();
  		})
	}
	
内部的，resolve函数会去mongo数据库中查询给定的用户名，如果不存在就会新建这个用户。但这些细节并不需要你关心。

调用这个mutation如下：

	curl -XPOST -H 'Content-Type:application/graphql'  -d ‘mutation M { upsertUser(name: “newUser”) { name, caught, created } }' http://localhost:3000/graphql
	
参数`mutation`很重要，后面要跟一个name--这里，name就是`M`，如果你删除了`mutation`，意味着你告诉GraphQL你想在root query中执行upsertUser（这是不存在的）。如果你省略了`M`,GraphQL会报语法错误告诉你需要给定一个name。

我们要实现的第二个mutation是获取pokemon--这里的参数是用户名和pokemon名。我们的mutation非常简单：

	caughtPokemon: {
  		type: UserType,
  		args: {
    		name: {
      			description: 'The name of the user',
      			type: new GraphQLNonNull(GraphQLString)
    		},
    		pokemon: {
      			description: 'The name of the Pokemon that was caught',
      			type: new GraphQLNonNull(GraphQLString)
    		}
  		},
  		resolve: (obj, {name, pokemon}) => {
    		return someLogicReturningAPromise();
 		 })
	}
	
调用这个mutation如下：

	curl -XPOST -H 'Content-Type:application/graphql'  -d ‘mutation M { caughtPokemon(name: “newUser” pokemon: “Snorlax") { name, caught, created } }' http://localhost:3000/graphql
	
###Closing

GraphQL的文档和生态圈仍处于婴儿时期，但就我们目前在一些会议和其发布的文档来看，它是很简单灵活的。从我在这篇文章中所获得的经验中来看是非常值得令人激动的。但更令我激动的是与Relay的结合。我非常乐观的认为，这些工具将会加速开发和减少代码，让我从死板的后端和数据中解脱出来。


###读后感

看完这两篇文章，我觉得大家都应该大致了解GraphQL的用法了吧，当然这里面肯定还有很多高级特性没有看到，不过GraphQL官方说明书可真是好长好长啊！！

其实我觉得，可以使用GraphQL作为中间件，后端依然请求rest api，这可能是目前我觉得最稳妥的方案，毕竟关于GraphQL我们还有太多的未知，而且GraphQL还存在不少变数！