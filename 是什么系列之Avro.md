今天来关注一下Avro，目的是想接触一下跨端RPC中间件中关于数据编解码及传输的相关技术，这和我目前负责的项目很有关系！那么先从网上找一些相关的文献来给自己科普一下~

Avro是Hadoop的一个子项目，也是Apache中的一个独立项目，它是一个基于二进制数据传输高性能的中间件。在Hadoop的其他项目中（Hbase，Hive）的客户端与服务端的数据传输中被大量采用。听上去很给力啊？！

Avro是一个数据序列化的系统，它可以将数据结构或对象转化成便于存储或传输的格式，Avro设计之初就定位为用来支持数据密集型应用，适用于远程或本地大规模数据的存储于交换，Avro支持的编程语言包括：C/C++，JAVA，Python，Ruby，PHP，C#，它的特点有：

1. 丰富的数据结构类型
2. 快速可压缩的二进制数据形式（对数据二进制序列化后可节约数据存储空间和网络传输带宽）
3. 存储持久数据的文件容器
4. 可实现远程过程调用(RPC)
5. 简单的动态语言结合功能

Avro和动态语言结合后，读写数据文件和使用RPC协议都不需要生成代码，而代码生成作为一种可选的优化只需要在静态类型语言中实现。

Avro依赖于模式（Schema），通过模式定义各种数据结构，只有确定了模式才能对数据进行解释，所以在数据的序列化和反序列化之前，必须先确定模式的结构。同时可动态加载相关数据的模式，数据的读写都使用模式，这使得数据之间不存在任何其他标识（类比Thrift），这样就减少了开销，使得序列化快速又轻巧，同时这种数据及模式的自我描述也方便了动态脚本语言的使用。

当数据存储到文件中时，它的模式也随之存储，这样任何程序都可以对文件进行处理。如果读取数据时使用的模式与写入时使用的模式不同，也容易解决，因为读取和写入的模式都是已知的。看下面这个规则：

<table>
<thead>
<tr class="header">
<th align="left">新模式</th>
<th align="left">Writer</th>
<th align="left">Reader</th>
<th align="left">规则</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">增加了字段</td>
<td align="left">采用旧的模式（新增前）</td>
<td align="left">采用新的模式</td>
<td align="left">Reader对新字段会使用其默认值（Writer不会为新增的字段赋值）</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">采用新的模式（新增后）</td>
<td align="left">采用旧的模式</td>
<td align="left">Reader会忽略掉新曾的字段，Writer会为新增字段赋值</td>
</tr>
<tr class="odd">
<td align="left">减少了字段</td>
<td align="left">采用旧的模式（减少前）</td>
<td align="left">采用新的模式</td>
<td align="left">Reader会忽略已经被删除的字段</td>
</tr>
<tr class="odd">
<td align="left"></td>
<td align="left">采用新的模式（减少后）</td>
<td align="left">采用旧的模式</td>
<td align="left">如果旧模式里被删除的那个字段有默认值，那么Reader会采用，否则，Reader会报错</td>
</tr>
</tbody>
</table>

类型
---
数据类型标准化的意义在于：

1. 使不同系统对相同的数据能够正确解析
2. 有利于数据序列化/反序列化

Avro定义了几种简单数据类型，包括：null，boolean，int，long，float，double，bytes，string。简单数据类型有类型名称定义，不包含属性信息，如：
	{"type": "string"}

Avro定义了六种复杂数据类型，每一种复杂数据类型都具有独特的属性，如下：

**Records**：

	{
		"type": "record",
		"name": "名称json字符串[必填]",
		"namespace": "命名空间json字符串[选填]",
		"doc": "文档json字符串[选填]"
		"aliases": "别名json字符串数组[选填]",
		"fields": [	//json数组[必填]
			{
				"name": "字段名",
				"type": "类型",
				"default": "默认值",
				"order": "字段顺序"
			},
			....
		]
	}

例如：

	{
		"type": "record",
		"name": "test",
		"fields": [
			{"name": "a", "type": "long"},
			{"name": "b", "type": "string"}
		]
	}

**Enums**：

	{
		"type": "enum",
		"name": "名称json字符串[必填]",
		"namespace": "命名空间json字符串[选填]",
		"doc": "文档json字符串[选填]"
		"aliases": "别名json字符串数组[选填]",
		"symbols": [ //json数组[必填]
			"值json字符串[必填]，所有值必须是唯一的"
		]
	}

例如：
	
	{
		"type": "enum",
		"name": "test",
		"symbols": ["k", "a", "z", "a", "ff"]
	}

**Arrays**：

	{
		"type": "array",
		"items": "子元素模式"
	}

例如：
	
	{
		"type": "array",
		"items": "long"
	}

**Maps**：

	{
		"type": "map",
		"values": "值元素模式"
	}


**Fixed**：

	{
		"type": "fixed",
		"name": "名称json字符串[必填]",
		"namespace": "命名空间json字符串[选填]",
		"aliases": "别名json字符串数组[选填]",
		"size" : "整型，指定每个值的字节数[必填]"
	}

**Unions**： json数组

