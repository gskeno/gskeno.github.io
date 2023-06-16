---
layout: post
title: RoaringBitmap的FormatSpec
tags:
- RoaringBitmap
categories: RoaringBitmap
description: RoaringBitmap的存储格式FormatSpec
---

本文主要介绍`RoaringBitmap`的序列化与反序列化。

<!-- more -->

## 通用存储格式
回顾下，**RoaringBitmap**内部实现最重要的组件就是**Container**， 当前共有三种实现，分别是
*ArrayContainer*，*BitmapContainer* ， *RunContainer*。

**RoaringBitmap**序列化后，其字节态下的内存布局可分为以下四个部分，先引入一张作者提供的内存布局示意图

<img alt="layout" src="/assets/img/roaringbitmap-format-spec.jpg" width="500"/>

### Cookie header 
在这个部分，你可以了解到当前要序列化的`RoaringBitmap`对象内部是否有用到**RunContainer**容器。

如果没有用到RunContainer，则前4个字节可以写入`魔数`即`SERIAL_COOKIE_NO_RUNCONTAINER=12346`，再接着4个字节写入容器的个数`size`，共计8个字节。

如果有用到RunContainer，则前4个字节可以写入`SERIAL_COOKIE | ((size - 1) << 16)`，其中SERIAL_COOKIE=`12347`，其后写入`(size + 7) / 8`
个字节，当其内部的某个bit位值为1时，表示该位对应的Container是RunContainer。


### Descriptive header
容器的描述性头，对于每个Container的描述，使用4个字节。高16位用来表达该Container所对应的key，低16位用来表达该Container内部元素的个数(实际存储的是元素个数cardinality - 1)

### Offset header
每个Container实体相对于**Cookie header**位置的相对偏移量描述。当以下两个条件任一成立时(不可能同时成立，因为这两个条件互斥)，**OffSet header**才会存储每个Container所在位置相对Cookie header的偏移量。

- 条件一: <u>roaringBitmap内部没有使用RunContainer。</u>
- 条件二: <u>roaringBitmap内部有使用RunContainer，且不少于4个。</u>

### Container storage
Container容器实体存储，到这里就会遍历存储每个Container内存储的数据。


