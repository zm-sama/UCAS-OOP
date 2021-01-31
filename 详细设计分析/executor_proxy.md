#  ExecutorServiceProxy

**executor服务的分布式实现。**

```java
public class ExecutorServiceProxy
        extends AbstractDistributedObject<DistributedExecutorService>
        implements IExecutorService {
```

继承自抽象分布类（一个很抽象的类，只提供四个基本方法），向下实现了最终呈现给外部的接口：IExecutorService 

其提供了其他方法，例如在特定成员上执行任务、在特定密钥的所有者上执行任务、在多个成员上执行任务以及使用回调侦听执行结果。

同时支持裂脑保护分离脑保护 其中的配置文件在集群版本3.10及更高版本。

将其中的主要方法整理为表格形式有：

| 修改器和类型                | 方法和描述                                                   |
| :-------------------------- | :----------------------------------------------------------- |
| `void`                      | `execute(Runnable command, MemberSelector memberSelector)`在随机选择的成员上执行任务。 |
| `void`                      | `executeOnAllMembers(Runnable command)`在所有已知群集成员上执行任务。 |
| `void`                      | `executeOnKeyOwner(Runnable command, Object key)`在指定密钥的所有者上执行任务。 |
| `void`                      | `executeOnMember(Runnable command, Member member)`在指定成员上执行任务。 |
| `void`                      | `executeOnMembers(Runnable command, Collection<Member> members)`在每个指定成员上执行任务。 |
| `void`                      | `executeOnMembers(Runnable command, MemberSelector memberSelector)`在每个选定成员上执行任务。 |
| `LocalExecutorStats`        | `getLocalExecutorStats()`返回与此执行器服务相关的本地统计信息。 |
| `<T> void`                  | `submit(Callable<T> task, ExecutionCallback<T> callback)`将任务提交给随机成员。 |
| `<T> Future<T>`             | `submit(Callable<T> task, MemberSelector memberSelector)`将任务提交给随机选择的成员，并返回表示该任务的 Future。 |
| `<T> void`                  | `submit(Callable<T> task, MemberSelector memberSelector, ExecutionCallback<T> callback)`将任务提交给随机选择的成员。 |
| `<T> void`                  | `submit(Runnable task, ExecutionCallback<T> callback)`将任务提交给随机成员。 |
| `<T> void`                  | `submit(Runnable task, MemberSelector memberSelector, ExecutionCallback<T> callback)`将任务提交给随机选择的成员。 |
| `<T> Map<Member,Future<T>>` | `submitToAllMembers(Callable<T> task)`将任务提交给所有群集成员，并返回成员-未来对的映射，表示每个成员上的任务即将完成。 |
| `<T> void`                  | `submitToAllMembers(Callable<T> task, MultiExecutionCallback callback)`将任务提交给所有群集成员。 |
| `void`                      | `submitToAllMembers(Runnable task, MultiExecutionCallback callback)`将任务提交给所有群集成员。 |
| `<T> Future<T>`             | `submitToKeyOwner(Callable<T> task, Object key)`将任务提交给指定密钥的所有者，并返回表示该任务的 Future。 |
| `<T> void`                  | `submitToKeyOwner(Callable<T> task, Object key, ExecutionCallback<T> callback)`将任务提交到指定密钥的所有者。 |
| `<T> void`                  | `submitToKeyOwner(Runnable task, Object key, ExecutionCallback<T> callback)`将任务提交给指定密钥的所有者。 |
| `<T> Future<T>`             | `submitToMember(Callable<T> task, Member member)`将任务提交到指定成员并返回表示该任务的 Future。 |
| `<T> void`                  | `submitToMember(Callable<T> task, Member member, ExecutionCallback<T> callback)`将任务提交到指定成员。 |
| `<T> void`                  | `submitToMember(Runnable task, Member member, ExecutionCallback<T> callback)`将任务提交到指定成员。 |
| `<T> Map<Member,Future<T>>` | `submitToMembers(Callable<T> task, Collection<Member> members)`向给定成员提交任务，并返回成员-未来对的返回图，表示每个成员的任务即将完成 |
| `<T> void`                  | `submitToMembers(Callable<T> task, Collection<Member> members, MultiExecutionCallback callback)`将任务提交到指定成员。 |
| `<T> Map<Member,Future<T>>` | `submitToMembers(Callable<T> task, MemberSelector memberSelector)`向选定的成员提交任务，并返回成员-未来对的映射，表示每个成员上的任务即将完成。 |
| `<T> void`                  | `submitToMembers(Callable<T> task, MemberSelector memberSelector, MultiExecutionCallback callback)`将任务提交到所选成员。 |
| `void`                      | `submitToMembers(Runnable task, Collection<Member> members, MultiExecutionCallback callback)`将任务提交到指定成员。 |
| `void`                      | `submitToMembers(Runnable task, MemberSelector memberSelector, MultiExecutionCallback callback)`将任务提交到所选成员。 |

