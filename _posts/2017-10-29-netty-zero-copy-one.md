---
layout: post
title: Netty 之 Zero-copy 的实现（上）
excerpt: "维基百科中对 Zero-copy 的解释是:零拷贝技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。"
modified: 2017-10-29 23:10:00 +08:00
categories: articles
tags: [Java, Netty]
comments: true
share: true
---

维基百科中对 `Zero-copy` 的解释是

> 零拷贝技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。

维基百科里提到的零拷贝是在硬件和操作系统层面的，而本文主要介绍的是Netty在应用层面的优化。不过需要注意的是，零拷贝并非字面意义上的没有内存拷贝，而是避免多余的拷贝操作，即使是系统层的零拷贝也有从设备到内存，内存到设备的数据拷贝过程。

Netty 的零拷贝体现在以下几个方面

* `ByteBuf` 的 `slice` 操作并不会拷贝一份新的 `ByteBuf` 内存空间，而是直接借用原来的 `ByteBuf` ，只是独立地保存读写索引。
* Netty 提供了 `CompositeByteBuf` 类，可以将多个 `ByteBuf` 组合成一个逻辑上的 `ByteBuf` 。
* Netty 的 `FileRegion` 中包装了 `NIO` 的 `FileChannel.transferTo()`方法，该方法在底层系统支持的情况下会调用 `sendfile` 方法，从而在传输文件时避免了用户态的内存拷贝。
* Netty 的 `PooledDirectByteBuf` 等类中封装了 `NIO` 的 `DirectByteBuffer` ，而 `DirectByteBuffer` 是直接在 jvm 堆外分配的内存，省去了堆外内存向堆内存拷贝的开销。

下面来简单介绍下这几种方式。

## slice

以下以 `AbstractUnpooledSlicedByteBuf` 为例讲解 `slice` 的零拷贝原理，至于内存池化的实现 `PooledSlicedByteBuf` ，因为内存池要通过引用计数来控制内存的释放，所以代码里会出现很多与本文主题无关的逻辑，这里就不拿来举栗子了。

```java 
//切片ByteBuf的构造函数，其中字段adjustment为切片ByteBuf相对于被切片ByteBuf的偏移量，两个ByteBuf共用一块内存空间,字段buffer为实际存储数据的ByteBuf
AbstractUnpooledSlicedByteBuf(ByteBuf buffer, int index, int length) {
    super(length);
    checkSliceOutOfBounds(index, length, buffer);//检查slice是否越界
    
    if (buffer instanceof AbstractUnpooledSlicedByteBuf) {
        //如果被切片ByteBuf也是AbstractUnpooledSlicedByteBuf对象
        this.buffer = ((AbstractUnpooledSlicedByteBuf) buffer).buffer;
        adjustment = ((AbstractUnpooledSlicedByteBuf) buffer).adjustment + index;
    } else if (buffer instanceof DuplicatedByteBuf) {
        //如果被切片ByteBuf为DuplicatedByteBuf对象，则用unwrap得到实际存储数据的ByteBuf赋值buffer
        this.buffer = buffer.unwrap();
        adjustment = index;
    } else {
        //如果被切片ByteBuf为一般ByteBuf对象，则直接赋值buffer
        this.buffer = buffer;
        adjustment = index;
    }

    initLength(length);
    writerIndex(length);
}
```

以上为 `AbstractUnpooledSlicedByteBuf` 类的构造函数，比较简单，就不详细介绍了。

```java
@Override
public ByteBuf getBytes(int index, ByteBuffer dst) {
    checkIndex0(index, dst.remaining());//检查是否越界
    unwrap().getBytes(idx(index), dst);
    return this;
}

@Override
public ByteBuf unwrap() {
    return buffer;
}

private int idx(int index) {
    return index + adjustment;
}
```

这是 `AbstractUnpooledSlicedByteBuf` 重载的 `getBytes` 方法，可以看到 `AbstractUnpooledSlicedByteBuf` 是直接在封装的 `ByteBuf` 上取的字节，但是重新计算了索引，加上了相对偏移量。

## CompositeByteBuf

在有些场景里，我们的数据会分散在多个 `ByteBuf` 上，但是我们又希望将这些 `ByteBuf` 聚合在一个 `ByteBuf` 里处理。这里最直观的想法是将所有 `ByteBuf` 的数据拷贝到一个 `ByteBuf` 上，但是这样会有大量的内存拷贝操作，产生很大的CPU开销。

