这次我们要科普的，是Thrift，它是Facebook的一个开源项目，定位为一个跨语言的远程服务调用框架。

目前流行的服务调用方式有很多种，例如基于SOAP消息格式的Web Service，基于JSON消息格式的RESTful服务等，其中所用到的数据传输格式包括XML，JSON等，然后XML相对体积太大，传输效率底，JSON体积小，但还不够完善。针对高并发，大数据量和多语言的需求，传输二进制格式的Thrift更有优势。

Thrift有一个代码生成器来对它所定义的IDL文件自动生成服务代码框架，用户只需要在其之上进行二次开发即可，对底层的RPC通信等都是透明的。目前它支持的语言有C++，Java，Python，PHP，Ruby，Erlang，Perl，Haskell，C#，Cocoa，Smalltalk，等。（妈蛋，真强悍啊！）

现在看一个Thrift的IDL定义，Hello.thrift文件：

	namespace java service.demo
	service Hello{
		string helloString(1:string para)
		i32 helloInt(1:i32 para)
		bool helloBoolean(1:bool para)
		void helloVoid()
		string helloNull()
	}

上面的这段代码定义了了服务Hello的5个方法，每个方法包含方法名，参数表和返回类型，每个参数包含参数序号，参数类型及参数名。使用Thrift工具编译Hello.thrift，就会生成响应的Hello.java文件，该文件包含了在Hello.thrift文件中描述的服务Hello的接口定义：Hello.Iface接口，以及服务调用的底层通信细节，包括客户端的调用逻辑Hello.Client以及服务器端的处理逻辑Hello.Processor，用于构建客户端和服务器端的功能。

