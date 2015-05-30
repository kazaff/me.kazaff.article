妈蛋，实在是顶不住部分文章的排版，看着太让人心焦了~~所以，对一篇文章重新排版一下，希望能帮助到其他人~

先发出[原文链接](http://xinklabi.iteye.com/blog/837435)，供眼神好的童鞋~~

下面是小弟排版后的内容：

Java在运行已编译完成的类时，是通过java虚拟机来装载和执行的，java虚拟机通过操作系统命令`JAVA_HOME"bin/"java -option`来启动，`option`为虚拟机参数， `JAVA_HOME`为JDK安装路径，通过虚拟机参数可对虚拟机的运行状态进行调整，掌握参数的含义可对虚拟机的运行模式有更深入的理解。

如何查看参数列表
---
虚拟机参数分为基本和扩展两类，在命令行中输入`JAVA_HOME"bin/"java`就可得到基本的参数列表，在命令行中输入`JAVA_HOME"bin/"java -X`就可以得到扩展参数列表。

基本参数说明
---

**-client/-server**

这两个参数用于设置虚拟机使用何种运行模式，`client`模式启动比较快，但运行时性能和内存管理效率不如`server`模式，通常用于客户端应用程序。相反，`server`模式启动稍慢，但可获得更高的运行性能。

在win上，缺省的虚拟机类型为`client`模式，如果要使用`server`模式就需要在启动虚拟机时加上`-server`参数，以获得更高性能。对服务器应用，推荐采用`server`模式，尤其是多个cpu的系统。

在Linux，Solaris上缺省采用的是`server`模式。

**-hotspot**

含义与`client`相同，jdk1.4以前使用的参数，现在已经不再使用，取而代之的是`client`。

**-classpath/-cp**

虚拟机在运行一个类时需要将其装入内存，搜索类的方式和顺序如下：

1. Bootstrap classes
2. Extension Classes
3. User Classes

`Bootstrap`中的路径是虚拟机自带的jar或zip文件，虚拟机首选搜索这些包文件，用下面这种方式可得到虚拟机搜索的路径：

	System.getProperty("sun.boot.class.path")

`Extension`位于jre的`lib/ext`目录下的jar文件，虚拟机在搜索完Bootstrap后就搜索该目录下的jar文件，用下面这种方式可得到虚拟机使用的Extension搜索路径：

	System.getProperty("java.ext.dirs")

`User`类搜索顺序为：

1. `-classpath`指定的路径
2. 环境变量`CLASSPATH`
3. 当前目录

在使用`-classpath/-cp`时，多个目录之间用分号分隔。推荐使用该命令来指定虚拟机要搜索的类路径，而不是依赖环境变量，以减少多个项目同时使用环境变量时存在的潜在冲突（多版本库）。

可在运行时通过下面的代码获取虚拟机查找类的路径：

	System.getProperty("java.class.path")

**-D<propertyName>=value**

在虚拟机的系统属性中设置属性名/值对，运行在此虚拟机上的应用程序可用：

	System.getProperty("属性名")

得到value的值。

如果value中有空格，则需要用双引号将该值括起来，如：`-Dname="kazaf f"`。

该参数通常用于设置系统级全局变量值，如配置文件路径，保证该属性在程序中任何地方都可访问。


**-verbose[:class|gc|jni]**

在输出设备上显示虚拟机运行信息。 `-verbose`和`-verbose:class`含义相同，表示输出虚拟机装入的类的信息，格式如下：

	[Loaded java.io.FilePermission$1 from shared objects file]

当虚拟机报告类找不到或类冲突时，用此参数来查看虚拟机装入类的情况。

`-verbose:gc`用于在虚拟机发生内存回收时在输出设备上显示信息，格式如下：

	[Full GC 268K->168K(1984K), 0.0187390 secs]

`-verbose:jni`用于在虚拟机调用native方法时在设备上输出显示信息，格式如下：
	
	[Dynamic-linking native method HelloNative.sum ... JNI]

该参数用于监视虚拟机调用本地方法的情况，在发生jni错误时可以为诊断提供便利。

**-ea[:<packagename>...|:<classname>]/-enableassertions[:<packagename>...|:<classname>]**

从jdk1.4开始，java可支持断言机制，用于诊断运行时问题。通常在测试阶段使断言有效，在线上运行时不需要运行断言。断言后的表达式的值是一个逻辑值，为`true`时断言不运行，为`false`时断言抛出`java.lang.AssertionError`错误。

这个参数就是用来设置虚拟机是否启动断言机制，缺省时虚拟机关闭断言机制，用`-ea`可打开断言机制，不加`packagename`和`classname`表示运行所有包和类中的断言，如果希望只是运行某些包或类中的断言，可将包名或类名加到`-ea`之后，例如：`-ea:com.foo.util`。

**-da[:<packagename>...|:<classname>]/-disableassertions[:<packagename>...|:<classname>]**
用来设置虚拟机关闭断言处理，用法与`-ea`类似。

**-esa/-enablessystemassertions**

设置虚拟机开启系统类的断言。

**-dsa/-disablesystemassertions**

设置虚拟机关闭系统类的断言。

**-agentlib:<libname>[=<options>]**

该参数是jdk5新引入的，用于虚拟机装载本地代理库。其中`libname`为本地代理库文件名，虚拟机的搜索路径为环境变量`path`中的路径，`options`为传给本地库启动时的参数，多个参数之间用逗号分隔。

在win平台中，虚拟机搜索本地库后缀名名为`.dll`，在Unix上则为`.so`文件。例如可以用`-agentlib:hprof`来获取虚拟机的运行情况。可用`-agentlib:hprof=help`来得到使用帮助列表（确保在win平台下的jre的`lib`目录下存在`hprof.dll`文件）。

**-agentpath:<jarpath>[=<options>]**

设置虚拟机按全路径装载本地库，不再搜索`PATH`中的路径，其他功能同`-agentlib`。

**-javaagent:<jarpath>[=<options>]**

虚拟机启动时装入java语言设备代理。`jarpath`文件中的`mainfest`文件必须有`Agent-Class`属性。代理类要实现

	public static void premain(String agentArgs, Instrumentation inst)

方法。当虚拟机初始化时，将按照代理类的说明顺序调用`premain`方法。

扩展参数说明
---

**-Xmixed**

设置`-client`模式虚拟机对使用频率高的方法进行`Just-In-Time`编译和执行，对其他方法使用解释方式执行，该方式是虚拟机缺省模式。

**-Xint**

设置`-client`模式下运行的虚拟机以解释方式执行类的字节码，不将字节码编译为本机码。

**-Xbootclasspath[/a|/p]:path**

改变虚拟机装载缺省系统运行包`rt.jar`的路径，从`-Xbootclasspath`中设定的搜索路径中装载运行类。除非你自己能写一个运行时，否则不会用到这个参数。

其中`/a`将在缺省搜索路径后加上path中的搜索路径，而`/p`在缺省路径前先搜索path中的路径。

**-Xnoclassgc**

关闭虚拟机对class的垃圾回收功能。

**-Xincgc**

启动增量垃圾收集器，缺省是关闭的。增量垃圾收集器能减少偶尔发生的长时间的垃圾回收造成的暂停时间。但增量垃圾收集器和应用程序并发执行，因此会占用部分CPU在应用程序上的功能。

**-Xloggc:<file>**

将虚拟机每次垃圾回收的信息写到日志文件中，文件名由`file`指定，内容和`-verbose:gc`输出内容相同。

**-Xbatch**

虚拟机的缺省运行方式是在后台编译类代码，然后在前台执行代码，使用该参数将关闭虚拟机后台编译，在前台编译完成后再执行。

**-Xms<size>**

设置虚拟机可用内存堆的初始大小，缺省单位为字节。该大小为1024的整数倍并且要大于1MB，可用k(K)或m(M)为单位来设置较大的内存数。初始堆的大小为2MB。例如：`-Xms6400K`、`-Xms256M`

**-Xmx<size>**

设置虚拟机内存堆的最大可用大小，缺省单位为字节，缺省堆最大值为64MB。该值必须为1024整数倍并且要大于2MB，可用k(K)或m(M)为单位来设置较大的内存数。

当应用程序申请了大内存，运行时虚拟机抛出`java.lang.OutOfMemoryError: Java heap space`错误，就需要用改参数设置更大的内存数。

**-Xss<size>**

设置线程栈的大小，缺省单位为字节，通常操作系统分配给线程栈的缺省大小为1MB。另外，也可以在java中创建线程对象时设置栈的大小，构造函数原型为：

	Thread(ThreadGroup group, Runnable target, String name, long stackSzie)

**-Xprof**

输出CPU运行时的诊断信息。

**-Xfuture**

对类文件进行严格格式检查，以保证类代码符合类代码规范。为保证向后兼容，虚拟机缺省不进行严格的格式检查。

**-Xrs**

减少虚拟机中操作系统的信号的使用。该参数通常在虚拟机以后台服务方式运行时使用（如Serlet）。

**-Xcheck:jni**

对jni函数执行检查。

**-Xshare:off**

不尝试使用共享类的数据

**-Xshare:auto**

在可能的情况下使用共享类的数据，默认值。

**-Xshare:on**

要求使用共享类的数据，否则运行失败。