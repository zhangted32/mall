# Redis 快速学习指南（面试版）

> 目标：一天内掌握 Redis 核心知识点，应对面试

---

## 1. Redis 基础

### 1.1 什么是 Redis？

- **Redis**（Remote Dictionary Server）是一个开源的内存数据库
- **特点**：
  - 内存存储，读写速度极快（10万+ QPS）
  - 支持持久化（RDB、AOF）
  - 支持多种数据结构
  - 支持事务、Lua脚本
  - 支持主从复制、哨兵、集群

### 1.2 为什么使用 Redis？

| 场景 | 说明 |
|------|------|
| **缓存** | 减少数据库压力，提高响应速度 |
| **会话管理** | 分布式Session共享 |
| **计数器** | 点赞数、访问量（原子操作） |
| **排行榜** | 利用ZSet实现 |
| **消息队列** | 利用List实现简单队列 |
| **分布式锁** | 利用SetNX实现 |

---

## 2. 数据结构

### 2.1 五大基本数据结构

| 数据结构 | 说明 | 常用命令 | 项目应用 |
|----------|------|----------|----------|
| **String** | 字符串 | SET/GET/INCR/DECR | 验证码、Session、计数器 |
| **Hash** | 哈希表 | HSET/HGET/HGETALL | 用户信息、商品详情 |
| **List** | 双向链表 | LPUSH/RPUSH/LPOP/RPOP | 消息队列、最近浏览 |
| **Set** | 无序集合 | SADD/SMEMBERS/SINTER | 标签、去重 |
| **ZSet** | 有序集合 | ZADD/ZRANGE/ZREVRANGE | 排行榜、热门商品 |

### 2.2 项目中的 Redis 使用

```java
// 项目路径: mall-common/src/main/java/com/macro/mall/common/service/RedisService.java
public interface RedisService {
    // String 操作
    void set(String key, Object value, long time);
    Object get(String key);
    Long incr(String key, long delta);
    
    // Hash 操作
    Object hGet(String key, String hashKey);
    void hSet(String key, String hashKey, Object value);
    Map<Object, Object> hGetAll(String key);
    
    // Set 操作
    Long sAdd(String key, Object... values);
    Set<Object> sMembers(String key);
    
    // List 操作
    Long lPush(String key, Object value);
    List<Object> lRange(String key, long start, long end);
}
```

### 2.3 实现类

```java
// 项目路径: mall-common/src/main/java/com/macro/mall/common/service/impl/RedisServiceImpl.java
public class RedisServiceImpl implements RedisService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public void set(String key, Object value, long time) {
        redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
    }
    
    @Override
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }
    
    @Override
    public Long incr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }
}
```

---

## 3. 缓存策略

### 3.1 缓存穿透

**问题**：查询一个不存在的数据，每次都穿透到数据库

**解决方案**：
1. **空值缓存**：缓存 null 值，设置短过期时间
2. **布隆过滤器**：过滤不存在的 key

```java
public UmsAdmin getAdminByUsername(String username) {
    // 1. 先查缓存
    String key = "ums:admin:" + username;
    UmsAdmin admin = (UmsAdmin) redisService.get(key);
    if (admin != null) {
        return admin;
    }
    
    // 2. 查数据库
    admin = adminMapper.getAdminByUsername(username);
    
    // 3. 缓存结果（包括空值）
    if (admin != null) {
        redisService.set(key, admin, 3600);
    } else {
        // 缓存空值，防止穿透
        redisService.set(key, "", 60);
    }
    return admin;
}
```

### 3.2 缓存击穿

**问题**：热点 key 过期瞬间，大量请求穿透到数据库

**解决方案**：
1. **互斥锁**：用 Redis 分布式锁保证只有一个线程去数据库
2. **永不过期**：设置永不过期，后台异步更新