## 正文

### 次要方法：

`check sync` 用于阻止不在进行中的任务重载当前系统，通过计算时间的方式，如果不在当前允许时间范围，set为0，阻止任务提交

```java
    private boolean checkSync() {
        boolean sync = false;
        long last = lastSubmitTime;
        long now = Clock.currentTimeMillis();
        if (last + SYNC_DELAY_MS < now) {
            CONSECUTIVE_SUBMITS.set(this, 0);
        } else if (CONSECUTIVE_SUBMITS.incrementAndGet(this) % SYNC_FREQUENCY == 0) {
            sync = true;
        }
        lastSubmitTime = now;
        return sync;
    }

```

根据键值获取分进程id号：

```java
    private <T> int getTaskPartitionId(Callable<T> task) {
        if (task instanceof PartitionAware) {
            Object partitionKey = ((PartitionAware) task).getPartitionKey();
            if (partitionKey != null) {
                return getNodeEngine().getPartitionService().getPartitionId(partitionKey);
            }
        }
        return random.nextInt(partitionCount);
    }
```

`invoke`方法：将任务加入对应的队列，包括带时间参数和不带时间参数的两个版本，其中带时间参数的还涉及到了wait函数，分别罗列如下：

```java
    @Nonnull
    @Override
    public <T> List<Future<T>> invokeAll(@Nonnull Collection<? extends Callable<T>> tasks)
            throws InterruptedException {
        checkNotNull(tasks, "tasks must not be null");
        List<Future<T>> futures = new ArrayList<>(tasks.size());//创建序列
        List<Future<T>> result = new ArrayList<>(tasks.size());//创建序列
        for (Callable<T> task : tasks) {
            futures.add(submit(task));
        }
        for (Future<T> future : futures) {
            result.add(completedSynchronously(future, getNodeEngine().getSerializationService()));
        }//将对应项加入序列
        return result;
    }
```

带时间参数更为复杂：

```java
@Nonnull
    @Override
    public <T> List<Future<T>> invokeAll(@Nonnull Collection<? extends Callable<T>> tasks,
                                         long timeout,
                                         @Nonnull TimeUnit unit) {
        checkNotNull(unit, "unit must not be null");
        checkNotNull(tasks, "tasks must not be null");

        long timeoutNanos = unit.toNanos(timeout);
        List<Future<T>> futures = new ArrayList<>(tasks.size());//同样创建序列
        boolean done = false;
        try {
            for (Callable<T> task : tasks) {
                long startNanos = Timer.nanos();
                int partitionId = getTaskPartitionId(task);
                futures.add(submitToPartitionOwner(task, partitionId, true));
                timeoutNanos -= Timer.nanosElapsed(startNanos);//完成一个任务后，时间片自减
            }
            if (timeoutNanos <= 0L) {
                return futures;//最终超过时间上限，则失败
            }

            done = wait(timeoutNanos, futures);//需要等待直到时间片结束
            return futures;
        } catch (Throwable t) {
            logger.severe(t);
            return futures;
        } finally {
            if (!done) {
                cancelAll(futures);//若没有最后的完成标志，则取消剩余的任务
            }
        }
    }
```

