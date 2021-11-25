
---
type: docs
title: "RateLimiter"
linkTitle: "RateLimiter"
weight: 1
---

常见的限流算法有令牌桶算法，漏桶算法，与计数器算法。本文主要对三个算法的基本原理及 Google Guava 包中令牌桶算法的实现 RateLimiter 进行介绍，下一篇文章介绍最近写的一个以 RateLimiter 为参考的分布式限流实现及计数器限流实现。

## 令牌桶算法

令牌桶算法的原理就是以一个恒定的速度往桶里放入令牌，每一个请求的处理都需要从桶里先获取一个令牌，当桶里没有令牌时，则请求不会被处理，要么排队等待，要么降级处理，要么直接拒绝服务。当桶里令牌满时，新添加的令牌会被丢弃或拒绝。

令牌桶算法的处理示意图如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211121234648.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

令牌桶算法主要是可以控制请求的平均处理速率，它允许预消费，即可以提前消费令牌，以应对突发请求，但是后面的请求需要为预消费买单（等待更长的时间），以满足请求处理的平均速率是一定的。

## 漏桶算法

漏桶算法的原理是水（请求）先进入漏桶中，漏桶以一定的速度出水（处理请求），当水流入速度大于流出速度导致水在桶内逐渐堆积直到桶满时，水会溢出（请求被拒绝）。

漏桶算法的处理示意图如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211121234739.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

漏桶算法主要是控制请求的处理速率，平滑网络上的突发流量，请求可以以任意速度进入漏桶中，但请求的处理则以恒定的速度进行。

## 计算器算法

计数器算法是限流算法中最简单的一种算法，限制在一个时间窗口内，至多处理多少个请求。比如每分钟最多处理10个请求，则从第一个请求进来的时间为起点，60s的时间窗口内只允许最多处理10个请求。下一个时间窗口又以前一时间窗口过后第一个请求进来的时间为起点。常见的比如一分钟内只能获取一次短信验证码的功能可以通过计数器算法来实现。

## Guava RateLimiter

Guava是Google开源的一个工具包，其中的RateLimiter是实现了令牌桶算法的一个限流工具类。

如下测试代码示例了RateLimiter的用法：

```java
public static void main(String[] args) {
    RateLimiter rateLimiter = RateLimiter.create(1); //创建一个每秒产生一个令牌的令牌桶
    for(int i=1;i<=5;i++) {
        double waitTime = rateLimiter.acquire(i); //一次获取i个令牌
        System.out.println("acquire:" + i + " waitTime:" + waitTime);
    }
}
```

运行后，输出如下，

```shell
acquire:1 waitTime:0.0
acquire:2 waitTime:0.997729
acquire:3 waitTime:1.998076
acquire:4 waitTime:3.000303
acquire:5 waitTime:4.000223
```

第一次获取一个令牌时，等待0s立即可获取到（这里之所以不需要等待是因为令牌桶的预消费特性），第二次获取两个令牌，等待时间1s，这个1s就是前面获取一个令牌时因为预消费没有等待延到这次来等待的时间，这次获取两个又是预消费，所以下一次获取（取3个时）就要等待这次预消费需要的2s了，依此类推。可见预消费不需要等待的时间都由下一次来买单，以保障一定的平均处理速率（上例为1s一次）。

RateLimiter有两种实现：

1. SmoothBursty： 令牌的生成速度恒定。使用 `RateLimiter.create(double permitsPerSecond)` 创建的是 SmoothBursty 实例。
2. SmoothWarmingUp：令牌的生成速度持续提升，直到达到一个稳定的值。WarmingUp，顾名思义就是有一个热身的过程。使用 `RateLimiter.create(double permitsPerSecond, long warmupPeriod, TimeUnit unit)` 时创建就是 SmoothWarmingUp 实例，其中 warmupPeriod 就是热身达到稳定速度的时间。

类结构如下：

<div align="center"> <img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20211121235011.png" style="display:block;width:80%;" alt="NAME" align=center /> </div>

