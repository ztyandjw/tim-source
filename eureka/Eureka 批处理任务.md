#Eureka 批处理任务


队列的一些知识

只有LinkedBlockingQueue与ArrayBlockingQueue支持有界队列

add与offer区别， 当队列满了， add抛出uncheck exception， offer返回false

put仅当是阻塞队列才可以使用

remove和poll区别，poll支持超时，如果没有元素返回null， remove报错

take同put


####1.新任务触发入口

PeerAwareInstanceRegistryImpl#replicateToPeers

* peerEurekaNodes.getPeerEurekaNodes() //TODO

```bash
private void replicateToPeers(Action action, String appName, String id,InstanceInfo info
                              InstanceStatus newStatus, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // 假设是自己，continue
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            //**注释1**
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
```

PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers


```java

private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                //**注释1**
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    }
}

```

以register为例子

PeerEurekaNode#register

* expiretime 计算任务过期时间

* taskId("register", info)  计算task的ID, requestType + '#' + appName + '/' + id;

* batchingDispatcher:  private final TaskDispatcher<String, ReplicationTask> batchingDispatcher;  下面详解

* InstanceReplicationTask 节点与节点同步的任务，下面详解

```bash

public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    //private final TaskDispatcher<String, ReplicationTask> batchingDispatcher;
    batchingDispatcher.process(
        taskId("register", info),
        new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
            public EurekaHttpResponse<Void> execute() {
                return replicationClient.register(info);
            }
        },
        expiryTime
    );
}
```

TaskDispatcher接口，没有具体实现类

    public interface TaskDispatcher<ID, T> {

        void process(ID id, T task, long expiryTime);

        void shutdown();
    }


PeerEurekaNode#构造方法

通过TaskDispatchers.createBatchingTaskDispatcher方法 构造TaskDispatcher

```java
PeerEurekaNode(PeerAwareInstanceRegistry registry, String targetHost, String serviceUrl,
                                     HttpReplicationClient replicationClient, EurekaServerConfig config,
                                     int batchSize, long maxBatchingDelayMs,
                                     long retrySleepTimeMs, long serverUnavailableSleepTimeMs) {
        this.registry = registry;
        this.targetHost = targetHost;
        this.replicationClient = replicationClient;

        this.serviceUrl = serviceUrl;
        this.config = config;
        this.maxProcessingDelayMs = config.getMaxTimeForReplication();

        String batcherName = getBatcherName();
        ReplicationTaskProcessor taskProcessor = new ReplicationTaskProcessor(targetHost, replicationClient);
        this.batchingDispatcher = TaskDispatchers.createBatchingTaskDispatcher(
                batcherName,
                config.getMaxElementsInPeerReplicationPool(),
                batchSize,
                config.getMaxThreadsForPeerReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
        this.nonBatchingDispatcher = TaskDispatchers.createNonBatchingTaskDispatcher(
                targetHost,
                config.getMaxElementsInStatusReplicationPool(),
                config.getMaxThreadsForStatusReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
    }

```

TaskDispatchers#createBatchingTaskDispatcher

taskDispatcher任务分发器，关联任务接收执行器， 任务执行器

taskDispatcher调用process，其实调用的是任务接收执行器的process方法

```java
public static <ID, T> TaskDispatcher<ID, T> createBatchingTaskDispatcher(String id,
                                                                             int maxBufferSize,
                                                                             int workloadSize,
                                                                             int workerCount,
                                                                             long maxBatchingDelay,
                                                                             long congestionRetryDelayMs,
                                                                             long networkFailureRetryMs,
                                                                             TaskProcessor<T> taskProcessor) {
        final AcceptorExecutor<ID, T> acceptorExecutor = new AcceptorExecutor<>(
                id, maxBufferSize, workloadSize, maxBatchingDelay, congestionRetryDelayMs, networkFailureRetryMs
        );
        final TaskExecutors<ID, T> taskExecutor = TaskExecutors.batchExecutors(id, workerCount, taskProcessor, acceptorExecutor);
        return new TaskDispatcher<ID, T>() {
            @Override
            public void process(ID id, T task, long expiryTime) {
                acceptorExecutor.process(id, task, expiryTime);
            }

            @Override
            public void shutdown() {
                acceptorExecutor.shutdown();
                taskExecutor.shutdown();
            }
        };
    }
```


