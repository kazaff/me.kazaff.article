这次的目标是缓存，没错，绝壁常用的一个知识点，我们怎么能不了解一下它的内部实现源码呢？！

dubbo的官方描述很[简洁](http://alibaba.github.io/dubbo-doc-static/Result+Cache-zh.htm)，好的封装就是这么强大， 让你用起来丝毫不费力。我们今天就费力的看一下dubbo是如何提供cache功能的。有想直接使用的童鞋，就可以跳过下面内容直面看官方提供的[简单例子](https://github.com/alibaba/dubbo/tree/master/dubbo-test/dubbo-test-examples/src/main/java/com/alibaba/dubbo/examples/cache)。

按照SPI的要求，我们从配置文件中可以看到dubbo提供的三种缓存接口的入口：

	threadlocal=com.alibaba.dubbo.cache.support.threadlocal.ThreadLocalCacheFactory
	lru=com.alibaba.dubbo.cache.support.lru.LruCacheFactory
	jcache=com.alibaba.dubbo.cache.support.jcache.JCacheFactory

先来看一下dubbo提供的`AbstractCacheFactory`的细节：

	public abstract class AbstractCacheFactory implements CacheFactory {
    
	    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<String, Cache>();
	
	    public Cache getCache(URL url) {
	        String key = url.toFullString();
	        Cache cache = caches.get(key);
	        if (cache == null) {
	            caches.put(key, createCache(url));
	            cache = caches.get(key);
	        }
	        return cache;
	    }
	
	    protected abstract Cache createCache(URL url);
	
	}

很直观的看得出，该类完成了具体cache实现的实例化工作（注意`getCache`的返回类型Cache，该接口规范了不同缓存的实现），接下来我们就分三部分来具体看一下不同的缓存接口的具体实现。




ThreadLocal
---

如果你的配置如下：
	
	<dubbo:reference interface="com.foo.BarService" cache="threadlocal" />

那就表明你使用的是该类型的缓存，根据SPI机制，会执行下面这个工厂类：

	public class ThreadLocalCacheFactory extends AbstractCacheFactory {

	    protected Cache createCache(URL url) {
	        return new ThreadLocalCache(url);
	    }
	}

注意该类继承了上面提到的`AbstractCacheFactory`。可以看出，真正实例化的具体缓存层实现是`ThreadLocalCache`类型。由于此类型是基于线程本地变量的，所以非常简单：

	public class ThreadLocalCache implements Cache {

	    private final ThreadLocal<Map<Object, Object>> store;
	
	    public ThreadLocalCache(URL url) {
	        this.store = new ThreadLocal<Map<Object, Object>>() {
	            @Override
	            protected Map<Object, Object> initialValue() {
	                return new HashMap<Object, Object>();
	            }
	        };
	    }
	
	    public void put(Object key, Object value) {
	        store.get().put(key, value);
	    }
	
	    public Object get(Object key) {
	        return store.get().get(key);
	    }
	}

这里注意的是，为了遵循接口定义才需要初始化时传入`url`参数，但其实该类型的缓存实现是完全不需要额外参数的。

最后要叮嘱的是，该缓存应用场景为：

> 比如一个页面渲染，用到很多portal，每个portal都要去查用户信息，通过线程缓存，可以减少这种多余访问。

场景描述的核心内容是**当前请求的上下文**，可以结合dubbo的线程模型来更好的消化这一点。也许我们以后还会单独来分析这个主题。




LRU
---

类似ThreadLocal，我们就不再重复列举对应的工厂方法了，直接看`LruCache`类的实现：

	public class LruCache implements Cache {
	    
	    private final Map<Object, Object> store;
	
	    public LruCache(URL url) {
	        final int max = url.getParameter("cache.size", 1000);   //定义了缓存的容量
	        this.store = new LinkedHashMap<Object, Object>() {
	            private static final long serialVersionUID = -3834209229668463829L;
	            @Override
	            protected boolean removeEldestEntry(Entry<Object, Object> eldest) { //jdk提供的接口，用于移除最旧条目的需求
	                return size() > max;
	            }
	        };
	    }
	
	    public void put(Object key, Object value) {
	        synchronized (store) {  //注意这里的同步条件
	            store.put(key, value);
	        }
	    }
	
	    public Object get(Object key) {
	        synchronized (store) {  //注意这里的同步条件
	            return store.get(key);
	        }
	    }
	}

相比ThreadLocal，可以看出，该类型的缓存是跨线程的，也匹配我们常见的缓存场景。





JCache
---

对于我这种java新手，什么是JCache，显然需要科普一下，这里给出了我找到的几篇不错的文章：[官府](http://docs.oracle.com/middleware/1213/coherence/tutorial/jcache.htm#COHTU1006)，[草根](http://stackoverflow.com/questions/25506110/memcached-vs-memcache-vs-jcache)，[小栗子](http://java.dzone.com/articles/introduction-jcache-jsr-107)，[注解篇](https://spring.io/blog/2014/04/14/cache-abstraction-jcache-jsr-107-annotations-support)，[中文完美篇](http://jinnianshilongnian.iteye.com/blog/2001040)。由于内容太多，我就不胡乱翻译了~~

由于这部分的代码太简单，节省篇幅就不列源码了。不过我们的项目缓存是基于redis的，而我并没有找到支持JCache的redis客户端，不知道大家有没有推荐的啊~？？




如何解析“cache”属性
---

那么，cache层的逻辑是如何一步一步“注入”到我们的业务逻辑里呢？这还是要追溯到dubbo的[过滤器](http://blog.kazaff.me/2015/02/06/dubbo%E7%9A%84%E6%8B%A6%E6%88%AA%E5%99%A8%E5%92%8C%E7%9B%91%E5%90%AC%E5%99%A8/)上，我们知道在dubbo初始化指定protocol的时候，会使用装饰器模式把所有需要加载的过滤器封装到目标protocol上，这个细节指引我来查看`ProtocolFilterWrapper`类：

	refer() --->  buildInvokerChain（）
							|
							V

	private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (filters.size() > 0) {
            for (int i = filters.size() - 1; i >= 0; i --) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    public Result invoke(Invocation invocation) throws RpcException {
                        return filter.invoke(next, invocation);
                    }

                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }


注意`ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);`这一行，单步调试可以得知它会返回所有需要“注入”的Filter逻辑，当然也包含我们关注的缓存：`com.alibaba.dubbo.cache.filter.CacheFilter`。

注意看该类声明的开头：

	@Activate(group = {Constants.CONSUMER, Constants.PROVIDER}, value = Constants.CACHE_KEY)

这一行是关键哟，上面提到的`getActivateExtension`方法就是靠这一行注解工作的。dubbo以这种设计风格完成了大多数的功能，所以对于研究dubbo源码的童鞋，一定要多多注意。

经历了这一圈下来，所有过滤器就已经注入到我们的服务当中了。





业务层如何使用cache
---

最后再来仔细看一下`com.alibaba.dubbo.cache.filter.CacheFilter`类的invoke方法：

	public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (cacheFactory != null && ConfigUtils.isNotEmpty(invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.CACHE_KEY))) {
            Cache cache = cacheFactory.getCache(invoker.getUrl().addParameter(Constants.METHOD_KEY, invocation.getMethodName()));
            if (cache != null) {
                String key = StringUtils.toArgumentString(invocation.getArguments());
                if (cache != null && key != null) {
                    Object value = cache.get(key);
                    if (value != null) {
                        return new RpcResult(value);
                    }
                    Result result = invoker.invoke(invocation);
                    if (! result.hasException()) {
                        cache.put(key, result.getValue());
                    }
                    return result;
                }
            }
        }
        return invoker.invoke(invocation);
    }

可以看出，这里根据不同的配置会初始化并使用不同的缓存实现，好了，关于缓存的分析就到此为止。