序列化/反序列化
---
Avro指定两种数据序列化编码方式：binary和json。使用二进制编码会高效序列化，并且序列化后得到的结果会比较小，而json一般用于调试系统或基于web应用。

简单数据类型：

<table>
<thead>
<tr class="header">
<th align="left">类型</th>
<th align="left">编码</th>
<th align="left">例子</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left">null</td>
<td align="left">0字节</td>
<td align="left">Null</td>
</tr>
<tr>
<td align="left">boolean</td>
<td align="left">1字节</td>
<td align="left">{true: 1, false: 0}</td>
</tr>
<tr>
<td align="left">int/long</td>
<td align="left">variable-length zig-zag coding</td>
<td align="left"></td>
</tr>
<tr>
<td align="left">float</td>
<td align="left">4字节</td>
<td align="left">Java’s floatToIntBits</td>
</tr>
<tr>
<td align="left">double</td>
<td align="left">8字节</td>
<td align="left">Java’s floatToIntBits</td>
</tr>
<tr>
<td align="left">bytes</td>
<td align="left">一个表示长度的long值，后跟指定长度的字节序列</td>
<td align="left"></td>
</tr>
<tr>
<td align="left">string</td>
<td align="left">一个表示长度的long值，后跟UTF-8字符集的指定长度的字节序列</td>
<td align="left">“foo”:{3,f,o,o}</td>
</tr>
</tbody>
</table>

复杂数据类型：

**records**类型会按字段声明的顺序串连编码值，例如下面这个record schema：
	
	{
		"type": "record",
		"name": "test",
		"fields": [
			{"name": "a", "type": "long"},
			{"name": "b", "type": "string"}
		]
	}

实例化这个record，假设给a字段赋值27(编码为0x36)，给b字段赋值“foo”(06 66 6f 6f，注意第一个06表示字符串长度3的编码值)，那么这个record编码结果为： 36 06 66 6f 6f

**enum**被编码为一个int，比如：
	
	{
		"type": "enum",
		"name": "test",
		"symbols": ["A", "B", "C", "D"]
	}

这将被编码为一个取值范围为[0，3]的int，0表示A，3表示D。

**arrays**编码为block序列，每个block包含一个long的count值，紧跟着的是array items，一个block的count为0表示该block是array的结尾。

**maps**编码为block序列，每个block包含一个long的count值，紧跟着的是key/value对，一个block的count为0表示该block是map的结尾。

**union**编码以一个long值开始，表示后面的数据时union中的哪种数据类型。

**fixed**编码为指定数目的字节。