AcceptorExecutor<ID, T>#process(ID id, T task, long expiryTime)


    private final BlockingQueue<TaskHolder<ID, T>> acceptorQueue = new LinkedBlockingQueue<>();
    

acceptedTask仅仅volatile，可能会有线程安全问题

```java
    void process(ID id, T task, long expiryTime) {
        acceptorQueue.add(new TaskHolder<ID, T>(id, task, expiryTime));
        acceptedTasks++;
    }
```


TaskHolder<ID, T>


```java
class TaskHolder<ID, T> {

    private final ID id;
    private final T task;
    private final long expiryTime;
    private final long submitTimestamp;

    TaskHolder(ID id, T task, long expiryTime) {
        this.id = id;
        this.expiryTime = expiryTime;
        this.task = task;
        this.submitTimestamp = System.currentTimeMillis();
    }

    public ID getId() {
        return id;
    }

    public T getTask() {
        return task;
    }

    public long getExpiryTime() {
        return expiryTime;
    }

    public long getSubmitTimestamp() {
        return submitTimestamp;
    }
}

```


####2、 处理 acceptorQueue中的taskholder


AcceptorExecutor中的acceptor的初始化

    private final Thread acceptorThread;

    ThreadGroup threadGroup = new ThreadGroup("eurekaTaskExecutors");
    this.acceptorThread = new Thread(threadGroup, new AcceptorRunner(), "TaskAcceptor-" + id);
    this.acceptorThread.setDaemon(true);
    this.acceptorThread.start();

AcceptorRunner#run

* isShutdown() @

* drainInputQueues  @

* 计算scheduleTime 下一次执行时间，scheduletime如果比现在小，说明要执行了

* trafficShaper 流量整形  @

* assignBatchWork @

```java
class AcceptorRunner implements Runnable {
        @Override
        public void run() {
            long scheduleTime = 0;
            while (!isShutdown.get()) {
                try {
                    drainInputQueues();

                    int totalItems = processingOrder.size();

                    long now = System.currentTimeMillis();
                    if (scheduleTime < now) {
                        scheduleTime = now + trafficShaper.transmissionDelay();
                    }
                    if (scheduleTime <= now) {
                        assignBatchWork();
                        assignSingleItemWork();
                    }

                    // If no worker is requesting data or there is a delay injected by the traffic shaper,
                    // sleep for some time to avoid tight loop.
                    if (totalItems == processingOrder.size()) {
                        Thread.sleep(10);
                    }
                } catch (InterruptedException ex) {
                    // Ignore
                } catch (Throwable e) {
                    // Safe-guard, so we never exit this loop in an uncontrolled way.
                    logger.warn("Discovery AcceptorThread error", e);
                }
            }
        }

```




----------

1. isShutdown, 多线程的balking模式

```java
private final AtomicBoolean isShutdown = new AtomicBoolean(false);
```

调用shutdown方法， 将cas值设置为true， 打破循环，进入interruptException

```java
void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            acceptorThread.interrupt();
        }
    }
```

2. drainInputQueues

* 循环条件， 只要有一个成立，久会一直循环， 接收队列不为空， 重试队列不为空， 等待队列为空，只有接收队列为空，重试队列为空，等待队列不为空，才会跳出循环

* drainReprocessQueue @
* drainAcceptorQueue @

* 进入三空状态， 随便做点事情，为了等一段时间， 但没有直接sleep，

```java
private void drainInputQueues() throws InterruptedException {
            do {
                drainReprocessQueue();
                drainAcceptorQueue();

                if (!isShutdown.get()) {
                    // If all queues are empty, block for a while on the acceptor queue
                    if (reprocessQueue.isEmpty() && acceptorQueue.isEmpty() && pendingTasks.isEmpty()) {
                        TaskHolder<ID, T> taskHolder = acceptorQueue.poll(10, TimeUnit.MILLISECONDS);
                        if (taskHolder != null) {
                            appendTaskHolder(taskHolder);
                        }
                    }
                }
            } while (!reprocessQueue.isEmpty() || !acceptorQueue.isEmpty() || pendingTasks.isEmpty());
        }

```


