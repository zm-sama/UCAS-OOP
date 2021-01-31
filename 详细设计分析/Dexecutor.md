# DistributedExecutorService

**为分布式实现的基础，在proxy中调用最后呈现给使用者**

**分析主要结合代码为主**

#### 前置知识：ConcurrentHashMap

ConcurrentHashMap是Java1.5中引用的一个线程安全的支持高并发的HashMap集合类。

其方法和特点为：采用了非常精妙的"分段锁"策略，ConcurrentHashMap的主干是个Segment数组。Segment继承了ReentrantLock，所以它就是一种可重入锁（ReentrantLock)。在ConcurrentHashMap，一个Segment就是一个子哈希表，Segment里维护了一个HashEntry数组，并发环境下，对于不同Segment的数据进行操作是**不用考虑锁竞争的** 。

比较常用的方法：

- ConcurrentHashMap的put方法：根据hash值找到对应的segment，segment是分段锁。
- ConcurrentHashMap的get方法：count>0,hash找到HashEntry,hash相等并且key相同，若取value为null，加锁重新获取。
- ConcurrentHashMap的remove方法：加锁，每删除一个元素就将那之前的元素克隆一边。因为设置为第一次next之后不能再改变。
- ConcurrentHashMap的size（）方法：2次不锁住segment方式统计各个segment的大小，若count发生变化，采用加锁方式统计。modCount变量，在put，remove和clean方法里操作元素，modcount加1.

### 前置知识：裂脑保护

所谓的脑裂，就是指在主从集群中，同时有两个主节点，它们都能接收写请求。而脑裂最直接的影响，就是客户端不知道应该往哪个主节点写入数据，结果就是不同的客户端会往不同的主节点上写入数据。而且，严重的话，脑裂会进一步导致数据丢失。

hazelcast提供了一种保护方法，可以防止这种裂脑的发生

具体内容由于和主体方法没有太大关联，这里就略去不提了。

## 正文：

`CallableProcessor.responseFlag`的更新：

```java
    private static final AtomicReferenceFieldUpdater<Processor, Boolean> RESPONSE_FLAG =
            AtomicReferenceFieldUpdater.newUpdater(Processor.class, Boolean.class, "responseFlag");

    // package-local access to allow test to inspect the map's values
    final ConcurrentMap<String, ExecutorConfig> executorConfigCache = new ConcurrentHashMap<String, ExecutorCodnfig>();
```

### 定义对象：

定义的所有对象如下： 其中shutdownExecuto使用的方式较为巧妙 是jdk1.6中加入的新特性

```java
    private NodeEngine nodeEngine;
    private ExecutionService executionService;
    private final ConcurrentMap<UUID, Processor> submittedTasks = new ConcurrentHashMap<>();//利用前述的concurrenthashmap创建提交任务
    private final Set<String> shutdownExecutors
            = Collections.newSetFromMap(new ConcurrentHashMap<>());
	//关闭方法 生成对Map进行包装的Set。
	//这个Set和被包装的Map拥有相同的key顺序（遍历Set调用的还是Map的keySet），相同的并发特性（也就是说如果对ConcurrentHashMap进行包装，得到的Set也将线程安全）。
	//本质上来说，这个工厂方法（newSetFromMap）就是提供了一个和Map实现相对应的Set实现。

    private final ExecutorStats executorStats = new ExecutorStats();//创建executor开始的总类
    private final ConcurrentMap<String, Object> splitBrainProtectionConfigCache = new ConcurrentHashMap<String, Object>();
    private final ContextMutexFactory splitBrainProtectionConfigCacheMutexFactory = new ContextMutexFactory();
    private final ConstructorFunction<String, Object> splitBrainProtectionConfigConstructor =
            new ConstructorFunction<String, Object>() {
                @Override
                public Object createNew(String name) {
                    ExecutorConfig executorConfig = nodeEngine.getConfig().findExecutorConfig(name);
                    String splitBrainProtectionName = executorConfig.getSplitBrainProtectionName();
                    return splitBrainProtectionName == null ? NULL_OBJECT : splitBrainProtectionName;
                }
            };
	//上面三个对象都是脑裂保护用
    private ILogger logger;
```

