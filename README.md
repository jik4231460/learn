JetCache主要通过@Cached和@CreateCache实现缓存，@Cached是在接口方法或者类方法上添加缓存，一般以参数为key，以返回值为value存入缓存中。@CreateCache是直接创建一个缓存实例，然后调用put(T key， T value)、get(T key)等方法实现缓存。

（1）如果是SpringBoot框架开发：

pom文件：

<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-starter-redis</artifactId>
    <version>2.5.6</version>
</dependency>
application.yml文件：

jetcache:
  statIntervalMinutes: 15
  areaInCacheName: false
  local:
    default:
      type: linkedhashmap
      keyConvertor: fastjson
  remote:
    default:
      type: redis
      keyConvertor: fastjson
      valueEncoder: java
      valueDecoder: java
      poolConfig:
        minIdle: 5
        maxIdle: 20
        maxTotal: 50
      host: 127.0.0.1
      port: 6379
启动类：

EnableMethodCache，EnableCreateCacheAnnotation这两个注解分别激活Cached和CreateCache注解，其他和标准的Spring Boot程序是一样的。这个类可以直接main方法运行。

package com.company.mypackage;
 
import com.alicp.jetcache.anno.config.EnableCreateCacheAnnotation;
import com.alicp.jetcache.anno.config.EnableMethodCache;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
 
@SpringBootApplication
@EnableMethodCache(basePackages = "com.company.mypackage")  //激活@Cached
@EnableCreateCacheAnnotation       //激活@CreateCache
public class MySpringBootApp { 
    public static void main(String[] args) {
        SpringApplication.run(MySpringBootApp.class);
    }
}
（2）如果没有使用SpringBoot：

pom文件：

<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-anno</artifactId>
    <version>2.5.6</version>
</dependency>
<dependency>
    <groupId>com.alicp.jetcache</groupId>
    <artifactId>jetcache-redis</artifactId>
    <version>2.5.6</version>
</dependency>
配置JetCacheConfig类，激活@CreateCache和@Cached注解。

package com.company.mypackage;
 
import java.util.HashMap;
import java.util.Map;
 
import com.alicp.jetcache.anno.CacheConsts;
import com.alicp.jetcache.anno.config.EnableCreateCacheAnnotation;
import com.alicp.jetcache.anno.config.EnableMethodCache;
import com.alicp.jetcache.anno.support.GlobalCacheConfig;
import com.alicp.jetcache.anno.support.SpringConfigProvider;
import com.alicp.jetcache.embedded.EmbeddedCacheBuilder;
import com.alicp.jetcache.embedded.LinkedHashMapCacheBuilder;
import com.alicp.jetcache.redis.RedisCacheBuilder;
import com.alicp.jetcache.support.FastjsonKeyConvertor;
import com.alicp.jetcache.support.JavaValueDecoder;
import com.alicp.jetcache.support.JavaValueEncoder;
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.util.Pool;
 
@Configuration
@EnableMethodCache(basePackages = "com.company.mypackage")
@EnableCreateCacheAnnotation
public class JetCacheConfig {
 
    @Bean
    public Pool<Jedis> pool(){
        GenericObjectPoolConfig pc = new GenericObjectPoolConfig();
        pc.setMinIdle(2);
        pc.setMaxIdle(10);
        pc.setMaxTotal(10);
        return new JedisPool(pc, "localhost", 6379);
    }
 
    @Bean
    public SpringConfigProvider springConfigProvider() {
        return new SpringConfigProvider();
    }
 