-------------------
2.1. AcceptorRunner#drainReprocessQueue

reprocessQueue是一个双端阻塞队列
pendingtask只存放id，并且为了避免重复， 所以引入pendingtasks

        private final BlockingDeque<TaskHolder<ID, T>> reprocessQueue = new LinkedBlockingDeque<>();
        
        private final Map<ID, TaskHolder<ID, T>> pendingTasks = new HashMap<>();

        private final Map<ID, TaskHolder<ID, T>> pendingTasks = new HashMap<>();


* reprocessQueue不为空且没有满，进入循环 这里满为10000个

* pollLast 是拿走最新进来的，并不是最早进去的，因为重试队列新的比老的值钱

* 假设过期了，或者任务id相同，丢弃

* processingOrder将重试任务放在队头，为了当放入新任务发现满了，把这个队首的给删了

* 退出while循环发现pendingorder满了，重试队列清空

        private void drainReprocessQueue() {
            long now = System.currentTimeMillis();
            //pendingTasks数量大于等于maxBufferSize
            while (!reprocessQueue.isEmpty() && !isFull()) {
                TaskHolder<ID, T> taskHolder = reprocessQueue.pollLast();
                ID id = taskHolder.getId();
                //过期了，不要了这个task
                if (taskHolder.getExpiryTime() <= now) {
                    expiredTasks++;
                //任务id相同覆盖
                } else if (pendingTasks.containsKey(id)) {
                    overriddenTasks++;
                } else {
                    //map存入，这个map作用是1、让order只存id，2、覆盖相同id的任务
                    pendingTasks.put(id, taskHolder);
                    //order放到队首
                    processingOrder.addFirst(id);
                }
            }
            //如果pendingOrder满了，reprocessQueue东西不要了
            if (isFull()) {
                queueOverflows += reprocessQueue.size();
                reprocessQueue.clear();
            }
        }

2.2 AcceptorExecutor#drainAcceptorQueue

```java
private void drainAcceptorQueue() {
            while (!acceptorQueue.isEmpty()) {
                appendTaskHolder(acceptorQueue.poll());
            }
        }

```

AcceptorExecutor#appendTaskHolder

* 假设pendingorder满了，pendingorder poll队首，map remove该元素

* 通过put判断是否以前存在改task， 不存在，进行添加


```java
private void appendTaskHolder(TaskHolder<ID, T> taskHolder) {
            if (isFull()) {
                pendingTasks.remove(processingOrder.poll());
                queueOverflows++;
            }
            TaskHolder<ID, T> previousTask = pendingTasks.put(taskHolder.getId(), taskHolder);
            if (previousTask == null) {
                processingOrder.add(taskHolder.getId());
            } else {
                overriddenTasks++;
            }
        }

```

-------

3、流量整型

TrafficShaper流量整形类,

构造方法， congestionRetryDelayMs=1000 networkFailureRetryMs =100

当调用失败，注册结果，记录上一次失败时间


```java
class TrafficShaper {

    /**
     * Upper bound on delay provided by configuration.
     */
    private static final long MAX_DELAY = 30 * 1000;

    private final long congestionRetryDelayMs;
    private final long networkFailureRetryMs;

    private volatile long lastCongestionError;
    private volatile long lastNetworkFailure;

    TrafficShaper(long congestionRetryDelayMs, long networkFailureRetryMs) {
        this.congestionRetryDelayMs = Math.min(MAX_DELAY, congestionRetryDelayMs);
        this.networkFailureRetryMs = Math.min(MAX_DELAY, networkFailureRetryMs);
    }

    void registerFailure(ProcessingResult processingResult) {
        if (processingResult == ProcessingResult.Congestion) {
            lastCongestionError = System.currentTimeMillis();
        } else if (processingResult == ProcessingResult.TransientError) {
            lastNetworkFailure = System.currentTimeMillis();
        }
    }

```

执行结果enum

    enum ProcessingResult {
        Success, Congestion, TransientError, PermanentError
    }