关键属性及方法解析（以 SmoothBursty 为例）：

### 关键属性

```java
/** 桶中当前拥有的令牌数. */
double storedPermits;

/** 桶中最多可以保存多少秒存入的令牌数 */
double maxBurstSeconds;

/** 桶中能存储的最大令牌数，等于storedPermits*maxBurstSeconds. */
double maxPermits;

/** 放入令牌的时间间隔*/
double stableIntervalMicros;

/** 下次可获取令牌的时间点，可以是过去也可以是将来的时间点*/
private long nextFreeTicketMicros = 0L;
```

### 关键方法

调用 `RateLimiter.create(double permitsPerSecond)` 方法时，创建的是 SmoothBursty 实例，默认设置 maxBurstSeconds 为 1s。SleepingStopwatch 是guava中的一个时钟类实现。

```java
@VisibleForTesting
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
        RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
}

SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
    super(stopwatch);
    this.maxBurstSeconds = maxBurstSeconds;
}
```

并通过调用 `SmoothBursty.doSetRate(double, long)` 方法进行初始化，该方法中:

1. 调用 `resync(nowMicros)` 对 storedPermits 与 nextFreeTicketMicros 进行了调整——如果当前时间晚于 nextFreeTicketMicros，则计算这段时间内产生的令牌数，累加到 storedPermits 上，并更新下次可获取令牌时间 nextFreeTicketMicros 为当前时间。
2. 计算 stableIntervalMicros 的值，1/permitsPerSecond。
3. 调用 `doSetRate(double, double)` 方法计算 maxPermits 值（maxBurstSeconds*permitsPerSecond），并根据旧的 maxPermits 值对 storedPermits 进行调整。

源码如下所示

```java
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
        resync(nowMicros);
        double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
        this.stableIntervalMicros = stableIntervalMicros;
        doSetRate(permitsPerSecond, stableIntervalMicros);
}

/** Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time. */
void resync(long nowMicros) {
        // if nextFreeTicket is in the past, resync to now
        if (nowMicros > nextFreeTicketMicros) {
        double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
        storedPermits = min(maxPermits, storedPermits + newPermits);
        nextFreeTicketMicros = nowMicros;
        }
}

@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        double oldMaxPermits = this.maxPermits;
        maxPermits = maxBurstSeconds * permitsPerSecond;
        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
                // if we don't special-case this, we would get storedPermits == NaN, below
                storedPermits = maxPermits;
        } else {
                storedPermits =
                        (oldMaxPermits == 0.0)
                                ? 0.0 // initial state
                                : storedPermits * maxPermits / oldMaxPermits;
        }
}
```

调用 `acquire(int)` 方法获取指定数量的令牌时，

1. 调用 `reserve(int)` 方法，该方法最终调用 `reserveEarliestAvailable(int, long)` 来更新下次可取令牌时间点与当前存储的令牌数，并返回本次可取令牌的时间点，根据该时间点计算需要等待的时间
2. 阻塞等待1中返回的等待时间
3. 返回等待的时间（秒）

源码如下所示

```java
/** 获取指定数量（permits）的令牌，阻塞直到获取到令牌，返回等待的时间*/
@CanIgnoreReturnValue
public double acquire(int permits) {
        long microsToWait = reserve(permits);
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

final long reserve(int permits) {
        checkPermits(permits);
        synchronized (mutex()) {
                return reserveAndGetWaitLength(permits, stopwatch.readMicros());
        }
}

/** 返回需要等待的时间*/
final long reserveAndGetWaitLength(int permits, long nowMicros) {
        long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
        return max(momentAvailable - nowMicros, 0);
}

/** 针对此次需要获取的令牌数更新下次可取令牌时间点与存储的令牌数，返回本次可取令牌的时间点*/
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
        resync(nowMicros); // 更新当前数据
        long returnValue = nextFreeTicketMicros;
        double storedPermitsToSpend = min(requiredPermits, this.storedPermits); // 本次可消费的令牌数
        double freshPermits = requiredPermits - storedPermitsToSpend; // 需要新增的令牌数
        long waitMicros =
                storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
                        + (long) (freshPermits * stableIntervalMicros); // 需要等待的时间

        this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros); // 更新下次可取令牌的时间点
        this.storedPermits -= storedPermitsToSpend; // 更新当前存储的令牌数
        return returnValue;
}
```

