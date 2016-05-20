之前一直都是做SPA项目，不过最近由于种种原因，需要返璞归真来基于jquery做项目！但无奈之前一直使用像ng或react这样的大杀器在编程，思维上已经有了一定的模式，突然回到jquery时代，感觉不会写代码了！真是感叹前端发展的迅猛啊~

其实react已算很轻了，但jsx语法我感觉确实是一个坎，虽然在我看来是精华也是趋势，但在业务量面前我实在不忍心再给团队成员加砝码了！

之所以考虑让项目基于jquery，是因为目前的前端开发人员和我所在的城市的前端开发人员整体程度比较低（只熟悉jquery），所以这么抉择，虽然保守，但让项目可以更快的完成，也有利于扩员，算是一种权衡！

但jquery确实太老了，只用jquery显然不能满足项目的需求，尤其是我这种生活在美好的数据绑定环境下的人，确实无法接受老旧的dom操作！

我在github上搜索："jquery data binding"，推荐给我的一款基于jquery的数据绑定库是真尼玛难用，所以我就不贴链接了，怕被打脸！总之，我们的目的是尽量找学习成本低，侵入性低的库，哪怕功能比较弱一些，哪怕性能比较差一些（我们的项目是内部系统，so~）！

结果，偶然间哥注意到了Vue.js，完美！！！学习成本非常的低，而且十分的优雅，社区也很繁荣，简直了！那么接下来就是掌握一下它的使用方法，由于它出生在和react差不多一个时代，所以思路和用法都类似，看看官方文档就能上手做项目了！（希望没有大坑在等我~）

单用vue，肯定也还是不够的，既然我们引入了它，自然要发挥它最大功效，回到今天的主题：表单验证！（别嫌我啰嗦，我就是话多！）