TrafficShaper#transmissionDelay

计算现在时间和上次出错时间差值，和约定重试时间比较，返回还要等待的时间

```java
    long transmissionDelay() {
        //lastCongestionError上一次发生拥挤错误的时间
        //lastNetworkFailure上一次发生网络错误的时间
        if (lastCongestionError == -1 && lastNetworkFailure == -1) {
            return 0;
        }
        long now = System.currentTimeMillis();
        if (lastCongestionError != -1) {
            //上一次出错到现在的时间间隔
            long congestionDelay = now - lastCongestionError;
            if (congestionDelay >= 0 && congestionDelay < congestionRetryDelayMs) {
                return congestionRetryDelayMs - congestionDelay;
            }
            lastCongestionError = -1;
        }
        //同理拥挤，这里理论上不可能都为-1啊
        else if (lastNetworkFailure != -1) {
            long failureDelay = now - lastNetworkFailure;
            if (failureDelay >= 0 && failureDelay < networkFailureRetryMs) {
                return networkFailureRetryMs - failureDelay;
            }
            lastNetworkFailure = -1;
        }
        return 0;
    }

```

-------------------

5、AcceptorExecutor#assignBatchWork

将processingorder的任务放到执行队列

* hasEnoughTasksForNextBatch @

* batchWorkRequests.tryAcquire(1)   信号量， 限制线程数量 @

* Math.min（250， processingorder的数量）， 一次batch最多250条记录

* 将task整成一个list，塞进batchworkerqueue， 这是一个并发阻塞队列 链表形式//    private final BlockingQueue<List<TaskHolder<ID, T>>> batchWorkQueue = new LinkedBlockingQueue<>();


```java
void assignBatchWork() {
            if (hasEnoughTasksForNextBatch()) {
                if (batchWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    int len = Math.min(maxBatchingSize, processingOrder.size());
                    List<TaskHolder<ID, T>> holders = new ArrayList<>(len);
                    while (holders.size() < len && !processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            holders.add(holder);
                        } else {
                            expiredTasks++;
                        }
                    }
                    if (holders.isEmpty()) {
                        batchWorkRequests.release();
                    } else {
                        batchSizeMetric.record(holders.size(), TimeUnit.MILLISECONDS);
                        batchWorkQueue.add(holders);
                    }
                }
            }
        }

```

------------

5.1 hasEnoughTasksForNextBatch

AcceptorExecutor#hasEnoughTasksForNextBatch

* 假设processingOrder为空， 返回false， 假设tasks的size大于maxbuffersize， 返回true

* processingOrder拿出队首元素，从holder里面找出来， 查询他等了多久了，如果超过500ms， 返回true

* 攒一波（量） 或者 时间达到

```java
private boolean hasEnoughTasksForNextBatch() {
            if (processingOrder.isEmpty()) {
                return false;
            }
            if (pendingTasks.size() >= maxBufferSize) {
                return true;
            }

            TaskHolder<ID, T> nextHolder = pendingTasks.get(processingOrder.peek());
            long delay = System.currentTimeMillis() - nextHolder.getSubmitTimestamp();
            return delay >= maxBatchingDelay;
        }

```



----------------

5.2 batchWorkRequests

信号量只有0.为啥后面可以accquire（1) 呢？  

    private final Semaphore batchWorkRequests = new Semaphore(0);
    
    
TaskExecutors#batchExecutors

```java
static <ID, T> TaskExecutors<ID, T> batchExecutors(final String name,
                                                       int workerCount,
                                                       final TaskProcessor<T> processor,
                                                       final AcceptorExecutor<ID, T> acceptorExecutor) {
        final AtomicBoolean isShutdown = new AtomicBoolean();
        final TaskExecutorMetrics metrics = new TaskExecutorMetrics(name);
        return new TaskExecutors<>(new WorkerRunnableFactory<ID, T>() {
            @Override
            public WorkerRunnable<ID, T> create(int idx) {
                return new BatchWorkerRunnable<>("TaskBatchingWorker-" +name + '-' + idx, isShutdown, metrics, processor, acceptorExecutor);
            }

        }, workerCount, isShutdown);
    }
```