`acquire(int)` 方法是获取不到令牌时一直阻塞，直到获取到令牌，`tryAcquire(int,long,TimeUnit)` 方法则是在指定超时时间内尝试获取令牌，如果获取到或超时时间到则返回是否获取成功

1. 先判断是否能在指定超时时间内获取到令牌，通过 `nextFreeTicketMicros <= timeoutMicros + nowMicros` 是否为true来判断，即可取令牌时间早于当前时间加超时时间则可取（预消费的特性），否则不可获取。
2. 如果不可获取，立即返回false。
3. 如果可获取，则调用 `reserveAndGetWaitLength(permits, nowMicros)` 来更新下次可取令牌时间点与当前存储的令牌数，返回等待时间（逻辑与前面相同），并阻塞等待相应的时间，返回true。

源码如下所示

```java
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
        long timeoutMicros = max(unit.toMicros(timeout), 0);
        checkPermits(permits);
        long microsToWait;
        synchronized (mutex()) {
                long nowMicros = stopwatch.readMicros();
                if (!canAcquire(nowMicros, timeoutMicros)) { //判断是否能在超时时间内获取指定数量的令牌
                        return false;
                } else {
                        microsToWait = reserveAndGetWaitLength(permits, nowMicros);
                }
        }
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
        return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros; //只要可取时间小于当前时间+超时时间，则可获取（可预消费的特性！）
}

@Override
final long queryEarliestAvailable(long nowMicros) {
        return nextFreeTicketMicros;
}
```

以上就是 SmoothBursty 实现的基本处理流程。注意两点：

1. RateLimiter 通过限制后面请求的等待时间，来支持一定程度的突发请求——预消费的特性。
2. RateLimiter 令牌桶的实现并不是起一个线程不断往桶里放令牌，而是以一种延迟计算的方式（参考`resync`函数），在每次获取令牌之前计算该段时间内可以产生多少令牌，将产生的令牌加入令牌桶中并更新数据来实现，比起一个线程来不断往桶里放令牌高效得多。（想想如果需要针对每个用户限制某个接口的访问，则针对每个用户都得创建一个RateLimiter，并起一个线程来控制令牌存放的话，如果在线用户数有几十上百万，起线程来控制是一件多么恐怖的事情）

## 单机局限

本文介绍了限流的三种基本算法，其中令牌桶算法与漏桶算法主要用来限制请求处理的速度，可将其归为限速，计数器算法则是用来限制一个时间窗口内请求处理的数量，可将其归为限量（对速度不限制）。

Guava 的 RateLimiter 是令牌桶算法的一种实现，但 RateLimiter 只适用于单机应用，在分布式环境下就不适用了。虽然已有一些开源项目可用于分布式环境下的限流管理，如阿里的Sentinel，但对于小型项目来说，引入Sentinel可能显得有点过重。

## 分布式实现

基于 Redis 脚本分别实现：

- 基于RateLimiter令牌桶算法的限速控制（严格限制访问速度）
- 基于Lua脚本的限量控制（限制一个时间窗口内的访问量，对访问速度没有严格限制）

### 限速控制

#### 1. 令牌桶模型

首先定义令牌桶模型，与RateLimiter中类似，包括几个关键属性与关键方法。其中关键属性定义如下，