### 次要方法：

最开始的初始化任务，初始化了节点、整体服务程序、日志以及调用 `metric库`的监控系统的初始化：

```java
    @Override
    public void init(NodeEngine nodeEngine, Properties properties) {
        this.nodeEngine = nodeEngine;
        this.executionService = nodeEngine.getExecutionService();
        this.logger = nodeEngine.getLogger(DistributedExecutorService.class);

        ((NodeEngineImpl) nodeEngine).getMetricsRegistry().registerDynamicMetricsProvider(this);
    }
```

封装了 `shutdown` 时调用的 `reset` 函数，主要功能是清空所有的线程池内容（包括cache）,重置executor启动部分等，代码如下：

```java
    public void reset() {
        shutdownExecutors.clear();
        submittedTasks.clear();
        executorStats.clear();
        executorConfigCache.clear();
    }
```

分布式执行时，`cancel`方法通过前置知识中的concurrentmap中的remove在集合中取消掉这一个进程的位置；再进行多次判断，需要在确实进程不空+收到了取消例外的标志+数据有效的情况下才真正取消

```java
    public boolean cancel(UUID uuid, boolean interrupt) {
        Processor processor = submittedTasks.remove(uuid);//调用remove
        if (processor != null && processor.cancel(interrupt)) {
            if (processor.sendResponse(new CancellationException())) {
                if (processor.isStatisticsEnabled()) {
                    executorStats.cancelExecution(processor.name);
                }
                return true;//仅当
            }
        }
        return false;
    }
```

`shutdownExecutor`中调用java库函数的取消，同时在进程config_cache中移除该进程：

```java
    public void shutdownExecutor(String name) {
        executionService.shutdownExecutor(name);
        shutdownExecutors.add(name);
        executorConfigCache.remove(name);
    }
```

同时与代理的交互主要通过创建分布式对象完成：

```java
    @Override
    public ExecutorServiceProxy createDistributedObject(String name, UUID source, boolean local) {
        return new ExecutorServiceProxy(name, nodeEngine, this);
    }
```

与前述的初始化任务相对应对的是摧毁进程，主要操作基本类似，去除所占空间，最后取消裂脑保护：

```java
    @Override
    public void destroyDistributedObject(String name, boolean local) {
        shutdownExecutors.remove(name);
        executionService.shutdownExecutor(name);
        executorStats.removeStats(name);
        executorConfigCache.remove(name);
        splitBrainProtectionConfigCache.remove(name);
    }
```

由于引入了cache系统，当需要查找config信息的时候要么选择查找，要么选择创建并将其加入缓存池，代码及注释如下：

```java
    /**
     * Locate the {@code ExecutorConfig} in local {@link
     * #executorConfigCache} or find it from {@link
     * NodeEngine#getConfig()} and cache it locally.
     */
    private ExecutorConfig getOrFindExecutorConfig(String name) {
        ExecutorConfig cfg = executorConfigCache.get(name);
        if (cfg != null) {
            return cfg;
        } else {
            cfg = nodeEngine.getConfig().findExecutorConfig(name);
            ExecutorConfig executorConfig = executorConfigCache.putIfAbsent(name, cfg);
            return executorConfig == null ? cfg : executorConfig;
        }
    }

```

有关裂脑保护的部分

### 主要方法：

#### Processor方法：继承自java库函数

```java
private final class Processor extends FutureTask implements Runnable {
```

内部参数如下：

```java
        //is being used through the RESPONSE_FLAG. Can't be private due to reflection constraint.
        volatile Boolean responseFlag = Boolean.FALSE;

        private final String name;
        private final UUID uuid;
        private final Operation op;
        private final String taskToString;
        private final long creationTime = Clock.currentTimeMillis();
        private final boolean statisticsEnabled;
```