WorkerRunnableFactory

```java
    @FunctionalInterface
    interface WorkerRunnableFactory<ID, T> {
        WorkerRunnable<ID, T> create(int idx);
    }
```

WorkerRunnable

```java
abstract static class WorkerRunnable<ID, T> implements Runnable {
        final String workerName;
        final AtomicBoolean isShutdown;
        final TaskExecutorMetrics metrics;
        final TaskProcessor<T> processor;
        final AcceptorExecutor<ID, T> acceptorExecutor;

        WorkerRunnable(String workerName,
                       AtomicBoolean isShutdown,
                       TaskExecutorMetrics metrics,
                       TaskProcessor<T> processor,
                       AcceptorExecutor<ID, T> acceptorExecutor) {
            this.workerName = workerName;
            this.isShutdown = isShutdown;
            this.metrics = metrics;
            this.processor = processor;
            this.acceptorExecutor = acceptorExecutor;
        }

        String getWorkerName() {
            return workerName;
        }
    }

```

TaskDispatchers#createBatchingTaskDispatcher

生成了taskExecutor 由于传入了workerCount=20

```java
public static <ID, T> TaskDispatcher<ID, T> createBatchingTaskDispatcher(String id,
                                                                             int maxBufferSize,
                                                                             int workloadSize,
                                                                             int workerCount,
                                                                             long maxBatchingDelay,
                                                                             long congestionRetryDelayMs,
                                                                             long networkFailureRetryMs,
                                                                             TaskProcessor<T> taskProcessor) {
        final AcceptorExecutor<ID, T> acceptorExecutor = new AcceptorExecutor<>(
                id, maxBufferSize, workloadSize, maxBatchingDelay, congestionRetryDelayMs, networkFailureRetryMs
        );
        final TaskExecutors<ID, T> taskExecutor = TaskExecutors.batchExecutors(id, workerCount, taskProcessor, acceptorExecutor);
        return new TaskDispatcher<ID, T>() {
            @Override
            public void process(ID id, T task, long expiryTime) {
                acceptorExecutor.process(id, task, expiryTime);
            }

            @Override
            public void shutdown() {
                acceptorExecutor.shutdown();
                taskExecutor.shutdown();
            }
        };
    }

```

TaskExecutors#构造方法

执行thread线程启动，调用run， 20次，所以里面release了20下

```java
TaskExecutors(WorkerRunnableFactory<ID, T> workerRunnableFactory, int workerCount, AtomicBoolean isShutdown) {
        this.isShutdown = isShutdown;
        this.workerThreads = new ArrayList<>();

        ThreadGroup threadGroup = new ThreadGroup("eurekaTaskExecutors");
        for (int i = 0; i < workerCount; i++) {
            WorkerRunnable<ID, T> runnable = workerRunnableFactory.create(i);
            Thread workerThread = new Thread(threadGroup, runnable, runnable.getWorkerName());
            workerThreads.add(workerThread);
            workerThread.setDaemon(true);
            workerThread.start();
        }
    }

```


#### 真正的worker执行线程

BatchWorkerRunnable#run

* 1、getWork()  @

* 2.getTasksOf  @

* 3、 processor.process 最重要的部分  @

* 4、  当返回Congestion与TransientError  @

```java
@Override
        public void run() {
            try {
                while (!isShutdown.get()) {
                    List<TaskHolder<ID, T>> holders = getWork();
                    metrics.registerExpiryTimes(holders);

                    List<T> tasks = getTasksOf(holders);
                    ProcessingResult result = processor.process(tasks);
                    switch (result) {
                        case Success:
                            break;
                        case Congestion:
                        case TransientError:
                            acceptorExecutor.reprocess(holders, result);
                            break;
                        case PermanentError:
                            logger.warn("Discarding {} tasks of {} due to permanent error", holders.size(), workerName);
                    }
                    metrics.registerTaskResult(result, tasks.size());
                }
            } catch (InterruptedException e) {
                // Ignore
            } catch (Throwable e) {
                // Safe-guard, so we never exit this loop in an uncontrolled way.
                logger.warn("Discovery WorkerThread error", e);
            }
        }

```

----