```java
public UmsAdmin getAdminByUsername(String username) {
    String key = "ums:admin:" + username;
    UmsAdmin admin = (UmsAdmin) redisService.get(key);
    
    if (admin != null) {
        return admin;
    }
    
    // 互斥锁，防止击穿
    String lockKey = "lock:admin:" + username;
    if (redisService.setIfAbsent(lockKey, "1", 30)) {
        try {
            admin = adminMapper.getAdminByUsername(username);
            if (admin != null) {
                redisService.set(key, admin, 3600);
            }
        } finally {
            redisService.del(lockKey);
        }
    }
    
    return admin;
}
```

### 3.3 缓存雪崩

**问题**：大量 key 同时过期，数据库压力剧增

**解决方案**：
1. **随机过期时间**：给每个 key 设置不同的过期时间
2. **多级缓存**：本地缓存 + Redis 缓存
3. **预热缓存**：提前加载热点数据

---

## 4. 持久化

### 4.1 RDB（快照）

**原理**：定时将内存数据快照写入磁盘

**优点**：
- 文件小，恢复快
- 性能影响小

**缺点**：
- 可能丢失最后一次快照后的数据
- 大内存时 fork 子进程开销大

**配置**：
```
save 900 1    # 900秒内至少1个key变化
save 300 10   # 300秒内至少10个key变化
save 60 10000 # 60秒内至少10000个key变化
```

### 4.2 AOF（Append Only File）

**原理**：记录所有写操作命令，恢复时重放

**优点**：
- 数据安全性高（可配置每秒/每次写入）
- 文件可读性好

**缺点**：
- 文件体积大
- 恢复速度慢

**配置**：
```
appendonly yes          # 开启AOF
appendfsync everysec    # 每秒刷盘（推荐）
# appendfsync always    # 每次写入刷盘（最安全）
# appendfsync no        # 操作系统决定刷盘（最快）
```

### 4.3 选择建议

| 场景 | 推荐方案 |
|------|----------|
| 追求性能，可容忍少量数据丢失 | RDB |
| 追求数据安全 | AOF |
| 生产环境 | RDB + AOF 同时开启 |

---

## 5. 分布式锁

### 5.1 基于 Redis 实现

```java
public boolean tryLock(String key, String value, long expireTime) {
    return redisTemplate.opsForValue().setIfAbsent(key, value, expireTime, TimeUnit.SECONDS);
}

public void unlock(String key, String value) {
    String currentValue = (String) redisTemplate.opsForValue().get(key);
    if (value.equals(currentValue)) {
        redisTemplate.delete(key);
    }
}
```

### 5.2 Redisson 分布式锁（推荐）

```java
// 引入依赖
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.17.5</version>
</dependency>

// 使用
@Autowired
private RedissonClient redissonClient;

public void doSomething() {
    RLock lock = redissonClient.getLock("order:lock:123");
    try {
        // 尝试获取锁，最多等待10秒，锁自动过期30秒
        if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
            // 业务逻辑
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    } finally {
        lock.unlock();
    }
}
```

---

## 6. 项目中的 Redis 应用

### 6.1 用户缓存

```java
// 项目路径: mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java
@Service
public class UmsAdminCacheServiceImpl implements UmsAdminCacheService {
    
    @Autowired
    private RedisService redisService;
    
    private static final String ADMIN_KEY_PREFIX = "ums:admin:";
    private static final String RESOURCE_LIST_KEY_PREFIX = "ums:resourceList:";
    
    @Override
    public void delAdmin(Long adminId) {
        String key = ADMIN_KEY_PREFIX + adminId;
        redisService.del(key);
    }
    
    @Override
    public void setAdmin(UmsAdmin admin) {
        String key = ADMIN_KEY_PREFIX + admin.getId();
        redisService.set(key, admin, 86400);
    }
    
    @Override
    public UmsAdmin getAdmin(Long adminId) {
        String key = ADMIN_KEY_PREFIX + adminId;
        return (UmsAdmin) redisService.get(key);
    }
}
```

### 6.2 订单号生成

```java
// 项目路径: mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java
@Override
public String generateOrderSn(Long memberId) {
    String key = "oms:orderId";
    long orderId = redisService.incr(key, 1);
    return DateUtil.format(new Date(), "yyyyMMdd") 
           + String.format("%012d", orderId);
}
```