带时间参数的invoke方法中，较为关键的 `wait`函数：

```java
    private <T> boolean wait(long timeoutNanos, List<Future<T>> futures) throws InterruptedException {
        boolean done = true;
        for (int i = 0, size = futures.size(); i < size; i++) {
            long startNanos = Timer.nanos();
            Object value;
            try {
                Future<T> future = futures.get(i);//get方法拥有"状态依赖"的内在特性，因而调用者不需要知道任务的状态，此外在任务提交和获得结果中包含的安全发布属性也确保了这个方法是线程安全的。Future.get的异常处理代码将处理两个可能的问题：任务遇到一个Exception，时间片耗尽。
                value = future.get(timeoutNanos, TimeUnit.NANOSECONDS);
            } catch (ExecutionException e) {
                value = e;
            } catch (TimeoutException e) {//时间片耗尽，则返回失败
                done = false;
                for (int l = i; l < size; l++) {
                    Future<T> f = futures.get(i);
                    if (f.isDone()) {//若部分完成，则将已完成的任务放入future中
                        futures.set(l, completedSynchronously(f, getNodeEngine().getSerializationService()));
                    }
                }
                break;
            }

            futures.set(i, InternalCompletableFuture.newCompletedFuture(value));
            timeoutNanos -= Timer.nanosElapsed(startNanos);
        }
        return done;
    }
```

`cancel ALL`调用了future中的cancel方法取消所有list中的futuren型任务，较为简单，就不再多提。

`invoke Any`由于任意执行，所以只需要throw出奇怪的例外处理就行，内部就一行

后续还有很多细碎的判断方法，这里略去，基本看一眼就知道他在分析什么。

**值得注意的方法：** shutdown和shutdownNow:

- shutdown：停止，先将任务提交到关闭操作（后续调用方法包中的shutdown方法关闭）,最后用上最开始的关闭例外处理传给waitddl处理，相当于正在执行的任务会继续执行下去，没有被执行的则中断。
- shutdownNow: 最后额外加了返回一个空表，相当于强行停止所有任务，正在执行的任务也会被直接中断。

```java
    public void shutdown() {
        NodeEngine nodeEngine = getNodeEngine();
        Collection<Member> members = nodeEngine.getClusterService().getMembers();
        OperationService operationService = nodeEngine.getOperationService();
        Collection<Future> calls = new LinkedList<>();

        for (Member member : members) {
            Future f = submitShutdownOperation(operationService, member);
            calls.add(f);
        }
        waitWithDeadline(calls, 3, TimeUnit.SECONDS, shutdownExceptionHandler);// 
        //Calculate timeouts for whole operation and per future. If corresponding TimeUnits not set assume
        // the default of TimeUnit.SECONDS
    }

    private InvocationFuture<Object> submitShutdownOperation(OperationService operationService, Member member) {
        ShutdownOperation op = new ShutdownOperation(name);
        return operationService.invokeOnTarget(getServiceName(), op, member.getAddress());
    }
/**************************************************************************************************/
    @Nonnull
    @Override
    public List<Runnable> shutdownNow() {
        shutdown();
        return Collections.emptyList();
    }

```

最后是一个选择成员的函数，其中select方法经过笔者四层的深度搜索，发现在 `cluster.MemberSelector.java`方法中给出了一定的注释的select 但是似乎也没什么特殊的选择，可能就是一种随机的选择，重点在于选择的时候进行了附加的判断，而不是选择的方式：

```java
    private List<Member> selectMembers(@Nonnull MemberSelector memberSelector) {
        checkNotNull(memberSelector, "memberSelector must not be null");
        List<Member> selected = new ArrayList<>();
        Collection<Member> members = getNodeEngine().getClusterService().getMembers();
        for (Member member : members) {
            if (memberSelector.select(member)) {//以为很特殊，但实际很正常
                selected.add(member);
            }
        }
        if (selected.isEmpty()) {
            throw new RejectedExecutionException("No member selected with memberSelector[" + memberSelector + "]");//错误处理
        }
        return selected;
    }
```