![](http://pic.yupoo.com/kazaff_v/DT5eHGLc/132ea.jpg)

如图所示，图中黄色部分是用户实现的业务逻辑，褐色部分是根据Thrift定义的服务接口描述文件生成的客户端和服务器端代码框架，红色部分是根据Thrift文件生成代码实现数据的读写操作。红色部分以下是Thrift的传输体系、协议以及底层I/O通信，使用Thrift可以很方便的定义一个服务并且选择不同的传输协议和传输层而不用重新生成代码。

Thrift服务器包含用于绑定协议和传输层的基础架构，它提供阻塞、非阻塞、单线程和多线程的模式运行在服务器上，可以配合服务器/容器一起运行，可以和现有的J2EE服务器/Web容器无缝的结合。

![](http://pic.yupoo.com/kazaff_v/DT5mKsO9/zTkIx.png)

该图所示是HelloServiceServer启动的过程以及服务被客户端调用时，服务器的响应过程。从图中我们可以看到，程序调用了TThreadPoolServer的serve方法后，server进入阻塞监听状态，其阻塞在TServerSocket的accept方法上。当接收到来自客户端的消息后，服务器发起一个新线程处理这个消息请求，原线程再次进入阻塞状态。在新线程中，服务器通过TBinaryProtocol协议读取消息内容，调用HelloServiceImpl的helloVoid方法，并将结果写入helloVoid_result中传回客户端。

![](http://pic.yupoo.com/kazaff_v/DT5qrcyO/VFbKS.png)

该图所示是HelloServiceClient调用服务的过程以及接收到服务器端的返回值后处理结果的过程。从图中我们可以看到，程序调用了Hello.Client的helloVoid方法，在helloVoid方法中，通过send_helloVoid方法发送对服务的调用请求，通过recv_helloVoid方法接收服务处理请求后返回的结果。

上述过程对应的代码如下：

服务器端：

	package service.server; 
	import org.apache.thrift.TProcessor; 
	import org.apache.thrift.protocol.TBinaryProtocol; 
	import org.apache.thrift.protocol.TBinaryProtocol.Factory; 
	import org.apache.thrift.server.TServer; 
	import org.apache.thrift.server.TThreadPoolServer; 
	import org.apache.thrift.transport.TServerSocket; 
	import org.apache.thrift.transport.TTransportException; 
	import service.demo.Hello; 
	import service.demo.HelloServiceImpl; 
	
	 public class HelloServiceServer { 
	    /** 
	     * 启动 Thrift 服务器
	     * @param args 
	     */ 
	    public static void main(String[] args) { 
	        try { 
	            // 设置服务端口为 7911 
	            TServerSocket serverTransport = new TServerSocket(7911); 
	            // 设置协议工厂为 TBinaryProtocol.Factory 
	            Factory proFactory = new TBinaryProtocol.Factory(); 
	            // 关联处理器与 Hello 服务的实现
	            TProcessor processor = new Hello.Processor(new HelloServiceImpl()); 
	            TServer server = new TThreadPoolServer(processor, serverTransport, 
	                    proFactory); 
	            System.out.println("Start server on port 7911..."); 
	            server.serve(); 
	        } catch (TTransportException e) { 
	            e.printStackTrace(); 
	        } 
	    } 
	 } 

客户端：

	package service.client; 
	import org.apache.thrift.TException; 
	import org.apache.thrift.protocol.TBinaryProtocol; 
	import org.apache.thrift.protocol.TProtocol; 
	import org.apache.thrift.transport.TSocket; 
	import org.apache.thrift.transport.TTransport; 
	import org.apache.thrift.transport.TTransportException; 
	import service.demo.Hello; 
	
	 public class HelloServiceClient { 
	 /** 
	     * 调用 Hello 服务
	     * @param args 
	     */ 
	    public static void main(String[] args) { 
	        try { 
	            // 设置调用的服务地址为本地，端口为 7911 
	            TTransport transport = new TSocket("localhost", 7911); 
	            transport.open(); 
	            // 设置传输协议为 TBinaryProtocol 
	            TProtocol protocol = new TBinaryProtocol(transport); 
	            Hello.Client client = new Hello.Client(protocol); 
	            // 调用服务的 helloVoid 方法
	            client.helloVoid(); 
	            transport.close(); 
	        } catch (TTransportException e) { 
	            e.printStackTrace(); 
	        } catch (TException e) { 
	            e.printStackTrace(); 
	        } 
	    } 
	 } 

HelloServiceImpl.java：

	package service.demo; 
	import org.apache.thrift.TException; 
	public class HelloServiceImpl implements Hello.Iface { 
	    @Override 
	    public boolean helloBoolean(boolean para) throws TException { 
	        return para; 
	    } 
	    @Override 
	    public int helloInt(int para) throws TException { 
	        try { 
	            Thread.sleep(20000); 
	        } catch (InterruptedException e) { 
	            e.printStackTrace(); 
	        } 
	        return para; 
	    } 
	    @Override 
	    public String helloNull() throws TException { 
	        return null; 
	    } 
	    @Override 
	    public String helloString(String para) throws TException { 
	        return para; 
	    } 
	    @Override 
	    public void helloVoid() throws TException { 
	        System.out.println("Hello World"); 
	    } 
	 } 

数据类型
---
Thrift脚本可定义的数据类型包括以下几种：

* 基本类型
	* bool：对应java的boolean，true或false
	* byte：对应java的byte，8位有符号整数
	* i16：对应java的short，16位有符号整数
	* i32：对应java的int，32位有符号整数
	* i64：对应java的long，64位有符号整数
	* double：对应java的double，64位浮点数
	* string：对应java的String，未知编码文本或二进制字符串
* 结构体类型
	* struct：定义公共的对象，对应Java的一个JavaBean，类似C语言中的结构体定义
* 容器类型
	* list：对应java的ArrayList
	* set：对应java的HashSet
	* map：对应java的HashMap
* 异常类型
	* exception：对应java的Exception
* 服务类型
	* service：对应服务的类

协议
---
Thrift可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本（text）和二进制（binary）传输协议，为节约带宽，提高传输效率，一般情况下使用二进制类型的传输协议为主。常用的协议如下：

* TBinaryProtocol：二进制编码格式传输协议
* TCompactProtocol：高效率的，密集的二进制编码格式传输协议
* TJSONProtocol：使用JSON的数据编码传输协议
* TSimpleJSONProtocol：只提供JSON只写的协议，适用于通过脚本语言解析

传输层
---
常用的传输层有：

* TSocket：使用阻塞式I/O进行传输
* TFramedTransport：使用非阻塞方式，按照块的大小进行传输，类似JAVA的NIO
* TNonblockingTransport：使用非阻塞方式，用于构建异步客户端

部署
---

![](http://pic.yupoo.com/kazaff_v/DT5yMjza/15i6nB.jpg)

从图中可以看出，客户端和服务端部署时，都需要用到公共的jar包和java文件（图中绿色区域）。

上文内容抄于[这里](http://www.ibm.com/developerworks/cn/java/j-lo-apachethrift/)。

可以看出，Thrift比Avro要简单，至少从例子的角度上来看，RPC的调用方式更直观一些~