#Buffers
**本章包括**

 - ByteBuf
 - ByteBufHolder
 - ByteBufAllocator
 - 在这些接口上分配和执行操作
当你想传输数据的时候必定会涉及到缓存。Java的NIO API有在即的Buffer类，如同我们在前些章节中所讨论的那样，该实现相当的有局限性而且不是最优的。使用JDK的ByteBuffer经常是既沉重又复杂。缓存是一个重要的组件，提供需要的层次是必须的任务，应该是API的一部分。
幸运的是Netty提供了很强大的缓存实现，用来代表一串字节数组，帮助你操作原始字节或者定制POJO。新夫人缓存类型ByteBuf在Netty中等价于JDK中的ByteBuffer。ByteBuf的任务是在Netty管线中传递数据。它从基础设计来解决JDK中的ByteBuffer的问题，来满足网络应用开发者的日常问题，使得它们更加高效。这种方式要比JDK的ByteBuffer更具有优势。在本书的其余部分，我会把Netty的缓存接口和实现叫做数据容器，来和Java的实现进行区分，Java的实现继续叫做Java 缓存API。
在本章，你将会了解Betty的缓存API，它比JDK的缓存实现有哪些优势，什么使它如此强大，它为什么比JDK的缓存API更灵活。你将对如何访问Netty框架中交换的数据有更深入的了解，以及如何使用它。本章也是后续章节的基础，在Netty中缓存API无处不在。
因为穿梭于Netty的ChannelPipeline和ChannelHandler的实现，所以在Netty应用的日常开发中很普遍。ChannelPipeline和ChannelHandler将在第6章中详细讨论。

##Buffer API

Netty的缓存API包含两个接口：

 - ByteBuf
 - ByteBufHolder

Netty使用引用计数技术来决定什么时候释放Buf和申请的资源安全，知道Netty使用引用计数很有用，所有的都自动完成。这样允许Netty使用池和其它技巧进行加速，保持内存使用在理智的水平上。你不需要做任何事情使得这一切发生：开发Netty应用的时候，试图尽快的处理和释放池化的资源。
Netty的缓存API提供了多项优势：

 - 如果需要，你可以定义自己的缓存类型。
 - 通过内部的组合缓存类型，实现了透明的零拷贝。
 - 容量会根据需要进行扩展，像StringBuffer那样
 - 不需要调用flip()在读、写模式之间切换
 - 读、写索引的分割
 - 方法链
 - 引用技术
 - 池化

我们在下一部分深入的看看这些部分，包括池化。我们先看看字节容器。

##ByteBuf-字节数据容器
当你需要和远程节点例如数据库进行通讯的时候，则通讯需要通过字节来完成。由于这个还有其它他原因，需要一个高效、方便和易于使用的数据结构，Netty的ByteBuf实现满足这些需求，还有使得它成为一个完美的数据容器，最优化了存储和操作字节数组。
ByteBuf是一个数据容器，允许你高效的从它添加或者获取字节数组。为了使得它更容易操作，它使用了两个索引：一个读一个写。这允许你顺序的读取数据，并且跳回去重新读取。你所要做的只是调整读索引和再一次的读取。

