# Pipeline

## 问题

1. netty是如何判断ChannelHandler类型的？
2. 对于ChannelHandler的添加应该遵循什么样的顺序？
3. 用户手动触发事件传播，不同的触发方式有什么样的区别？

Pileline类结构图

pipeline能够传播inbound和outbound事件

pipeline维护ChannelHandlerContext的双向链表结构，并对外提供新增，删除ChannelHandler的行为

![1554995047377](/home/collapsar/.config/Typora/typora-user-images/1554995047377.png)

## pipeline创建

1. pipeline在创建Channel的时候被创建

```java
protected AbstractChannel(Channel parent, ChannelId id) {
    this.parent = parent;
    this.id = id;
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
/**
 * Returns a new {@link DefaultChannelPipeline} instance.
 */
protected DefaultChannelPipeline newChannelPipeline() {
    // 将channel本身传递进pipeline中
    return new DefaultChannelPipeline(this);
}
```

2. Pipeline中的两大哨兵：head和tail

```java
protected DefaultChannelPipeline(Channel channel) {
    // 传入关联的Channel
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    // 默认创建两个哨兵节点
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

## Pipeline节点数据结构

ChannelHandlerContext

能够传播事件

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {

    /**
     * Return the {@link Channel} which is bound to the {@link ChannelHandlerContext}.
     */
    Channel channel();

    /**
     * Returns the {@link EventExecutor} which is used to execute an arbitrary task.
     */
    EventExecutor executor();

    /**
     * The unique name of the {@link ChannelHandlerContext}.The name was used when then {@link ChannelHandler}
     * was added to the {@link ChannelPipeline}. This name can also be used to access the registered
     * {@link ChannelHandler} from the {@link ChannelPipeline}.
     */
    String name();

    /**
     * The {@link ChannelHandler} that is bound this {@link ChannelHandlerContext}.
     */
    ChannelHandler handler();

    /**
     * Return {@code true} if the {@link ChannelHandler} which belongs to this context was removed
     * from the {@link ChannelPipeline}. Note that this method is only meant to be called from with in the
     * {@link EventLoop}.
     */
    boolean isRemoved();

    @Override
    ChannelHandlerContext fireChannelRegistered();

    @Override
    ChannelHandlerContext fireChannelUnregistered();

    @Override
    ChannelHandlerContext fireChannelActive();

    @Override
    ChannelHandlerContext fireChannelInactive();

    @Override
    ChannelHandlerContext fireExceptionCaught(Throwable cause);

    @Override
    ChannelHandlerContext fireUserEventTriggered(Object evt);

    @Override
    ChannelHandlerContext fireChannelRead(Object msg);

    @Override
    ChannelHandlerContext fireChannelReadComplete();

    @Override
    ChannelHandlerContext fireChannelWritabilityChanged();

    @Override
    ChannelHandlerContext read();

    @Override
    ChannelHandlerContext flush();

    /**
     * Return the assigned {@link ChannelPipeline}
     */
    ChannelPipeline pipeline();

    /**
     * Return the assigned {@link ByteBufAllocator} which will be used to allocate {@link ByteBuf}s.
     */
    ByteBufAllocator alloc();

    /**
     * @deprecated Use {@link Channel#attr(AttributeKey)}
     */
    @Deprecated
    @Override
    <T> Attribute<T> attr(AttributeKey<T> key);

    /**
     * @deprecated Use {@link Channel#hasAttr(AttributeKey)}
     */
    @Deprecated
    @Override
    <T> boolean hasAttr(AttributeKey<T> key);
}
```

## 添加删除ChannelHandler

```java
@Override
public final ChannelPipeline addLast(ChannelHandler... handlers) {
    return addLast(null, handlers);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h);
    }

    return this;
}

// 实际添加操作
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        // 判断是否handler是否已添加过，非共享handler不能多pipeline重复添加
        checkMultiplicity(handler);

        // 创建context，若name为空，系统自动生成，否则name不能重复
        newCtx = newContext(group, filterName(name, handler), handler);

        // 添加context
        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            // 设置context是否已添加的标识位，此处标识为正在添加
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

判断是否重复添加

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        // 若不是可共享的handler且已经添加过的，抛出异常
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                h.getClass().getName() +
                " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```

创建节点

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}

DefaultChannelHandlerContext(
    DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
    super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
}

// 通过handler判断context类型
private static boolean isInbound(ChannelHandler handler) {
    return handler instanceof ChannelInboundHandler;
}
private static boolean isOutbound(ChannelHandler handler) {
    return handler instanceof ChannelOutboundHandler;
}
```

添加至链表

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```



删除节点

使用场景：权限校验

```java
@Override
public final <T extends ChannelHandler> T remove(Class<T> handlerType) {
    return (T) remove(getContextOrDie(handlerType)).handler();
}
```

找到节点

```java
private AbstractChannelHandlerContext getContextOrDie(Class<? extends ChannelHandler> handlerType) {
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handlerType);
    if (ctx == null) {
        throw new NoSuchElementException(handlerType.getName());
    } else {
        return ctx;
    }
}

@Override
public final ChannelHandlerContext context(Class<? extends ChannelHandler> handlerType) {
    if (handlerType == null) {
        throw new NullPointerException("handlerType");
    }

    AbstractChannelHandlerContext ctx = head.next;
    for (;;) {
        if (ctx == null) {
            return null;
        }
        if (handlerType.isAssignableFrom(ctx.handler().getClass())) {
            return ctx;
        }
        ctx = ctx.next;
    }
}
```

链表的删除

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail;

    synchronized (this) {
        remove0(ctx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we remove the context from the pipeline and add a task that will call
        // ChannelHandler.handlerRemoved(...) once the channel is registered.
        if (!registered) {
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }

        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx);
    return ctx;
}

// 标准的双向链表删除操作
private static void remove0(AbstractChannelHandlerContext ctx) {
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    prev.next = next;
    next.prev = prev;
}
```

回调删除handler事件

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        try {
            ctx.handler().handlerRemoved(ctx);
        } finally {
            ctx.setRemoved();
        }
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```

## 事件和异常的传播

inBount事件的传播

何为inBound事件以及ChannelInboundHandler

ChannelRead事件的传播

SimpleInboundHandler

ChannelHandler 类结构

![1554991462393](/home/collapsar/.config/Typora/typora-user-images/1554991462393.png)

ChannelHandler 定义了添加和删除的回调，异常捕获

CHannelInbound 定义了一些 channel 被注册，激活等一些回调，事件触发后的回调



outBound事件的传播，触发事件，被动

何为outBound事件以及ChannelOutBoundHandler

write事件的传播