之前用过不少表单验证插件，但实话说，看完vue-validator的[官方文档](http://vuejs.github.io/vue-validator/)，真的让我非常的震撼，基本上属于扩展性非常强的插件了，作者考虑到了绝大多数适配的场景，堪称完美！

不过，在实际使用的时候，还是有一些文档上没有写太详细的小坑，我的任务就是把我使用的时候碰到的问题记录下来，我们开始吧！

##### 安装

由于我没有使用任何加载框架，只是简单的在浏览器上使用vue和vue-validator，所以我们只需要直接加载它们即可，不需要做额外的工作，这里我使用的是[bootcdn](http://www.bootcdn.cn/)提供的cdn，代码如下：

```html
<script src="//cdn.bootcss.com/vue/1.0.17/vue.js"></script>
<script src="//cdn.bootcss.com/vue-validator/2.0.0-alpha.22/vue-validator.js"></script>
```

需要注意的是加载顺序和版本，我测试加载vue-validator的1.X版本的话并不会自动装载到vue中，不清楚是版本的事儿还是其它问题，总之，使用最新的版本即可。

##### 绑定model

最新的vue-validator下，在使用时已经不支持下面的写法：

```html
<input
	type="text"
	v-model="test"
	v-validate="{required:true}"/>
```

会提示：
	Uncaught TypeError: Cannot read property 'replace' of undefined

所以你只能老老实实的写成：

```html
<input
	type="text"
	v-validate:test="{required:true}"/>
```

##### 结构复杂的model如何绑定

我的例子中model的结构比较恶心，是这样的：

```javascript
var blockBox = new Vue({
	el: "#blockBox",
	data: {
		table: {
			rowA: {
				columnA: "a-a",
				columnB: 'a-b',
				columnC: 'a-c'
			},
			rowB: {
				columnA: 'b-a',
				columnB: 'b-b',
				columnC: 'b-c'
			}
		}
	});
```

那我总不能写成这样吧：

```html
<input
	type="text"
	v-validate:table.rowA.columnA="{required:true}"/>
```
丑不说，何况这么写也是错误的！

怎么办？按照[官方的说法](http://vuejs.github.io/vue-validator/en/structure.html)应该这么写：

```html
<input
	class="form-control"
	type="text"
	v-model="table.rowA.columnA"
	v-validate:message-aa="{required:true}"/>
<div class="errors">
	<p v-if="$blockBoxValidator.messageAa.required" class="lang" data-key="blockBoxAA" />
</div>
```
完美，注意我在`v-validate:message-aa`中的写法！千万不能写成驼峰式的，如`v-validate:messageAA`，这么做会报错哟~原因我才和vue解析有关，这应该牵扯到了html属性的格式规范问题！总之你避免这么写就ok了！

##### 和jquery配合

我不想说我下面的这个场景是否合理，但为了快糙猛的把项目demo搞出来，我遇到了下面的问题。

场景是我们的项目需要多语言支持，界面上能看到的元素（不包括用户输入）都应该支持切换语言，包括表单错误提醒！目前我所使用的切换语言的方法很简单：

```html
<p v-if="$blockBoxValidator.message_aa.required" class="lang" data-key="blockBoxAA" />
<script type="text/javascript">
$(function(){
	//多语言处理
	$(".lang").each(function(node){
		$(this).html($langCFG[$(this).data("key")]);
	});
});
</script>
```
利用jquery的`domready`事件来第一时间替换需要显示的文字！这就有个问题出现了，若你在`domready`事件触发前就初始化了vue和vue-validator，根据表单验证逻辑很可能导致`v-if`指令直接干掉表示错误信息的那个`p`标签，所以自然也不会正确的执行多语言替换的逻辑。

在不考虑花时间研究vue-validator生命周期的前提下，我们如何更快的绕过这个问题呢？第一感觉是，将初始化vue和vue-validator的工作也放在`domready`中（但注意一定要放在多语言处理的代码之后）。不过你还会遇到另外一个问题，这个时候你在浏览器调试终端中按照vue官方的例子打印你创建的`vm`时，会报错提示变量定义不存在~具体原因不清楚，不过我们依然可以搞定这个问题，最终的代码如下：

```html
<p v-if="$blockBoxValidator.message_aa.required" class="lang" data-key="blockBoxAA" />
<script type="text/javascript">
var blockBox = null;	//变量声明，这样能保证在调试终端中可以直接使用'blockBox.$log()'
$(function(){
	//多语言处理
	$(".lang").each(function(node){
		$(this).html($langCFG[$(this).data("key")]);
	});

	//放在多语言处理下面，且在domready事件内
	blockBox = new Vue({
		el: "#blockBox",
		data: {
			table: {
				rowA: {
					columnA: "a-a",
					columnB: 'a-b',
					columnC: 'a-c'
				},
				rowB: {
					columnA: 'b-a',
					columnB: 'b-b',
					columnC: 'b-c'
				}
			}
		}
	);
});
</script>
```

理论上，你只要把你想用js做的业务逻辑代码都放在`domready`事件的回调方法中的vue初始化之后，就可以像和单独使用vue写代码一样的流程来进行开发了！

##### 页面初次加载显示问题

这应该不是vue-validator的问题，这应该属于vue的问题域。之前angular推荐的解决方案是提供了一个指令来监听模版解析完毕的事件。
我搜了一圈没找到官方或第三方提供的类似功能的实现，不过自己来做也不是什么难事儿：

```html
<body id="body" style="display:none">
{{test}}
</body>
<script type="text/javascript">
new Vue({
	el:"#body",
	ready: function(){
		$("#body").show();
	}
});
</script>
```
其实思路就是借助vue组件的生命周期hook来及时的修改css样式，你也可以添加loading动画等高级特效，我懒我就不弄了~

##### $resetValidation()

重置验证的场景是这样的：当你的表单被用户合法修改保存后，用户并没有跳转到其他页面，仍然在当前页面下，由于表单内容已经和最原始的版本不一样，你需要resetValidation一下从而使vue-validator的各个状态匹配业务场景（包括但不限于`dirty`,`modified`等状态）。不过这里有两个小插曲：

1. 我直接使用的bootcdn提供的版本导致js报错说$resetValidation()未定义，切换成官方的最新版本即可解决；
2. 应该是跟vue的[vm更新机制](http://vuejs.org.cn/guide/reactivity.html#异步更新队列)有关:

```javascript
var that = this;
setTimeout(function(){
	that.$resetValidation();
},0);
```
否则呢，你无法看到期望的结果~

##### lazy的正确使用方法

有些表单的初始化数据需要异步获取，太正常不过了，对吧？vue-validator提供了lazy模式，刚好匹配这种场景。所以，你只需要在`validator`标签上添加`lazy`属性即可，[官方例子](http://vuejs.github.io/vue-validator/en/lazy.html)简单明了，似乎没有什么要解释的！不过，问题来了，官方的例子代码是基于vue的动态组件写的，动态组件才有`activate`事件，而如果你像我一样是用在普通组件下的，那么自然不会触发`activate`事件，那回调逻辑自然不会被执行，`this.$activateValidator()`不会执行的话，看看vue-validator会提示什么：

```
[vue-validator] validator element directive need to specify 'name' param attribute: (e.g. <validator name="validator1">...</validator>)
```
看起来提示的非常不准确，让我排查了好久，甚至去联系作者来查看这个问题！其实你只要确保在正确的时机调用`$activateValidator()`方法就可以了，那什么叫正确的时机：

- 普通组件的`ready`事件回调中
- 动态组件的`activate`事件回调中

否则你一定会看到上面的那个提醒，特别提醒，下面的写法依然会看到提醒：

```javascript
ready: function(){
	setTimeout(function(){
		this.$activateValidator();
	}.bind(this), 3000);
},
```

这应该是作者好心办的"错事儿"吧，他要求你只能在组件渲染前把该做的都做了（数据获取），不然你就是在自讨苦吃~当然，这种设计哲学并非没有道理，全看权衡和喜好！

---

好了，先总结到这里，如果以后有啥新的结论，再说~
