昨天碰到个[问题](http://segmentfault.com/q/1010000003899542)，GG了半天也没有发现一篇对应主题的文章，最后还是在react-router的github官方求助，才被热心的大牛给上了一课，其实这也都怪自己没有耐心仔细阅读react-router的官方资料，下面就来简单汉化一下官方针对这个问题的相关解释。

以下内容翻译自官方文档，原文链接：[https://github.com/rackt/react-router/blob/master/docs/guides/advanced/ComponentLifecycle.md](https://github.com/rackt/react-router/blob/master/docs/guides/advanced/ComponentLifecycle.md)

---

##Component Lifecycle

理解router在其生命周中会触发哪些hooks调用对于实现你的应用场景非常重要，常见的场景是何时抓取数据。

在router中组件的生命周期和react中定义的并没有什么不同，让我们来假设一个场景，其route配置如下：

	<Route path="/" component={App}>
  		<IndexRoute component={Home}/>
  		<Route path="invoices/:invoiceId" component={Invoice}/>
  		<Route path="accounts/:accountId" component={Account}/>
	</Route>
	
###Lifecycle hooks when routing

1. 我们假设用户访问应用的"/"页面。

	| 组件 | 生命周期内的hooks调用 |
    |-----------|------------------------|
    | App | `componentDidMount` |
    | Home | `componentDidMount` |
    | Invoice | N/A |
    | Account | N/A |
    
2. 然后，用户从"/"页面跳转到"/invoice/123"

	| 组件 | 生命周期内的hooks调用 |
    |-----------|------------------------|
    | App | `componentWillReceiveProps`, `componentDidUpdate` |
    | Home | `componentWillUnmount` |
    | Invoice | `componentDidMount` |
    | Account | N/A | 

	- `App`组件由于之前已经被渲染过，但是由于将会从router中接受到新的props（`children`，`params`，`location`等），所以会调用其`componentWillReceiveProps`和`componentDidUpdate`方法
	- `Home`组件此时已经不再需要了，所以它会被卸载
	- `Invoice`组件此时将会被首次渲染
	
3. 再然后，用户从"/invoice/123"页面跳转到"/invoice/789"

	| 组件 | 生命周期内的hooks调用 |
    |-----------|------------------------|
    | App | componentWillReceiveProps, componentDidUpdate |
    | Home | N/A |
    | Invoice | componentWillReceiveProps, componentDidUpdate |
    | Account | N/A |
	
	这次所有需要的组件之前都已经被加载了，此时它们都会接收到来自router传递的新props。
	
4. 最后，用户从"/invoice/789"页面跳转访问"/accounts/123"

	| 组件 | 生命周期内的hooks调用 |
    |-----------|------------------------|
    | App | componentWillReceiveProps, componentDidUpdate |
    | Home | N/A |
    | Invoice | componentWillUnmount |
    | Account | componentDidMount |

###Fetching Data

在使用router时有很多种方法来获取数据，其中最简单的一种方式就是利用组件的生命周期hooks，并保存数据在组件的state中（译者：这一点在Redux设计理念中被强烈反对，事实证明即便是不依赖state，这种方式照样可行，我们后面会给出例子）。我们已经知道了当router切换时组件的生命周期hooks调用过程，我们来为之前的`Invoice`组件来实现一个简单的获取数据逻辑。

	let Invoice = React.createClass({

  		getInitialState () {
    		return {
      			invoice: null
    		}
  		},

  		componentDidMount () {
    		// fetch data initially in scenario 2 from above
    		this.fetchInvoice()
  		},

  		componentDidUpdate (prevProps) {
    		// respond to parameter change in scenario 3
    		let oldId = prevProps.params.invoiceId
    		let newId = this.props.params.invoiceId
    		if (newId !== oldId)
      			this.fetchInvoice()
  		},

  		componentWillUnmount () {
    		// allows us to ignore an inflight request in scenario 4
    		this.ignoreLastFetch = true
  		},

  		fetchInvoice () {
    		let url = `/api/invoices/${this.props.params.invoiceId}`
    		this.request = fetch(url, (err, data) => {
      		if (!this.ignoreLastFetch)
        		this.setState({ invoice: data.invoice })
    		})
  		},

  		render () {
    		return <InvoiceView invoice={this.state.invoice}/>
  		}
	})
	
---

##译者总结

在本篇开头我的那篇问题贴中已描述场景，由于我一直以来对react的生命周期hooks错误的理解，导致死活无法优雅的实现兼容浏览器前进后退操作的组件。经过上面内容的学习，一切都不在话下。

首先，我之前总是习惯使用`componentWillMount`方法来初始化组件的状态，事实证明这种写法并非适用于所有场景（尽管使用它也可以，但至少不是更多人推荐的）。其次，尽可能避免让组件包含太多会话状态，保证组件的纯净。

在我的代码中，更改后的完美版本如下：

（算了，想了想这部分代码的附加逻辑太多，不适合用来阐述上述观点，还是不贴了～）