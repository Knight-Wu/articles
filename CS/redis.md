# 分布式限流
```
-- token_bucket.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local capacity = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 获取现有 token 状态
local last_tokens = tonumber(redis.call("HGET", key, "tokens") or capacity)
local last_time = tonumber(redis.call("HGET", key, "timestamp") or now)

-- 补充token 到令牌桶: 现在到上次获取token 的时间间隔 * 每单位时间间隔需要新增的token(refill_rate)
local delta = math.max(0, now - last_time)
local refill = math.floor(delta * refill_rate)
local current_tokens = math.min(capacity, last_tokens + refill)

-- 没超过限流, 返回 0
if current_tokens < requested then 
  return 0
else
  redis.call("HSET", key, "tokens", current_tokens - requested)
  redis.call("HSET", key, "timestamp", now)
  redis.call("EXPIRE", key, 2)
  return 1
end

```
实际上还可以请求的token 计算下次不被限流的时间, 思路: 时间间隔 = 请求的token / 单位时间的产生的token , 下次时间 = 时间间隔 + now
# 分布式锁
SET lock_key client_id NX PX 30000

lock_key：表示这把锁的名称。

client_id：唯一标识符, 用于标识当前客户端（释放锁时会验证）。

NX：仅当 key 不存在时才设置成功（实现加锁互斥）。

PX 30000：设置过期时间，防止死锁（单位是毫秒）

## 释放分布式锁
使用lua 脚本保证原子操作, 检查是否是自己加的锁, 防止误删别人的锁, 只要删除之前先检查是不是自己加的锁, 就算超时了回来检查已经是别人的锁, 也不会有问题. 
```
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end

```

## 持有锁的
## lua 脚本为什么是原子的
一是因为redis 是单线程的, 二是lua 脚本保证脚本的命令执行过程中不能插入其他命令, 那么单线程, 顺序执行, 就可以要么全部成功要么失败的原子性. 
# 单线程
Redis 单线程指的是「接收客户端请求->解析请求->进行数据读写等操作->发送数据给客户端」这个过程是由一个线程(主线程)来完成的, 不管你写哪个db, 有多少个db, 都是单线程,
因为是纯内存操作, cpu 通常不是瓶颈, 内存和网络通常是瓶颈, 一个普通的linux 服务器可以提供单机百万的qps

## 利用单机多核性能
可以在一台服务器起多个redis 实例, 采用shard 集群的方式. 

* 网络处理是多线程

虽然 Redis的主要工作(网络I/O和执行命令)一直是单线程模型,但是在Redis 6.0版本之后,也采用
了多个I/O线程来处理网络请求,这是因为随着网络硬件的性能提升,Redis的性能瓶颈有时会出现在网
络I/O的处理上。

所以为了提高网络I/O的并行度,Redis 6.0对于网络I/O采用多线程来处理。但是对于命令的执行,
Redis 仍然使用单线程来处理

Redis 官方表示,Redis 6.0 版本引入的多线程I/O特性对性能提升至少是一倍以上。

* 后台任务多线程
![image](https://github.com/user-attachments/assets/5ee4793f-44e8-4554-bcaa-4ffe6fc46aa2)

# 持久化
## AOF 日志
重启之后读取日志来恢复内存中的数据, 但是日志过多会导致重启恢复过慢. 
![image](https://github.com/user-attachments/assets/38491845-cbdd-4ad3-ad21-adc43c41278a)
![image](https://github.com/user-attachments/assets/fd43b380-a448-49af-aaab-8933abb941de)

写日志的方式, 先执行写命令再写日志, 仍然由主线程处理, 可能会出现执行完命令之后就挂掉的情况所以还是会出现数据没有被持久化的情况, 所以不能靠 redis 作为唯一持久化的存储, 需要数据库来保证持久化. 

### 日志持久化到磁盘的策略
![image](https://github.com/user-attachments/assets/0b63232d-6929-44b9-8558-fc412ce9952d)

## RDB 快照
记录的是内存的快照, 而不是像AOF 文件记录的是命令操作的日志. 可以配置每隔一段时间自动记录快照, 但是是全量快照, 不是增量所以会影响性能. 
可以在主线程中直接记录快照会阻塞写入, 也可以使用子进程来创建快照, 跟父进程共享内存, 如果父进程发生了修改会采用写时复制技术避免冲突. 


## 混合持久化
这个思想是数据持久化的一般思路, es 也是类似的
![image](https://github.com/user-attachments/assets/81283f0e-b34b-4ee1-816b-8a796bdc3e50)

# 缓存雪崩
当大量缓存数据在同一时间过期(失效)时,如果此时有大量的用户请求,都无法在Redis中处
理,于是全部请求都直接访问数据库,从而导致数据库的压力骤增,严重的会造成数据库宕机,从而形成
一系列连锁反应.

1. 将缓存失效时间随机打散
2. 设置缓存不过期(可以不说, 麻烦), 异步更新

# 缓存击穿
如果缓存中的某个热点数据过期了,此时大量的请求访问了该热点数据,就无法从缓存中读取,直接访问
数据库,数据库很容易就被高并发的请求冲垮,这就是缓存击穿的问题。

1. 不给热点数据设置失效时间, 异步更新

# 缓存穿透
缓存和db 都不存在的数据
1. 应用层面过滤非法请求
2. 缓存设置空值
3. 应用层使用布隆过滤器

# 缓存更新策略
## 旁路缓存(cache aside)
适用redis 和mysql 的更新, 适合读多写少, 
![image](https://github.com/user-attachments/assets/5f65d96d-d370-4e31-b995-344543e69db2)
最能保证缓存和数据库一致性的方法: 先更新数据库再删除缓存

* 但是还是有极端情况导致数据不一致
假如某个用户数据在缓存中不存在,请求A读取数据时从数据库中查询到年龄为20,在未写入缓存中时
另一个请求B更新数据。它更新数据库中的年龄为21,并且清空缓存。这时请求A把从数据库中读到的
年龄为20的数据写入到缓存中。
最终用户缓存数据是20 , 数据库是21.

* 可以加上缓存过期时间来兜底, 过一段时间后就一致了
![image](https://github.com/user-attachments/assets/2862d46c-f69e-4867-8586-d3be369de96f)

## read through, write through(读穿, 写穿)
![image](https://github.com/user-attachments/assets/9e3c82b7-c61d-451d-a769-69aea360bfb3)

## 写回(write back)
适合写多的场景, 只写缓存, 异步更新数据库, 但是会有数据丢失风险. 

## 数据库日志异步更新
由binlog 异步更新缓存, 

# 延迟队列
例如多久没付款取消订单
![image](https://github.com/user-attachments/assets/c4a1c3d0-f8fa-45ac-bd9d-44a19f6c38ab)