    @Bean
    public GlobalCacheConfig config(SpringConfigProvider configProvider, Pool<Jedis> pool){
        Map localBuilders = new HashMap();
        EmbeddedCacheBuilder localBuilder = LinkedHashMapCacheBuilder
                .createLinkedHashMapCacheBuilder()
                .keyConvertor(FastjsonKeyConvertor.INSTANCE);
        localBuilders.put(CacheConsts.DEFAULT_AREA, localBuilder);
 
        Map remoteBuilders = new HashMap();
        RedisCacheBuilder remoteCacheBuilder = RedisCacheBuilder.createRedisCacheBuilder()
                .keyConvertor(FastjsonKeyConvertor.INSTANCE)
                .valueEncoder(JavaValueEncoder.INSTANCE)
                .valueDecoder(JavaValueDecoder.INSTANCE)
                .jedisPool(pool);
        remoteBuilders.put(CacheConsts.DEFAULT_AREA, remoteCacheBuilder);
 
        GlobalCacheConfig globalCacheConfig = new GlobalCacheConfig();
        globalCacheConfig.setConfigProvider(configProvider);
        globalCacheConfig.setLocalCacheBuilders(localBuilders);
        globalCacheConfig.setRemoteCacheBuilders(remoteBuilders);
        globalCacheConfig.setStatIntervalMinutes(15);
        globalCacheConfig.setAreaInCacheName(false);
 
        return globalCacheConfig;
    }
}
方法一：创建缓存实例

通过@Cached直接创建一个缓存实例，默认超时100s

@CreateCache(expire = 100)
private Cache<Long, UserDO> userCache;
调用api方法实现缓存：

V get(K key)
void put(K key, V value);
boolean putIfAbsent(K key, V value); //多级缓存MultiLevelCache不支持此方法
boolean remove(K key);
<T> T unwrap(Class<T> clazz);//2.2版本前，多级缓存MultiLevelCache不支持此方法
Map<K,V> getAll(Set<? extends K> keys);
void putAll(Map<? extends K,? extends V> map);
void removeAll(Set<? extends K> keys);
这些方法和JSR107的javax.cache.Cache接口一致，下面是jetCache特有的API：

V computeIfAbsent(K key, Function<K, V> loader)
当key对应的缓存不存在时，使用loader加载。通过这种方式，loader的加载时间可以被统计到。

V computeIfAbsent(K key, Function<K, V> loader, boolean cacheNullWhenLoaderReturnNull)
当key对应的缓存不存在时，使用loader加载。cacheNullWhenLoaderReturnNull参数指定了当loader加载出来时null值的时候，是否要进行缓存（有时候即使是null值也是通过很繁重的查询才得到的，需要缓存）。

V computeIfAbsent(K key, Function<K, V> loader, boolean cacheNullWhenLoaderReturnNull, long expire, TimeUnit timeUnit)
当key对应的缓存不存在时，使用loader加载。cacheNullWhenLoaderReturnNull参数指定了当loader加载出来时null值的时候，是否要进行缓存（有时候即使是null值也是通过很繁重的查询才得到的，需要缓存）。expire和timeUnit指定了缓存的超时时间，会覆盖缓存的默认超时时间。

void put(K key, V value, long expire, TimeUnit timeUnit)
put操作，expire和timeUnit指定了缓存的超时时间，会覆盖缓存的默认超时时间。

AutoReleaseLock tryLock(K key, long expire, TimeUnit timeUnit)
boolean tryLockAndRun(K key, long expire, TimeUnit timeUnit, Runnable action)
非堵塞的尝试获取一个锁，如果对应的key还没有锁，返回一个AutoReleaseLock，否则立即返回空。如果Cache实例是本地的，它是一个本地锁，在本JVM中有效；如果是redis等远程缓存，它是一个不十分严格的分布式锁。锁的超时时间由expire和timeUnit指定。多级缓存的情况会使用最后一级做tryLock操作。用法如下：

  // 使用try-with-resource方式，可以自动释放锁
  try(AutoReleaseLock lock = cache.tryLock("MyKey",100, TimeUnit.SECONDS)){
     if(lock != null){
        // do something
     }
  }
上面的代码有个潜在的坑是忘记判断if(lock!=null)，所以一般可以直接用tryLockAndRun更加简单

  boolean hasRun = tryLockAndRun("MyKey",100, TimeUnit.SECONDS), () -> {
    // do something
  };
tryLock内部会在访问远程缓存失败时重试，会自动释放，而且不会释放不属于自己的锁，比你自己做这些要简单。当然，基于远程缓存实现的任何分布式锁都不会是严格的分布式锁，不能和基于ZooKeeper或Consul做的锁相比。

还有大写的API：

V get(K key)这样的方法虽然用起来方便，但有功能上的缺陷，当get返回null的时候，无法断定是对应的key不存在，还是访问缓存发生了异常，所以JetCache针对部分操作提供了另外一套API，提供了完整的返回值，包括：

