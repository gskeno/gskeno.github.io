---
layout: post
title: guava中的限流器
tags:
- guava
- concurrent-programming
categories: concurrent-programming
description: RateLimiter的工作原理
---

限流器一般有几种工作原理(限流算法)？每种算法的特点是什么？guava中的`RateLimiter`用的是哪种算法？
本文主要介绍guava中的限流器工作原理。

<!-- more -->

## 基本工作原理图
<img src="/assets/img/guava-ratelimiter.png" width="500"/>

- 令牌桶的容量(可以装载的令牌数量)是有限的，1秒内允许通过的请求数作为令牌桶的容量。
  
- 一个请求过来，一般来说，必须获取到要求的令牌数量，才可放行通过，但是这不是绝对的，`令牌可以预支`。


## 类图以及使用分析
<img src="/assets/img/guava-ratelimiter-diagram.jpg" width="800"/>

有两个实现类，分别是`SmoothBursty`，可用于恒定速度场景(可偶尔有尖峰)。
`SmoothWarmingUp`，可用于预热场景。

### SmoothBursty
```java
    @Test
    public void testSmoothBurstyUse() throws InterruptedException {
        // 每秒可以通过5个请求，也即每秒放入5个令牌
        RateLimiter rateLimiter = RateLimiter.create(5);
        // 这里睡眠2s，令牌桶里也不会有10个令牌；
        // 因为框架中 maxBurstSeconds 强制设置为1，表示令牌桶只会保留1s的令牌数即5个
        // 令牌桶容量就是5
        Thread.sleep(2000);

        int requests = 20;
        CountDownLatch countDownLatch = new CountDownLatch(requests);
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                try {
                    countDownLatch.countDown();
                    countDownLatch.await();
                    // 前6个请求耗时都为0
                    // 前5个请求耗时为0，是因为桶里有5个令牌
                    // 第6个请求耗时为0，是因为"预支"原理，先将令牌预先取到，不耗时；
                    // 由于平均0.2s才生产1个令牌，所以下次请求获取令牌需要等待0.2s
                    System.out.println(Thread.currentThread().getName() + ":" + rateLimiter.acquire(1));
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        };
        ExecutorService service = Executors.newFixedThreadPool(requests);
        for (int i = 0; i < requests; i++) {
            service.submit(runnable);
        }
        service.awaitTermination(60, TimeUnit.SECONDS);
    }
```
**源码解读**直接看下方的参考文档即可，这里不再赘述。

`Smooth`原意平稳，上方示例中，如果每0.2s过来一个请求，将会很平稳。

`Bursty`原意突发，脉冲，上方示例中，如果1s内没有请求，将会保留1s内对应投放数量的令牌，在加上当前1s内若有大量请求涌入，则理论上在当前这`1s内我们有2倍令牌桶容量的令牌`，所以遇到突发流量时，我们的qps理论可支持2倍日常qps。

### SmoothWarmingUp
预热场景下，因为代码是冷的，启动时，qps不宜过高，随着时间推移，qps逐渐升高到一定程度后，保持稳定速度运行。

也即启动时，获取令牌的用时较长；随后，获取令牌锁时间逐渐变短，最后趋于稳定。

<img src="/assets/img/guava-ratelimiter-smooth-warm-up.jpg" width="800"/>

- 在预热期生成一个令牌的耗时是平稳期的3倍。
- 临界值令牌数`thresholdPermits`是最大令牌数`maxPermits`的一半。
- 最大令牌数`maxPermits`是框架内部计算得到。


```java
    @Test
    public void testWarmUpRateLimiter() throws InterruptedException {
        // 每秒生成5个令牌，预热期设置为2s(即2s后令牌生成速度变为0.2s生成一个，在此之前，生成速度更慢) ，预热期令牌生成速度会低于稳定期
        RateLimiter rateLimiter = RateLimiter.create(5, 2, TimeUnit.SECONDS);
        Thread.sleep(2000);
        for (int i = 0; i < 20; i++) {
            //0.0
            //0.553163   开始时获取令牌耗时最长，随后逐渐减小
            //0.476724
            //0.396308
            //0.319991
            //0.237281
            //0.198168    2s后，获取令牌耗时 趋于稳定
            //0.195875
            //0.196701
            //... 
            System.out.println(rateLimiter.acquire(1));
        }
    }
```


## 思考

- 真实生产世界里，一般不会调用`acquire`阻塞方法，因为如果等待时间过长，线程资源迟迟得不到释放，就会创建越来越多的请求线程，系统负载压力逐渐升高，因此大多应该使用 `tryAcquire(int permits, long timeout, TimeUnit unit)`方法。如果不能在指定时间内获取到令牌，返回false；否则，等待相应的时间获取到令牌并返回true。

- 多个并发请求同时到达时，都会去尝试拿去令牌，则令牌的计数可能会有线程安全问题。acquire方法是阻塞方法，一定要拿到令牌，在计算获取令牌需要的等待时间时(内部会先读后写已存储的令牌数)，会加锁。

- `漏桶算法`的限流原理是什么？业界有开源的技术实现方案吗？
  
- sentinel的限流原理是什么？(滑动窗口)




参考

- [RateLimiter](https://github.com/gskeno/guava/blob/note1/guava/src/com/google/common/util/concurrent/RateLimiter.java)
- [SmoothRateLimiter](https://github.com/gskeno/guava/blob/note1/guava/src/com/google/common/util/concurrent/SmoothRateLimiter.java)
- [令牌桶算法原理及应用](https://mp.weixin.qq.com/s/yk-b6JB-IC5G_MyDCf4Q8Q)
- [逐行拆解Guava限流器RateLimiter](https://zhuanlan.zhihu.com/p/439682111?utm_id=0)
- [uber-go 漏桶限流器](https://www.cyhone.com/articles/analysis-of-uber-go-ratelimit/)
- [sentinel](https://sentinelguard.io/zh-cn/index.html)
- [sentinel 滑动窗口](https://zhuanlan.zhihu.com/p/383064126)
- [sentinel wiki](https://github.com/alibaba/Sentinel/wiki/Sentinel-%E6%A0%B8%E5%BF%83%E7%B1%BB%E8%A7%A3%E6%9E%90)

