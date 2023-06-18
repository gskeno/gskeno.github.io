---
layout: post
title: nio的Buffer api如何使用
tags:
- nio
categories: nio
description: nio的Buffer api如何使用
---

本文主要介绍Java NIO中Buffer API的使用方式，属于科普型介绍文章。

<!-- more -->

## Buffer
Buffer就是一个线性有限的存储**primitive**类型数据的容器。除了其内部存储的数据(比如byte数组)外，核心的参数有三个`capacity`,`limit`,`position` 。

> capacity: 容器里存储的元素数量，非负数，永远不会变。
> 
> limit: 容器里第一个禁止读(或写)的位置，非负数， 永远 <= capacity。
> 
> position: 容器里当前读(或写)的位置，非负数，永远 <= limit。

### mark and reset
`mark`标记，就是用来做为标记的，当对一个Buffer进行读(或写)时，可以执行*mark*方法，此时就会将当前的`position`赋值给mark，这就相当于做了一个标记。
```java
    /**
     * Sets this buffer's mark at its position.
     *
     * @return  This buffer
     */
    public final Buffer mark() {
        mark = position;
        return this;
    }
```
那么什么时候能用到这个标记*mark*呢? `reset`方法可以用到，当执行*reset*方法时，会将*mark*值赋值给*position*，之后从*position*这个位置开始读(或写)，注意，*reset*并不会改变mark的值。
```java
    /**
     * Resets this buffer's position to the previously-marked position.
     *
     * <p> Invoking this method neither changes nor discards the mark's
     * value. </p>
     *
     * @return  This buffer
     *
     * @throws  InvalidMarkException
     *          If the mark has not been set
     */
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
```

### clear,flip,rewind
> clear，原意为清理。position重置为0，limit重置为capacity, mark重置为-1。

```java
    /**
     * Clears this buffer.  The position is set to zero, the limit is set to
     * the capacity, and the mark is discarded.
     *
     * <p> Invoke this method before using a sequence of channel-read or
     * <i>put</i> operations to fill this buffer.  For example:
     *
     * <blockquote><pre>
     * buf.clear();     // Prepare buffer for reading
     * in.read(buf);    // Read data</pre></blockquote>
     *
     * <p> This method does not actually erase the data in the buffer, but it
     * is named as if it did because it will most often be used in situations
     * in which that might as well be the case. </p>
     *
     * @return  This buffer
     */
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```

-----

> flip，原意为翻转。limit重置为当前position，position重置为0，mark重置为-1。对于buffer而言，`写完之后切读`，适合调用此方法。

```java
    /**
     * Flips this buffer.  The limit is set to the current position and then
     * the position is set to zero.  If the mark is defined then it is
     * discarded.
     *
     * <p> After a sequence of channel-read or <i>put</i> operations, invoke
     * this method to prepare for a sequence of channel-write or relative
     * <i>get</i> operations.  For example:
     *
     * <blockquote><pre>
     * buf.put(magic);    // Prepend header
     * in.read(buf);      // Read data into rest of buffer
     * buf.flip();        // Flip buffer
     * out.write(buf);    // Write header + data to channel</pre></blockquote>
     *
     * <p> This method is often used in conjunction with the {@link
     * java.nio.ByteBuffer#compact compact} method when transferring data from
     * one place to another.  </p>
     *
     * @return  This buffer
     */
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
```

------


> rewind，原意为倒带，退回。position重置为0，mark标记重置为-1，使用此方法可以从头开始重新读或写。

```java
    /**
     * Rewinds this buffer.  The position is set to zero and the mark is
     * discarded.
     *
     * <p> Invoke this method before a sequence of channel-write or <i>get</i>
     * operations, assuming that the limit has already been set
     * appropriately.  For example:
     *
     * <blockquote><pre>
     * out.write(buf);    // Write remaining data
     * buf.rewind();      // Rewind buffer
     * buf.get(array);    // Copy data into array</pre></blockquote>
     *
     * @return  This buffer
     */
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
```

## ByteBuffer
对于每个原生类型(`primitive`)，除了Boolean，都有一个对应的`Buffer`子类，比如*IntBuffer*，*LongBuffer*，而最常见最常用的就是`ByteBuffer`，以字节为容器基本元素。

### slice
slice，原意是切片。调用`byteBuffer.slip()`方法，将会从*byteBuffer*的*position*位置作为`切点`，切分出一个新的*ByteBuffer*并返回，新旧buffer在该公共切片上的数据
是`共享`的。

看一下`HeaperBuffer`中的实现
```java
    public ByteBuffer slice() {
        return new HeapByteBuffer(hb,
                                        -1,
                                        0,
                                        this.remaining(),
                                        this.remaining(),
                                        this.position() + offset);
    }

    protected HeapByteBuffer(byte[] buf,
                                   int mark, int pos, int lim, int cap,
                                   int off)
    {
        super(mark, pos, lim, cap, buf, off);
    }

    ByteBuffer(int mark, int pos, int lim, int cap,  
                 byte[] hb, int offset)
    {
        super(mark, pos, lim, cap);
        this.hb = hb;
        this.offset = offset;
    }
```
可以看到*ByteBuffer*内部引入了`offset`的概念，且当执行*sliece*方法后，新的*ByteBuffer*与原*ByteBuffer*共享内部字节数组`byte[] hb`下标*offset*之后的数据。


## 总结

- NIO中的ByteBuffer内部保存一个字节数组，通过几个内部变量`position`，`limit`，`capacity`来存储当前字节数组的使用状态，同时也提供了一系列方法，如`clear`,`flip`,`rewind`,`slip`来帮助使用者操作字节。

- 可以通过一些开源库，如[RoaringBitmap](https://github.com/RoaringBitmap/RoaringBitmap)来深入学习理解其使用方式。


参考

- [javadoc Buffer](https://docs.oracle.com/javase/8/docs/api/java/nio/Buffer.html)
- [javadoc ByteBuffer](https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html)