## 序列化
看一段官方的[测试代码](https://github.com/RoaringBitmap/RoaringBitmap/blob/master/examples/src/main/java/SerializeToDiskExample.java),
```java
public static void main(String[] args) throws IOException {
        RoaringBitmap rb = new RoaringBitmap();
        for (int k = 0; k < 100000; k+= 1000) {
            rb.add(k);
        }
        for (int k = 100000; k < 200000; ++k) {
            rb.add(3*k);
        }
        for (int k = 700000; k < 800000; ++k) {
            rb.add(k);
        }
        String file1 = "bitmapwithoutruns.bin";
        try (DataOutputStream out = new DataOutputStream(new FileOutputStream(file1))) {
          rb.serialize(out);
        }
        rb.runOptimize();
        String file2 = "bitmapwithruns.bin";
        try (DataOutputStream out = new DataOutputStream(new FileOutputStream(file2))) {
          rb.serialize(out);
        }
        // verify
        RoaringBitmap rbtest = new RoaringBitmap();
        try (DataInputStream in = new DataInputStream(new FileInputStream(file1))) {
          rbtest.deserialize(in);
        }
        if(!rbtest.equals(rb)) throw new RuntimeException("bug!");
        try (DataInputStream in = new DataInputStream(new FileInputStream(file2))) {
          rbtest.deserialize(in);
        }
        if(!rbtest.equals(rb)) throw new RuntimeException("bug!");
        System.out.println("Serialized bitmaps to "+file1+" and "+file2);
    }
````
该段代码构建了一个*RoaringBitmap*对象，并序列化到输出文件中(可选是否进行步长压缩)，并尝试进行反序列化重建*RoaringBitmap*对象。

序列化的核心代码如下(加了个人注释)
```java
  /**
   * Serialize.
   *
   * The current bitmap is not modified.
   *
   * @param out the DataOutput stream
   * @throws IOException Signals that an I/O exception has occurred.
   */
  public void serialize(DataOutput out) throws IOException {
    int startOffset = 0;
    boolean hasrun = hasRunContainer();
    // Cookie header部分开始
    if (hasrun) {
      out.writeInt(Integer.reverseBytes(SERIAL_COOKIE | ((size - 1) << 16)));
      // tips, 每一个字节有8bit, 对于任意一个container，可以通过位标记判断其是否属于RunContainer
      byte[] bitmapOfRunContainers = new byte[(size + 7) / 8];
      // 存储runFlagBitSets
      for (int i = 0; i < size; ++i) {
        if (this.values[i] instanceof RunContainer) {
          bitmapOfRunContainers[i / 8] |= (1 << (i % 8)); // 某个bit位值为1，就表示其映射的Conatiner是RunContainer
        }
      }
      out.write(bitmapOfRunContainers);
      if (this.size < NO_OFFSET_THRESHOLD) {
        // RunContainer个数小于4个时，Offset header部分为空即可
        startOffset = 4 + 4 * this.size + bitmapOfRunContainers.length;
      } else {
        // RunContainer个数小于4个时，Offset header部分占用了 4 * this.size个字节
        startOffset = 4 + 8 * this.size + bitmapOfRunContainers.length;
      }
    } else { // backwards compatibility
      out.writeInt(Integer.reverseBytes(SERIAL_COOKIE_NO_RUNCONTAINER)); // 将int以字节为单位进行反转
      out.writeInt(Integer.reverseBytes(size));// 桶<->容器， 个数进行反转
      startOffset = 4 + 4 + 4 * this.size + 4 * this.size;
    }
    // Descriptive header部分开始
    for (int k = 0; k < size; ++k) {
      out.writeShort(Character.reverseBytes(this.keys[k]));
      out.writeShort(Character.reverseBytes((char) (this.values[k].getCardinality() - 1)));
    }
    // Offset header部分开始
    if ((!hasrun) || (this.size >= NO_OFFSET_THRESHOLD)) {
      // writing the containers offsets
      for (int k = 0; k < this.size; k++) {
        out.writeInt(Integer.reverseBytes(startOffset));
        startOffset = startOffset + this.values[k].getArraySizeInBytes();
      }
    }
    // Container storage部分开始
    for (int k = 0; k < size; ++k) {
      values[k].writeArray(out);
    }
  }
```

## 反序列化
序列化之后，继续看反序列化代码
```java
  /**
   * Deserialize. If the DataInput is available as a byte[] or a ByteBuffer, you could prefer
   * relying on {@link #deserialize(ByteBuffer)}. If the InputStream is &gt;= 8kB, you could prefer
   * relying on {@link #deserialize(DataInput, byte[])};
   *
   * @param in the DataInput stream
   * @throws IOException Signals that an I/O exception has occurred.
   * @throws InvalidRoaringFormat if a Roaring Bitmap cookie is missing.
   */
  public void deserialize(DataInput in) throws IOException {
    this.clear();
    // little endian 序列化生成文件时，使用的是小端序，这里需要再还原
    final int cookie = Integer.reverseBytes(in.readInt());
    if ((cookie & 0xFFFF) != SERIAL_COOKIE && cookie != SERIAL_COOKIE_NO_RUNCONTAINER) {
      throw new InvalidRoaringFormat("I failed to find a valid cookie.");
    }
    // 从Cookie header中解析出 容器的个数。注意，这里是无符号右移
    this.size = ((cookie & 0xFFFF) == SERIAL_COOKIE) ? (cookie >>> 16) + 1
        : Integer.reverseBytes(in.readInt());
    // logically we cannot have more than (1<<16) containers.
    if(this.size > (1<<16)) {
      throw new InvalidRoaringFormat("Size too large");
    }
    if ((this.keys == null) || (this.keys.length < this.size)) {
      this.keys = new char[this.size];
      this.values = new Container[this.size];
    }

    // 从Cookie header中尝试解析出 哪些容器是RunContainer
    byte[] bitmapOfRunContainers = null;
    boolean hasrun = (cookie & 0xFFFF) == SERIAL_COOKIE;
    if (hasrun) {
      bitmapOfRunContainers = new byte[(size + 7) / 8];
      in.readFully(bitmapOfRunContainers);
    }

    final char[] keys = new char[this.size];
    final int[] cardinalities = new int[this.size];
    final boolean[] isBitmap = new boolean[this.size];
    for (int k = 0; k < this.size; ++k) {
      // 从Descriptive header中解析出每个容器container相关联的key(标的高16位)
      // 以及每个容器内标的数量
      keys[k] = Character.reverseBytes(in.readChar());
      cardinalities[k] = 1 + (0xFFFF & Character.reverseBytes(in.readChar()));
      // 容器内元素个数>4096个时，可以武断地任务容器类型是BitmapContainer
      isBitmap[k] = cardinalities[k] > ArrayContainer.DEFAULT_MAX_SIZE;
      // 但是，其也有可能是RunContainer
      if (bitmapOfRunContainers != null && (bitmapOfRunContainers[k / 8] & (1 << (k % 8))) != 0) {
        isBitmap[k] = false;
      }
    }
    // 这里很奇妙，跳过了offset header部分，这里记录的是每个container的位置
    if ((!hasrun) || (this.size >= NO_OFFSET_THRESHOLD)) {
      // skipping the offsets
      in.skipBytes(this.size * 4);
    }
    // 遍历读取每个container，类型不同，反序列化重建的方式也不同
    // Reading the containers
    for (int k = 0; k < this.size; ++k) {
      Container val;
      if (isBitmap[k]) {
        final long[] bitmapArray = new long[BitmapContainer.MAX_CAPACITY / 64];
        // little endian
        for (int l = 0; l < bitmapArray.length; ++l) {
          bitmapArray[l] = Long.reverseBytes(in.readLong());
        }
        val = new BitmapContainer(bitmapArray, cardinalities[k]);
      } else if (bitmapOfRunContainers != null
          && ((bitmapOfRunContainers[k / 8] & (1 << (k % 8))) != 0)) {
        // cf RunContainer.writeArray()
        int nbrruns = (Character.reverseBytes(in.readChar()));
        final char[] lengthsAndValues = new char[2 * nbrruns];

        for (int j = 0; j < 2 * nbrruns; ++j) {
          lengthsAndValues[j] = Character.reverseBytes(in.readChar());
        }
        val = new RunContainer(lengthsAndValues, nbrruns);
      } else {
        final char[] charArray = new char[cardinalities[k]];
        for (int l = 0; l < charArray.length; ++l) {
          charArray[l] = Character.reverseBytes(in.readChar());
        }
        val = new ArrayContainer(charArray);
      }
      this.keys[k] = keys[k];
      this.values[k] = val;
    }
  }
```

## 总结
- 1个bug
> [RoaringFormatSpec](https://github.com/RoaringBitmap/RoaringFormatSpec) Cookie header的描述，经本人提出issue后，作者
> 将`The cookie header spans either 32 bits or 64 bits` 修改为了`The cookie header spans either 64 bits or 32 bits followed by a variable number of bytes`

- 2个细节
> a,大小端序。 默认情况下，java生成的文件为大端序，而其他语言读取文件时，大多都是按照小端序来读取的，所以在序列化时，都有使用`reverseBytes`方法将字节序列转化为小端序，读取时也同样适用了`reverseBytes`方法。
> 
> b,offset header。序列化时，其可能占用了部分字节，存储了每个容器的位置，但是反序列化时，并没有利用到，而是直接跳过。[lemire](https://lemire.me/en/#about)在他的文章中说是为了便于
> 随机访问任意一个container，但是当我们并不需要该功能时，这对于内存来说是一个浪费。






参考

- [github RoaringFormatSpec](https://github.com/RoaringBitmap/RoaringFormatSpec/tree/master#standard-32-bit-roaring-bitmap)
