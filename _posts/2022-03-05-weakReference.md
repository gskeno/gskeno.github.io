---
layout: post
title: 弱引用WeakReference
tags:
- java
- guava
categories: java
description: WeakReference
---

Java中有四种引用类型，`强引用`, `软引用`, `弱引用`, `虚引用`。

> 软引用，[SoftReference](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/SoftReference.html), 当jvm内存不足时，垃圾回收器才会回收软引用对象。

> 弱引用，[WeakReference](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/WeakReference.html)，垃圾回收时，`肯定会回收弱引用对象`。

本文主要介绍弱引用。

<!-- more -->

## 使用示例
创建弱引用对象后，立马执行垃圾回收`System.gc`，下述执行不会报错，`Integer 对象 128` 被回收掉，只剩下`空壳子weakReference`。后文会将WeakReference内部的referent称为`原对象`，而自身称为引用对象。
```java
    @Test
    public void testWeak(){
         WeakReference<Object> weakReference = new WeakReference<>(Integer.valueOf(128));
        Assert.assertNotNull(weakReference.get());
        // gc后弱引用 引用的对象被回收
        System.gc();
        Assert.assertNull(weakReference.get());
    }
```
> tips
> 1. 如果128换成127会怎样?
> 2. 如果referent不是Integer,而是字符串会怎样？比如 new WeakReference<>(new String("abce")) 和 new WeakReference<>("abce") 会有什么不同？


另外，当一个对象**只有弱引用对象指向它**时，GC发生的时候该对象才能被回收。如下:
```java
    @Test
    public void testWeakOnGC(){
        Object o = new Object();
        WeakReference<Object> weakReference1 = new WeakReference<>(o, null);
        Assert.assertNotNull(weakReference1.get());
        System.gc();
        // 此时，原对象还有栈中的o指向它
        Assert.assertNotNull(weakReference1.get());
        
        // 原对象只剩下弱引用对象weakReference1指向了
        o = null;
        System.gc();
        Assert.assertNull(weakReference1.get());
    }
```

### 引用队列
在创建弱引用对象时，可以为其关联一个[引用队列](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/ReferenceQueue.html)。当原对象被回收时，引用对象会被放入引用队列。示例如下:
```java
    @Test
    public void testQueue() throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        WeakReference<Object> weakReference = new WeakReference<>(new String("abcd"), queue);
        Assert.assertNotNull(weakReference.get());
        // gc后 原对象被回收
        System.gc();
        // 原对象确实被回收，即String("abce")被回收
        Assert.assertNull(weakReference.get());
        Thread.sleep(1000);
        // 引用对象 被放入队列中
        Assert.assertTrue(weakReference == queue.poll());
    }
```

>`思考` 

**原对象被回收了，引用对象只是个壳子，引用对象还在队列中，但是引用对象内部的属性referent已经是null了，引用对象在这里还有什么作用呢？**


