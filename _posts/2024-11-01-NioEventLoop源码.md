---
title: NioEventLoop源码
date: 2024-11-01 20:30:00 +0800
categories: [网络编程, Netty]
tags: [netty]     
---

# NioEventLoop源码

## 继承结构

![NioEventLoop.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/NioEventLoop.png)

## 初始化过程

入口：io.netty.channel.nio.NioEventLoopGroup#newChild

方法入参：

`executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());`

`args[0] = SelectorProvider.provider();`

`args[1] = DefaultSelectStrategyFactory.INSTANCE;`

`args[2] = RejectedExecutionHandlers.reject();`

![image.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/image.png)

上面newChild方法最后一行调用NioEventLoop构造器创建实例并将当前NioEventLoopGroup设置到NioEventLoop的parent属性（实际上该属性被其父类声明）

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory taskQueueFactory, EventLoopTaskQueueFactory tailTaskQueueFactory) {
    super(parent, executor, false, newTaskQueue(taskQueueFactory), newTaskQueue(tailTaskQueueFactory),
        rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}

private SelectorTuple openSelector() {
		//通过jdk底层方法获取当前环境下可用的java原生Selector对象
		final Selector unwrappedSelector;
		try {
			  unwrappedSelector = provider.openSelector();
		} catch (IOException e) {
		    throw new ChannelException("failed to open a new selector", e);
		}

		//尝试加载类：sun.nio.ch.SelectorImpl
		Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
		    @Override
		    public Object run() {
		        try {
		            return Class.forName(
		                    "sun.nio.ch.SelectorImpl",
		                    false,
		                    PlatformDependent.getSystemClassLoader());
		        } catch (Throwable cause) {
		            return cause;
		        }
		    }
		});

		//如果当前环境下是不存在sun.nio.ch.SelectorImpl类，
		//或者当前环境下的Selector对象不是sun.nio.ch.SelectorImpl或其子类
    if (!(maybeSelectorImplClass instanceof Class) ||
        // ensure the current selector implementation is what we can instrument.
        !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
        if (maybeSelectorImplClass instanceof Throwable) {
            Throwable t = (Throwable) maybeSelectorImplClass;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
        }
        return new SelectorTuple(unwrappedSelector);
    }

		// 如果 Selector 是 SelectorImpl 类型，则尝试进行优化
    // 因为 sun.nio.ch.SelectorImpl 在高并发下有 epoll bug，导致 Selector#select 方法空轮询
    // Netty 会通过反射机制修改内部的 SelectedKeys 集合，将其替换成线程安全、优化的集合
    final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;
    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

    Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            try {
                Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                    // Let us try to use sun.misc.Unsafe to replace the SelectionKeySet.
                    // This allows us to also do this in Java9+ without any extra flags.
                    long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                    long publicSelectedKeysFieldOffset =
                            PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                    if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                        PlatformDependent.putObject(
                                unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                        PlatformDependent.putObject(
                                unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                        return null;
                    }
                    // We could not retrieve the offset, lets try reflection as last-resort.
                }

                Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                if (cause != null) {
                    return cause;
                }
                cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                if (cause != null) {
                    return cause;
                }

                selectedKeysField.set(unwrappedSelector, selectedKeySet);
                publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                return null;
            } catch (NoSuchFieldException e) {
                return e;
            } catch (IllegalAccessException e) {
                return e;
            }
        }
    });

        if (maybeException instanceof Exception) {
            selectedKeys = null;
            Exception e = (Exception) maybeException;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
            return new SelectorTuple(unwrappedSelector);
        }
        selectedKeys = selectedKeySet;
        logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
        return new SelectorTuple(unwrappedSelector,
                                 new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
    }
```

上面构造NioEventLoop的过程的最后三行

`final SelectorTuple selectorTuple = openSelector();`
`this.selector = selectorTuple.selector;`
`this.unwrappedSelector = selectorTuple.unwrappedSelector;`

首先获取当前环境的Selector实例，然后判断是否是sun.nio.ch.SelectorImpl类型。如果是则通过反射对该实例进行修改优化（因为 sun.nio.ch.SelectorImpl 在高并发下有 epoll bug，导致 Selector#select 方法空轮询）并封装成netty自己的Selector设置到`SelectorTuple.selector` 变量中，`SelectorTuple.unwrappedSelector` 变量中存储的则是jdk提供的原始Selector实例。

我们继续追踪父类构造过程：

```java
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
    tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
}
```

```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    this.executor = ThreadExecutorMap.apply(executor, this);
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

```java
protected AbstractScheduledEventExecutor(EventExecutorGroup parent) {
    super(parent);
}
```

```java
private final EventExecutorGroup parent;

protected AbstractEventExecutor(EventExecutorGroup parent) {
    this.parent = parent;
}
```

截止目前NioEventLoop及相关父类属性值情况如下：

