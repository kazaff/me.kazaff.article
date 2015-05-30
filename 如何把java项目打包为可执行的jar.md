这几天用java写了一个后台服务，就是把office文件转换成pdf。实现挺简单的，直接调用openoffice服务即可，大致做法网上有很多教程，这里就不多扯了，今天主要是说一下项目打包的事儿。


用Idea打包
---

前段时间是直接用IDE提供的支持把java Web项目打包成war。一切都是那么的顺利，可现在想要的是直接把项目打包成jar，而且需要的是直接在终端中输入`java -jar xxxx.jar`即可运行！

一开始也打算尝试使用IDE提供的打包方式，可总是失败，jar是生成了，不过运行时总提示类缺失，明明已经把所有依赖的第三方jar包都copy了一份，可死活就是找不到，十分怀疑是`MANIFEST.MF`文件的配置问题。

总之使用了几种网上找的方式都出现各种问题，所以暂时放弃了Idea打包jar的想法，有经验的童鞋可以来[这里](http://segmentfault.com/q/1010000002429996)捧场，一经测试，立马送分哟~。


用Maven打包
---

要说吧，这算是大家最认可的方式，不过maven打包也有好多种插件，这让我这种新手很是头疼。

**方案一：**

	<plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>2.2</version>
        <configuration>
            <archive>
                <manifest>
                    <mainClass>me.kazaff.Main</mainClass>
                </manifest>
            </archive>
            <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>
        </configuration>
        <executions>
            <execution>
                <id>make-assembly</id>
                <phase>package</phase>
                <goals>
                    <goal>single</goal>
                </goals>
            </execution>
        </executions>
    </plugin>

这会将所有依赖的包解压后重新和自己的项目包合并打入jar包中，至于这么做的优劣可以留给你自己去体会，这么做的理由我猜应该和jar包查找classpatch有关系吧，否则为什么要这么奇葩呢？

**方案二：**

	<plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <configuration>
            <archive>
                <manifest>
                    <addClasspath>true</addClasspath>
                    <classpathPrefix>lib/</classpathPrefix>
                    <mainClass>me.kazaff.Main</mainClass>
                </manifest>
            </archive>
        </configuration>
    </plugin>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
            <execution>
                <id>copy</id>
                <phase>package</phase>
                <goals>
                    <goal>copy-dependencies</goal>
                </goals>
                <configuration>
                    <outputDirectory>./target/lib</outputDirectory>
                </configuration>
            </execution>
        </executions>
    </plugin>

这种方式相对方案一，似乎好了那么一些，它会把你的项目打成一个独立的jar包，并在jar包所在的目录中建立一个lib文件夹，并把所有三方依赖jar包拷贝进去，这还不算完，生成的那个jar包中的`MANIFEST.MF`会配置好`Class-Path`属性。

需要注意的是，如果你想直接在终端中运行该jar包，必须保证终端进入到项目jar包所在的目录下，否则相对路径会导致依赖的三方jar包找不到哟~~

好吧，如果你像我一样记性不好，又需要将打包的jar文件拷贝到其他机器上部署的话，很容易忘记lib文件夹！这里有一个进阶的打包方法：[传送门](http://www.xeclipse.com/?p=1470)。


**方案三：**

那有没有一种方法，可以让我们把项目真正的打包成单一可执行的jar包呢？而且这个jar包中的结构不会像方案一那样混乱，而是把所有第三方依赖包以jar包的方式存储。这难道成了奢望么？

油腻的湿姐并非我一个人在寻找：[队友](http://www.gznote.com/2014/07/maven%E4%B8%ADmaven-assembly-plugin%E7%9A%84%E4%BD%BF%E7%94%A8.html)。不过，按照他的最后一个办法打包出来的jar是无法直接执行的，因为其中`MANIFEST.MF`缺少`Main-Class`和`Class-Path`属性，他的这种方式如果是打包成一个非执行jar包的话应该还是不错的。

所以，如你所见，方案三并不存在，至少我不知道应该怎么做！有办法的朋友可以在文章下面留言，不胜感激！



java读取jar包中的资源文件
---

打包后发现原先在代码中直接通过路径读取的配置文件再也找不到了，可是打开jar包明明看得到啊！

查了一下资料，才知道到底发生了什么，看这里：[传送门](http://ppjava.com/?p=1205)，[传送门2](http://www.iteye.com/topic/483115)。