```java
public interface MemberSelector {

    /**
     * Decides if the given member will be part of an operation or not.
     *
     * @param member the member instance to decide upon
     * @return true if the member should take part in the operation, false otherwise
     */
    boolean select(Member member);

}
```



### 主要方法：

**根据execute中不同参数决定在怎样的成员上执行任务**

**根据submit中不同参数决定将任务提交给哪些成员**

但execute返回值倒是void 所以只有6种

但submit返回值也有可能是future型 或者map型 所以总共有多达20种不同的submit方法。

代码高度相似，但主要通过**编写主方法，剩余方法改写的方式实现** ，以submit to member为例：

首先编写了主方法，其中包含了各种可能用上的信息，包括uuid 实际地址，节点启动，等

同时根据sync检测信息给出了报错信息

参考上面的方法列表，三种不同的方法都通过 `@override`方式实现覆盖，减少了代码量，同时减少后期维护成本。

```java
    private <T> void submitToMember(@Nonnull Data taskData,
                                    @Nonnull Member member,
                                    @Nullable ExecutionCallback<T> callback) {
        checkNotNull(member, "member must not be null");
        checkNotShutdown();

        NodeEngine nodeEngine = getNodeEngine();
        UUID uuid = newUnsecureUUID();
        MemberCallableTaskOperation op = new MemberCallableTaskOperation(name, uuid, taskData);
        OperationService operationService = nodeEngine.getOperationService();
        Address address = member.getAddress();
        //完成了对象的创建
        InvocationFuture<T> future = operationService
                .createInvocationBuilder(DistributedExecutorService.SERVICE_NAME, op, address)
                .invoke();
        if (callback != null) {
            future.whenCompleteAsync(new ExecutionCallbackAdapter<>(callback))
                    .whenCompleteAsync((v, t) -> {
                        if (t instanceof RejectedExecutionException) {
                            callback.onFailure(t);
                        }//错误处理
                    });
        }
    }

    @Override
    public <T> void submitToMember(@Nonnull Callable<T> task,
                                   @Nonnull Member member,
                                   @Nullable ExecutionCallback<T> callback) {
        checkNotNull(task, "task must not be null");
        checkNotShutdown();

        Data taskData = getNodeEngine().toData(task);
        submitToMember(taskData, member, callback);
    }
```

其余高度相似部分就不再贴代码分析，下面给出来自官网的（经过个人不太靠谱的翻译）的方法信息：、

> https://docs.hazelcast.org/docs/4.0.2/javadoc/com/hazelcast/core/IExecutorService.html

- #### 执行

  ```
  void execute(@Nonnull
               Runnable command,
               @Nonnull
               MemberSelector memberSelector)
  ```

  在随机选择的成员上执行任务。

  - **参数：**

    `command`- 在随机选择的成员上执行的任务

    `memberSelector`- 会员选择

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 执行密钥所有者

  ```
  void executeOnKeyOwner(@Nonnull
                         Runnable command,
                         @Nonnull
                         Object key)
  ```

  在指定密钥的所有者上执行任务。

  - **参数：**

    `command`- 在指定密钥的所有者上执行的任务

    `key`- 指定的键



- #### 执行成员

  ```
  void executeOnMember(@Nonnull
                       Runnable command,
                       @Nonnull
                       Member member)
  ```

  在指定成员上执行任务。

  - **参数：**

    `command`- 在指定成员上执行的任务

    `member`- 指定成员



- #### 执行成员

  ```
  void executeOnMembers(@Nonnull
                        Runnable command,
                        @Nonnull
                        Collection<Member> members)
  ```

  在每个指定成员上执行任务。

  - **参数：**

    `command`- 在指定成员上执行的任务

    `members`- 指定成员



- #### 执行成员

  ```
  void executeOnMembers(@Nonnull
                        Runnable command,
                        @Nonnull
                        MemberSelector memberSelector)
  ```

  在每个选定成员上执行任务。

  - **参数：**

    `command`- 在每个选定成员上执行的任务

    `memberSelector`- 会员选择

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 执行所有成员

  ```
  void executeOnAllMembers(@Nonnull
                           Runnable command)
  ```

  在所有已知群集成员上执行任务。

  - **参数：**

    `command`- 在所有已知群集成员上执行的任务