CacheGetResult<V> GET(K key);
MultiGetResult<K, V> GET_ALL(Set<? extends K> keys);
CacheResult PUT(K key, V value);
CacheResult PUT(K key, V value, long expireAfterWrite, TimeUnit timeUnit);
CacheResult PUT_ALL(Map<? extends K, ? extends V> map);
CacheResult PUT_ALL(Map<? extends K, ? extends V> map, long expireAfterWrite, TimeUnit timeUnit);
CacheResult REMOVE(K key);
CacheResult REMOVE_ALL(Set<? extends K> keys);
CacheResult PUT_IF_ABSENT(K key, V value, long expireAfterWrite, TimeUnit timeUnit);
这些方法的特征是方法名为大写，与小写的普通方法对应，提供了完整的返回值，用起来也稍微繁琐一些。例如：

CacheGetResult<OrderDO> r = cache.GET(orderId);
if( r.isSuccess() ){
    OrderDO order = r.getValue();
} else if (r.getResultCode() == CacheResultCode.NOT_EXISTS) {
    System.out.println("cache miss:" + orderId);
} else if(r.getResultCode() == CacheResultCode.EXPIRED) {
    System.out.println("cache expired:" + orderId));
} else {
    System.out.println("cache get error:" + orderId);
}
属性值说明：

属性	默认值	说明
area	“default”	如果需要连接多个缓存系统，可在配置多个cache area，这个属性指定要使用的那个area的name
name	未定义	指定缓存的名称，不是必须的，如果没有指定，会使用类名+方法名。name会被用于远程缓存的key前缀。另外在统计中，一个简短有意义的名字会提高可读性。如果两个@CreateCache的name和area相同，它们会指向同一个Cache实例
expire	未定义	该Cache实例的默认超时时间定义，注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取无穷大
timeUnit	TimeUnit.SECONDS	指定expire的单位
cacheType	CacheType.REMOTE	缓存的类型，包括CacheType.REMOTE、CacheType.LOCAL、CacheType.BOTH。如果定义为BOTH，会使用LOCAL和REMOTE组合成两级缓存
localLimit	未定义	如果cacheType为CacheType.LOCAL或CacheType.BOTH，这个参数指定本地缓存的最大元素数量，以控制内存占用。注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取100
serialPolicy	未定义	如果cacheType为CacheType.REMOTE或CacheType.BOTH，指定远程缓存的序列化方式。JetCache内置的可选值为SerialPolicy.JAVA和SerialPolicy.KRYO。注解上没有定义的时候会使用全局配置，如果此时全局配置也没有定义，则取SerialPolicy.JAVA
keyConvertor	未定义	指定KEY的转换方式，用于将复杂的KEY类型转换为缓存实现可以接受的类型，JetCache内置的可选值为KeyConvertor.FASTJSON和KeyConvertor.NONE。NONE表示不转换，FASTJSON通过fastjson将复杂对象KEY转换成String。如果注解上没有定义，则使用全局配置。
方法二：创建方法缓存

使用@Cached方法可以为一个方法添加上缓存，@CacheUpdate用于更新缓存，@CacheInvalidate用于移除缓存元素。JetCache通过Spring AOP生成代理，来支持缓存功能。注解可以加在接口方法上也可以加在类方法上，但需要保证是个Spring bean。

@Cached：系统调用该接口方法时检测到@Cached标签，首先会根据key去调用get方法获取value值，如果存在value值则直接将值返回，如果不存在key，则会执行代码查询结果，并自动调用get方法将返回值存入缓存中。

public interface UserService {
    @Cached(name="userCache.", key="#userId", expire = 3600)
    User getUserById(long userId);

    @CacheUpdate(name="userCache.", key="#user.userId", value="#user")
    void updateUser(User user);

    @CacheInvalidate(name="userCache.", key="#userId")
    void deleteUser(long userId);
}
key使用Spring的SpEL脚本来指定。如果要使用参数名（比如这里的key="#userId"），项目编译设置target必须为1.8格式，并且指定javac的-parameters参数，否则就要使用key="args[0]"这样按下标访问的形式。

@CacheUpdate和@CacheInvalidate的name和area属性必须和@Cached相同，name属性还会用做cache的key前缀。

