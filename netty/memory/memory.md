堆外内存分配比堆内存分配性能更差

堆外内存难以控制，如果内存泄漏，那么很难排查 

为什么在执行网络IO或者文件IO时,推荐使用堆外内存?

1. 使用DirectBuffer就会少一次内存拷贝。如果是非DirectBuffer,JDK会先创建一个DirectBuffer,再去执行真正的写操作.这是因为当我们把一个地址通过JNI传递给底层的C库的时候,有一个基本的要求,就是这个地址上的内容不能失效.然而,在GC管理下的对象是会在Java堆中移动的.也就是说,有可能我把一个地址传给底层的write,但是这段内存却因为gc整理内存而失效了.所以必须把待发送的数据放在一个GC管不着的地方,所以再每次调用native方法之前,先将数据拷贝一份到堆外内存,再使用堆外内存传送数据.

2. 可以减少gc的压力

## Arena

顾名思义，PoolArena负责缓存池的全局调度，它负责在上层组织和管理所有的chunk以及subpage单元。为了减少多线程请求内存池时的同步处理，Netty默认提供了cpu核数*2个PoolArena实例。

> We use 2 *available processors by default to reduce contention as we use 2* available processors for the number of EventLoops in NIO and EPOLL as well. If we choose a smaller number we will run into hot spots as allocation and de-allocation needs to be synchronized on the PoolArena.

PoolArena 内部对chunk与subpage的内存组织方式如下图：

![1564636926961](/home/collapsar/.config/Typora/typora-user-images/1564636926961.png)

应用层的内存分配主要通过如下实现，但最终还是委托给PoolArena实现。

```
PooledByteBufAllocator.DEFAULT.directBuffer(128);
```

由于netty通常应用于高并发系统，不可避免的有多线程进行同时内存分配，可能会极大的影响内存分配的效率，为了缓解线程竞争，可以通过创建多个poolArena细化锁的粒度，提高并发执行的效率。

先看看poolArena的内部结构：

![1564672056813](/home/collapsar/.config/Typora/typora-user-images/1564672056813.png)

poolArena提供了两种方式进行内存分配：

1. PoolSubpage用于分配小于8k的内存；

- tinySubpagePools：用于分配小于512字节的内存，默认长度为32，因为内存分配最小为16，每次增加16，直到512，区间[16，512)一共有32个不同值；
- smallSubpagePools：用于分配大于等于512字节的内存，当size>=512时，size成倍增长512->1024->2048->4096->8192,默认长度为4；
- tinySubpagePools和smallSubpagePools中的元素都是默认subpage。

```java
// 512 / 16
static final int numTinySubpagePools = 512 >>> 4;        
tinySubpagePools = newSubpagePoolArray(numTinySubpagePools);        
for (int i = 0; i < tinySubpagePools.length; i ++) {            
    tinySubpagePools[i] = newSubpagePoolHead(pageSize);        
}
// 当size>=512时，size成倍增长512->1024->2048->4096,默认长度是4
numSmallSubpagePools = pageShifts - 9;        
smallSubpagePools = newSubpagePoolArray(numSmallSubpagePools);        
for (int i = 0; i < smallSubpagePools.length; i ++) {            
    smallSubpagePools[i] = newSubpagePoolHead(pageSize);        
}
```

2. poolChunkList用于分配大于8k的内存；

- qInit：存储内存利用率0-25%的chunk
- q000：存储内存利用率1-50%的chunk
- q025：存储内存利用率25-75%的chunk
- q050：存储内存利用率50-100%的chunk
- q075：存储内存利用率75-100%的chunk
- q100：存储内存利用率100%的chunk

![1564672145288](/home/collapsar/.config/Typora/typora-user-images/1564672145288.png)

1. qInit前置节点为自己，且minUsage=Integer.MIN_VALUE，意味着一个初分配的chunk，在最开始的内存分配过程中(内存使用率<25%)，即使完全释放也不会被回收，会始终保留在内存中。
2. q000没有前置节点，当一个chunk进入到q000列表，如果其内存被完全释放，则不再保留在内存中，其分配的内存被完全回收。

