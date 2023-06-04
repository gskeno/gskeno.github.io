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


## 1.基本用法

单进程下多线程并发访问共享资源时，需要使用锁来控制资源的访问顺序，本篇介绍synchronized锁。
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