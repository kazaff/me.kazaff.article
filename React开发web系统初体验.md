大热React的这段时间里，刚好碰到我找工作，所以画了一些时间学了一下react，用其开发了一个小web系统，下面主要是介绍一下其中的一些收获。


React ＋ ？ 
---
这里我想说的是，只用react你是不足以搭建一个web系统的前端的，保守的说，你还需要：

1. 路由：react-router
2. ajax：我使用的是jQuery
3. 项目文档结构：

![](http://pic.yupoo.com/kazaff/EU3r7OaO/pa2RS.jpg)

你会发现上图中存在两种带描述后缀的文件：XXXAction.js 和 XXXStore.js，之所以这么分，是因为项目里还是用了[reflux](http://blog.kazaff.me/2015/05/24/React%E5%92%8Cflux%E5%88%9D%E5%B0%9D%E5%BF%83%E5%BE%97/)，这个理念我们之前已经说过了。

除此之外，所有页面文件的后缀名我使用`.jsx`，其他都是`.js`，我认为这样有助于快速的了解对应文件的作用。


上述文件结构其实不算很灵活，实际开发中碰到有一些通用需求的页面，又不是太适合放在components里，就显得无法适配了，除此之外，components中的可复用web控件，理应不包含数据获取相关业务逻辑，但由于时间关系，我打破了这个规则，这样这些控件就无法直接用于其他项目了。。。以后开发需要注意～


组件化思维
---

之前在使用angular的时候，指令其实是推荐使用的，但是你也完全可以不考虑组件模式，而react就不行了，你无时无刻不被强迫思考组件这个概念，因为react提供的语法本身就无时无刻体现了该模式，这一点我认为是极好的。

可以说虽然react不像angular那样提供大量的内容需要开发者掌握，但想要用好react，必须建立起组件思维，官方提供的demo中就强调了如何把原先思考页面的布局改为思考页面的组件，听起来简单但实际做的时候还是会有些不适应。


语法小贴士
---

1. 由于有babel这样的好东西，所以推荐从今天起就开始使用es6
2. 好好学一下webpack的用法
3. 那些不用来影响视图的变量就不要放在state中
4. 组件嵌套时，props的值不要在组件内部做二次赋值，这样会导致react数据绑定“失效”，下面我给个例子：

		let Inside = React.createClass({
  			componentWillMount(){
    			this.setState({
      				a: this.props.params.a,
    			});
  			},
  
  			render(){
    			return (
      				<div>{this.state.a}</div>
    			);
  			},
		});

		let Outside = React.createClass({
  			getInitialState(){
    			return {
      				data: {
        				a: 1,
      				},
    			};
  			},
  
  			render(){
    			return (
      				<Inside params={this.state.data} />
    			);
  			},
		});

看上去按说应该在外层每次修改`this.state.data.a`的时候内层都应该立刻显示最新的a值，但其实不然，原因其实很简单，内层`componentWillMount`方法只会在组件第一次加载时调用一次，然后就没有然后了。

聪明的童鞋应该知道怎么改了，我就不多说了。




最后
---

我给出我项目的依赖库：

	"devDependencies": {
      "babel-core": "^5.8.20",
      "babel-loader": "^5.3.2",
      "css-loader": "^0.15.6",
      "file-loader": "^0.8.4",
      "html-webpack-plugin": "^1.6.0",
      "react-hot-loader": "^1.2.8",
      "style-loader": "^0.12.3",
      "url-loader": "^0.5.6",
      "webpack": "^1.10.5",
      "webpack-dev-server": "^1.10.1",
      "amazeui": "^2.4.0",
      "amazeui-react": "latest"
   	},
    "dependencies": {
      "react": "^0.13.3",
      "react-notification-system": "^0.1.14",
      "react-router": "^0.13.3",
      "reflux": "^0.2.11"
    }
    
下面是项目的webpack配置文件：

	var path = require('path');
	var webpack = require('webpack');
	var HtmlWebpackPlugin = require('html-webpack-plugin');
	var node_modules = path.resolve(__dirname, 'node_modules');
	
	var deps = [
  		//'react/dist/react.min.js',
  		'react-router/umd/ReactRouter.min.js',
  		//'amazeui-react/dist/amazeui.react.min.js',
	];

	var config = {
  		entry: [
    		path.resolve(__dirname, "app/app.jsx"),
    		"webpack/hot/dev-server",
  		],
  		resolve: {
    	alias: {}
  		},
  		output: {
    		path: path.resolve(__dirname, "build"),
    		filename: "bundle.js",
  		},
  		plugins: [
    		/*new webpack.optimize.UglifyJsPlugin({
      			compress:{
        			warnings: false,
      			},
    		}),
    		new webpack.optimize.CommonsChunkPlugin('common', 'common.js'),*/
    		new HtmlWebpackPlugin({
      			inject: true,
      			template: 'app/index.html',
    		}),
    		new webpack.NoErrorsPlugin(),
  		],
  		module: {
    		loaders: [
      		{
        		test: /\.jsx?$/,
        		//loaders: ['react-hot','babel']
        		loader: 'babel'
      		},
      		{
        		test: /\.css$/,
        		loader: 'style!css',
      		},
      		{
        		test: /\.(png|jpg|eot|ttf|woff|woff2)$/,
        		loader: 'url',
      		}
    		],
    		noParse: [],
  		},
  		debug: true,
  		devtool: 'eval-cheap-module-source-map',
  		devServer: {
    		contentBase: path.resolve(__dirname, "build"),
    		historyApiFallback: true
  		}
	};

	deps.forEach(function (dep) {
  		var depPath = path.resolve(node_modules, dep);
  		config.resolve.alias[dep.split(path.sep)[0]] = depPath;
  		config.module.noParse.push(depPath);
	});

	module.exports = config;


实际使用下来，感觉react要比ng强很多，同时也非常期待ng2的问世。
