# 缓存设计

## 1 缓存穿透

    缓存穿透是指查询**一条不存在的数据**，缓存层和存储层都不会命中。

    通过出于容错考虑，如果存储层查不到的数据，不会写入缓存层中。就会出现缓存穿透，每次请求都需要去存储层进行查询，失去了包含存储层意义。



    造成缓存穿透的原因：

- 自身业务代码或者数据出现问题

- 恶意攻击，通过大量空命中查询给存储层压力

### 1.2 缓存穿透解决方案

1.  缓存空对象

    将缓存层和存储层中都没有查询到的数据也写入缓存，一定需要设置一个合适的过期时间

```java
// 缓存空对象例子
Object get(String key){
    Object cache = redis.get(key);
    
    if (null == cache) {
        cache = mysql.get(key);
        redis.set(key,cache);
        if(null == cache){
            redis.expire(key, 300);
        }
    }
    return cache;
}
```

2.  使用布隆过滤器

## 2 缓存击穿（缓存失效）

    缓存击穿是指**大量热点数据在同一时间失效**，大量的请求同时请求失效数据，所有的请求都要去存储层查询，数据库瞬间压力增大。

### 2.1 缓存击穿解决方案

    在批量增加缓存时，需要将这一批缓存的过期时间设置为一个时间段内的不同时间。

```java
// 缓存设置过期时间例子 固定时间+随机时间
Object get(String key){
    Object cache = redis.get(key);
    
    if (null == cache) {
        cache = mysql.get(key);
        redis.set(key,cache);
        if(null == cache){
            long expireTime = 300 + Radom.nextInt(300);
            redis.expire(key, expireTime);
        }
    }
    return cache;
}
```

## 3 缓存雪崩

    缓存雪崩是指缓存层故障或者宕机，所有的请求都流向后端服务，失去了缓存层承载大量流量，后端服务也会支撑不住并发量造成宕机。

### 3.1 缓存雪崩解决方案

1.  保证缓存层服务高可用（例如使用redis cluster）

2. 使用隔离组件为后端限流熔断并降级（例如使用sentinel和hystrix）

3. 需要提前演练设定缓存雪崩预定方案
