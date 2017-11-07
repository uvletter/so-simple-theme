---
layout: post
title: Netty 之 Zero-copy 的实现（下）
excerpt: 上一篇说到了 `CompositeByteBuf` ，这一篇接着上篇的内容讲下去。
modified: 2017-11-07 21:57:00 +08:00
categories: articles
tags: [Java, Netty]
comments: true
share: true
---


上一篇说到了 `CompositeByteBuf` ，这一篇接着上篇的内容讲下去。

## FileRegion

让我们看一个Netty官方的例子

```java
// netty-netty-4.1.16.Final\example\src\main\java\io\netty\example\file\FileServerHandler.java
public void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
    RandomAccessFile raf = null;
    long length = -1;
    try {
        raf = new RandomAccessFile(msg, "r");
        length = raf.length();
    } catch (Exception e) {
        ctx.writeAndFlush("ERR: " + e.getClass().getSimpleName() + ": " + e.getMessage() + '\n');
        return;
    } finally {
        if (length < 0 && raf != null) {
            raf.close();
        }
    }

    ctx.write("OK: " + raf.length() + '\n');
    if (ctx.pipeline().get(SslHandler.class) == null) {
        // SSL not enabled - can use zero-copy file transfer.
        ctx.write(new DefaultFileRegion(raf.getChannel(), 0, length));
    } else {
        // SSL enabled - cannot use zero-copy file transfer.
        ctx.write(new ChunkedFile(raf));
    }
    ctx.writeAndFlush("\n");
}
```

可以看到在没开启SSL的情况下handler是通过 `DefaultFileRegion` 传输文件的，`DefaultFileRegion` 是 `FileRegion` 接口的一个实现，在 `FileRegion` 的接口说明上可以看到这样一行：

> A region of a file that is sent via a *Channel* which supports <a href="http://en.wikipedia.org/wiki/Zero-copy">zero-copy file transfer</a>.

其内部实现是通过 Java NIO 的 `FileChannel.transferTo()` 方法。

先让我们看一段传输文件的一般写法吧。

```java
File.read(file, buf, len);
Socket.send(socket, buf, len);
```

尽管上面的代码看起来很简单，但在内部实际包含了4次用户态-内核态上下文切换，和4次数据拷贝。

![上下文切换示意图][2]

![数据拷贝示意图][1]

其中步骤有：

1. `read()` 调用导致了一次用户态到内核态的上下文切换，在内部，一个 `sys_read()` （或等价函数）被执行来从文件中读取数据。第一次拷贝是由 `DMA` 引擎将数据从磁盘文件存储到内核地址空间缓冲区。

2. 被请求长度的数据从内核的读缓冲区拷贝到用户缓冲区，并且 `read()` 调用返回。这个返回导致又一次从内核态到用户态的上下文切换。现在数据是存储在用户地址空间缓冲区。

3. `send()` 调用引起了一次从用户态到内核态的上下文切换。第三次拷贝又一次将数据放进内核地址空间缓冲区，尽管这一次是放进另一个不同的缓冲区，和目标socket联系在一起。

4. `send()` 系统调用返回，产生了第四次上下文切换。第四次拷贝由 `DMA` 引擎独立异步地将数据从内核缓冲区传递给协议引擎。

看到这里可能有些读者会问，`read()` 函数为什么不直接将数据拷贝到用户地址空间的缓冲区，而要经内核地址空间的缓冲区转一次手，这不是白白多了一次拷贝操作吗？

对IO函数有了解的童鞋肯定知道，在IO函数的背后有一个缓冲区 `buffer` ，我们平常的读和写操作并不是直接和底层硬件设备打交道，而是通过一块叫缓冲区的内存区域缓存数据来间接读写。我们知道，和CPU、高速缓存、内存比，磁盘、网卡这些设备属于慢速设备，交换一次数据要花很多时间，同时会消耗总线传输带宽，所以我们要尽量降低和这些设备打交道的频率，而使用缓冲区中转数据就是为了这个目的。

引用参考资料2中的话：

> Using the intermediate buffer on the read side allows the kernel buffer to act as a "readahead cache" when the application hasn't asked for as much data as the kernel buffer holds. This significantly improves performance when the requested data amount is less than the kernel buffer size. The intermediate buffer on the write side allows the write to complete asynchronously. 