在这个例子中，我们在队列中获取到弱引用对象后，没有任何事情可以做。但在另一个场景下我们是可以做些事情的，比如 [WeakHashMap](https://docs.oracle.com/javase/8/docs/api/java/util/WeakHashMap.html)。

另外，我们还要注意到，代码中有`sleep 1s`，这是因为弱引用对象入队列是一个单独线程`ReferenceHandler`来执行，其优先级为10，比main线程高(main线程5)，但这**并不意味**着弱引用对象入队列**一定早于**工作线程将弱引用对象从队列中取出来。因此，我们这里睡眠1s。

### WeakHashMap
WeakHashMap是一个弱引用HashMap，其中的Entry节点就是弱引用对象(WeakReference的子类)，这个弱引用对象不同于父类WeakReference，内部除了拥有**原对象**(Entry.key)，还拥有`value`, `Entry next`等成员，这些成员可不会被垃圾回收器回收，因为Entry实例还持有其引用。
```java
    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;
        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
    }
```
ok，那么垃圾回收时，一个Entry实例的key被回收了，但是value却没有回收，value这个时候并没有什么作用了(***忽略遍历所有entry的value这种无意义的场景***)，业务上不应该在访问这个value，这value应该被清除，甚至这个Entry都应该被清除。

对的，确实应该这样。注意到一个事实,
<font color=red>Entry是一个弱引用对象，当其内部原对象被GC回收时，
Entry这个弱引用对象，是会被放入引用队列里的。
我们可以将其从引用队列中取出来，将其value置为空，并将该节点从链表中移除。</font>
如下所示:
```java
    /**
     * Expunges stale 擦除陈旧的 entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        // 当垃圾节点在链表的首部时
                        if (prev == e)
                            table[i] = next;
                        // 当垃圾节点不在链表的首部时
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```

移除无效的entry节点发生在 <font color = red> 求容量，遍历entry节点，扩容 </font> 这三个场景

看一个测试用例
```java
    /**
     * WeakHashMap内部的Entry继承WeakReference，实现了Map.Entry,
     * 当只有WeakHashMap引用key，key不再被其他对象引用时，
     * gc回收时会回收该key,且Entry节点被放入到引用队列中，
     * size()方法会诱发无效Entry节点的删除
     */
    @Test
    public void testWeakHashMap() throws InterruptedException {
        WeakHashMap<String,String> whm = new WeakHashMap<>();
        String key1 = new String("张三丰");
        String value1 = "武当创始人";
        String key2 = new String("郭襄");
        String value2 = "峨眉创始人";
        whm.put(key1, value1);
        whm.put(key2, value2);
        Assert.assertEquals(whm.size() , 2);
        // key1 不再被强引用
        key1 = null;
        // gc后，弱引用Entry节点被放入到引用队列中
        System.gc();
        // 确保弱引用Entry真的已经被放入到队列中
        Thread.sleep(1000);
        // size()方法，会诱发"无效"Entry节点的删除
        Assert.assertEquals(whm.size(), 1);
    }
```

## guava中的WeakKey
guava中可以基于弱引用，软引用，建立Cache。以弱引用Key为例，当执行GC时，Key被回收，而无效的
Entry节点在之后有另一个Entry节点作用在同一个segment上时将被删除。测试用例如下
```java
    @Test
    public void testWeakReferenceBaseEviction() throws InterruptedException {
        Cache<String, String> cache = CacheBuilder.newBuilder().weakKeys().removalListener(
                new RemovalListener<String, String>() {
                    @Override
                    public void onRemoval(RemovalNotification<String, String> notification) {
                        // 如果value是大对象，要主动将其失效掉
                        System.out.println(notification.getKey() + ":" + notification.getValue()
                                + " is " + notification.wasEvicted());
                    }
                }).build();

        String key = UUID.randomUUID().toString();
        String value = UUID.randomUUID().toString();
        cache.put(key, value);
        Assert.assertEquals(cache.asMap().toString(), "{" + key + "=" + value + "}");
        // map中的key只有弱引用，可以被会收
        key = null;
        System.gc();
        Thread.sleep(1000);
        // 得到 {}， key已经被垃圾回收，但是相应的entry还没有被清除
        Assert.assertEquals(cache.asMap().toString(), "{}");
        Assert.assertTrue(cache.size() == 1);
        // 引用队列是基于每个segment上的，这里的put(或者get)操作如果与上一个entry不在一个segment上时，上一个segment中
        // 陈旧的节点不会被影响到，失效entry不会被移除，上面的监听器也不会有响应。
        cache.put(UUID.randomUUID().toString(), UUID.randomUUID().toString());
        Assert.assertTrue(cache.size() == 1);
    }
```
- asMap打印时，会过滤掉无效的节点，但是不会删除该节点。
- 当put，get操作时，会定位到一个segment，其上的无效节点才会被删除。
- size方法获取的时cacheMap中节点的个数(各个segment上的节点个数相加)，
  无效的节点还未删除时，也会纳入统计


guava cache中的一些核心代码片段如下
```java
  /**
   * Gets the value from an entry. Returns null if the entry is invalid, partially-collected, 部分成员被垃圾回收
   * loading, or expired. Unlike {@link Segment#getLiveValue} this method does not attempt to clean
   * up stale entries. As such it should only be called outside a segment context, such as during
   * iteration. 该方法不会清除陈旧的entry,只能在segment上下文外部调用，比如遍历cache场景
   */
  // LocalCache.getLiveValue()
  @CheckForNull
  V getLiveValue(ReferenceEntry<K, V> entry, long now) {
    // key被弱引用时，GC后，key被回收，值为null
    if (entry.getKey() == null) {
      return null;
    }
    // value被弱引用时，GC后，value 被回收，值为null
    V value = entry.getValueReference().get();
    if (value == null) {
      return null;
    }

    if (isExpired(entry, now)) {
      return null;
    }
    return value;
  }

    /**
     * Gets the value from an entry. Returns null if the entry is invalid, partially-collected,
     * loading, or expired.
     */
    // Segment.getLiveValue 会清除内部无效Entry
    V getLiveValue(ReferenceEntry<K, V> entry, long now) {
      if (entry.getKey() == null) {
        tryDrainReferenceQueues();
        return null;
      }
      V value = entry.getValueReference().get();
      if (value == null) {
        tryDrainReferenceQueues();
        return null;
      }

      if (map.isExpired(entry, now)) {
        tryExpireEntries(now);
        return null;
      }
      return value;
    }

```

## 总结

- 当发生GC时，弱引用指向的原对象如果不再被其他对象所引用，则原对象会被垃圾回收。
- 可以为弱引用对象关联一个引用队列，当原对象被垃圾回收时，弱引用对象(WeakReference)会被放入
  引用队列。(注意:放入引用队列里的是弱引用对象，原对象已经被回收。)
- 弱引用对象在GC时被放入引用队列是`java.lang.ref.Reference.ReferenceHandler`做的，
  其优先级比main主线程还高，但是这并不意味着原对象一被GC，弱引用对象立马就会被放入引用队列，
  这期间可能有<font color=red>时间差 </font>。
- WeakHashMap基于引用队列，找到弱引用对象，其属性key已经被垃圾回收，WeakHashMap会主动将value
  置为null，帮助GC。
- guava中的cache可以基于弱引用做数据逐出，cache内部中的每个segment都会有一个相应的
  ReferenceQueue，当弱引用对象指向的原对象被垃圾回收时，弱引用对象将被放入引用队列中，
  以供后来合适机会将entry节点从segement中删除。

参考

- [java8 docs](https://docs.oracle.com/javase/8/docs/api/)
- [reference-based-eviction](https://github.com/google/guava/wiki/CachesExplained#reference-based-eviction)