```java
@Data
public class RedisPermits {

    /**
     * 最大存储令牌数
     */
    private double maxPermits;
    /**
     * 当前存储令牌数
     */
    private double storedPermits;
    /**
     * 添加令牌的时间间隔/毫秒
     */
    private double intervalMillis;
    /**
     * 下次请求可以获取令牌的时间，可以是过去（令牌积累）也可以是将来的时间（令牌预消费）
     */
    private long nextFreeTicketMillis;

    //...
```

关键方法定义与RateLimiter也大同小异，方法注释基本已描述各方法用途，不再赘述。

```java
    /**
     * 构建Redis令牌数据模型
     *
     * @param permitsPerSecond     每秒放入的令牌数
     * @param maxBurstSeconds      maxPermits由此字段计算，最大存储maxBurstSeconds秒生成的令牌
     * @param nextFreeTicketMillis 下次请求可以获取令牌的起始时间，默认当前系统时间
     */
    public RedisPermits(double permitsPerSecond, double maxBurstSeconds, Long nextFreeTicketMillis) {
        this.maxPermits = permitsPerSecond * maxBurstSeconds;
        this.storedPermits = maxPermits;
        this.intervalMillis = TimeUnit.SECONDS.toMillis(1) / permitsPerSecond;
        this.nextFreeTicketMillis = nextFreeTicketMillis;
    }

    /**
     * 基于当前时间，若当前时间晚于nextFreeTicketMicros，则计算该段时间内可以生成多少令牌，将生成的令牌加入令牌桶中并更新数据
     */
    public void resync(long nowMillis) {
        if (nowMillis > nextFreeTicketMillis) {
            double newPermits = (nowMillis - nextFreeTicketMillis) / intervalMillis;
            storedPermits = Math.min(maxPermits, storedPermits + newPermits);
            nextFreeTicketMillis = nowMillis;
        }
    }

    /**
    * 保留指定数量令牌，并返回需要等待的时间
    */
    public long reserveAndGetWaitLength(long nowMillis, int permits) {
        resync(nowMillis);
        double storedPermitsToSpend = Math.min(permits, storedPermits); // 可以消耗的令牌数
        double freshPermits = permits - storedPermitsToSpend; // 需要等待的令牌数
        long waitMillis = (long) (freshPermits * intervalMillis); // 需要等待的时间

        nextFreeTicketMillis = LongMath.saturatedAdd(nextFreeTicketMillis, waitMillis);
        storedPermits -= storedPermitsToSpend;
        return waitMillis;
    }

    /**
    * 在超时时间内，是否有指定数量的令牌可用
    */
    public boolean canAcquire(long nowMillis, int permits, long timeoutMillis) {
        return queryEarliestAvailable(nowMillis, permits) <= timeoutMillis;
    }

    /**
     * 指定数量令牌数可用需等待的时间
     *
     * @param permits 需保留的令牌数
     * @return 指定数量令牌可用的等待时间，如果为0或负数，表示当前可用
     */
    private long queryEarliestAvailable(long nowMillis, int permits) {
        resync(nowMillis);
        double storedPermitsToSpend = Math.min(permits, storedPermits); // 可以消耗的令牌数
        double freshPermits = permits - storedPermitsToSpend; // 需要等待的令牌数
        long waitMillis = (long) (freshPermits * intervalMillis); // 需要等待的时间

        return LongMath.saturatedAdd(nextFreeTicketMillis - nowMillis, waitMillis);
    }
```

#### 2. 令牌桶控制类

Guava RateLimiter中的控制都在RateLimiter及其子类中（如SmoothBursty），本处涉及到分布式环境下的同步，因此将其解耦，令牌桶模型存储于Redis中，对其同步操作的控制放置在如下控制类，其中同步控制使用到了分布式锁。