接下去看看poolArena如何实现内存的分配，实现如下：

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity);
    // capacity < pageSize
    if (isTinyOrSmall(normCapacity)) {
        int tableIdx;
        PoolSubpage<T>[] table;
        // < 512
        boolean tiny = isTiny(normCapacity);
        if (tiny) {
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }

        final PoolSubpage<T> head = table[tableIdx];

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);

                if (tiny) {
                    allocationsTiny.increment();
                } else {
                    allocationsSmall.increment();
                }
                return;
            }
        }
        allocateNormal(buf, reqCapacity, normCapacity);
        return;
    }
    if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        allocateNormal(buf, reqCapacity, normCapacity);
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        // 直接通过ByteBuf分配
        allocateHuge(buf, reqCapacity);
    }
}
```

1、默认先尝试从poolThreadCache中分配内存，PoolThreadCache利用ThreadLocal的特性，消除了多线程竞争，提高内存分配效率；首次分配时，poolThreadCache中并没有可用内存进行分配，当上一次分配的内存使用完并释放时，会将其加入到poolThreadCache中，提供该线程下次申请时使用。
2、如果是分配小内存，则尝试从tinySubpagePools或smallSubpagePools中分配内存，如果没有合适subpage，则采用方法allocateNormal分配内存。
3、如果分配一个page以上的内存，直接采用方法allocateNormal分配内存。

allocateNormal实现如下：

```java
// Method must be called inside synchronized(this) { ... } block
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {    
	if (q050.allocate(buf, reqCapacity, normCapacity) || 
        q025.allocate(buf, reqCapacity, normCapacity) ||        
        q000.allocate(buf, reqCapacity, normCapacity) || 
        qInit.allocate(buf, reqCapacity, normCapacity) ||        
        q075.allocate(buf, reqCapacity, normCapacity)) {        
    	return;    
	}    
    // Add a new chunk.    
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);    
    long handle = c.allocate(normCapacity);    
    assert handle > 0;    
    c.initBuf(buf, handle, reqCapacity);    
    qInit.add(c);
}
```

第一次进行内存分配时，chunkList没有chunk可以分配内存，需通过方法newChunk新建一个chunk进行内存分配，并添加到qInit列表中。如果分配如512字节的小内存，除了创建chunk，还有创建subpage，PoolSubpage在初始化之后，会添加到smallSubpagePools中，其实并不是直接插入到数组，而是添加到head的next节点。下次再有分配512字节的需求时，直接从smallSubpagePools获取对应的subpage进行分配。

![1564673710924](/home/collapsar/.config/Typora/typora-user-images/1564673710924.png)

分配内存时，为什么不从内存使用率较低的q000开始？在chunkList中，我们知道一个chunk随着内存的释放，会往当前chunklist的前一个节点移动。

q000存在的目的是什么？
 q000是用来保存内存利用率在1%-50%的chunk，那么这里为什么不包括0%的chunk？
 直接弄清楚这些，才好理解为什么不从q000开始分配。q000中的chunk，当内存利用率为0时，就从链表中删除，直接释放物理内存，避免越来越多的chunk导致内存被占满。

想象一个场景，当应用在实际运行过程中，碰到访问高峰，这时需要分配的内存是平时的好几倍，当然也需要创建好几倍的chunk，如果先从q0000开始，这些在高峰期创建的chunk被回收的概率会大大降低，延缓了内存的回收进度，造成内存使用的浪费。

那么为什么选择从q050开始？
 1、q050保存的是内存利用率50%~100%的chunk，这应该是个折中的选择！这样大部分情况下，chunk的利用率都会保持在一个较高水平，提高整个应用的内存利用率；
 2、qinit的chunk利用率低，但不会被回收；
 3、q075和q100由于内存利用率太高，导致内存分配的成功率大大降低，因此放到最后；

## Chunk

Netty中底层的内存分配和回收管理主要由PoolChunk实现，PoolChunk是Netty内存申请的最小值，默认16M,page 8k.在PoolChunk内部维护一棵平衡二叉树memoryMap管理.

二叉树管理page

```java
PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize, int offset) {
    unpooled = false;
    this.arena = arena;
    // Chunk块的内存地址
    this.memory = memory;
    // 8192
    this.pageSize = pageSize;
    // 13
    this.pageShifts = pageShifts;
    // 11
    this.maxOrder = maxOrder;
    // 16M
    this.chunkSize = chunkSize;
    this.offset = offset;
    // 默认12,用于标记二叉树中每个page已分配
    unusable = (byte) (maxOrder + 1);
    log2ChunkSize = log2(chunkSize);
    // page大小的掩码,用于判断申请大小是否>=pages
    subpageOverflowMask = ~(pageSize - 1);
    freeBytes = chunkSize;

    assert maxOrder < 30 : "maxOrder should be < 30, but is: " + maxOrder;
    // 默认maxOrder = 11，最多 2^11　个page
    maxSubpageAllocs = 1 << maxOrder;

    // 二叉树初始化
    // 生成(２^12 - 1 + 1) 个数组，记录每个节点的状态，但是数组第０个不使用
    memoryMap = new byte[maxSubpageAllocs << 1];
    depthMap = new byte[memoryMap.length];
    int memoryMapIndex = 1;
    //　从第一层开始给每层节点数值
    for (int d = 0; d <= maxOrder; ++ d) {
        //　每一层的节点树，２^(层数-1)
        int depth = 1 << d;
        for (int p = 0; p < depth; ++ p) {
            memoryMap[memoryMapIndex] = (byte) d;
            depthMap[memoryMapIndex] = (byte) d;
            memoryMapIndex ++;
        }
    }

    // 创建一个subpage数组,长度和page一样,默认为2048
    subpages = newSubpageArray(maxSubpageAllocs);
}
```

| **节点深度** | **节点个数** | **每个节点大小**     |
| ------------ | ------------ | -------------------- |
| 0            | 1            | chunkSize            |
| 1            | 2            | chunkSize/2          |
| d            | 2^d          | chunkSize/2^d        |
| ...          | ...          | ...                  |
| maxOrder     | 2^maxOrder   | chunkSize/2^maxOrder |

树的最大的深度为maxOrder+1，默认maxOrder的值为11，因此一个chunk中默认有2^11=2048个page,计算下来每个page是8k。因此当线程请求一个chunkSize/2^k大小的内存时，只需要在树的第k层中从左往右找到一个满足条件的节点即可，什么是满足条件的呢，就是当前该节点和该节点的所有子节点都是可分配的状态。

#### Page分配

在现有的chunk上分配

```java
// Method must be called inside synchronized(this) { ... } block
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
    // 查找已有的chunk中是否能分配
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    // 若不能分配,则创建Chunk
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
    long handle = c.allocate(normCapacity);
    assert handle > 0;
    //　初始化ByteBuf
    c.initBuf(buf, handle, reqCapacity);
    qInit.add(c);
}

