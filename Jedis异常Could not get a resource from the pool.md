今天用`jmeter`对项目中的相关接口进行压力测试，发现一个问题：

> 严重: Servlet.service() for servlet [mvc-dispatcher] in context with path [] threw exception
java.io.IOException: org.springframework.web.util.NestedServletException: Request processing failed; nested exception is org.springframework.data.redis.RedisConnectionFailureException: Cannot get Jedis connection; nested exception is redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool

字面意思来看，应该是redis连接池没有可申请的连接了。当然，在低并发测试时是不会看到这个报错的，大概在我把并发数调到200左右时会狂报这个错误！

根据网上前辈们的总结，无非就是调整`Jedis`连接池的配置，redis本身的连接数的配置等等。这里我就不再复述了，我只想说，按照这些指示做了一边后问题并没任何好转。

我用的是工作机，操作系统是WIN7 64位，redis用的是2.8.3 win64版，jedis的版本是2.6.1（网上说2.1.0之前的版本要手动归还连接），应用中操作redis的api是依赖spring-data封装的，配置如下：

	<!-- 会话持久层redis配置 -->
    <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxIdle" value="${redis.maxIdle}" />
        <property name="maxTotal" value="${redis.maxTotal}" />
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}" />
        <property name="testOnBorrow" value="${redis.testOnBorrow}" />
        <property name="testOnReturn" value="${redis.testOnReturn}" />
    </bean>

    <bean id="jedisConnFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="hostName" value="${redis.host}" />
        <property name="port" value="${redis.port}" />
        <property name="password" value="${redis.password}" />
        <property name="usePool" value="true" />
        <property name="poolConfig" ref="poolConfig" />
        <property name="database" value="${redis.db}" />
    </bean>

    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="jedisConnFactory" />
        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
        </property>
        <property name="valueSerializer">
            <bean class="org.springframework.data.redis.serializer.JdkSerializationRedisSerializer" />
        </property>
    </bean>

redis连接池相关的参数如下：

	#redis配置
	redis.maxIdle=100
	redis.maxTotal=1000
	redis.maxWaitMillis=2000
	redis.testOnBorrow=true
	redis.testOnReturn=true

中规中矩，而且按照我配置的相关参数，设置的并发500绝逼不应该报这个错误的！其中一个细节是，当应用狂报错时，redis-cli命令也无法连接到redis服务了，也就是说redis本身的配置根本就不匹配我在应用中连接池的配置，这应该是导致问题的根本原因，简单地说就是：

> redis本身的连接数上限必须大于应用连接池中的最大连接数

可本尊看了一下redis.conf文件，其中`maxClients`的值为0，也就是不限制。这尼玛如何是好？

突然想到多年之前，小弟我刚接触redis的时候，就看到过一个大牛曾经说过：**redis的win版本基本上就是个玩具**。

抱着试一试的心态（<s>我也购买了两盒...</s>），我试着让应用直连centos中的redis服务，尼玛直接就解决了，或者说出现的情况至少和配置相吻合，而且在linux下redis-server在启动之初就会提醒你关于连接数的配置，很贴心。

好吧，最好再延伸一个老知识点，如果你想把redis的连接数设置的很大，很可能会超过目前配置的系统可使用文件句柄数上限，这时你的配置就会失效，那如何突破这个瓶颈呢，你需要了解一下这个命令： [ulimit](http://linuxguest.blog.51cto.com/195664/362366/)。

PS：redis在linux上跑起来，就像吃了枪药，打了鸡血，磕了军粮丸一样，根本停不下来啊！