```java
@Slf4j
public class RedisRateLimiter {

    /**
     * 获取一个令牌，阻塞一直到获取令牌，返回阻塞等待时间
     *
     * @return time 阻塞等待时间/毫秒
     */
    public long acquire(String key) throws IllegalArgumentException {
        return acquire(key, 1);
    }

    /**
     * 获取指定数量的令牌，如果令牌数不够，则一直阻塞，返回阻塞等待的时间
     *
     * @param permits 需要获取的令牌数
     * @return time 等待的时间/毫秒
     * @throws IllegalArgumentException tokens值不能为负数或零
     */
    public long acquire(String key, int permits) throws IllegalArgumentException {
        long millisToWait = reserve(key, permits);
        log.info("acquire {} permits for key[{}], waiting for {}ms", permits, key, millisToWait);
        try {
            Thread.sleep(millisToWait);
        } catch (InterruptedException e) {
            log.error("Interrupted when trying to acquire {} permits for key[{}]", permits, key, e);
        }
        return millisToWait;
    }

    /**
     * 在指定时间内获取一个令牌，如果获取不到则一直阻塞，直到超时
     *
     * @param timeout 最大等待时间（超时时间），为0则不等待立即返回
     * @param unit    时间单元
     * @return 获取到令牌则true，否则false
     * @throws IllegalArgumentException
     */
    public boolean tryAcquire(String key, long timeout, TimeUnit unit) throws IllegalArgumentException {
        return tryAcquire(key, 1, timeout, unit);
    }

    /**
     * 在指定时间内获取指定数量的令牌，如果在指定时间内获取不到指定数量的令牌，则直接返回false，
     * 否则阻塞直到能获取到指定数量的令牌
     *
     * @param permits 需要获取的令牌数
     * @param timeout 最大等待时间（超时时间）
     * @param unit    时间单元
     * @return 如果在指定时间内能获取到指定令牌数，则true,否则false
     * @throws IllegalArgumentException tokens为负数或零，抛出异常
     */
    public boolean tryAcquire(String key, int permits, long timeout, TimeUnit unit) throws IllegalArgumentException {
        long timeoutMillis = Math.max(unit.toMillis(timeout), 0);
        checkPermits(permits);

        long millisToWait;
        boolean locked = false;
        try {
            locked = lock.lock(key + LOCK_KEY_SUFFIX, WebUtil.getRequestId(), 60, 2, TimeUnit.SECONDS);
            if (locked) {
                long nowMillis = getNowMillis();
                RedisPermits permit = getPermits(key, nowMillis);
                if (!permit.canAcquire(nowMillis, permits, timeoutMillis)) {
                    return false;
                } else {
                    millisToWait = permit.reserveAndGetWaitLength(nowMillis, permits);
                    permitsRedisTemplate.opsForValue().set(key, permit, expire, TimeUnit.SECONDS);
                }
            } else {
                return false;  //超时获取不到锁，也返回false
            }
        } finally {
            if (locked) {
                lock.unLock(key + LOCK_KEY_SUFFIX, WebUtil.getRequestId());
            }
        }
        if (millisToWait > 0) {
            try {
                Thread.sleep(millisToWait);
            } catch (InterruptedException e) {

            }
        }
        return true;
    }

    /**
     * 保留指定的令牌数待用
     *
     * @param permits 需保留的令牌数
     * @return time 令牌可用的等待时间
     * @throws IllegalArgumentException tokens不能为负数或零
     */
    private long reserve(String key, int permits) throws IllegalArgumentException {
        checkPermits(permits);
        try {
            lock.lock(key + LOCK_KEY_SUFFIX, WebUtil.getRequestId(), 60, 2, TimeUnit.SECONDS);
            long nowMillis = getNowMillis();
            RedisPermits permit = getPermits(key, nowMillis);
            long waitMillis = permit.reserveAndGetWaitLength(nowMillis, permits);
            permitsRedisTemplate.opsForValue().set(key, permit, expire, TimeUnit.SECONDS);
            return waitMillis;
        } finally {
            lock.unLock(key + LOCK_KEY_SUFFIX, WebUtil.getRequestId());
        }
    }

    /**
     * 获取令牌桶
     *
     * @return
     */
    private RedisPermits getPermits(String key, long nowMillis) {
        RedisPermits permit = permitsRedisTemplate.opsForValue().get(key);
        if (permit == null) {
            permit = new RedisPermits(permitsPerSecond, maxBurstSeconds, nowMillis);
        }
        return permit;
    }

    /**
     * 获取redis服务器时间
     */
    private long getNowMillis() {
        String luaScript = "return redis.call('time')";
        DefaultRedisScript<List> redisScript = new DefaultRedisScript<>(luaScript, List.class);
        List<String> now = (List<String>)stringRedisTemplate.execute(redisScript, null);
        return now == null ? System.currentTimeMillis() : Long.valueOf(now.get(0))*1000+Long.valueOf(now.get(1))/1000;
    }

    //...
}
```