@Cached注解和@CreateCache的属性非常类似，但是多几个：

属性	默认值	说明
area	“default”	如果在配置中配置了多个缓存area，在这里指定使用哪个area
name	未定义	指定缓存的唯一名称，不是必须的，如果没有指定，会使用类名+方法名。name会被用于远程缓存的key前缀。另外在统计中，一个简短有意义的名字会提高可读性。
key	未定义	使用SpEL指定key，如果没有指定会根据所有参数自动生成。
expire	未定义	超时时间。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为无穷大
timeUnit	TimeUnit.SECONDS	指定expire的单位
cacheType	CacheType.REMOTE	缓存的类型，包括CacheType.REMOTE、CacheType.LOCAL、CacheType.BOTH。如果定义为BOTH，会使用LOCAL和REMOTE组合成两级缓存
localLimit	未定义	如果cacheType为LOCAL或BOTH，这个参数指定本地缓存的最大元素数量，以控制内存占用。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为100
localExpire	未定义	仅当cacheType为BOTH时适用，为内存中的Cache指定一个不一样的超时时间，通常应该小于expire
serialPolicy	未定义	指定远程缓存的序列化方式。可选值为SerialPolicy.JAVA和SerialPolicy.KRYO。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为SerialPolicy.JAVA
keyConvertor	未定义	指定KEY的转换方式，用于将复杂的KEY类型转换为缓存实现可以接受的类型，当前支持KeyConvertor.FASTJSON和KeyConvertor.NONE。NONE表示不转换，FASTJSON可以将复杂对象KEY转换成String。如果注解上没有定义，会使用全局配置。
enabled	true	是否激活缓存。例如某个dao方法上加缓存注解，由于某些调用场景下不能有缓存，所以可以设置enabled为false，正常调用不会使用缓存，在需要的地方可使用CacheContext.enableCache在回调中激活缓存，缓存激活的标记在ThreadLocal上，该标记被设置后，所有enable=false的缓存都被激活
cacheNullValue	false	当方法返回值为null的时候是否要缓存
condition	未定义	使用SpEL指定条件，如果表达式返回true的时候才去缓存中查询
postCondition	未定义	使用SpEL指定条件，如果表达式返回true的时候才更新缓存，该评估在方法执行后进行，因此可以访问到#result
@CacheInvalidate注解说明：

属性	默认值	说明
area	“default”	如果在配置中配置了多个缓存area，在这里指定使用哪个area，指向对应的@Cached定义。
name	未定义	指定缓存的唯一名称，指向对应的@Cached定义。
key	未定义	使用SpEL指定key
condition	未定义	使用SpEL指定条件，如果表达式返回true才执行删除，可访问方法结果#result
@CacheUpdate注解说明：

属性	默认值	说明
area	“default”	如果在配置中配置了多个缓存area，在这里指定使用哪个area，指向对应的@Cached定义。
name	未定义	指定缓存的唯一名称，指向对应的@Cached定义。
key	未定义	使用SpEL指定key
value	未定义	使用SpEL指定value
condition	未定义	使用SpEL指定条件，如果表达式返回true才执行更新，可访问方法结果#result
使用@CacheUpdate和@CacheInvalidate的时候，相关的缓存操作可能会失败（比如网络IO错误），所以指定缓存的超时时间是非常重要的。

@CacheRefresh注解说明：

属性	默认值	说明
refresh	未定义	刷新间隔
timeUnit	TimeUnit.SECONDS	时间单位
stopRefreshAfterLastAccess	未定义	指定该key多长时间没有访问就停止刷新，如果不指定会一直刷新
refreshLockTimeout	60秒	类型为BOTH/REMOTE的缓存刷新时，同时只会有一台服务器在刷新，这台服务器会在远程缓存放置一个分布式锁，此配置指定该锁的超时时间
@CachePenetrationProtect注解：

当缓存访问未命中的情况下，对并发进行的加载行为进行保护。 当前版本实现的是单JVM内的保护，即同一个JVM中同一个key只有一个线程去加载，其它线程等待结果。
————————————————
版权声明：本文为CSDN博主「伪学霸1」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/m0_37637141/article/details/82417230
