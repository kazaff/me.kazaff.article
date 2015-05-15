搞了一段时间的JAVA，又要再一次回归前端技术了，其实这么说也不准确，因为现在的前端技术已经囊括了太多~

这次回来，是为了征服**移动端**，也很可能最后一次为目前的公司征战沙场了（有种说不出道不明预感~）！我一直坚信，应用级别的APP开发套件迟早一定是H5的天下，跨平台永远是人类追求的目标，这个应该是毋庸置疑的！谁能做到真的“跨平台”，谁就能笼络更多的人心。

目前我看到的发展史是：

	Native --> Hybrid --> H5

虽然这路程并不平坦，反反复复坎坎坷坷，但大方向还是非常明确的~

扯了那么多，今次的主题是AMD和CMD，之所以纠结这两个概念，是因为目前项目中计划开发一些用于封装逻辑的SDK，那么，就需要我们提供各种语言和平台下的SDK，当然这个范围一开始并不会太大，毕竟能力有限。

前端，移动端，Node端我打算实现基于JS的SDK，这就要求在项目模块化组织时选择一个通用性更强的标准，一开始打算使用RequireJS，因为目前公司的项目中使用的就是它，不过，它是基于AMD概念的，也就是说更偏重于前端，不过，还好，RequireJS可以兼容基于CMD概念实现的模块，下面我看一个例子：


	Project
		|-index.html
		|-main.js
		|-scripts
		|	|-require.js
		|-lib
			|-foo
			|	|-main.js
			|-bar
				|-main.js

index.html

	<!DOCTYPE html>
	<html>
	<head><title></title></head>
	<body>
	<script data-main="./main" src="scripts/require.js"></script>
	</body>
	</html>

main.js

	require.config({
		baseUrl: "./lib", //所有模块的base URL,注意，以.js结尾的文件加载时不会使用该baseUrl
		packages:["foo","bar"],	//需要把所有CMD的模块都声明在这里
		waitSeconds: 10   //waitSeconds是指定最多花多长等待时间来加载一个JavaScript文件，用户不指定的情况下默认为7秒
	});
	
	require(["foo"],function(foo){
		console.log("test");
		foo.log();
	});

lib/foo/main.js

	define(function(require, exports, module){
		exports.name = "foo";
		exports.log = function(){
			console.log(this.name);
		}
	
		var bar = require("bar");
		bar.log();
	
	});

lib/bar/main.js

	define(function(require, exports, module){
		exports.name = "bar";
		exports.log = function(){
			console.log(this.name);
		}
	});

代码很简单，直接放在web目录下测试即可，运行看一下控制台的输出：

	bar
	test
	foo

留意一下输出的顺序，“bar”甚至再“test”之前，这是为什么呢？理由很简单，因为`foo`模块被main.js依赖，所以requireJS会先加载并执行它，这个时候发现`foo`模块又依赖`bar`模块，所以会先去加载`bar`模块，加载完毕后执行`foo`模块中写的逻辑，这个时候会打印出“bar”，接下来，main.js的回调逻辑会执行，打印“test”，然后再调用“foo.log()”打印最后的“foo”。

这样，我们就可以把SDK以CMD规范来编写，将来用于多种场景~

参考：

[RequireJS 中文网](http://www.requirejs.cn/#show-last-Point)

[CMD 模块定义规范](https://github.com/seajs/seajs/issues/242)