其中：

1. acquire 是阻塞方法，如果没有可用的令牌，则一直阻塞直到获取到令牌。
2. tryAcquire 则是非阻塞方法，如果在指定超时时间内获取不到指定数量的令牌，则直接返回false，不阻塞等待。
3. getNowMillis 获取Redis服务器时间，避免业务服务器时间不一致导致的问题，如果业务服务器能保障时间同步，则可从本地获取提高效率。

#### 3. 令牌桶控制工厂类

工厂类负责管理令牌桶控制类，将其缓存在本地，这里使用了Guava中的Cache，一方面避免每次都新建控制类提高效率，另一方面通过控制缓存的最大容量来避免像用户粒度的限流占用过多的内存。

```java
public class RedisRateLimiterFactory {

    private PermitsRedisTemplate permitsRedisTemplate;
    private StringRedisTemplate stringRedisTemplate;
    private DistributedLock distributedLock;

    private Cache<String, RedisRateLimiter> cache = CacheBuilder.newBuilder()
            .initialCapacity(100)  //初始大小
            .maximumSize(10000) // 缓存的最大容量
            .expireAfterAccess(5, TimeUnit.MINUTES) // 缓存在最后一次访问多久之后失效
            .concurrencyLevel(Runtime.getRuntime().availableProcessors()) // 设置并发级别
            .build();

    public RedisRateLimiterFactory(PermitsRedisTemplate permitsRedisTemplate, StringRedisTemplate stringRedisTemplate, DistributedLock distributedLock) {
        this.permitsRedisTemplate = permitsRedisTemplate;
        this.stringRedisTemplate = stringRedisTemplate;
        this.distributedLock = distributedLock;
    }

    /**
     * 创建RateLimiter
     *
     * @param key              RedisRateLimiter本地缓存key
     * @param permitsPerSecond 每秒放入的令牌数
     * @param maxBurstSeconds  最大存储maxBurstSeconds秒生成的令牌
     * @param expire           该令牌桶的redis tty/秒
     * @return RateLimiter
     */
    public RedisRateLimiter build(String key, double permitsPerSecond, double maxBurstSeconds, int expire) {
        if (cache.getIfPresent(key) == null) {
            synchronized (this) {
                if (cache.getIfPresent(key) == null) {
                    cache.put(key, new RedisRateLimiter(permitsRedisTemplate, stringRedisTemplate, distributedLock, permitsPerSecond,
                            maxBurstSeconds, expire));
                }
            }
        }
        return cache.getIfPresent(key);
    }
}
```

#### 4. 注解支持

定义注解 @RateLimit 如下，表示以每秒rate的速率放置令牌，最多保留burst秒的令牌，取令牌的超时时间为timeout，limitType用于控制key类型，目前支持：

1. IP, 根据客户端IP限流
2. USER, 根据用户限流，对于Spring Security可从SecurityContextHolder中获取当前用户信息，如userId
3. METHOD, 根据方法名全局限流，className.methodName，注意避免同时对同一个类中的同名方法做限流控制，否则需要修改获取key的逻辑
4. CUSTOM，自定义，支持表达式解析，如#{id}, #{user.id}

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimit {
    String key() default "";
    String prefix() default "rateLimit:"; //key前缀
    int expire() default 60; // 表示令牌桶模型RedisPermits redis key的过期时间/秒
    double rate() default 1.0; // permitsPerSecond值
    double burst() default 1.0; // maxBurstSeconds值
    int timeout() default 0; // 超时时间/秒
    LimitType limitType() default LimitType.METHOD;
}
```

通过切面的前置增强来为添加了 @RateLimit 注解的方法提供限流控制，如下

```java
@Aspect
@Slf4j
public class RedisLimitAspect {
    //...

