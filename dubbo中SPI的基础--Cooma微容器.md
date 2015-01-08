断断续续研究[dubbo](http://alibaba.github.io/dubbo-doc-static/Home-zh.htm)其实有一段时间了，总是搁浅的原因和工作安排有很大关系，不过我始终保持着学习它的兴趣。随着[dubbox](http://dangdangdotcom.github.io/dubbox/)项目的更新，我决定再一次尝试学习这个项目。

之前把玩的时候自己甚至连一本完整的javaEE资料都没有看完过，虽然现在也还是入门级新手，不过在其他项目中的工作也让我有了比之前更多的积累，应该可以看得更透一些。

从哪里开始呢？我个人认为应该从dubbo的SPI理念开始，由于dubbo的项目Leader一开始就把其定义成一个方便扩展的服务框架，所以在dubbo的架构设计中始终保持了良好的依赖扩展机制：**微内核+插件**。简而言之就是让扩展者可以和项目开发者拥有一样的灵活度，这也是dubbo得以迅速流行的一个必要条件。

要想实现这种自由度，除了在架构分层组件上要保持高内聚低耦合外，底层也需要一套强大的类管理工具。在javaEE世界里，把这份工作做到极致的也已经有成熟的标准规范：OSGi。不过OSGi并不完全适配dubbo的需求，而且这玩意儿也有些过于重了，所以在此基础上，dubbo结合JDK标准的SPI机制设计出来一个轻量级的实现：[Cooma](https://github.com/alibaba/cooma/wiki)。

这篇文章，我就打算从Cooma说起，官方介绍的已经非常详细了，不过它在从dubbo独立出来发布之前是做过修改优化的，在dubbo项目中使用时可能会存在些许的不同，我们就从dubbo内部来研读这部分实现的代码，并结合dubbo中的上下文来了解一下dubbo是如何使用SPI的。

我们把目标定位在dubbo的这个包上：

> com.alibaba.dubbo.common.extension

看一下这个包的目录结构：

	com.alibaba.dubbo.common.extension
	 |
	 |--factory
	 |     |--AdaptiveExtensionFactory   #稍后解释
	 |     |--SpiExtensionFactory        #稍后解释
	 |
	 |--support
	 |     |--ActivateComparator
	 |
	 |--Activate  #自动激活加载扩展的注解
	 |--Adaptive  #自适应扩展点的注解
	 |--ExtensionFactory  #扩展点对象生成工厂接口
	 |--ExtensionLoader   #扩展点加载器，扩展点的查找，校验，加载等核心逻辑的实现类
	 |--SPI   #扩展点注解

我们通过对照dubbo如何使用扩展点机制来完成扩展点工厂实例的选择与加载来了解一下扩展点实现的细节，这句话很拗口，有点递归的味道，我们不妨直接从代码中来理解：

	public class ExtensionLoader<T> {
		...
		private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
		private final Class<?> type;		
		...
		@SuppressWarnings("unchecked")
	    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
	        if (type == null)
	            throw new IllegalArgumentException("Extension type == null");
	        if(!type.isInterface()) {
	            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
	        }
	        if(!withExtensionAnnotation(type)) {
	            throw new IllegalArgumentException("Extension type(" + type + 
	                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
	        }
	        
	        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
	        if (loader == null) {
	            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
	            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
	        }
	        return loader;
	    }
	
	    private ExtensionLoader(Class<?> type) {
	        this.type = type;
	        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
	    }		
		...
	}

我们主要看`ExtensionLoader`构造方法，其中它初始化了`type`和`objectFactory`，前者为要作为扩展点的接口类型，后者表示要如何获取指定名称的扩展点实例（工厂类），目前dubbo提供了2个实现类，上面在包结构图上已经标注过了。

	@SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if(createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            }
            else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }

上面这个方法是用来获取自适应扩展类实例的，但其实它只是封装了一层缓存而已，真正完成创建实例的是`createAdaptiveExtension`方法。由于调用关系太深，请看下面的图：

![](http://pic.yupoo.com/kazaff/Elc1xnHk/7eCGH.png)

上图中给出的路径，缺少了查找扩展点实现的细节，也就是并没有展开`getExtensionClasses`方法，该方法会根据指定位置的配置文件扫描并解析拿到所有可用的扩展点实现，代码如下：

	private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
	}

可见它也只是封装了一层缓存而已，我们继续深挖`loadExtensionClasses`方法：

	// 此方法已经getExtensionClasses方法同步过。
    private Map<String, Class<?>> loadExtensionClasses() {
		//检查并获取该接口类型声明的默认扩展点实现
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if(defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if(value != null && (value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if(names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if(names.length == 1) cachedDefaultName = names[0];
            }
        }
        
		//去三个指定的位置查找配置文件并解析拿到扩展点键值映射关系
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadFile(extensionClasses, DUBBO_DIRECTORY);
        loadFile(extensionClasses, SERVICES_DIRECTORY);

        return extensionClasses;
    }

我们先来看一下配置文件的格式：

	adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
	spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory

`loadFile`方法会从指定位置（`META-INF/dubbo/internal/`）根据指定接口类型（`type`）为文件名称查找目标配置文件，然后解析并校验，最终拿到匹配的扩展点类的所有`Class`实例。对应上面给出的配置文件，也就是`AdaptiveExtensionFactory`和`SpiExtensionFactory`，它们已经在包结构图上提到过了。

现在我们来着重看一下这两个类，它们到底是做什么用的呢？首先，`AdaptiveExtensionFactory`定义上有`@Adaptive`注解标识，很明显，它就是自适应扩展点的实现，从`loadFile`方法中可以留意到：**同一个接口类型只能存在一个自适应扩展点实现**：

	@Adaptive
	public class AdaptiveExtensionFactory implements ExtensionFactory {
	    
    	private final List<ExtensionFactory> factories;
    
	    public AdaptiveExtensionFactory() {
	        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
	        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
	        for (String name : loader.getSupportedExtensions()) {
	            list.add(loader.getExtension(name));
	        }
	        factories = Collections.unmodifiableList(list);
	    }

	    //从这个方法定义来看，这个自适应扩展点实现类并没有做任何事儿，唯一的工作就是把真正获取扩展点实例的逻辑依次交给
	    //框架中声明的所有ExtensionFactory扩展点实例，默认也就是SpiExtensionFactory
	    public <T> T getExtension(Class<T> type, String name) {
	        for (ExtensionFactory factory : factories) {
	            T extension = factory.getExtension(type, name);
	            if (extension != null) {
	                return extension;
	            }
	        }
	        return null;
	    }
	}

可以看到，`AdaptiveExtensionFactory`把逻辑委托给`SpiExtensionFactory`来做，而后者又是怎么做的呢：

	public class SpiExtensionFactory implements ExtensionFactory {

	    public <T> T getExtension(Class<T> type, String name) {
	        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
	            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
	            if (loader.getSupportedExtensions().size() > 0) {
					//获取自适应扩展点实例，这是dubbo默认的行为，
					//也可以自己写一个ExtensionFactory来按照要求加载扩展点
	                return loader.getAdaptiveExtension();   
	            }
	        }
	        return null;
	    }
	}

而`objectFactory`（真实工作的也就是`SpiExtensionFactory.getExtension`）只是用在ExtendLoader的注入方法（`injectExtension`）中，该方法用于为选定的扩展点实现注入相关的其他扩展点实例。

目前为止，我们已经大概了解在dubbo内部，是以什么样的规则来使用扩展点机制，也为以后学习dubbo的其它方面提供了基础。