而 `CompositeByteBuf` 可以很好地解决这个问题，正如名字一样，这是一个复合 `ByteBuf` ，内部由很多的 `ByteBuf` 组成，但 `CompositeByteBuf` 给它们做了一层封装，可以直接以 `ByteBuf` 的接口操作它们。

```java
/**
 * Precondition is that {@code buffer != null}.
 */
private int addComponent0(boolean increaseWriterIndex, int cIndex, ByteBuf buffer) {
    assert buffer != null;
    boolean wasAdded = false;
    try {
        //检查新增的component的索引是否合法
        checkComponentIndex(cIndex);

        //buffer的长度
        int readableBytes = buffer.readableBytes();

        // No need to consolidate - just add a component to the list.
        @SuppressWarnings("deprecation")
        //统一为大端ByteBuf
        Component c = new Component(buffer.order(ByteOrder.BIG_ENDIAN).slice());
        if (cIndex == components.size()) {
            //如果索引等于components的大小，则加在components尾部
            wasAdded = components.add(c);
            if (cIndex == 0) {
                //如果components中只有一个元素
                c.endOffset = readableBytes;
            } else {
                //如果components中有多个元素
                Component prev = components.get(cIndex - 1);
                c.offset = prev.endOffset;
                c.endOffset = c.offset + readableBytes;
            }
        } else {
            //如果新的ByteBuf是插在components中间
            components.add(cIndex, c);
            wasAdded = true;
            if (readableBytes != 0) {
                //如果components的大小不为0,则依次更新cIndex之后的所有components的offset和endOffset
                updateComponentOffsets(cIndex);
            }
        }
        if (increaseWriterIndex) {
            //如果要更新writerIndex
            writerIndex(writerIndex() + buffer.readableBytes());
        }
        return cIndex;
    } finally {
        if (!wasAdded) {
            //如果没添加成功，则释放ByteBuf
            buffer.release();
        }
    }
}
```

这是添加一个新的 `ByteBuf` 的逻辑，核心是 `offset` 和 `endOffset` ，分别指代一个   `ByteBuf` 在 `CompositeByteBuf` 中开始和结束的索引，它们是唯一标记了这个 `ByteBuf` 在 `CompositeByteBuf` 中的位置。

弄清楚了这个，我们会发现上面的代码无外乎做了两件事：
1. 把 `ByteBuf` 封装成 `Component` 加到 `components` 合适的位置上
2. 使 `components` 里的每个 `Component` 的 `offset` 和 `endOffset` 值都正确

```java
@Override
public CompositeByteBuf getBytes(int index, ByteBuf dst, int dstIndex, int length) {
    //检查索引是否越界
    checkDstIndex(index, length, dstIndex, dst.capacity());
    if (length == 0) {
        return this;
    }

    //用二分搜索查找index对应的Component在components中的索引
    int i = toComponentIndex(index);
    //循环读直至length为0
    while (length > 0) {
        Component c = components.get(i);
        ByteBuf s = c.buf;
        int adjustment = c.offset;
        //取length和ByteBuf剩余字节数中的较小值
        int localLength = Math.min(length, s.capacity() - (index - adjustment));
        //开始索引为index - c.offset，而不是0
        s.getBytes(index - adjustment, dst, dstIndex, localLength);
        index += localLength;
        dstIndex += localLength;
        length -= localLength;
        i ++;
    }
    return this;
}

/**
 * Return the index for the given offset
 */
public int toComponentIndex(int offset) {
    checkIndex(offset);

    for (int low = 0, high = components.size(); low <= high;) {
        int mid = low + high >>> 1;
        Component c = components.get(mid);
        if (offset >= c.endOffset) {
            low = mid + 1;
        } else if (offset < c.offset) {
            high = mid - 1;
        } else {
            return mid;
        }
    }

    throw new Error("should not reach here");
}
```

同样是以 `getBytes` 为例，可以看到是先将 `index` 转换成对应 `Component` 在 `components` 中的索引，然后从这个 `Component` 开始往后循环取字节，直到读完。这里有个小trick，因为 `components` 是有序排列的，所以 `toComponentIndex` 做索引转换时没有直接遍历，而是用的二分查找。

今天写得有点累了，留个坑，下一篇再填上。