![](http://pic.yupoo.com/kazaff_v/DSBloaTw/wq4vh.jpg)

上图表示的是Avro本地序列化和反序列化流程，Avro将用户定义的模式和具体的数据编码成二进制序列存储在对象容器文件中，例如用户定义了包含学号，姓名，院系和电话的学生模式，而Avro对其进行编码后存储在student.db文件中，其中存储数据的模式放在文件头的元数据中，这样读取的模式即使与写入的模式不同，也可以迅速的读取数据。假如另一个程序需要获取学生的姓名和电话，只需要定义包含姓名和电话的学生模式，然后用此模式去读取容器文件中的数据即可。

排序
---
Avro为数据定义了一个标准的排列顺序，“比较”在很多时候都是经常被使用的对象之间的炒作，标准定义可以进行方便有效的比较和排序，同时标准的定义可以方便对Avro的二进制编码数据直接进行排序而不需要反序列化。

只有当数据项包含相同的Schema时，数据之间的比较才有意义，比较按照Schema深度优先，从左至右的顺序递归进行，找到第一个不匹配即可终止比较。

两个拥有相同模式的项的比较按照以下规则进行：

1. null：总是相等
2. int,long,float：按照数值大小
3. boolean：flase在true之前
4. string：按照字典顺序
5. bytes，fixed：按照byte的字典顺序
6. array：按照元素的字典顺序
7. enum：按照符号在枚举中的位置
8. record：按照域的字典顺序，如果指定了以下属性：
	1. ascending：域值的顺序不变
	2. descending：域值的顺序颠倒
	3. ignore：排序时忽略域值
9. map：不可进行排序比较

对象容器文件
---
Avro定义了一个简单的对象容器文件格式，一个文件对应一个模式，所有存储在文件中的对象都是根据模式写入的，对象按照块进行存数，块可以采用压缩的方式存储。为了在进行MapReduce处理的时候有效的切分文件，在块之间采用了同步记号。

![](http://pic.yupoo.com/kazaff_v/DSBxUm94/VtHbS.jpg)

一个文件有两部分组成：文件头(Header)和一个或多个文件的数据块(Data Block)。而头信息由由三部分构成：四个字节的前缀，文件Meta-data信息和随机生成的16字节同步标记符。目前Avro支持的Meta-data有两种：schema和codec。

codec表示对后面的文件数据块采用何种压缩方式。Avro的实现都需要支持下面两种压缩方式：null（不压缩）和deflate（使用Deflate算法压缩数据块）。除了文档中认定的两种Meta-data，用户也可以自定义适用于自己的Meta-data，这里用long型来表示有多少个Meta-data数据对，也是让用户在实际应用中可以定义足够的Meta-data信息。

对于每对Meta-data信息，都有一个string型的key（要以“avro.”为前缀）和二进制编码后的value。由于对象可以组织成不同的块，使用时就可以不经过反序列化而对某个数据块进行操作。还可以由数据块数，对象数和同步标记符来定位损坏的块以确保数据完整性。

RPC
===

当在RPC中使用Avro时，服务器和客户端可以在握手连接时交换模式，服务器和客户端有彼此全部的模式，因此相同命名字段，缺失字段和多余字段等信息之间通信中需要处理的一致性问题就可以解决。

客户端希望同服务器端交互时，就需要交换双方通信的协议，它类似于模式，需要双方来协商定义，在Avro中被称为消息。

![](http://pic.yupoo.com/kazaff_v/DSCXbgqa/Jzy9D.jpg)

上图所示，协议中定义了用于传输的消息，消息使用框架后放入缓冲区中进行传输，由于传输的开始就交换了各自的协议定义，因此数据就能够正确解析。所谓传输的开始，也就是很重要的握手阶段。

握手的过程是确保Server和Client获得对方的Schema定义，从而使Server能够正确反序列化请求信息，Client能够正确反序列化响应信息。通常，Server/Client会缓存最后使用到的一些协议格式，所以大多数情况下，握手过程不需要交换整个Schema文本。

所有的RPC请求和响应处理都建立在已经完成握手的基础上，对于无状态的连接，所有的请求响应之前都附有一次握手过程，而对于有状态的连接，一次握手完成，整个连接的生命期内都有效。

下面看一下具体握手过程：

1. Client发起HandshakeRequest，其中包含Client本身SchemaHash值和对应Server端的SchemaHash值，例如：（clientHash!=null,clientProtocol=null,serverHash!=null），如果本地缓存有ServerHash值则直接填充，如果没有则通过猜测填充；
2. Server用如下之一的HandshakeResponse响应Client请求：
	1. （match=BOTH,serverProtocol=null,serverHash=null）：表示当Client发送正确的ServerHash值且Server缓存中存在相应的ClientHash，则握手过程完成；
	2. （match=CLIENT,serverProtocol!=null,serverHash!=null）：表示当Server缓存有Client的Schema，但Client请求中的ServHash值不正确。Server发送Server端的Schema数据和相应的Hash值，此次握手完成。
	3. （match=NONE）：表示当Client发送的ServerHash不正确且Server端没有Client的Schema缓存。这种情况下Client需要重新提交请求信息（clientHash!=null,clientProtocol!=null,serverHash!=null），Server根据实际情况给予正确的响应，握手完成。

握手过程使用的Schema结构如下：

	{
		"type": "record",
		"name": "HandshakeRequest",
		"namespace": "org.apache.avro.ipc",
		"fields": [
			{
				"name": "clientHash",
				"type": {
					"type": "fixed",
					"name": "MD5",
					"size": 16
				}
			},
			{"name": "clientProtocol", "type": ["null", "string"]},
			{"name": "serverHash", "type": "MD5"},
			{
				"name": "meta", 
				"type": [
					"null",
					{
						"type": "map",
						"values": "bytes"
					}
				]
			}
		]
	}

	{
		"type": "record",
		"name": "HandshakeResponse",
		"namespace": "org.apache.avro.ipc",
		"fields": [
			{
				"name": "match", 
				"type": {
					"type": "enum",
					"name": "HandshakeMatch",
					"symbols": ["BOTH", "CLIENT", "NONE"]
				}
			},
			{"name": "serverProtocol", "type": ["null", "string"]},
			{
				"name": "serverHash",
				"type": {
					"type": "fixed",
					"name": "MD5",
					"size": 16
				}
			},
			{
				"name": "meta", 
				"type": [
					"null",
					{
						"type": "map",
						"values": "bytes"
					}
				]
			}
		]
	}


消息从客户端发送到服务器端需要经过传输层（Transport Layer），它发送消息并接受服务器端响应。到达传输层的数据就已经是二进制数据了，通常以HTTP作为传输模型，数据以POST方式发送给对方，在Avro中，消息被封装成一组Buffer，如下图所示：

![](http://pic.yupoo.com/kazaff_v/DSCZH27c/9Qe0j.jpg)

每个Buffer以四个字节开头，中间是多个字节的数据，最后以一个空缓冲区结尾。

协议定义
---
Avro协议描述了RPC接口，和Schema一样，用JSON定义，有以下属性：

* protocol，必需，协议名称
* namespace，可选，协议的命名空间
* doc，可选，描述这个协议的字符串
* types，可选，定义协议类型名称列表
* messages，可选，一个json对象，key是message名称，value是json对象
	* doc，可选，对消息的说明
	* request，参数列表，其中参数拥有名字和类型
	* response，响应数据的schema
	* error，可选，用一个declared union来描述，error类型的定义和record一样，除了它使用error，而不是record
	* one-way，可选，布尔类型，当response 类型是null，并且没有列出error时，one-way parameter只能是true


实例
---
可以参考这篇文章，写得非常详细：[这里](http://www.cnblogs.com/agoodegg/p/3309041.html)