long allocate(int normCapacity) {
    // 判断>= pageSize
    if ((normCapacity & subpageOverflowMask) != 0) {
        // 分配连续pages
        return allocateRun(normCapacity);
    } else {
        // 分配page,在其中分配subpage
        return allocateSubpage(normCapacity);
    }
}
```

分配连续pages(>=1)

```java
private long allocateRun(int normCapacity) {
    //　log2(normCapacity) - pageShifts　计算需要几个page
    // pageShifts是13,因为只有申请大小超过2^13（8192,8K）时才会分配连续page
    int d = maxOrder - (log2(normCapacity) - pageShifts);
    // 在二叉树中进行节点分配
    int id = allocateNode(d);
    if (id < 0) {
        return id;
    }
    freeBytes -= runLength(id);
    return id;
}

// 计算val二进制中第一个为1的位数
// log2(000000000010) = 2
private static int log2(int val) {
    // compute the (0-based, with lsb = 0) position of highest set bit i.e, log2
    return INTEGER_SIZE_MINUS_ONE - Integer.numberOfLeadingZeros(val);
}

/**
 * Algorithm to allocate an index in memoryMap when we query for a free node
 * at depth d
 *
 * @param d depth
 * @return index in memoryMap
 */
private int allocateNode(int d) {
    int id = 1;
    int initial = - (1 << d); // has last d bits = 0 and rest all = 1
    // 1. 从根节点开始遍历
    byte val = value(id);
    // 根节点被分配
    if (val > d) {
        return -1;
    }
    while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
        // 匹配下一层
        id <<= 1;
        val = value(id);
        // 若已分配则寻找兄弟树
        if (val > d) {
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
                                                                  value, id & initial, d);
    // 将该节点置为已使用
    setValue(id, unusable);
    // 逐级更新父节点
    updateParentsAlloc(id);
    return id;
}

