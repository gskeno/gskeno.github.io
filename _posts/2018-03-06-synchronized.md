---
layout: post
title: synchronized锁
tags:
- concurrent-programming
categories: concurrent-programming
description: synchronized锁使用
---

本文主要介绍java中`synchronized`关键字的使用及原理。

<!-- more -->
单进程下多线程并发访问共享资源时，需要使用锁来控制资源的访问顺序，本篇介绍synchronized锁。

## 基本用法

```java
void func(){
    synchronized(lockObject){
        ....
        if(conditionSatisfy){
            doWork();
            lockObject.notifyAll();
        }else{
            lockObject.wait(waitSeconds);
            doAny();
        }
    }
}
```


- 获取到锁(lockObject)的线程，才会进入到**synchronized**代码块，同一时刻只会有一个线程获取到锁。
- 获取到锁的线程，才可以调用wait、notify、notifyAll方法。
- 调用wait方法的线程将会释放锁，且该方法阻塞，当有以下几种情况时，才会返回。
  - 其他线程调用 lockObject.notifyAll()。
  - 其他线程调用 lockObject.notify，且当前被阻塞的线程刚好被唤醒。
  - 其他线程中断 当前被阻塞的线程。
  - 当前线程睡眠 waitSeconds时间后，方法返回。

## 案例

### zk的discovery阶段

zookeeper在leadElection(phase0)阶段之后，会进入discovery(phase1)节点，这个阶段中，要求刚选举出来的leader节点在有限时间内与集群中`大多数的节点连接上`，否则
leader节点崩溃，需要再次取选取leader节点。
```java
    // 该方法只会被两处地方调用到
    // 1. Leader类在phase0阶段结束后，leader调用该方法阻塞，直到过半节点与leader连接上才会返回
    // 2. leader与每个learner连接上时，都会生成一个LearnerHandler对象,其会调用leader.getEpochToPropose方法
    // 这涉及到多个线程，最终当有多数节点与leader连接成功时，才会返回；或者超时抛出异常
    public long getEpochToPropose(long sid, long lastAcceptedEpoch) throws InterruptedException, IOException {
        // 获取锁，同一时间多个线程只有一个能获取锁成功
        synchronized(connectingFollowers) {
            // leader不需要再等待新的learner连接，因为已经有半数以上连接成功
            if (!waitingForNewEpoch) {
                return epoch;
            }
            // epoch比所有learner的lastAcceptedEpoch还要大1
            if (lastAcceptedEpoch >= epoch) {
                epoch = lastAcceptedEpoch+1;
            }
            // leader连接上的follower数量
            connectingFollowers.add(sid);
            QuorumVerifier verifier = self.getQuorumVerifier();
            // leader连接上过半的集群节点
            if (connectingFollowers.contains(self.getId()) && 
                                            verifier.containsQuorum(connectingFollowers)) {
                waitingForNewEpoch = false;
                self.setAcceptedEpoch(epoch);
                // 唤起所有等待的线程
                connectingFollowers.notifyAll();
            } else {
                long start = System.currentTimeMillis();
                long cur = start;
                long end = start + self.getInitLimit()*self.getTickTime();
                while(waitingForNewEpoch && cur < end) {
                    // 等待指定时间，放弃锁
                    connectingFollowers.wait(end - cur);
                    cur = System.currentTimeMillis();
                }
                // 理想情况下，这里被唤醒的线程会发现，已经过半节点与leader成功连接上
                if (waitingForNewEpoch) {
                    throw new InterruptedException("Timeout while waiting for epoch from quorum");        
                }
            }
            return epoch;
        }
    }
```
以上代码，多个线程都会在该方法上阻塞住，直至某个线程中，learner与leader连接成功后，达到过半要求，所有线程才会从方法返回。

## 原理
每个Java对象，都有一个与之关联的监视器(Monitor)对象。
解释: JVM中的对象内存布局可以分为以下3块区域, 对象头`[MarkWord, Klass Pointer, ArrayLength(可选)]`，`示例数据[Instance Data]`，`对齐填充[Padding]`。
<img src="/assets/img/synchronized-markword.png" width="500"/>

当线程进入同步代码块的时候，如果此同步对象(锁对象)没有被锁定，那么虚拟机首先会在`当前线程的栈中创建Lock Record(锁记录)`，用于
存储锁对象的Mark Word的拷贝。

当线程能成功获取到synchronized的对象锁时, MarkWord锁标示位为10，指针指向监视器对象ObjectMonitor。
<img  src="/assets/img/synchronized-relation.png" width="500"/>

对象监视器(ObjectMonitor)在锁竞争情况下的状态转换示意如下
- 未获取到锁的线程，进入到_EntryList队列中等待获取到锁。
- 获取到锁的线程，ObjectMonitor设置_owner为该线程。
- 获取到锁的线程，wait后释放锁，进入_WaitSet集合被唤醒。
  <img alt="线程进入synchronized状态转换示意图" src="/assets/img/synchronized-state-transfer.png" width="500"/>

## 目录


参考

1. [Java面试一站到底]()
2. [synchronized对象头结构](https://www.cnblogs.com/xiaofuge/p/13895226.html)