今天跟同事讨论一个前端页面显示图片时的路径问题，当系统的前后台有不同的域名时，这个问题就会出现！

常用的html编辑器是可以定义图片的显示路径的，但这不是我们要讨论的重点。重点是，**如何保证内容中的图片相对路径可以在不同的域名下显示**。

好吧，这是一个比较讨厌的问题！一开始，我试图在前端显示的时候正则取到内容中的 *img* ，为其动态添加指定的域名！这明显是个简单除暴的方案~

有没有其他方案可以更优雅一些呢？答案是：必须的！

这里要介绍到一个 *html* 标签：[base](http://www.w3school.com.cn/tags/tag_base.asp)，搞前端的童鞋对这个肯定不陌生，有了它，我们就可以给页面上所有相对路径的标签提供根路径了，确实是一个一劳永逸的做法！

不过，只用它，还不行，为什么呢？可能和我们的具体项目有关，我们的前台中显示的文章内容目前是 *ajax* 获取的，这部分异步获取的数据才是我们要更改路径的目标，除此之外的资源路径不能被影响，有点混乱吧？其实很简单，看下面这种结构：
	
	<body>
		<div>
			这里面使用相对路径的img必须基于指定域名
		</div>
		<div>
			而这里面使用相对路径的img不能被上面的div指定的域名所影响
		</div>
	</body>

就是上面这种要求！好吧，只能引入 *iframe* 了，我们把第一个 *div* 里的内容丢入一个动态创建的 *iframe* 中，这样就能单独为其指定 *base* 了！

ok，既然方案就在眼前，剩下的就是上代码了：
	
	<html>
		<head>
		</head>
		<body>
			<div id="iframe"></div>
			<div>
				<img src="kazaff.png" /> /* 这个相对路径的图片不能被影响 */
			</div>
		
			<script src="jquery-1.11.0.js"></script>
			<script>
				$(function(){
					var iframe = $("<iframe style='border:0'></iframe>");	//创建一个iframe
					$("#iframe").append(iframe);	//插入到对应的div
					
					$(iframe[0].contentWindow.document.head).html("<base href='http://v4.vcimg.com/' />");	//给iframe加入base标签，用来定义根路径
					iframe.contents().find("body").append("<img src='global/images/logo.png' />");	//给iframe插入一个相对路径的图片
				});
			</script>
		</body>
	</html>