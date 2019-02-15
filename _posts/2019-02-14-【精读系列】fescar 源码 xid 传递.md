---
layout: post
title: 【精读系列】fescar 源码 xid 传递
date: 2019-02-14 19:08:13.000000000 +09:00
categories: 精读系列
keyword: fescar
---

### 精读目的

如果之前有了解过 fescar，那么你应该对 xid 不会感到陌生。如果你不是很了解 fescar，也不清楚 xid 在其中的作用，我这里从 fescar 官方 wiki 中截了一张图来你来了解一下 xid 在其中的重要性。
<br>
![xid 在 fescar 的重要性](https://zhimo-1252091368.cos.ap-chengdu.myqcloud.com/blog/20190215103948.png)
<br>
所以本篇文章的目的就为了让你对 xid 的生成，传递有一个

### 阅读内容

我们先看一下 xid 在 TM 端的代码，先从 `begin()` 方法开始。

```java
@Override
public void begin(int timeout, String name) throws TransactionException {
    ...
    xid = transactionManager.begin(null, null, name, timeout);
    status = GlobalStatus.Begin;
    RootContext.bind(xid);
    ...
    }
}
```

TC端生成一个 xid，生成规则是 ip : port : transactionId，transactionId 的生成规则如下。TC 端生成 xid 后，返回给 TM 端。

```java
private static AtomicLong UUID = new AtomicLong(1000);
private static int serverNodeId = 1;
private static final long UUID_INTERNAL = 2000000000;

public static long generateUUID() {
    long id = UUID.incrementAndGet();
    if (id > UUID_INTERNAL * serverNodeId) {
        synchronized (UUID) {
            if (UUID.get() >= id) {
                id -= UUID_INTERNAL;
                UUID.set(id);
            }
        }
    }
    return id;
}
```

绑定到 ThreadLocal 中

```java
private static ContextCore CONTEXT_HOLDER = ContextCoreLoader.load();

public static void bind(String xid) {
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("bind " + xid);
    }
    CONTEXT_HOLDER.put(KEY_XID, xid);
}
```

```java
public class ContextCoreLoader {

    private static class ContextCoreHolder {
        private static ContextCore INSTANCE;

        static {
            // TODO:暂时不考虑这段代码，后续再回来研究
            ContextCore contextCore = EnhancedServiceLoader.load(ContextCore.class);
            if (contextCore == null) {
                // ThreadLocal
                contextCore = new ThreadLocalContextCore();
            }
            INSTANCE = contextCore;
        }
    }

    /**
     * Load context core.
     *
     * @return the context core
     */
    public static ContextCore load() {
        return ContextCoreHolder.INSTANCE;
    }
}
```

接下来我们看 dubbo 自定义透传 xid 的 filter代码。不要慌，下面有拆分讲解。

```java
@Activate(group = { Constants.PROVIDER, Constants.CONSUMER }, order = 100)
public class TransactionPropagationFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionPropagationFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        String xid = RootContext.getXID();
        String rpcXid = RpcContext.getContext().getAttachment(RootContext.KEY_XID);
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("xid in RootContext[" + xid + "] xid in RpcContext[" + rpcXid + "]");
        }
        boolean bind = false;
        if (xid != null) {
            RpcContext.getContext().setAttachment(RootContext.KEY_XID, xid);
        } else {
            if (rpcXid != null) {
                RootContext.bind(rpcXid);
                bind = true;
                if (LOGGER.isDebugEnabled()) {
                    LOGGER.debug("bind[" + rpcXid + "] to RootContext");
                }
            }
        }
        try {
            return invoker.invoke(invocation);

        } finally {
            if (bind) {
                String unbindXid = RootContext.unbind();
                if (LOGGER.isDebugEnabled()) {
                    LOGGER.debug("unbind[" + unbindXid + "] from RootContext");
                }
                if (!rpcXid.equalsIgnoreCase(unbindXid)) {
                    LOGGER.warn("xid in change during RPC from " + rpcXid + " to " + unbindXid);
                    if (unbindXid != null) {
                        RootContext.bind(unbindXid);
                        LOGGER.warn("bind [" + unbindXid + "] back to RootContext");
                    }
                }
            }
        }
    }
}
```

上面是 dubbo 透传 xid 的自定义 filter 代码，我们看到第一行代码就是 `@Activate(group = { Constants.PROVIDER, Constants.CONSUMER }, order = 100)`。下面分别会对 provider 和 consumer 拆分讲解一下透传的原理。当然具体的 dubbo 源码我还没看，所以这里暂时不会涉及到 dubbo 源码。

如果是 consumer，通常 TM 一定是 consumer，且一定有 xid。删减后重要的代码如下所示
```java
@Activate(group = { Constants.CONSUMER}, order = 100)
public class TransactionPropagationFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionPropagationFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 获取 TM 放入上下文的 xid
        String xid = RootContext.getXID();
        // 将其 xid 放入其 RpcContext 中
        RpcContext.getContext().setAttachment(RootContext.KEY_XID, xid);
        return invoker.invoke(invocation);
    }
}
```

如果是 provider，当有 consumer 调用其 provider 暴露的方法，会调用如下删减后重要的代码。
```java
@Activate(group = { Constants.PROVIDER}, order = 100)
public class TransactionPropagationFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(TransactionPropagationFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 通过 rpcContext 获取从 TM 带来的 xid
        String rpcXid = RpcContext.getContext().getAttachment(RootContext.KEY_XID);
        // 绑定到本地 jvm 内存中
        RootContext.bind(rpcXid);
        return invoker.invoke(invocation);

        // 我这里把 try finally 删除了，这段代码会被执行
        // 解绑 xid，如果解绑的 xid 和 rpcContext 带来的 xid 不一致，再次绑定解绑的 xid
        String unbindXid = RootContext.unbind();
        if (!rpcXid.equalsIgnoreCase(unbindXid)) {
            LOGGER.warn("xid in change during RPC from " + rpcXid + " to " + unbindXid);
            if (unbindXid != null) {
                RootContext.bind(unbindXid);
                LOGGER.warn("bind [" + unbindXid + "] back to RootContext");
            }
        }
    }
}
```

总结一下。其实就是 TM 向 TC 发起全局事务开启，TC 给 TM 返回一个 xid，TM 将其 xid 绑定到 ThreadLocal 上，并通过 dubbo 自定义 filter通过消费服务的方式透传给服务方。

### 后续补充
1. undoLog 的实现
2. TC 锁的原理
3. 隔离性
4. RM 和 TC 的 netty 封装
