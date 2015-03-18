之前写过基于[WebUploader+Java/Php/Nodejs](http://blog.kazaff.me/2014/11/14/%E8%81%8A%E8%81%8A%E5%A4%A7%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0/)的文章，今天来结合dubbo聊一下文件上传的问题。

目前我们的项目中，文件上传功能是以独立服务的方式部署的，也就是说多个不同的应用可以向同一个文件上传API发起请求（可能需要跨域支持）。我个人是比较喜欢这样的部署方式，不过任何事都是有利弊的，这里就不展开讨论了。

复习完上面的内容后，我们开始今天的主题，那就是基于dubbo来完成文件上传服务的开发，这么做的动机也很单纯，就是**尽可能保持统一的设计哲学和实现方案**。统一的好处，从运维、监控等多方面也是非常有必要的，**任何自动化的前提是足够的规范**。

要想做到把之前写的java版本的文件上传功能集成到dubbo中，我们就得先了解一下dubbo的[协议细节](http://alibaba.github.io/dubbo-doc-static/Protocol+Reference-zh.htm)。不难发现，凡是以单一长连接提供通信的协议都不太适合大数据的传输，原因很简单，凡是单个客户端会长期占用通道传输的场景都会造成通道阻塞，你可以脑补高架上堵车的景象。

而我们这里提供文件上传服务要使用的协议是REST，也就是dubbox新增的一个方式，具体细节可以看[这里](https://github.com/kazaff/me.kazaff.article/blob/master/dubbox%E6%96%B0%E5%A2%9E%E7%9A%84REST%E5%8D%8F%E8%AE%AE%E5%88%86%E6%9E%90.md)和[这里](http://dangdangdotcom.github.io/dubbox/rest.html#show-last-Point)。其实和上面文档描述的`http://`协议很相似。

之前的请求流程如下图：

	client ---> tomcat ---> servlet ---> spring ---> springMVC ---> business code

dubbo化后的如下图：

	client ---> tmocat embed ---> servlet ---> dubbo/spring ---> resteasy ---> business code

不确定这么描述是否合适，但大致就是这样了，其中`springMVC`和`resteasy`做的事儿是一样的。我之所以犹豫，是因为不确定`dubbo`放在哪合适~~完成我们的预期需求，我们可以按照以下几步来走：

1. 编写Filter实现跨域请求（因为我们的前端全部都是Ajax调用）；
2. 配置以REST协议暴露dubbo服务；
3. 使用RESTEasy及JAX-RS规范提供的注解完成把请求参数映射成Bean；
4. 实现文件上传的业务逻辑（其实就是把原先的springMVC版的相关代码移植到RESTEasy对应位置）
5. 测试


剩下的任务就是实操了，我会更新[原先的github项目](https://github.com/kazaff/webuploaderDemo)，有兴趣的盆宇可以去关注一下。


