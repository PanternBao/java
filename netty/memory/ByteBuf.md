# ByteBuf

三个问题

1. 内存的类别有哪些？
2. 如何减少多线程内存分配之间的竞争？
3. 不同大小的内存是如何进行分配的？



内存与内存管理器的抽象

不同规格大小和不同类别的内存分配策略

内存的回收过程



ByteBuf结构及重要api

```text
+-------------------+------------------+------------------+
*      | discardable bytes |  readable bytes  |  writable bytes  |
*      |                   |     (CONTENT)    |                  |
*      +-------------------+------------------+------------------+
*      |                   |                  |                  |
*      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

read，write，set方法

mark   保存index指针 和reset（恢复mark的index）方法



ByteBuf分类

todo      类机构

Pooled和Unpooled：预先申请一大块和每次使用去申请

Unsafe和非Unsafe：直接通过JDK中的unsafe对象指针读取

Heap和Direct：堆外内存不收jvm管理，内存泄露



ByteBufAllocator分析

1. ByteBufAllocator功能

   主要从heap和direct维度分配

2. AbstractByteBufAllocator

3. ByteBufAllocator两大子类

![1555827910931](/home/collapsar/.config/Typora/typora-user-images/1555827910931.png)

## UnPooledByteBufAllocator分析

heap内存的分配

子类实现AbstractByteBufAllocator的newHeapBuffer类实现具体buffer的分配

unsafe的维度是netty自身判断

```java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    return PlatformDependent.hasUnsafe() ?
            new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
            new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}
```

InstrumentedUnpooledHeapByteBuf 继承于UnpooledHeapByteBuf，netty新增了对堆内存和堆外内存使用量的统计，兼容性类

UnpooledUnsafeHeapByteBuf继承于UnpooledHeapByteBuf，区别在于取数据的方式

```java
UnpooledUnsafeHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(alloc, initialCapacity, maxCapacity);
}
```



```java
public UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);

    checkNotNull(alloc, "alloc");

    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
            "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setArray(allocateArray(initialCapacity));
    setIndex(0, 0);
}
```

非unsafe获取方式

```java
@Override
public byte getByte(int index) {
    ensureAccessible();
    return _getByte(index);
}

@Override
protected byte _getByte(int index) {
    return HeapByteBufUtil.getByte(array, index);
}
```

HeapByteBufUtil.java

```java
static byte getByte(byte[] memory, int index) {
    return memory[index];
}
```

```java
static byte getByte(byte[] array, int index) {
    return PlatformDependent.getByte(array, index);
}
```

```java
// PlatformDependent.java
public static byte getByte(byte[] data, int index) {
    return PlatformDependent0.getByte(data, index);
}
// PlatformDependent0.java
static byte getByte(byte[] data, int index) {
    return UNSAFE.getByte(data, BYTE_ARRAY_BASE_OFFSET + index);
}
```



```java
@Override
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    final ByteBuf buf;
    if (PlatformDependent.hasUnsafe()) {
        buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}
```

```java
public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("initialCapacity: " + initialCapacity);
    }
    if (maxCapacity < 0) {
        throw new IllegalArgumentException("maxCapacity: " + maxCapacity);
    }
    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    // 调用jdk底层方式获取buffer
    setByteBuffer(allocateDirect(initialCapacity));
}

private void setByteBuffer(ByteBuffer buffer) {
    ByteBuffer oldBuffer = this.buffer;
    if (oldBuffer != null) {
        if (doNotFree) {
            doNotFree = false;
        } else {
            freeDirect(oldBuffer);
        }
    }

    this.buffer = buffer;
    tmpNioBuf = null;
    capacity = buffer.remaining();
}
```

```java
public UnpooledUnsafeDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("initialCapacity: " + initialCapacity);
    }
    if (maxCapacity < 0) {
        throw new IllegalArgumentException("maxCapacity: " + maxCapacity);
    }
    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setByteBuffer(allocateDirect(initialCapacity), false);
}

final void setByteBuffer(ByteBuffer buffer, boolean tryFree) {
    if (tryFree) {
        ByteBuffer oldBuffer = this.buffer;
        if (oldBuffer != null) {
            if (doNotFree) {
                doNotFree = false;
            } else {
                freeDirect(oldBuffer);
            }
        }
    }
    this.buffer = buffer;
    // 会获取分配的ByteBuffer的内存地址
    memoryAddress = PlatformDependent.directBufferAddress(buffer);
    tmpNioBuf = null;
    capacity = buffer.remaining();
}
```

## PooledByteBufAllocator分析

拿到线程局部缓存PoolThreadCache

在线程局部缓存的Area上进行内存分配

```java
@Override
protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
    // 线程局部缓存PoolThreadCache
    PoolThreadCache cache = threadCache.get();
    // 在线程局部缓存的Area上进行内存分配
    PoolArena<byte[]> heapArena = cache.heapArena;

    final ByteBuf buf;
    if (heapArena != null) {
        buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}