1、BatchWorkerRunnable#getWork

获取执行队列的每个元素，元素是list

```java
private List<TaskHolder<ID, T>> getWork() throws InterruptedException {
            BlockingQueue<List<TaskHolder<ID, T>>> workQueue = acceptorExecutor.requestWorkItems();
            List<TaskHolder<ID, T>> result;
            do {
                result = workQueue.poll(1, TimeUnit.SECONDS);
            } while (!isShutdown.get() && result == null);
            return (result == null) ? new ArrayList<>() : result;
        }

```

AcceptorExecutor#requestWorkItems

获取真正的执行队列

```java
    BlockingQueue<List<TaskHolder<ID, T>>> requestWorkItems() {
        batchWorkRequests.release();
        return batchWorkQueue;
    }

```

--------

2、getTasksOf

BatchWorkerRunnable#getTasksOf

将list<TaskHolder> 转为 List<Task>



```java
private List<T> getTasksOf(List<TaskHolder<ID, T>> holders) {
            List<T> tasks = new ArrayList<>(holders.size());
            for (TaskHolder<ID, T> holder : holders) {
                tasks.add(holder.getTask());
            }
            return tasks;
        }
```
-------------------

3、ReplicationTaskProcessor#process

* 3.1 ReplicationList list = createReplicationListOf(tasks) 将list<Task> 转换为ReplicationList

* 3.2 返回为503，返回congestion，拥堵错误， 1000ms等待   @

* 3.3 maybeReadTimeOut  TransientError 短暂错误 100  不是很理解 @

* 3.4 handleBatchResponse 最最重要


```java
@Override
    public ProcessingResult process(List<ReplicationTask> tasks) {
            ReplicationList list = createReplicationListOf(tasks);
            try {
            EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
            int statusCode = response.getStatusCode();
            if (!isSuccess(statusCode)) {
                if (statusCode == 503) {
                    logger.warn("Server busy (503) HTTP status code received from the peer {}; rescheduling tasks after delay", peerId);
                    return ProcessingResult.Congestion;
                } else {
                    // Unexpected error returned from the server. This should ideally never happen.
                    logger.error("Batch update failure with HTTP status code {}; discarding {} replication tasks", statusCode, tasks.size());
                    return ProcessingResult.PermanentError;
                }
            } else {
                handleBatchResponse(tasks, response.getEntity().getResponseList());
            }
        } catch (Throwable e) {
            if (maybeReadTimeOut(e)) {
                logger.error("It seems to be a socket read timeout exception, it will retry later. if it continues to happen and some eureka node occupied all the cpu time, you should set property 'eureka.server.peer-node-read-timeout-ms' to a bigger value", e);
                //read timeout exception is more Congestion then TransientError, return Congestion for longer delay
                return ProcessingResult.Congestion;
            } else if (isNetworkConnectException(e)) {
                logNetworkErrorSample(null, e);
                return ProcessingResult.TransientError;
            } else {
                logger.error("Not re-trying this exception because it does not seem to be a network exception", e);
                return ProcessingResult.PermanentError;
            }
        }
        return ProcessingResult.Success;
    }

```

------------------------------

补充
task.execute()

ReplicationTask#execute 是一个抽象方法， 都是匿名内部类实现他的， 这种编程方法非常好， 不需要写具体的类，而是在需要用到task的时候，用到的时候，生成匿名内部类，重写这个方法，看下面register示例

```java
public abstract EurekaHttpResponse<?> execute() throws Throwable;
```


PeerEurekaNode#register

需要用到这个类的时候再初始化，重写execute方法

```java
public void register(final InstanceInfo info) throws Exception {
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        batchingDispatcher.process(
                taskId("register", info),
                //构造方法protected 同一个包可以new
                new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.register(info);
                    }
                },
                expiryTime
        );
    }
```


----------------

3.1、ReplicationTaskProcessor#maybeReadTimeOut

do while循环获取exception e 与READ_TIME_OUT_PATTERN 进行match

    private static final Pattern READ_TIME_OUT_PATTERN = Pattern.compile(".*read.*time.*out.*");

