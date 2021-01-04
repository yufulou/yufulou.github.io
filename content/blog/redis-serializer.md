+++
title= "jackson做redis反序列化时，反序列化Map<Integer, T>时的问题"
date= 2021-01-04T14:14:38+08:00
categories = ["java", "jackson", "serialize"]
description = ""
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
type = "post"
+++

类似这个帖子提出的问题：https://stackoverflow.com/questions/55278446/redis-caching-mapinteger-string

即redis中事实上存储的都是字符串，包括key，但又需要反序列化成Map<Integer, String>类似的类型时，虽然能反序列化成功，但发现key的部分实际存储的还是字符串。

文中提出了如下解决办法：

![image](/img/blog/redis-serializer/solving-jackson-redis-serializer.png)

但这个办法有个问题，如果是Integer为Key，Value为其他类型还是会有问题。

故通过Class.getGenericSuperclass重新做了动态版的Serializer（参考：https://blog.csdn.net/xuhailiang0816/article/details/78403041）：
```java
public class CustomJsonSerializer extends Jackson2JsonRedisSerializer {

    public CustomJsonSerializer(Class type) {
        super(type);
    }

    @Override
    protected JavaType getJavaType(Class clazz) {
        if (List.class.isAssignableFrom(clazz)) {
            Type[] typeList = ((ParameterizedType)clazz.getGenericSuperclass()).getActualTypeArguments();
            if (typeList.length > 0) {
                Class clazzT = (Class)typeList[0];
                return TypeFactory.defaultInstance().constructCollectionType(List.class, clazzT);
            }
        } else if (Map.class.isAssignableFrom(clazz)) {
            Type[] typeList = ((ParameterizedType)clazz.getGenericSuperclass()).getActualTypeArguments();
            if (typeList.length > 1) {
                Class clazzKey = (Class)typeList[0];
                Class clazzVal = (Class)typeList[1];
                return TypeFactory.defaultInstance().constructMapType(Map.class, clazzKey, clazzVal);
            }
        }
        // 非map和list类型，返回默认构造器即可
        return TypeFactory.defaultInstance().constructType(clazz);
    }
}
```

然后在定义CacheManager的时候引入这个类(我还增加了key对应ttl的设置)：
```java
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {
    @Bean
    @SuppressWarnings(value = {"unchecked", "rawtypes"})
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = getCustomSerializer(Object.class);

        template.setValueSerializer(jackson2JsonRedisSerializer);
        // 使用StringRedisSerializer来序列化和反序列化redis的key值
        template.setKeySerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }

    /**
     * 指定 key 策略
     * 
     * @param redisConnectionFactory
     * @return
     */
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        return new RedisCacheManager(RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory),
            getRedisCacheConfigurationWithTtl(60, Object.class), getRedisCacheConfigurationMap());
    }

    private Map<String, RedisCacheConfiguration> getRedisCacheConfigurationMap() {
        Map<String, RedisCacheConfiguration> redisCacheConfigurationMap = new HashMap<>();
        for (CacheTtl ttlItem : CacheTtl.values()) {
            // 根据配置，对每个key设置指定的反序列化器和超时时间
            redisCacheConfigurationMap.put(ttlItem.getName(),
                getRedisCacheConfigurationWithTtl(ttlItem.getTtl(), ttlItem.getClazz()));
        }
        return redisCacheConfigurationMap;
    }

    private RedisCacheConfiguration getRedisCacheConfigurationWithTtl(Integer seconds, Class objectClass) {
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
        redisCacheConfiguration = redisCacheConfiguration
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(getCustomSerializer(objectClass)))
            .entryTtl(Duration.ofSeconds(seconds));

        return redisCacheConfiguration;
    }

    private CustomJsonSerializer getCustomSerializer(Class objectClass) {
        CustomJsonSerializer customSerializer = new CustomJsonSerializer(objectClass);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        customSerializer.setObjectMapper(om);
        return customSerializer;
    }

}
```

这里配置key对应的超时时间和反序列化类型，如果非需要特殊处理的Integer为key的Map，实际上配置Object.class即可，如果同时不需要超时时间，这里可以不做相应配置，会直接使用默认缓存超时时间，超时时间的配置也是为了集中管理
```java
@Getter
public enum CacheTtl {
    // menuId和appId的对应关系
    MENU_APP_MAP("menuAppMapCache", 60 * 30, Object.class),
    FULL_ROUTE_MAP("fullRouteMap", 60 * 60, (new HashMap<Integer, AuthServiceRoute>() {
    }).getClass());

    private final String name;
    private final Integer ttl;
    private final Class<?> clazz;

    CacheTtl(String name, Integer ttl, Class<?> clazz) {
        this.name = name;
        this.ttl = ttl;
        this.clazz = clazz;
    }
}
```

然后配置一个Constants：
```java
public class CacheConstants {

    public final static String MENU_APP_MAP_PREFIX = "menuAppMapCache";
    public final static String ROUTE_MAP_KEY = "fullRouteMap";

}
```


之后代码中再用到类似需要缓存的场景时，就可以这么用了：
```java
@Cacheable(CacheConstants.ROUTE_MAP_KEY)
public Map<Integer, AuthServiceRoute> getAuthServiceRouteMap() {
}
```

关于ttl的设置，还有个用spEL分析字符串的方案，还附带自动刷新缓存的功能，相当强大：https://www.jianshu.com/p/275cb42080d9