提供了两种构造方法，分别对应runnable和callable类型（代码仅展示一种）

```java

        private Processor(String name, UUID uuid,
                          @Nonnull Callable callable,
                          Operation op,
                          boolean statisticsEnabled) {
            //noinspection unchecked
            super(callable);
            this.name = name;
            this.uuid = uuid;
            this.taskToString = String.valueOf(callable);
            this.op = op;
            this.statisticsEnabled = statisticsEnabled;
        }
```

核心函数 run函数中调用监察函数计时，返回给最外部的接口 `executorStats` 同时也处理了例外，具体处理方式见下述代码及注释：

```java
        @Override
        public void run() {
            long start = Clock.currentTimeMillis();//计时开始
            if (statisticsEnabled) {
                executorStats.startExecution(name, start - creationTime);
            }//数据可用，记录开始执行
            Object result = null;
            try {
                super.run();
                if (!isCancelled()) {
                    result = get();
                }//非取消情况，得到结果
            } catch (Exception e) {
                logException(e);
                result = e;
            } //否则，得到例外信息，保存至结果
            	finally {//说明运行结束or取消，反正进程结束了
                if (uuid != null) {
                    submittedTasks.remove(uuid);
                }
                if (!isCancelled()) {
                    sendResponse(result);
                    if (statisticsEnabled) {
                        executorStats.finishExecution(name, Clock.currentTimeMillis() - start);
                    }//非取消，向外传递结果，同时记录结束执行到最外层接口
                }
            }
        }
```

还有三个小的分方法, 内容及实现方式直接在代码框中给出：

```java
        private void logException(Exception e) {
            if (logger.isFinestEnabled()) {
                logger.finest("While executing callable: " + taskToString, e);
            }
        }//发生例外时的日志，

        private boolean sendResponse(Object result) {
            if (RESPONSE_FLAG.compareAndSet(this, Boolean.FALSE, Boolean.TRUE)) {
                try {
                    op.sendResponse(result);
                } catch (HazelcastSerializationException e) {
                    op.sendResponse(e);
                }
                return true;
            }

            return false;
        }//发出结果，根据是否发生例外来选择发送结果/例外信息

        boolean isStatisticsEnabled() {
            return statisticsEnabled;
        }//返回布尔值，在外部函数调用时在，这个值由任务cfg信息确定
```

### execute方法：调用processor方法，完成最终运行

- 由于缓存cache的加入 需要用 `getOrFindExecutorConfig(name)`获取cfg信息
- 根据任务类型，二选一完成方法调用
- 尝试执行，如果返回错误，则向外发出错误信息类型，并将该uuid号从哈希表中移除（加锁）



```java
public <T> void execute(String name, UUID uuid,
                            @Nonnull T task, Operation op) {
        ExecutorConfig cfg = getOrFindExecutorConfig(name);
        if (cfg.isStatisticsEnabled()) {
            executorStats.startPending(name);
        }
        Processor processor;
        if (task instanceof Runnable) {
            processor = new Processor(name, uuid, (Runnable) task, op, cfg.isStatisticsEnabled());
        } else {
            processor = new Processor(name, uuid, (Callable) task, op, cfg.isStatisticsEnabled());
        }//注意数据是否有效是保存在cfg中
        if (uuid != null) {
            submittedTasks.put(uuid, processor);
        }//二选一

        try {
            executionService.execute(name, processor);
        } catch (RejectedExecutionException e) {
            if (cfg.isStatisticsEnabled()) {
                executorStats.rejectExecution(name);
            }
            logger.warning("While executing " + task + " on Executor[" + name + "]", e);
            if (uuid != null) {
                submittedTasks.remove(uuid);
            }
            processor.sendResponse(e);
        }
    }
```





## 总结

主体实现是调java库，但是不同的是hazelcast中给出了很多错误处理情况，方便开发者快速定位错误

同时在4.0版本中还额外加入了裂脑保护，进一步减少分布式并发执行时可能出错的概率，提高这种大型软件的稳定性