- #### 提交

  ```
  <T> Future<T> submit(@Nonnull
                       Callable<T> task,
                       @Nonnull
                       MemberSelector memberSelector)
  ```

  将任务提交给随机选择的成员，并返回表示该任务的 Future。

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 任务提交给随机选择的成员

    `memberSelector`- 会员选择

  - **返回：**

    a 未来表示任务即将完成

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交键所有者

  ```
  <T> Future<T> submitToKeyOwner(@Nonnull
                                 Callable<T> task,
                                 @Nonnull
                                 Object key)
  ```

  将任务提交给指定密钥的所有者，并返回表示该任务的 Future。

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给指定密钥所有者的任务

    `key`- 指定的键

  - **返回：**

    a 未来表示任务即将完成



- #### 提交到会员

  ```
  <T> Future<T> submitToMember(@Nonnull
                               Callable<T> task,
                               @Nonnull
                               Member member)
  ```

  将任务提交到指定成员并返回表示该任务的 Future。

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交到指定成员的任务

    `member`- 指定成员

  - **返回：**

    a 未来表示任务即将完成



- #### 提交会员

  ```
  <T> Map<Member,Future<T>> submitToMembers(@Nonnull
                                            Callable<T> task,
                                            @Nonnull
                                            Collection<Member> members)
  ```

  向给定成员提交任务，并返回成员-未来对的返回图，表示每个成员的任务即将完成

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给给定成员的任务

    `members`- 给定成员

  - **返回：**

    代表每个成员任务即将完成的成员-未来对的地图



- #### 提交会员

  ```
  <T> Map<Member,Future<T>> submitToMembers(@Nonnull
                                            Callable<T> task,
                                            @Nonnull
                                            MemberSelector memberSelector)
  ```

  向选定的成员提交任务，并返回成员-未来对的映射，表示每个成员上的任务即将完成。

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给选定成员的任务

    `memberSelector`- 会员选择

  - **返回：**

    代表每个成员任务即将完成的成员-未来对的地图

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交所有会员

  ```
  <T> Map<Member,Future<T>> submitToAllMembers(@Nonnull
                                               Callable<T> task)
  ```

  将任务提交给所有群集成员，并返回成员-未来对的映射，表示每个成员上的任务即将完成。

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给所有群集成员的任务

  - **返回：**

    代表每个成员任务即将完成的成员-未来对的地图



- #### 提交

  ```
  <T> void submit(@Nonnull
                  Runnable task,
                  @Nullable
                  ExecutionCallback<T> callback)
  ```

  将任务提交给随机成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 回调的响应类型

  - **参数：**

    `task`- 提交给随机成员的任务

    `callback`- 回调



- #### 提交

  ```
  <T> void submit(@Nonnull
                  Runnable task,
                  @Nonnull
                  MemberSelector memberSelector,
                  @Nullable
                  ExecutionCallback<T> callback)
  ```

  将任务提交给随机选择的成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - `T`- 回调的响应类型

  - **参数：**

    `task`- 提交给随机选择的成员的任务

    `memberSelector`- 会员选择

    `callback`- 回调

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交键所有者

  ```
  <T> void submitToKeyOwner(@Nonnull
                            Runnable task,
                            @Nonnull
                            Object key,
                            @Nonnull
                            ExecutionCallback<T> callback)
  ```

  将任务提交给指定密钥的所有者。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 回调的响应类型

  - **参数：**

    `task`- 提交给指定密钥所有者的任务

    `key`- 指定的键

    `callback`- 回调



- #### 提交到会员

  ```
  <T> void submitToMember(@Nonnull
                          Runnable task,
                          @Nonnull
                          Member member,
                          @Nullable
                          ExecutionCallback<T> callback)
  ```

  将任务提交到指定成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 回调的响应类型

  - **参数：**

    `task`- 提交到指定成员的任务

    `member`- 指定成员

    `callback`- 回调



