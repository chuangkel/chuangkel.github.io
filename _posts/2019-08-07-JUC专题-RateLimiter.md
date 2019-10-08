---
layout:     post
title:	JUC专题
subtitle: 	ReteLimiter
date:       2019-08-07
author:     chuangkel
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - java
---

# ReteLimiter

> ReteLimiter正如其名，速率控制。是google的一个qps控制器。

1. **TPS**：Transactions Per Second（每秒传输的事物处理个数），即服务器**每秒**处理的事务数。TPS包括一条消息入和一条消息出，加上一次用户数据库访问。（业务TPS = CAPS × 每个呼叫平均TPS）

TPS是软件测试结果的测量单位。一个事务是指一个客户机向服务器发送请求然后服务器做出反应的过程。客户机在发送请求时开始计时，收到服务器响应后结束计时，以此来计算使用的时间和完成的事务个数。

2. **QPS**：每秒查询率QPS是对一个特定的查询服务器在**规定时间内**所处理流量多少的衡量标准，在因特网上，作为域名系统服务器的机器的性能经常用每秒查询率来衡量。

对应fetches/sec，即每秒的响应请求数，也即是最大吞吐能力。

### 使用

```java
public static void main(String[] args) throws InterruptedException {
    RateLimiter rateLimiter = RateLimiter.create(5);
    ThreadPoolExecutor pools = new ThreadPoolExecutor(30, 30, 1000L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(10));
    for (int i = 0; i < 30; i++) {
        pools.submit(() -> {
            System.out.println(rateLimiter.acquire());
        });
    }
}
```



### 原理分析

```java
public double acquire(int permits) {
  long microsToWait = reserve(permits);//计算等待时间
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
```

```java
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  resync(nowMicros);
  long returnValue = nextFreeTicketMicros;
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  double freshPermits = requiredPermits - storedPermitsToSpend;
  long waitMicros = storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
      + (long) (freshPermits * stableIntervalMicros);

  try {
    this.nextFreeTicketMicros = LongMath.checkedAdd(nextFreeTicketMicros, waitMicros);
  } catch (ArithmeticException e) {
    this.nextFreeTicketMicros = Long.MAX_VALUE;
  }
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}
```



####  nextFreeTicketMicros

最简单的维持QPS速率的方式就是记住最后一次请求的时间，然后确保再次有请求过来的时候，已经经过了 1/QPS 秒。比如QPS是5 次/秒，只需要确保两次请求时间经过了200ms即可，如果刚好在100ms到达，就会再等待100ms,也就是说，如果一次性需要15个令牌，需要的时间为为3s。但是对于一个长时间没有请求的系统，这样的的设计方式有一定的不合理之处。考虑一个场景：如果一个RateLimiter,每秒产生1个令牌,它一直没有使用过，突然来了一个需要100个令牌的请求，选择等待100s再执行这个请求，显得不太明智，更好的处理方式为立即执行它，然后把接下来的请求推迟100s。

因而RateLimiter本身并不记下最后一次请求的时间，而是记下下一次期望运行的时间（nextFreeTicketMicros）。

> 这种方式带来的一个好处是，可以去判断等待的超时时间是否大于下次运行的时间，以使得能够执行，如果等待的超时时间太短，就能立即返回。