### 6.3 验证码存储

```java
// 项目路径: mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberServiceImpl.java
@Override
public CommonResult generateAuthCode(String telephone) {
    StringBuilder sb = new StringBuilder();
    Random random = new Random();
    for (int i = 0; i < 6; i++) {
        sb.append(random.nextInt(10));
    }
    String key = "ums:authCode:" + telephone;
    redisService.set(key, sb.toString(), 90); // 90秒过期
    return CommonResult.success(sb.toString());
}
```

---

## 7. 面试常问问题

### 7.1 Redis 和 Memcached 的区别？

**答案要点**：
| 特性 | Redis | Memcached |
|------|-------|-----------|
| 数据结构 | 丰富（String/Hash/List/Set/ZSet） | 仅 String |
| 持久化 | 支持（RDB/AOF） | 不支持 |
| 事务 | 支持 | 不支持 |
| 集群 | 原生支持 | 需第三方 |
| 内存管理 | 多种策略 | 固定大小 |

### 7.2 Redis 的过期策略？

**答案要点**：
1. **定时删除**：设置过期时间时创建定时器，到期立即删除（消耗CPU）
2. **惰性删除**：访问时检查是否过期，过期则删除（内存占用可能高）
3. **定期删除**：每隔一段时间扫描，删除过期 key（折中方案）

### 7.3 Redis 的内存淘汰策略？

**答案要点**：
| 策略 | 说明 |
|------|------|
| `volatile-lru` | 淘汰最近最少使用的过期 key |
| `volatile-ttl` | 淘汰即将过期的 key |
| `volatile-random` | 随机淘汰过期 key |
| `allkeys-lru` | 淘汰最近最少使用的所有 key |
| `allkeys-random` | 随机淘汰所有 key |
| `noeviction` | 不淘汰，写入时报错（默认） |

### 7.4 Redis 如何保证数据一致性？

**答案要点**：
1. **双写一致性**：先写数据库，再删缓存（推荐）
   - 问题：删缓存失败会导致数据不一致
   - 解决方案：重试机制 + 订阅 binlog 异步删缓存
2. **读穿一致性**：先读缓存，缓存不存在再读数据库并写入缓存

### 7.5 Redis 主从复制原理？

**答案要点**：
1. **全量同步**：从节点发送 SYNC 命令，主节点执行 BGSAVE 生成 RDB，发送给从节点
2. **增量同步**：主节点将写命令发送给从节点执行

### 7.6 Redis 集群模式？

**答案要点**：
1. **哨兵模式**：监控主节点，故障自动切换
2. **Cluster 模式**：数据分片存储，每个节点负责一部分槽位

### 7.7 Redis 为什么快？

**答案要点**：
1. **内存存储**：无需磁盘 IO
2. **单线程**：无锁竞争，避免线程切换开销
3. **高效数据结构**：底层用 C 实现，优化了数据结构
4. **IO 多路复用**：处理大量客户端连接

---

## 8. 快速学习清单

| 优先级 | 学习内容 | 时间建议 |
|--------|----------|----------|
| 1 | 五大数据结构及应用场景 | 30分钟 |
| 2 | 缓存三大问题（穿透/击穿/雪崩） | 30分钟 |
| 3 | 持久化机制（RDB/AOF） | 30分钟 |
| 4 | 分布式锁实现 | 30分钟 |
| 5 | Spring Data Redis 使用 | 30分钟 |
| 6 | 面试常问问题背诵 | 30分钟 |

---

## 9. 关键代码路径

| 路径 | 说明 |
|------|------|
| `mall-common/src/main/java/com/macro/mall/common/service/RedisService.java` | Redis服务接口 |
| `mall-common/src/main/java/com/macro/mall/common/service/impl/RedisServiceImpl.java` | Redis服务实现 |
| `mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java` | Redis配置 |
| `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminCacheServiceImpl.java` | 用户缓存 |
| `mall-portal/src/main/java/com/macro/mall/portal/service/impl/OmsPortalOrderServiceImpl.java` | 订单号生成 |
| `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberServiceImpl.java` | 验证码存储 |