| 类   | NioEventLoop                                                                                                                                  | SingleThreadEventLoop                                                                                 | SingleThreadEventExecutor                                                                                                              | AbstractEventExecutor                                           |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| 字段 | `this.provider = SelectorProvider.provider();`                                                                                                | // 初始化一个容量为Integer.MAX_VALUE的队列 `tailTasks = PlatformDependent.<Runnable>newMpscQueue()；` | `this.addTaskWakesUp = false;`                                                                                                         | //设置parent变量为NioEventLoopGroup对象 `this.parent = parent;` |
|      | `this.selectStrategy = DefaultSelectStrategyFactory.INSTANCE;`                                                                                |                                                                                                       | //DEFAULT_MAX_PENDING_EXECUTOR_TASKS默认值情况下取值为：INTEGER.MAX_VALUE `this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;` |                                                                 |
|      | //Netty基于java Selector自行优化封装的Selector对象   `this.selector = new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet)` |                                                                                                       | this.executor = ThreadExecutorMap.apply(executor, this);                                                                               |                                                                 |
|      | `this.unwrappedSelector = SelectorProvider.provider().openSelector()；`                                                                       |                                                                                                       | this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");                                                                      |                                                                 |
|      |                                                                                                                                               |                                                                                                       | //添加任务失败时的拒绝策略，当前设置为抛出异常的拒绝策略`this.rejectedExecutionHandler = RejectedExecutionHandlers.reject()`           |                                                                 |

## 事件循环核心run方法

```java
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
		        **//第1部分**
            int strategy;
            try {
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; // nothing on the calendar
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary selector wakeups
                        // so use of lazySet is ok (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }
            **//第1部分--end**
            
						**//第2部分--start**
            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }
            **//第2部分--end**

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
        } catch (Error e) {
            throw e;
        } catch (Throwable t) {
            handleLoopException(t);
        } finally {
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Error e) {
                throw e;
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}
```

我们将上述代码按照注释所示分成2个部分。

第1个部分的主要逻辑是：判断当前队列中是否有需要执行的任务。如果已经有待执行任务，为了能尽快进行执行则调用selectNow()方法无阻塞的从Selector中获取已就绪的io事件，或者根据下一个即将要执行的定时任务执行时间来使用select(long timeout)进行一定时间的等待。如果当前无任何任务需要执行则可以正常调用select()方法以阻塞的方式等待io就绪事件到达。

第2个部分的主要逻辑是：根据ioRatio属性配置（默认50，意味着每次循环执行io事件的时间和执行其他任务的时间比例为1:1) 来确定当次循环中处理io事件的时间和处理队列任务时间的比例。顺序是先执行io事件，再消费队列任务。

其中处理io事件所调用的方法是：`processSelectedKeys()` ,该方法主要用来对io就绪事件进行处理

## NioEventLoop生命周期

### 初始化

- 在创建NioEventLoopGroup对象时会创建指定数量（由NioEventLoopGroup构造器参数指定或使用默认数量）的NioEventLoop。并且在NioEventLoop的父类AbstractEventExecutor中通过parent变量来关联其和NioEventLoopGroup对象的关系。
- 在创建NioEventLoop对象时会初始化一个Selector，待后续channel注册到该EventLoop时会将channel注册到该Selector。channel会通过Selector注册其感兴趣的事件，当感兴趣的事件就绪时可以通过select()方法获取到该事件以进行消费
- 在创建NioEventLoop对象时同时也会创建2个任务队列
- 每个NioEventLoop对象对应着一个单一的线程
- 在初始化时并不会立即启动事件循环

### 启动

- 当向NioEventLoop注册channel时会触发第一次execute方法调用，此时由于执行注册的是另外的线程因此会执行startThread()方法来启动事件循环（run方法）。事件循环启动后即可消费具体的注册任务。

![image.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/image%201.png)

![image.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/image%202.png)

![image.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/image%203.png)

![image.png](/assets/images/NioEventLoop%E6%BA%90%E7%A0%81/image%204.png)

- `run()` 方法是一个无限循环，负责处理 I/O 事件、普通任务和定时任务。
- 启动后，`NioEventLoop` 就进入了“启动”状态，不断监听 `Selector` 上的事件，并对就绪的 `Channel` 执行相应的处理。

### 执行事件循环

- 事件循环负责以下几类任务：
    - **I/O 事件处理**：利用 `Selector` 的 `select()` 方法监听并处理连接、读、写等 I/O 事件。
    - **普通任务**：执行 `NioEventLoop` 内部或用户提交的任务，任务会被存入任务队列中，由事件循环线程按顺序执行。
    - **定时任务**：处理延时或定时任务，通过定时任务队列管理，按调度顺序执行。
- 事件循环还会根据当前是否有待处理任务，灵活调整 `Selector` 的 `select()` 超时时间或选择是否阻塞，以避免性能问题。

### 关闭

- 当 `NioEventLoop` 的 `shutdownGracefully()` 方法被调用时，生命周期进入关闭流程，等待当前任务完成并拒绝新任务。
- `NioEventLoop` 将停止接收新的连接请求和任务，并等待已经存在的任务和事件处理完成。
- 关闭期间会唤醒 `Selector`，从而使事件循环线程跳出 `select()` 阻塞，进入清理阶段。

### 清理资源

- 在退出事件循环后，`NioEventLoop` 会关闭 `Selector`，并释放相关资源。
- 最终，`NioEventLoop` 所管理的线程会彻底终止，生命周期完成。

### 关键方法

- **register(Channel, EventLoop)**：将 `Channel` 注册到 `NioEventLoop` 的 `Selector` 上，启动 `EventLoop` 的线程。
- **execute(Runnable task)**：添加任务到队列，由 `EventLoop` 线程执行。
- **shutdownGracefully()**：关闭 `EventLoop` 的方法，使其进入优雅关闭状态