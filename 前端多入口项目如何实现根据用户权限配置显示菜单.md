### 项目背景和限制条件

目标是要为一个ERP项目做前端实现，该项目前后端通信完全走REST，不过由于项目时间和人力有限，前端开发人员对目前主流的MVVM框架并不了解，所以之前的单入口思路就无法复用了。

这次前端实现采用的是多入口设计，且不使用html iframe方案。基于jquery的dom操作和ajax来完成大多数开发任务，引入lodash作为工具库来处理复杂数据结构。

再来说一下项目的概况，和大多数后台管理系统一样，这个ERP依然提供为用户配置操作权限的功能，这就要求前端界面需要根据用户的实际操作权限来响应哪些操作链接可以显示在界面上，哪些需要隐藏起来。当然，这只是为了界面显示做的配置，实际后端服务依然会判断用户权限的，不过界面上根据用户权限来控制菜单或按钮的显示情况，也是必不可少的。

除此之外，考虑到这个ERP功能是按照模块划分的，每个模块的功能相对独立，映射到项目结构上，我希望每个模块的相关代码（html，js，css等）都应该存放在各自的模块文件夹下，为将来维护提供良好的基础。

万幸的是，目前项目只针对chrome高版本，意味着我们可以不鸟兼容问题！！！

### 项目依赖

#### 运行依赖

- jquery
- lodash 4.8.2+
- sessionStorage

#### 编译依赖

- webpack
- node

具体依赖版本，可以看[package.json](https://github.com/kazaff/menuIfShow/blob/master/package.json)。

### 运行方式

将项目下载至本地web目录，然后执行下面的命令：

```shell
npm install
webpack --display-error-details
```

然后访问本地web服务。

注意：本模块只借助webpack编译所需config文件和项目发布时的压缩优化而已，后面会详细解释。

### 实现原理

由于前面介绍到，项目是多入口且不使用iframe，这会带来一个问题：每次页面跳转，理应都需要重新处理所有逻辑。

> 基于angular(1)的类似功能之前写过一个，放在[这里](https://github.com/kazaff/angular-build-seed)，有需要的同学可以去看看哈~。

为了避免不必要的性能损耗，我们将一些中间处理结果存在sessionStorage中，在用户会话有效期内可以起到缓存的目的。

下面用一个流程图来描述该组件大致的工作内容：

![](http://pic.yupoo.com/kazaff/FsxInLlx/H2X85.png)

这样，最大程度的避免了当页面跳转所造成的重新计算带来的损耗。

### 项目结构

根据设计，该项目按照功能模块来组织文件结构，文件结构骨架如下：

```
\
|---build	编译生成的文件夹
|---mock  
|		|---myAcl.json 用来模拟后端服务响应
|			
|---modules	模块文件夹
|			|---a  模块a
|			|		|---config.js 模块a的配置文件
|			|		|---index.html 模块a的html文件
|			|
|			|---b 模块b
|			|		|---config.js 模块b的配置文件
|			|		|---index.html 模块b的html文件
|
|---boot.js 该模块核心逻辑（上面的流程图描述的就是该js的逻辑）
|---index.html 非模块级别的html文件（例如实现login页面）

```

其它非业务文件在这里就不介绍了。

按照文件结构骨架，在实际项目开发时，应该按照划分好的模块为其在modules文件夹中创建一个独立的文件夹，将独属于该模块的资源全部放到该文件夹下。其中必须要存在一个config.js文件，用来配置该模块所包含的操作权限，具体格式可参考[这里](https://github.com/kazaff/menuIfShow/blob/master/modules%2Fa%2Fconfig.js)。

### 使用方式

该模块被封装成一个jquery插件：$.fwMenu：

```html

<script src="//cdn.bootcss.com/lodash.js/4.6.1/lodash.js"></script>
<script src="//cdn.bootcss.com/jquery/2.2.1/jquery.js"></script>
<script type="text/javascript">
	$(function(){
		var menu = new $.fwMenu({
			debug: true,
			node: $("#menu"),
			handleHtml: function(topMenus, getSonMenus){
				console.log(topMenus);
				console.log("================");
				console.log(getSonMenus(1));
				return "<ul class='test'><li>test</li><li>test</li></ul>";
			}
		}, function(instance){
			console.log($(".test").html());
		});
	});
</script>

```

初始化时所需的参数描述：

* debug: bool，表示是否无视缓存重新初始化所有数据，默认为false，可用于用户重新登录
*	node: jquery对象，表示主菜单dom，必填项，例如：$("#menu")
*	handleHtml: func(topMenus, sonMenus function(topMenuId))，用来创建menu html string，必填项，例如： function(data)，参数data代表finalConfig
* userToken: func，用来获取当前登录用户的sessionid
*	restApi: func，用来设置需要请求的后端服务地址
*	fetch: func，用来发送ajax请求的方法，返回一个jq ajax Promise对象
*	handleError:	func(status, error)，用来处理ajax异常

该插件，还提供了三个方法：

* getLinkById(id) : 根据给定操作编号，返回操作对应的json数据，用于非菜单的情况下获取操作链接信息（例如按钮）
* getLinkByAddress(prefix) : 根据地址栏url来返回对应的菜单项json数据，用于根据地址栏来控制对应菜单项的显示状态（例如展开，高亮等）
* rebuildMenu(handle) : 重新构建menuHtml，根据handle处理函数重新生成menuHtml（用于多语言切换等场景）


具体实现细节，推荐看项目的[boot.js代码](https://github.com/kazaff/menuIfShow/blob/master/boot.js)。

### 关于webpack编译

为了方便开发，每次修改modules中的config.js文件或html文件，webpack都会自动重新编译，并将最新结果放在build中。但并不完美。

#### Todo

目前不支持自动编译的操作如下：

- 非config.js，css，img等所需的资源文件，如果需要新增、编辑、删除该部分资源，不会触发自动编译

#### 推荐工作流程

1. 先根据需要在modules中建立好模块结构（包括要用到的config.js和html文件）
2. 执行`webpack -w`命令，进行日常开发工作
3. 每当不满足自动触发编译条件的事情发生时，先删除bulid文件夹，再手动重新执行`webpack -w`命令

该工作流程如果不满足你的需要，请自行完善。（PS：我是真的不了解webpack！！！抱歉~）