    @Before(value = "@annotation(rateLimit)")
    public void rateLimit(JoinPoint  point, RateLimit rateLimit) throws Throwable {
        String key = getKey(point, rateLimit.limitType(), rateLimit.key(), rateLimit.prefix());
        RedisRateLimiter redisRateLimiter = redisRateLimiterFactory.build(key, rateLimit.rate(), rateLimit.burst(), rateLimit.expire());
        if(!redisRateLimiter.tryAcquire(key, rateLimit.timeout(), TimeUnit.SECONDS)){
            ExceptionUtil.rethrowClientSideException(LIMIT_MESSAGE);
        }
    }

    //...
```

### 限量控制

#### 1. 限量控制类

限制一个时间窗口内的访问量，可使用计数器算法，借助Lua脚本执行的原子性来实现。

Lua脚本逻辑：

1. 以需要控制的对象为key（如方法，用户ID，或IP等），当前访问次数为Value，时间窗口值为缓存的过期时间
2. 如果key存在则将其增1，判断当前值是否大于访问量限制值，如果大于则返回0，表示该时间窗口内已达访问量上限，如果小于则返回1表示允许访问
3. 如果key不存在，则将其初始化为1，并设置过期时间，返回1表示允许访问

```java
public class RedisCountLimiter {

    private StringRedisTemplate stringRedisTemplate;

    private static final String LUA_SCRIPT = "local c \nc = redis.call('get',KEYS[1]) \nif c and redis.call('incr',KEYS[1]) > tonumber(ARGV[1]) then return 0 end"
            + " \nif c then return 1 else \nredis.call('set', KEYS[1], 1) \nredis.call('expire', KEYS[1], tonumber(ARGV[2])) \nreturn 1 end";

    private static final int SUCCESS_RESULT = 1;
    private static final int FAIL_RESULT = 0;

    public RedisCountLimiter(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    /**
     * 是否允许访问
     *
     * @param key redis key
     * @param limit 限制次数
     * @param expire 时间段/秒
     * @return 获取成功true，否则false
     * @throws IllegalArgumentException
     */
    public boolean tryAcquire(String key, int limit, int expire) throws IllegalArgumentException {
        RedisScript<Number> redisScript = new DefaultRedisScript<>(LUA_SCRIPT, Number.class);
        Number result = stringRedisTemplate.execute(redisScript, Collections.singletonList(key), String.valueOf(limit), String.valueOf(expire));
        if(result != null && result.intValue() == SUCCESS_RESULT) {
            return true;
        }
        return false;
    }

}
```

#### 2. 注解支持

定义注解 @CountLimit 如下，表示在period时间窗口内，最多允许访问limit次，limitType用于控制key类型，取值与 @RateLimit 同。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface CountLimit {
    String key() default "";
    String prefix() default "countLimit:"; //key前缀
    int limit() default 1;  // expire时间段内限制访问次数
    int period() default 1; // 表示时间段/秒
    LimitType limitType() default LimitType.METHOD;
}
```

同样采用前值增强来为添加了 @CountLimit 注解的方法提供限流控制，如下

```java
@Before(value = "@annotation(countLimit)")
public void countLimit(JoinPoint  point, CountLimit countLimit) throws Throwable {
    String key = getKey(point, countLimit.limitType(), countLimit.key(), countLimit.prefix());
    if (!redisCountLimiter.tryAcquire(key, countLimit.limit(), countLimit.period())) {
        ExceptionUtil.rethrowClientSideException(LIMIT_MESSAGE);
    }
}
```
