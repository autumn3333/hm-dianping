# 🍜 hm-dianping —— 黑马点评（Redis 实战项目）

一个本地生活点评类应用后端，完整实现了短信登录、商铺缓存、优惠券秒杀、达人探店、好友关注、附近商铺、用户签到等业务模块。项目的重点不是 CRUD，而是**把 Redis 的各类数据结构和分布式场景真实跑通**：缓存穿透/击穿/雪崩的三种解法、自研分布式锁到 Redisson、Lua 脚本 + Redis Stream 的异步秒杀、Set 交集算共同关注、GEO 找附近商铺、BitMap 做签到统计。

> 服务端口 `8081` · Spring Boot 2.3.12 · JDK 1.8

## 功能模块

| 模块 | 实现要点 |
|---|---|
| 短信登录 | 验证码存 Redis（2 分钟 TTL）；登录成功后生成随机 token 存 Redis（Hash 结构），前端放 `authorization` 请求头 |
| 商户查询缓存 | 空值缓存解决**穿透**；互斥锁与逻辑过期两套方案解决**击穿**（`queryWithMutex` / `queryWithLogicalExpire`）；随机 TTL 缓解**雪崩** |
| 全局唯一 ID | `RedisIdWorker`：时间戳（左移 32 位）+ Redis 自增序列号，`keyPrefix` 隔离不同业务 |
| 优惠券秒杀 | `seckill.lua` 在 Redis 内原子完成「校验库存 → 校验一人一单 → 扣减库存 → 写入 Stream」，再无阻塞地由后台线程消费下单 |
| 分布式锁 | 先自研 `SimpleRedisLock`（setnx + 过期时间 + Lua 释放锁），再替换为 Redisson（看门狗续期、可重入） |
| 达人探店 | 点赞用 zSet 存储（score 存时间戳），支持按点赞时间倒序的点赞排行榜 |
| 好友关注 | 关注关系用 Set 存储；`SINTER` 求共同关注；发布笔记时推送到粉丝收件箱（zSet） |
| 关注推送 Feed 流 | `queryBlogOfFollow` 基于 `ZREVRANGEBYSCORE` 滚动分页，返回 `ScrollResult`，解决时间戳相同时的分页错乱 |
| 附近商铺 | `GEOSEARCH` 按距离与坐标检索店铺，结果按距离排序分页 |
| 用户签到 | BitMap 存储签到记录，`setBit` 写入、按位统计连续签到天数 |

## 技术栈

| 层 | 技术 |
|---|---|
| 框架 | Spring Boot 2.3.12（JDK 1.8） |
| 持久层 | MyBatis-Plus 3.4.3 + MySQL 5.7+ |
| 缓存 / 分布式 | Redis（Lettuce 连接池）+ Redisson 3.13.6 |
| 工具库 | Hutool 5.7.17、Lombok、AspectJ |
| 前端 | 随项目提供的静态页面（Nginx 部署，见下） |

## 环境要求

- JDK 1.8+
- Maven 3.6+
- MySQL 5.7+
- Redis 5.0+（需支持 Stream 与 GEO，本机 `127.0.0.1:6379`）

## 快速开始

### 1. 初始化数据库

SQL 脚本位于 `src/main/resources/db/hmdp.sql`，会创建 `hmdp` 库及 11 张表（`tb_user`、`tb_shop`、`tb_blog`、`tb_voucher`、`tb_seckill_voucher`、`tb_voucher_order`、`tb_follow`、`tb_sign` 等）：

```bash
mysql -uroot -p < src/main/resources/db/hmdp.sql
```

### 2. 启动 Redis

```bash
redis-server
```

### 3. 修改配置

编辑 `src/main/resources/application.yaml`，确认数据源、Redis 地址与本机一致：

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/hmdp?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: 你的密码
  redis:
    host: 127.0.0.1
    port: 6379
```

### 4. 启动项目

```bash
mvn spring-boot:run
```

服务监听 `8081`，所有接口以 `/api` 为前缀（由前端静态页面 Nginx 反向代理）。

### 关于短信验证码

项目未接入真实短信服务商，`UserServiceImpl#sendCode` 生成 6 位随机验证码后写入 Redis，并通过 `log.debug` 打印到控制台。本地调试时从控制台日志取验证码即可：

```
发送短信验证码成功，验证码：123456
```