@Override
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    PoolThreadCache cache = threadCache.get();
    PoolArena<ByteBuffer> directArena = cache.directArena;

    final ByteBuf buf;
    if (directArena != null) {
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        buf = PlatformDependent.hasUnsafe() ?
                UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }

    return toLeakAwareBuffer(buf);
}
```

关于PoolThreadLocalCache

是allocator的一个内存类

```java
final class PoolThreadLocalCache extends FastThreadLocal<PoolThreadCache> {
    private final boolean useCacheForAllThreads;

    PoolThreadLocalCache(boolean useCacheForAllThreads) {
        this.useCacheForAllThreads = useCacheForAllThreads;
    }

    @Override
    protected synchronized PoolThreadCache initialValue() {
        // heapArenas和directArenas是allocator创建的时候创建
        final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas);
        final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

        Thread current = Thread.currentThread();
        if (useCacheForAllThreads || current instanceof FastThreadLocalThread) {
            return new PoolThreadCache(
                    heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                    DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
        }
        // No caching so just use 0 as sizes.
        return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
    }

    @Override
    protected void onRemoval(PoolThreadCache threadCache) {
        threadCache.free();
    }

    private <T> PoolArena<T> leastUsedArena(PoolArena<T>[] arenas) {
        if (arenas == null || arenas.length == 0) {
            return null;
        }

        PoolArena<T> minArena = arenas[0];
        for (int i = 1; i < arenas.length; i++) {
            PoolArena<T> arena = arenas[i];
            if (arena.numThreadCaches.get() < minArena.numThreadCaches.get()) {
                minArena = arena;
            }
        }

        return minArena;
    }
}
```

Allocator创建时

1.创建PoolThreadLocalCache

2.定义tinyCacheSize，smallCacheSize，normalCacheSize

```java
public PooledByteBufAllocator(boolean preferDirect, int nHeapArena, int nDirectArena, int pageSize, int maxOrder,
                              int tinyCacheSize, int smallCacheSize, int normalCacheSize,
                              boolean useCacheForAllThreads, int directMemoryCacheAlignment) {
    super(preferDirect);
    // 创建PoolThreadLocalCache
    threadCache = new PoolThreadLocalCache(useCacheForAllThreads);
    // 定义三个变量
    this.tinyCacheSize = tinyCacheSize;
    this.smallCacheSize = smallCacheSize;
    this.normalCacheSize = normalCacheSize;
    chunkSize = validateAndCalculateChunkSize(pageSize, maxOrder);

    if (nHeapArena < 0) {
        throw new IllegalArgumentException("nHeapArena: " + nHeapArena + " (expected: >= 0)");
    }
    if (nDirectArena < 0) {
        throw new IllegalArgumentException("nDirectArea: " + nDirectArena + " (expected: >= 0)");
    }

    if (directMemoryCacheAlignment < 0) {
        throw new IllegalArgumentException("directMemoryCacheAlignment: "
                + directMemoryCacheAlignment + " (expected: >= 0)");
    }
    if (directMemoryCacheAlignment > 0 && !isDirectMemoryCacheAlignmentSupported()) {
        throw new IllegalArgumentException("directMemoryCacheAlignment is not supported");
    }

    if ((directMemoryCacheAlignment & -directMemoryCacheAlignment) != directMemoryCacheAlignment) {
        throw new IllegalArgumentException("directMemoryCacheAlignment: "
                + directMemoryCacheAlignment + " (expected: power of two)");
    }

    int pageShifts = validateAndCalculatePageShifts(pageSize);

    if (nHeapArena > 0) {
        // 创建heapArena数组
        heapArenas = newArenaArray(nHeapArena);
        List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(heapArenas.length);
        for (int i = 0; i < heapArenas.length; i ++) {
            // 实现类PoolArena.HeapArena
            PoolArena.HeapArena arena = new PoolArena.HeapArena(this,
                    pageSize, maxOrder, pageShifts, chunkSize,
                    directMemoryCacheAlignment);
            heapArenas[i] = arena;
            metrics.add(arena);
        }
        heapArenaMetrics = Collections.unmodifiableList(metrics);
    } else {
        heapArenas = null;
        heapArenaMetrics = Collections.emptyList();
    }

    if (nDirectArena > 0) {
        directArenas = newArenaArray(nDirectArena);
        List<PoolArenaMetric> metrics = new ArrayList<PoolArenaMetric>(directArenas.length);
        for (int i = 0; i < directArenas.length; i ++) {
            // 实现类PoolArena.DirectArena
            PoolArena.DirectArena arena = new PoolArena.DirectArena(
                    this, pageSize, maxOrder, pageShifts, chunkSize, directMemoryCacheAlignment);
            directArenas[i] = arena;
            metrics.add(arena);
        }
        directArenaMetrics = Collections.unmodifiableList(metrics);
    } else {
        directArenas = null;
        directArenaMetrics = Collections.emptyList();
    }
    metric = new PooledByteBufAllocatorMetric(this);
}

private static <T> PoolArena<T>[] newArenaArray(int size) {
    return new PoolArena[size];
}
```

directArea分配direct内存的流程

从对象池中拿到PooledByteBuf进行复用