大意是说，在读一侧的中间缓冲区可以作为预读缓存显著提高当请求数据大小小于内核缓冲区大小时的读性能，在写一侧的中间缓冲区可以允许写操作异步完成。

不过，当读请求数据的大小大于内核缓冲区时这个策略本身会变成一个性能瓶颈，数据在到达应用程序前会在磁盘、内核缓冲区、用户缓冲区之间反复多次拷贝。

让我们重新思考下上面的过程，会发现第二次和第三次的拷贝其实是不必要的，我们为什么不直接从读缓冲区将数据传输到socket缓冲区呢？实际上这就是 `transferTo()` 所做的。

```java
public void transferTo(long position, long count, WritableByteChannel target);
```

 `transferTo()` 方法将数据从一个文件channel传输到一个可写channel。在内部它依赖于操作系统对 `Zero-copy` 的支持，在UNIX/Linux系统上， `transferTo()` 实际会调用 `sendfile()` 这个系统函数，将数据从一个文件描述符传输到另一个。

```c
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

![transferTo上下文切换][4]

![transferTo数据拷贝][3]

可以看到我们将上下文切换已经从4次减少到2次，同时把数据拷贝从4次减少到3次（只有1次 `CPU` 参与，另外2次 `DMA` 引擎完成），那么我们可不可以把这唯一一次CPU参与的数据拷贝也省掉呢？

如果网卡支持 `gather operations` 内核就可以进一步减少数据拷贝。在 Linux kernels 2.4 及更新的版本，socket 描述符已经为适应这个需求做了变化。现在这个方法不仅减少了上下文切换，而且消除了CPU参与的数据拷贝。API接口是一样的，但是实质已经发生了变化：

1. `transferTo()` 方法引起 `DMA` 引擎将文件内容拷贝到内核缓冲区。

2. 没有数据从内核缓冲区拷贝到socket缓冲区，只有携带位置和长度信息的描述符被追加到socket缓冲区上， `DMA` 引擎直接将内核缓冲区的数据传递到协议引擎，全程无需CPU拷贝数据。

![transferTo和gather operation数据拷贝][5]

到这里大家对 `transferTo()` 实现 `Zero-copy` 的原理应该很清楚了吧， `FileRegion` 是对 `transferTo()` 的一个封装，所以也是一样的。

## DirectByteBuffer

`DirectByteBuffer` 是 Java NIO 用于实现堆外内存的一个很重要的类，而 `Netty` 用 `DirectByteBuffer` 作为`PooledDirectByteBuf` 和 `UnpooledDirectByteBuf` 的内部数据容器（区别于 `HeapByteBuf` 直接用 `byte[]` 作为数据容器），以使用和操纵堆外内存。要了解 `DirectByteBuffer` 怎么实现 `Zero-copy`，我们要先了解 `DirectByteBuffer` 这个类和堆外内存。

![DirectByteBuffer继承关系][7]

`DirectByteBuffer` 类本身还是位于Java内存模型的堆中，堆内存是JVM可以直接管控、操纵的内存，而 `DirectByteBuffer` 中的 `unsafe.allocateMemory(size)` 是一个native方法，这个方法分配的是堆外内存，通过 C 的 `malloc` 来进行分配的。分配的内存是在系统本地的内存，并不在Java的内存中，也不属于JVM管控范围，所以在 `DirectByteBuffer` 一定会存在某种方式操纵堆外内存。

在 `DirectByteBuffer` 的父类 `Buffer` 中有个 `address` 属性：

```java
// Used only by direct buffers
// NOTE: hoisted here for speed in JNI GetDirectBufferAddress
long address;
```

`address` 只会被 `DirectByteBuffer` 使用到，之所以将 `address` 属性升级放在 `Buffer` 中，是为了在JNI调用 `GetDirectBufferAddress` 时提高效率。

`address` 表示分配的堆外内存的地址，JNI对这个堆外内存的操作都是通过这个 `address` 实现的。

在回答为什么堆外内存可以实现 `Zero-copy` 前，我们先要明确一个结论，那就是 **操作系统不能直接访问Java堆的内存区域**。

JNI方法访问的内存区域是一个已经确定的内存区域，如果该内存地址指向的是一个Java堆内存的话，在操作系统正在访问这个内存地址时，JVM在这个时候进行了GC操作，GC经常会进行先标记再压缩的操作，即将可回收的空间做标记，然后清空标记位置的内存，然后会进行一个压缩，压缩会涉及到对象的移动，以腾出一块更加完整、连续的内存空间，以容纳更大的新对象，但是这个移动的过程会使JNI调用的数据错乱。

为了解决上述的问题，一般会做一个堆内存与堆外内存之间数据拷贝的操作：比如我们要完成一个从文件中读数据到堆内存的操作，即 `FileChannelImpl.read(HeapByteBuffer)` ,这里实际上File I/O会将数据读到堆外内存中，然后堆外内存再将数据拷贝到堆内存，这样我们就读到了文件中的内容。

![FileChannelImpl.read][6]

```java
static int read(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
    if (var1.isReadOnly()) {
        throw new IllegalArgumentException("Read-only buffer");
    } else if (var1 instanceof DirectBuffer) {
        return readIntoNativeBuffer(var0, var1, var2, var4);
    } else {
        // 分配临时的堆外内存
        ByteBuffer var5 = Util.getTemporaryDirectBuffer(var1.remaining());

        int var7;
        try {
            // File I/O 操作会将数据读入到堆外内存中
            int var6 = readIntoNativeBuffer(var0, var5, var2, var4);
            var5.flip();
            if (var6 > 0) {
                // 将堆外内存的数据拷贝到堆外内存中
                var1.put(var5);
            }

            var7 = var6;
        } finally {
            // 里面会调用DirectBuffer.cleaner().clean()来释放临时的堆外内存
            Util.offerFirstTemporaryDirectBuffer(var5);
        }

        return var7;
    }
}
```

而写操作则反之，我们会将堆内存的数据先写到堆外内存，然后操作系统会将堆外内存的数据写入到堆内存。

如果我们直接使用堆外内存，即直接在堆外分配一块内存来存储数据，这样就可以避免堆内存和堆外内存之间的数据拷贝，进行I/O操作时直接将堆外内存地址传给JNI的I/O函数就好了。

这里引用一段 stackoverflow 里关于 [ByteBuffer.allocate() vs. ByteBuffer.allocateDirect()](https://stackoverflow.com/questions/5670862/bytebuffer-allocate-vs-bytebuffer-allocatedirect) 的讨论：

> Operating systems perform I/O operations on memory areas. These memory areas, as far as the operating system is concerned, are contiguous sequences of bytes. It's no surprise then that only byte buffers are eligible to participate in I/O operations. Also recall that the operating system will directly access the address space of the process, in this case the JVM process, to transfer the data. This means that memory areas that are targets of I/O perations must be contiguous sequences of bytes. In the JVM, an array of bytes may not be stored contiguously in memory, or the Garbage Collector could move it at any time. Arrays are objects in Java, and the way data is stored inside that object could vary from one JVM implementation to another.

这也是堆外内存 `DirectByteBuffer` 被引进的原因。

但是同时，创建和销毁一块堆外内存的花销要比堆内存昂贵得多，这是因为堆外内存的创建和销毁要通过系统相关的 native 方法，而不是在 Java 堆上直接由 JVM 操控。为了更有效地重用堆外内存，Netty 引入了内存池机制手动管理内存，这是一个 Java 版的 Jemalloc，后面有机会再写篇文章专门介绍这个，因为我现在也不是很懂（先挖个坑）。

## 总结

到这里关于 Netty 实现 `Zero-copy` 的4种机制，切片共用，组合缓冲区，操作系统层的零拷贝以及堆外内存已经介绍完了，因为本人也是最近刚开始学习 Netty 框架，对很多知识点掌握得还不是很通透，如果文章写得有什么不妥的地方还请大家不吝赐教。

## 参考

1) [对于 Netty ByteBuf 的零拷贝(Zero Copy) 的理解](https://segmentfault.com/a/1190000007560884)

2) [Efficient data transfer through zero copy](https://www.ibm.com/developerworks/library/j-zerocopy/)

3) [堆外内存 之 DirectByteBuffer 详解](http://www.jianshu.com/p/007052ee3773)


  [1]: /images/bVzP6n.gif
  [2]: /images/bVzP6i.gif
  [3]: /images/bVzP68.gif
  [4]: /images/bVzP60.gif
  [5]: /images/bVzP7j.gif
  [6]: /images/bVXW50.png
  [7]: /images/bVX1sJ.png