## 主要接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/user/code` | 发送手机验证码 |
| POST | `/user/login` | 验证码登录，返回 token |
| POST | `/user/logout` | 退出登录 |
| GET | `/user/me` | 获取当前登录用户 |
| POST | `/user/sign` | 用户签到 |
| GET | `/user/sign/count` | 查询本月连续签到天数 |
| GET | `/shop/{id}` | 查询商铺详情（走缓存） |
| GET | `/shop/of/type` | 按类型 + 坐标分页查询商铺（GEO） |
| PUT | `/shop` | 更新商铺（删除缓存） |
| POST | `/voucher` | 新增优惠券 |
| POST | `/voucher/seckill` | 新增秒杀券 |
| POST | `/voucher-order/seckill/{id}` | 秒杀下单（Lua + Stream） |
| PUT | `/blog/like/{id}` | 点赞 / 取消点赞 |
| GET | `/blog/likes/{id}` | 点赞排行榜 |
| GET | `/blog/of/follow` | 关注的人的笔记（滚动分页） |
| PUT | `/follow/{id}/{isFollow}` | 关注 / 取关 |
| GET | `/follow/common/{id}` | 共同关注 |

## 项目结构

```
src/main/java/com/hmdp/
├── config/            # MvcConfig（双拦截器注册）、Redisson、MyBatis
├── controller/        # 各业务 HTTP 入口
├── dto/               # Result 统一响应、UserDTO、ScrollResult（滚动分页）
├── entity/            # 数据库实体
├── mapper/            # MyBatis-Plus Mapper
├── service/impl/      # 业务实现（缓存、秒杀、点赞、Feed 流等核心逻辑）
└── utils/
    ├── CacheClient.java          # 缓存工具：普通写入 + 逻辑过期写入
    ├── SimpleRedisLock.java      # 自研分布式锁（setnx + Lua 解锁）
    ├── RedisIdWorker.java        # 全局唯一 ID 生成器
    ├── RefreshTokenInterceptor   # 拦截所有请求，续期 token 并写入 UserHolder
    ├── LoginInterceptor          # 校验登录态，未登录返回 401
    ├── UserHolder.java           # ThreadLocal 保存当前登录用户
    └── RedisConstants.java       # 所有 Redis key 与 TTL 常量

src/main/resources/
├── db/hmdp.sql         # 建库建表 + 初始化数据
├── seckill.lua         # 秒杀脚本：库存校验 + 一人一单 + 扣减 + 发消息
└── unlock.lua          # 分布式锁释放脚本：校验持有者后删除
```

## 关键实现说明

### 秒杀为什么能抗住并发

`seckill.lua` 把四步放在 Redis 单线程内一次性完成，避免多次网络往返带来的竞态：

```lua
local stockStr = redis.call('get', stockKey) or "0"   -- 库存不存在时按 0 处理，防 nil
local stock = tonumber(stockStr)
if stock <= 0 then return 1 end                        -- 库存不足
if redis.call('sismember', orderKey, userId) == 1 then
    return 2                                           -- 重复下单
end
redis.call('incrby', stockKey, -1)                     -- 扣减库存
redis.call('sadd', orderKey, userId)                   -- 标记已下单
redis.call('xadd', 'stream.orders', '*', 'userId', userId,
           'voucherId', voucherId, 'id', orderId)      -- 异步落库
return 0
```

脚本返回 `0` 表示抢购成功，接口立刻返回订单号；真正的写库动作由后台单线程消费 Redis Stream 完成，并配套 `pending-list` 重试机制（`handlePendingList`），保证订单最终一致。

### 缓存击穿的两种解法

- **互斥锁**（`queryWithMutex`）：未命中时用 `setnx` 抢锁，抢到的线程查库并回填，抢不到的线程短暂休眠重试。
- **逻辑过期**（`queryWithLogicalExpire`）：缓存里额外存一个过期时间字段，值本身永不过期。命中后发现逻辑过期，就丢给线程池异步重建，当前请求直接返回旧数据。`queryById` 默认走的就是这条路径。

### 双拦截器分工

`RefreshTokenInterceptor`（order 0）拦截 `/**`，负责从 `authorization` 请求头取 token、刷新 TTL、把用户放进 `UserHolder`；`LoginInterceptor`（order 1）只做「有没有登录」的判断，并排除登录注册等公开接口。请求结束时在 `afterCompletion` 清理 ThreadLocal，防止内存泄漏。

## 说明

- `application.yaml` 中的数据库与 Redis 配置仅供本地开发，请勿直接用于生产环境。
- 秒杀与缓存相关测试见 `src/test/java/com/hmdp/HmDianPingApplicationTests.java`。