###它是如何工作的
当一些东西被写入到ByteBuf中的时候，它所谓writerIndex会增加你所写的字节的数目。当你进行读操作，readerIndex会增加。你可以进行读取字节数组直到writerIndex和readerIndex相同。ByteBuf不可读，再次读取会导致IndexOutOfBoundsException异常，和数组的读取类似。
调用任何以read或者write开头的缓存方法都会自动的增加读或者写的索引。同样存在相对操作来设置和获取字节数组。这些操作不会移动索引，但是操作在给定的相对索引上。
一个ByteBuf可以设置最大容量值来设置最大数据容量，试图将写索引移除到容量之外会导致异常，默认限制是Integer.MAX_VALUE。
图5.2展示了ByteBuf的布局。
![16个字节的ByteBuf，读写索引初始值都为0](http://img.blog.csdn.net/20151019112243660)
**图5.1 16个字节的ByteBuf，读写索引初始值都为0**
如图5.1中展示的那样，ByteBuf和字节数组很相似，最显著的不同是添加了读和写索引来控制缓存数据的访问。
在后面的部分会更详细的介绍ByteBuf上的操作。现在，我们看一下你可能使用到的不同的ByteBuf类型。

###不同类型的ByteBuf

当使用Netty的时候，存在三种不同类型的ByteBuf（内部使用的还有很多种）。你可以自己实现，在这里超出范围。我们来看看你最感兴趣的已经提供的类型。

####HEAP BUFFERS
使用最多的ByteBuf类型是将数据存储在JVM的堆空间。这是通过将数据存在在支持的数组中完成的。这一类型可以快速分配，当你不使用池的时候也可以快速回收。它同样提供了直接访问支持数组的方式，这样使得更容易和历史代码进行交互。

**列表5.1 访问支持数组**

```
ByteBuf heapBuf = ...;
if (heapBuf.hasArray()) {#1
	byte[] array = heapBuf.array();#2
	int offset = heapBuf.arrayOffset() + heapBuf.position();#3
	int length = heapBuf.readableBytes();#4
	YourImpl.method(array, offset, length);#5
}
#1 检查ByteBuf是否是数组支持的
#2 获取支持数组的引用
#3 计算数组中第一个字节的偏移量
#4 获取可读字节的数目
#5 使用字节数组，偏移量和长度来调用方法
```

如果ByteBuf是“nonheap”类型，访问数组会导致UnsupportedOperationException异常。所以使用hasArray()检查ByteBuf是否是数组支持是一个好习惯。如果你以前使用过JDK的ByteBuffer这种模式应该很熟悉。

####DIRECT BUFFERS
另外一个ByteBuf实现是直接缓存。直接意味着它直接分配内存，它在堆内存之外。你在堆空间内看不到内存使用。当你计算你的应用的最大内存使用的时候一定要考虑进去，如何进行限制，只考虑堆空间是不够的。直接缓存是最优的当你通过套接字传输数据的时候。实际上，当你使用非直接缓存传输数据的时候，JVM会在发送它之前，在内部拷贝一份到直接缓存。
直接缓存的缺点是相对于堆缓存，直接缓存的分配和回收开销很大。这就是事Netty支持池化的原因，从而消除这一问题。另外一个缺点是你再也不能通过支持数组访问数据，如果历史代码非要使用数组，你可以在使用之前拷贝一份数据。
以下列表显示了如何获得数据数组，使用数组调用方法即使你无法直接访问支持数组。
**列表5.2 访问数据**

```
ByteBuf directBuf = ...;
if (!directBuf.hasArray()) {	#1
	int length = directBuf.readableBytes();	#2
	byte[] array = new byte[length];	#3
	directBuf.getBytes(array);	#4
	YourImpl.method(array, 0, array.length);	#5
}
#1 检查ByteBuf不是被数组直接支持
#2 获取可读取字节的数目
#3 分配长度为可读取数目的数组
#4 读取字节到数组中
#5 使用数组，偏移量和长度作为参数来调用方法
```
它需要更多工作，并且涉及一个拷贝操作。如果你希望访问数据，并且希望使用数组，你最好使用堆缓存。

####COMPOSITE BUFFERS
你可能遇到的最后一个ByteBuf实现是CompositeByteBuf。它的作用如同它的名字显示的那样；它允许组合不同的ByteBuf实例，并且提供了一个视图。你同样可以动态的添加和删除它们，所以它像一个列表。如果你以前使用过JDK的ByteBuffer，在那里你可没有这一特性。因为CompositeByteBuf只是其他ByteBuf的视图，<font color=red>hasArray()方法会返回false，因为它可能包含多个ByteBuf实例类型可能是直接的和非直接的类型。</font>


----------
<font color=red>
**备注（译者）**
</font>
<font size=1>
这一点不确切，它的源代码实现如下：
</font>

```
    @Override
    public boolean hasArray() {
        switch (components.size()) {
        case 0:
            return true;
        case 1:
            return components.get(0).buf.hasArray();
        default:
            return false;
        }
    }

```

----------

例如，一个消息可能由两部分组成：头部和主体。在一个模块化的应用中，这两部分可能由不同的模块产生，到消息被发送的时候装配。当然，你可以一直使用县共同的主体部分，只是修改头部。在这里不用每次都分配新的缓存很有意义。
这里使用CompositeByteBuf很完美，不需要内存拷贝，API和非组合缓存相同。
图5.2展示了CompositeByteBuf用于组合头部和主体
![持有头部和主体的CompositeByteBuf](http://img.blog.csdn.net/20151019134928038)
**图5.2 持有头部和主体的CompositeByteBuf**
如果你使用过JDK的ByteBuffer，这就不可能。组合两个ByteBuffer的方法是创建一个数组来持有它们或者创建一个新的ByteBuffer，将两者的内容拷贝到新创建的缓存中。以下的列表显示了这些。
**列表5.3 组合历史的JDK ByteBuffer**

```
// 使用数组来组合它们
ByteBuffer[] message = new ByteBuffer[] { header, body };
// 使用复制来合并它们
ByteBuffer message2 = ByteBuffer.allocate(
                          header.remaining()+ body.remaining());
message2.put(header);
message2.put(body);
message2.flip();
```

列表5.3中的两种方式都有缺点：处理数组，使得你不能使得API保持一致，如果两者都想支持。拷贝存在性能开销，它简单但不是最优的。
我们具体的看看CompositeByteBuf。以下列表给出视图。
**列表5.4 CompositeByteBuf实战**

```
CompositeByteBuf compBuf = ...;
ByteBuf heapBuf = ...;
ByteBuf directBuf = ...;
compBuf.addComponent(heapBuf, directBuf);	#1
.....
compBuf.removeComponent(0);	#2
for (ByteBuf buf: compBuf) {	#3
	System.out.println(buf.toString());
}
#1 向组合缓存中添加ByteBuf实例
#2 移除索引为0的ByteBuf(这里是heapBuf)
#3 遍历组合缓存中的所有ByteBuf实例
```

还存在更多的方法。Netty API文档很明确，你可以参照它，通过参照API文档，你可以很容易了解。
同样，因为CompositeByteBuf 的自身特性，你不能直接访问支持数组。获取数组方式的本地缓存类似。
**列表5.5 访问数据**

```
CompositeBytebuf compBuf = ...;
if (!compBuf.hasArray()) {	#1
	int length = compBuf.readableBytes();	#2
	byte[] array = new byte[length];	#3
	compBuf.getBytes(array);	#4
	YourImpl.method(array, 0, array.length);	#5
}
#1 检查是否存在支持数组，对于组合缓存返回false
#2 获取可读字节的数目
#3 分配长度为可读字节数目的字节数组
#4 将字节读到数组中
#5 通过数组，偏移量，长度调用方法
```
CompositeBytebuf是ByteBuf的子类，你可以像操作其它缓存那样操作它，但还可以进行其它的操作。
当使用CompositeBytebuf的时候，Netty将优化在套接字上的读、写操作。这意味着当读写套接字时使用聚合、分散不会招致性能问题或者JDK实现的内存泄漏问题。所有的这些都是在Netty核心来完成，所以你完全没必要考虑它，直到内部的一些优化实现也没有坏处。
当使用ByteBuffer的时候具体的类例如CompositeBytebuf并不出现。可见Netty的缓存API十分丰富相对于JDK的java.nio实现。

##ByteBuf的字节操作

ByteBuf提供了许多操作允许修改内容，或者直接读取。你会发现，它很像JDK的ByteBuffer，但作为增强，它提供了更好的用户体验和性能。
###随机访问索引
如同普通的基础字节数组，ByteBuf使用基于0的索引。这意味着第一个字节的索引是0，最后一个字节的数组是capacity-1。例如我可以遍历一个缓存的所有字节，不用考虑内部实现。
**列表 5.7 访问数据**

```
ByteBuf buffer = ...;
for (int i = 0; i < buffer.capacity(); i ++) {
	byte b = buffer.getByte(i);
	System.out.println((char) b);
}
```

注意这里的访问不会增加readerIndex和writerIndex索引。如果你想在操作的同时增加索引，你可以调用readerIndex(index) 和writerIndex(index) 。

###顺序访问索引

ByteBuf提供了两个指针变量来支持顺序读取和写操作-读操作的readerIndex和写操作writerIndex。而JDK的ByteBuffer只有一个，所以你需要调用flip()在读和写模式之间进行切换。图5.3显示了缓存如何被这两个真正分为三个区域。
![ByteBuf区域](http://img.blog.csdn.net/20151019144411898)
**图5.3 ByteBuf区域**

```
#1 可以丢弃的区域，它们已经被读取
#2 可读取的区域
#3 可写入的区域
```

###丢弃字节
可丢弃字节部分包含的字节已经被读操作读取，所以可以被丢弃，刚开始，这一部分的大小为0，当读操作执行的时候，它的大小随着readerIndex增加。这里的读操作指的是“read"操作，"get"操作不会移动readerIndex。通过调用discardReadBytes()可以回收没有用的空间。
图5.4展示了在调用discardReadBytes()前的各个部分。
![discardReadBytes调用前](http://img.blog.csdn.net/20151019145609046)
**图5.4 discardReadBytes调用前**

```
#1 可以丢弃的区域，它们已经被读取
#2 可读取的区域
#3 可写入的区域
```
你可以看到，可丢弃字节部分包含一些空间可以重复使用。这个可以通过调用discardReadBytes()获得。
图5.5显示了调用discardReadBytes()影响的部分。

![discardReadBytes调用后](http://img.blog.csdn.net/20151019145701733)
**图5.5 discardReadBytes调用后**

```
#1持有可读取内容，索引为0
#2可写入的部分，这一部分增长了原先丢弃区域的大小
```
注意调用discardReadBytes()并不能保证可写入字节内容。可写入字节在大多数情况下不会移动，也有可能被不同的数据填充，具体依赖于基础缓存实现。
你可能不断的调用discardReadBytes使得ByteBuf提供更多的写空间。注意discardReadBytes()很可能引起内存拷贝，你需要将可读字节移动到ByteBuf的开始处。这样的操作可能会影响性能，所以在需要它并且受益于它的时候使用。例如你需要尽快的释放内存的时候可以使用。

###可读取字节（具体内容）
这一区域是具体数据存储的区域任何以read或者skip开始的操作将会获取当前readerIndex的数据，并且索引增加读取字节数目。如果读操作的参数是ByteBuf，并且不存在目标索引，指定的目标缓存的writerIndex会一起增加。
如果剩下的内容不充足，会触发IndexOutOfBoundException异常。新分配的，包装的，拷贝的缓存的readerIndex的默认值为0。
以下列出如何读取所有可读数据。
**列表5.8 读取数据**

```
// Iterates the readable bytes of a buffer.
ByteBuf buffer = ...;
while (buffer.readable()) {
	System.out.println(buffer.readByte());
}
```

###可写入字节
这一部分是没定义区域，它需要填充。任何以write开头的操作会在writerIndex处写入数据，并且writerIndex会增加读取的数目。如果写入操作的参数是ByteBuf，并且没有指定源索引，则参数的readerIndex会一起增加。
入股没有剩余的可写区域，会触发IndexOutOfBoundException异常。新分配的缓存的writerIndex默认值是0。
以下列表限制向缓存中写入数据直到空间用完。
**列表5.9 写入数据**

```
// Fills the writable bytes of a buffer with random integers.
ByteBuf buffer = ...;
while (buffer.writableBytes() >= 4) {
	buffer.writeInt(random.nextInt());
}
```
###清除缓存索引
通过调用clear()你可以将writerIndex和readerIndex同时设为0.它不会清除缓存的内容，只是清除两个指针。<font color=red>语义上它和JDK中ByteBuffer.clear()不同。(JDK中也不会实际清除缓冲区的内容)</font>
我们来看一看它的功能。图5.6 显示了ByteBuf的三个不同区域。
![clear方法调用之前](http://img.blog.csdn.net/20151019160139478)
**图5.6 clear方法调用之前**
```
#1 可以丢弃的区域，它们已经被读取
#2 可读取的区域
#3 可写入的区域
```
像以前一样，它包含3个部分。当clear()调用的时候，你会发现变化。图5.7展示了调用clear()之后夫人ByteBuf。

![clear方法调用之后](http://img.blog.csdn.net/20151019160232205)
**图5.7 clear方法调用之后**

```
#1 现在ByteBuf和它的容量一样大，所有的东西都可写
```
和discardReadBytes()操作相比，clear()开销很小，因为它只调节指针的位置，而不需要拷贝内存。
###查询操作
不同的indexOf()方法帮助你定位满足一定条件的索引。和简单的静态字节搜索相比，复杂的动态连续搜索可以通过实现ByteBufProcessor来完成。
如果你解码边长数据，例如结尾符字符串，你会发现bytesBefore(byte)方法很有用。假设你在写一个应用，它需要和闪光套接字交互，它会使用结尾符字符串。使用bytesBefore()你就很容易的消耗数据，你就不要手动的杜宇每一个字节并且检查空字节。如果没有ByteBufProcessor，这些都需要你自己完成。这样更高效，因为在处理的过程中不需要范围检核。
###标识和重置
前面已经说明，每一个缓存存在两个标识索引。一个存储readerIndex，另外一个存储writerIndex。你可以通过调用reset方法重置这两个索引。他和InputStream中的mark很reset方法类似除了没有读的限制数目。
当然你可以通过代用readerIndex(int)和writerIndex(int)将索引指定到明确的位置上。记住将writerIndex和readerIndex设置到非法的位置上会引起IndexOutOfBoundException异常。
###衍生缓存

为了创建已有缓存的视图，你可以调用duplicate(),slice(),slice(int,int),readOnly()或者order(ByteOrder)。一个衍生缓存拥有独立的writerIndex，readerIndex和标识索引，但是共享内部数据，这一点和NIO中ByteBuffer类似。因为它共享内部数据，创建它开销很小，如果你只需要一部分可以使用slice()操作，这也是被推荐的。
如果需要现有缓存的一个完整副本，使用copy()或者copy(int,int)方法替代。下表显示如何分割ByteBuf。