```java
private static boolean maybeReadTimeOut(Throwable e) {
        do {
            if (IOException.class.isInstance(e)) {
                String message = e.getMessage().toLowerCase();
                Matcher matcher = READ_TIME_OUT_PATTERN.matcher(message);
                if(matcher.find()) {
                    return true;
                }
            }
            e = e.getCause();
        } while (e != null);
        return false;
    }
```

----------------------------
3.2、ReplicationTaskProcessor#isNetworkConnectException

```java
private static boolean isNetworkConnectException(Throwable e) {
        do {
            if (IOException.class.isInstance(e)) {
                return true;
            }
            e = e.getCause();
        } while (e != null);
        return false;
    }
```

--------------


3.4
ReplicationTaskProcessor#handleBatchResponse

* 3.4.1 第一个if 看看他的注解，蛮有意思的

* 3.4.2 andleBatchResponse  @

```java
    private void handleBatchResponse(List<ReplicationTask> tasks, List<ReplicationInstanceResponse> responseList) {
        if (tasks.size() != responseList.size()) {
            // This should ideally never happen unless there is a bug in the software.
            logger.error("Batch response size different from submitted task list ({} != {}); skipping response analysis", responseList.size(), tasks.size());
            return;
        }
        for (int i = 0; i < tasks.size(); i++) {
            handleBatchResponse(tasks.get(i), responseList.get(i));
        }
    }
```


-------------

3.4.2、

查看每个返回，假设是错误的，调用task的handleFailure，只有cacel 和 heartbeat有实现，用心跳举例


ReplicationTaskProcessor#handleBatchResponse

```java
private void handleBatchResponse(ReplicationTask task, ReplicationInstanceResponse response) {
        int statusCode = response.getStatusCode();
        if (isSuccess(statusCode)) {
            task.handleSuccess();
            return;
        }

        try {
            task.handleFailure(response.getStatusCode(), response.getResponseEntity());
        } catch (Throwable e) {
            logger.error("Replication task {} error handler failure", task.getTaskName(), e);
        }
    }

```

PeerEurekaNode#heartbeat

```java
public void heartbeat(final String appName, final String id,
                          final InstanceInfo info, final InstanceStatus overriddenStatus,
                          boolean primeConnection) throws Throwable {
        if (primeConnection) {
            // We do not care about the result for priming request.
            replicationClient.sendHeartBeat(appName, id, info, overriddenStatus);
            return;
        }
        ReplicationTask replicationTask = new InstanceReplicationTask(targetHost, Action.Heartbeat, info, overriddenStatus, false) {
            @Override
            public EurekaHttpResponse<InstanceInfo> execute() throws Throwable {
                return replicationClient.sendHeartBeat(appName, id, info, overriddenStatus);
            }

            @Override
            public void handleFailure(int statusCode, Object responseEntity) throws Throwable {
                super.handleFailure(statusCode, responseEntity);
                if (statusCode == 404) {
                    logger.warn("{}: missing entry.", getTaskName());
                    if (info != null) {
                        logger.warn("{}: cannot find instance id {} and hence replicating the instance with status {}",
                                getTaskName(), info.getId(), info.getStatus());
                        register(info);
                    }
                } else if (config.shouldSyncWhenTimestampDiffers()) {
                    InstanceInfo peerInstanceInfo = (InstanceInfo) responseEntity;
                    if (peerInstanceInfo != null) {
                        syncInstancesIfTimestampDiffers(appName, id, info, peerInstanceInfo);
                    }
                }
            }
        };
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        batchingDispatcher.process(taskId("heartbeat", info), replicationTask, expiryTime);
    }
```

----------------


4、 当发生短暂错误或者拥挤错误

AcceptorExecutor#reprocess

将List<TaskHolder> 一下子加入到reprocessQueue中，并且注册错误信息，用来流量整形

    private final BlockingDeque<TaskHolder<ID, T>> reprocessQueue = new LinkedBlockingDeque<>();

```java
    void reprocess(List<TaskHolder<ID, T>> holders, ProcessingResult processingResult) {
        reprocessQueue.addAll(holders);
        replayedTasks += holders.size();
        trafficShaper.registerFailure(processingResult);
    }
```


-----------------