private void updateParentsAlloc(int id) {
    while (id > 1) {
        int parentId = id >>> 1;
        byte val1 = value(id);
        //　取得id的左子数或右子树
        byte val2 = value(id ^ 1);
        byte val = val1 < val2 ? val1 : val2;
        setValue(parentId, val);
        id = parentId;
    }
}
```

比如2048节点被分配出去流程如下:

![1564659844051](/home/collapsar/.config/Typora/typora-user-images/1564659844051.png)

#### SubPage分配

上面分析了如何在poolChunk中分配一块大于pageSize的内存，但在实际应用中，存在很多分配小内存的情况，如果也占用一个page，明显很浪费。针对这种情况，Netty提供了PoolSubpage把poolChunk的一个page节点8k内存划分成更小的内存段，通过对每个内存段的标记与清理标记进行内存的分配与释放。

![1564664184077](/home/collapsar/.config/Typora/typora-user-images/1564664184077.png)

定位一个Subpage对象

```java
private long allocateSubpage(int normCapacity) {
    // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
    // This is need as we may add it back and so alter the linked-list structure.
    // Arena负责管理PoolChunk和PoolSubpage；
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
    synchronized (head) {
        // 先在在二叉树中找到匹配的节点，和poolChunk不同的是，只匹配叶子节点；
        int d = maxOrder; 
        // 叶子节点下标
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }

        // subpages在chunk创建时会生成
        final PoolSubpage<T>[] subpages = this.subpages;
        final int pageSize = this.pageSize;

        freeBytes -= pageSize;

        // 获得对应page的subpage的下标,相当于取模运算
        int subpageIdx = subpageIdx(id);
        PoolSubpage<T> subpage = subpages[subpageIdx];
        // 若为空则创建
        if (subpage == null) {
            // 主要信息,大小,对应的page,page中的偏移
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else {
            subpage.init(head, normCapacity);
        }
        return subpage.allocate();
    }
}
```

PoolSubpage初始化实现如下：

```java
PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
    // 对应的chunk
    this.chunk = chunk;
    // 对应chunk中二叉树的下标
    this.memoryMapIdx = memoryMapIdx;
    // 对应chunk中的内存偏移
    this.runOffset = runOffset;
    // page的大小
    this.pageSize = pageSize;
    // page默认8kb,subpage最小16b,long有64b位,即需要pageSize / 16 / 64个long数组
    bitmap = new long[pageSize >>> 10]; 
    // 根据分配值确定需要多少个bitmap元素
    init(head, elemSize);
}
```

下面通过分布申请4096和32大小的内存，说明如何确定bitmap length的值：

1. 比如，当前申请大小4096的内存，maxNumElems 和 numAvail 为2，说明一个page被拆分成2个内存段，2 >>> 6 = 0，且2 & 63 ！= 0，所以bitmapLength为1，说明只需要一个long就可以描述2个内存段状态。
2. 如果当前申请大小32的内存，maxNumElems 和 numAvail 为 256，说明一个page被拆分成256个内存段， 256>>> 6 = 4，说明需要4个long描述256个内存段状态。

```java
void init(PoolSubpage<T> head, int elemSize) {
    doNotDestroy = true;
    this.elemSize = elemSize;
    if (elemSize != 0) {
        // subpage的数量
        maxNumElems = numAvail = pageSize / elemSize;
        nextAvail = 0;
        // 实际需要多少个long数组存储subpage的分配装填
        bitmapLength = maxNumElems >>> 6;
        // maxNumElems 是否有不足64的部分,有则进1
        if ((maxNumElems & 63) != 0) {
            bitmapLength ++;
        }

        for (int i = 0; i < bitmapLength; i ++) {
            bitmap[i] = 0;
        }
    }
    addToPool(head);
}
```

下面看看subpage是如何进行内存分配的：

```java
long allocate() {
    if (elemSize == 0) {
        return toHandle(0);
    }

    if (numAvail == 0 || !doNotDestroy) {
        return -1;
    }

    // 获得下一个可用的 Subpage 在 bitmap 中的总体位置
    final int bitmapIdx = getNextAvail();
    
    // bitmap index
    int q = bitmapIdx >>> 6;
    // bit index
    int r = bitmapIdx & 63;
    assert (bitmap[q] >>> r & 1) == 0;
    // 将对应标识位设置为1
    bitmap[q] |= 1L << r;

    if (-- numAvail == 0) {
        removeFromPool();
    }

    return toHandle(bitmapIdx);
}

