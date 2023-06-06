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
### 对象头
每个Java对象，都有一个与之关联的监视器(Monitor)对象。
解释: JVM中的对象内存布局可以分为以下3块区域, 对象头`[MarkWord, Klass Pointer, ArrayLength(可选)]`，`示例数据[Instance Data]`，`对齐填充[Padding]`。
<img src="/assets/img/synchronized-markword.png" width="500"/>

### 偏向锁
biased lock，当虚拟机启用偏向锁且锁对象第一次被线程获取的时候，虚拟机将会把对象头中的标志位设为`01`，即偏向模式。

同时使用CAS操作把`获取到这个锁的线程的ID记录在对象的Mark Word`之中,如果CAS操作成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作。

当有另外一个线程去尝试获取这个锁时，`偏向模式就宣告结束`。根据锁对象目前是否处于被锁定的状态，撤销偏向（`Revoke Bias`）后恢复到未锁定（标志位为`01`）或轻量级锁定（标志位为`00`）的状态。

![](/assets/img/synchronized-markword-transfer.gif)

### 轻量级锁
接着看上一个图，如果在另一个线程B获取锁时，持有锁的线程A还没有释放锁，偏向锁会升级为轻量级锁。

虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（`Lock Record`）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word）

然后，虚拟机将使用CAS操作尝试将对象的`Mark Word更新为指向Lock Record的指针`。如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位（Mark Word的最后2bit）将转变为`00`，即表示此对象处于轻量级锁定状态。

如果这个更新操作失败了，虚拟机首先会`检查对象的Mark Word是否指向当前线程的栈帧`，如果只说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要`膨胀(inflate)为重量级锁`，锁标志的状态值变为`10`，`Mark Word中存储的就是指向重量级锁（互斥量）的指针`，后面等待锁的线程也要进入阻塞状态。

### 重量级锁
synchronized重量级是通过对象内部的一个Monitor（监视器）来实现的。Monitor本质是依赖于底层的操作系统的Mutex Lock来实现的。

通过操作系统实现线程之间的切换和调用需要从用户态转换到内核态，这个成本非常高，状态之间的转换需要相对比较长的时间，这就是synchronized在低版本JDK中效率低的原因。因此，这种依赖于操作系统Mutex Lock所实现的锁通常称为重量级锁。


对象监视器(ObjectMonitor)在锁竞争情况下的状态转换示意如下
- 未获取到锁的线程，进入到_EntryList队列中等待获取到锁。
- 获取到锁的线程，ObjectMonitor设置_owner为该线程。
- 获取到锁的线程，wait后释放锁，进入_WaitSet集合被唤醒。
  <img alt="线程进入synchronized状态转换示意图" src="/assets/img/synchronized-state-transfer.png" width="500"/>

## 总结
- synchronized的优化目的，是尽可能低成本的实现多线程之间同步操作。
- 锁只可以升级(偏向锁->轻量级锁->重量级锁)，不可以降级。

参考

1. [Java面试一站到底]()
2. [深入理解Java虚拟机]()
3. [synchronized对象头结构](https://www.cnblogs.com/xiaofuge/p/13895226.html)
4. [objectMonitor.hpp](https://github.com/gskeno/jdk/blob/master/src/hotspot/share/runtime/objectMonitor.hpp)
5. [synchronization openjdk wiki](https://wiki.openjdk.org/display/HotSpot/Synchronization)