- #### 提交会员

  ```
  void submitToMembers(@Nonnull
                       Runnable task,
                       @Nonnull
                       Collection<Member> members,
                       @Nonnull
                       MultiExecutionCallback callback)
  ```

  将任务提交到指定成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **参数：**

    `task`- 提交给指定成员的任务

    `members`- 指定成员

    `callback`- 回调



- #### 提交会员

  ```
  void submitToMembers(@Nonnull
                       Runnable task,
                       @Nonnull
                       MemberSelector memberSelector,
                       @Nonnull
                       MultiExecutionCallback callback)
  ```

  将任务提交到所选成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **参数：**

    `task`- 提交给选定成员的任务

    `memberSelector`- 会员选择

    `callback`- 回调

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交所有会员

  ```
  void submitToAllMembers(@Nonnull
                          Runnable task,
                          @Nonnull
                          MultiExecutionCallback callback)
  ```

  将任务提交给所有群集成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **参数：**

    `task`- 提交给所有群集成员的任务

    `callback`- 回调



- #### 提交

  ```
  <T> void submit(@Nonnull
                  Callable<T> task,
                  @Nullable
                  ExecutionCallback<T> callback)
  ```

  将任务提交给随机成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给随机成员的任务

    `callback`- 回调



- #### 提交

  ```
  <T> void submit(@Nonnull
                  Callable<T> task,
                  @Nonnull
                  MemberSelector memberSelector,
                  @Nullable
                  ExecutionCallback<T> callback)
  ```

  将任务提交给随机选择的成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给随机选择的成员的任务

    `memberSelector`- 会员选择

    `callback`- 回调

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交键所有者

  ```
  <T> void submitToKeyOwner(@Nonnull
                            Callable<T> task,
                            @Nonnull
                            Object key,
                            @Nullable
                            ExecutionCallback<T> callback)
  ```

  将任务提交到指定密钥的所有者。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给指定密钥所有者的任务

    `key`- 指定的键

    `callback`- 回调



- #### 提交到会员

  ```
  <T> void submitToMember(@Nonnull
                          Callable<T> task,
                          @Nonnull
                          Member member,
                          @Nullable
                          ExecutionCallback<T> callback)
  ```

  将任务提交到指定成员。调用方将收到通过执行调用的返回对象或失败（的话）通知任务的结果

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交到指定成员的任务

    `member`- 指定成员

    `callback`- 回调



- #### 提交会员

  ```
  <T> void submitToMembers(@Nonnull
                           Callable<T> task,
                           @Nonnull
                           Collection<Member> members,
                           @Nonnull
                           MultiExecutionCallback callback)
  ```

  将任务提交到指定成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给指定成员的任务

    `members`- 指定成员

    `callback`- 回调



- #### 提交会员

  ```
  <T> void submitToMembers(@Nonnull
                           Callable<T> task,
                           @Nonnull
                           MemberSelector memberSelector,
                           @Nonnull
                           MultiExecutionCallback callback)
  ```

  将任务提交到所选成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给选定成员的任务

    `memberSelector`- 会员选择

    `callback`- 回调

  - **抛出：**

    `RejectedExecutionException`- 如果未选择任何成员



- #### 提交所有会员

  ```
  <T> void submitToAllMembers(@Nonnull
                              Callable<T> task,
                              @Nonnull
                              MultiExecutionCallback callback)
  ```

  将任务提交给所有群集成员。多执行调用回对象将通知呼叫者每个任务的结果，并且当所有任务完成时，将调用多执行调用回MAP类型成员

  - **类型参数：**

    `T`- 可调用的结果类型

  - **参数：**

    `task`- 提交给所有群集成员的任务

    `callback`- 回调



- #### getlocalExcutorStats

  ```
  LocalExecutorStats getLocalExecutorStats()
  ```

  返回与此执行器服务相关的本地统计信息。

  - **返回：**

    与此执行器服务相关的本地统计信息