private int getNextAvail() {
    int nextAvail = this.nextAvail;
    if (nextAvail >= 0) {
        this.nextAvail = -1;
        return nextAvail;
    }
    return findNextAvail();
}

private int findNextAvail() {
    final long[] bitmap = this.bitmap;
    final int bitmapLength = this.bitmapLength;
    for (int i = 0; i < bitmapLength; i ++) {
        long bits = bitmap[i];
        // bitmap是否全部用完
        if (~bits != 0) {
            return findNextAvail0(i, bits);
        }
    }
    return -1;
}

// bits 仍有可用空间的bitmap, i bitmap的下标
private int findNextAvail0(int i, long bits) {
    final int maxNumElems = this.maxNumElems;
    final int baseVal = i << 6;

    // 每次右移一位和1与
    for (int j = 0; j < 64; j ++) {
        if ((bits & 1) == 0) {
            int val = baseVal | j;
            // 可能64位不是每一位都表示占用情况,可能其中33位
            if (val < maxNumElems) {
                return val;
            } else {
                break;
            }
        }
        // 右移一位
        bits >>>= 1;
    }
    return -1;
}
```

下面来看一下调用allocate的返回值的含义

```java
private long toHandle(int bitmapIdx) {
    return 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
}
```

- 低 32 bits ：`memoryMapIdx` ，可以判断所属 Chunk 的哪个 Page 节点，即 `memoryMap[memoryMapIdx]` 。

- 高 32 bits ：bitmapIdx，可以判断 Page 节点中的哪个 Subpage 的内存块，即bitmap[bitmapIdx],那么为什么会有0x4000000000000000L 呢？因为在 

  ```java
  PoolChunk#allocate(int normCapacity)
  ```

  中：

  - 如果分配的是 Page 内存块，返回的是 `memoryMapIdx` 。
  - 如果分配的是 Subpage 内存块，返回的是 `handle` 。**但但但是**，如果说 `bitmapIdx = 0` ，那么没有 `0x4000000000000000L` 情况下，就会和【分配 Page 内存块】冲突。因此，需要有 `0x4000000000000000L` 。
- 因为有了 `0x4000000000000000L`(最高两位为 `01` ，其它位为 `0` )，所以获取 `bitmapIdx` 时，通过 `handle >>> 32 & 0x3FFFFFFF` 操作。使用 `0x3FFFFFFF`( 最高两位为 `00` ，其它位为 `1` ) 进行消除 `0x4000000000000000L` 带来的影响。



回收

连续的内存区段加到缓存

```java
void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
    if (chunk.unpooled) {
        int size = chunk.chunkSize();
        destroyChunk(chunk);
        activeBytesHuge.add(-size);
        deallocationsHuge.increment();
    } else {
        SizeClass sizeClass = sizeClass(normCapacity);
        if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {
            // cached so not free it.
            return;
        }

        freeChunk(chunk, handle, sizeClass);
    }
}
```

标记连续的内存区段为未使用

```java
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
    final boolean destroyChunk;
    synchronized (this) {
        switch (sizeClass) {
        case Normal:
            ++deallocationsNormal;
            break;
        case Small:
            ++deallocationsSmall;
            break;
        case Tiny:
            ++deallocationsTiny;
            break;
        default:
            throw new Error();
        }
        destroyChunk = !chunk.parent.free(chunk, handle);
    }
    if (destroyChunk) {
        // destroyChunk not need to be called while holding the synchronized lock.
        destroyChunk(chunk);
    }
}
```

bytebuf加到对象池