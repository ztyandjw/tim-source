# eureka过期机制



####1.启动过期任务


AbstractInstanceRegistry#postInit

    private Timer evictionTimer = new Timer("Eureka-EvictionTimer", true);
    private final AtomicReference<EvictionTask> evictionTaskRef = new AtomicReference<EvictionTask>();




```java
protected void postInit() {
        //更替前一分钟和当前分钟的收到心跳数
        renewsLastMin.start();
        if (evictionTaskRef.get() != null) {
            evictionTaskRef.get().cancel();
        }
        evictionTaskRef.set(new EvictionTask());
        //开启过期任务
        evictionTimer.schedule(evictionTaskRef.get(),
                serverConfig.getEvictionIntervalTimerInMs(),
                serverConfig.getEvictionIntervalTimerInMs());
    }
```


EvictionTask#run

```java
private final AtomicLong lastExecutionNanosRef = new AtomicLong(0l);

        @Override
        public void run() {
            try {
                long compensationTimeMs = getCompensationTimeMs();
                logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
                evict(compensationTimeMs);
            } catch (Throwable e) {
                logger.error("Could not run the evict task", e);
            }
        }
    }
```

EvictionTask#getCompensationTimeMs

获取补偿时间,这里有点意思，时间用当前时间-上一次时间，再转换为millTime，结果与60000 相减，这个是补偿时间，减少cpu的一些误差

```java
long getCompensationTimeMs() {
            long currNanos = getCurrentTimeNano();
            long lastNanos = lastExecutionNanosRef.getAndSet(currNanos);
            if (lastNanos == 0l) {
                return 0l;
            }

            long elapsedMs = TimeUnit.NANOSECONDS.toMillis(currNanos - lastNanos);
            long compensationTime = elapsedMs - serverConfig.getEvictionIntervalTimerInMs();
            return compensationTime <= 0l ? 0l : compensationTime;
        }

        long getCurrentTimeNano() {  // for testing
            return System.nanoTime();
        }

```

AbstractInstanceRegistry#evict（long additionalLeaseMs） part1


* ！isLeaseExpirationEnabled 如果进入代码块，说明需要进行自我保护，不会执行后面evict逻辑

* lease.isExpired(additionalLeaseMs) 是否过期，下面详解

* 获取expiredLeases 过期实例list


```java
public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");

        if (!isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }

        // We collect first all expired items, to evict them in random order. For large eviction sets,
        // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
        // the impact should be evenly distributed across all applications.
        //过期租约expiredLeases
        List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
        for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
            Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
            if (leaseMap != null) {
                for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                    Lease<InstanceInfo> lease = leaseEntry.getValue();
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }
    }

```

Lease#isExpired(long additionalLeaseMs)

这里判断过期逻辑有点意思 先看下面renew的操作, 多加了个duration，后续竟然没有bugfix，匪夷所思

    public void renew() {
        lastUpdateTimestamp = System.currentTimeMillis() + duration;
    }

是否过期的逻辑evictionTimestamp > 0, 说明过期了，或者当前时间 大于lease最后操作时间 + duration + 补偿时间，这里补偿时间的意思是因为两次timer 调度时间间隔虽然是60s， 但是，其实两次时间存在时间差。 这里lastupdatetime + duration是要争着比当前时间大的，但是当前时间其实是可能多了计算出的补偿时间的大小，所以last+duration侧需要加上，否则吃亏了。哈哈

```java
    public boolean isExpired(long additionalLeaseMs) {
        return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
    }
```


AbstractInstanceRegistry#evict（long additionalLeaseMs） part2

* 计算当前注册实例数，下面详解

* 根据权重计算要保留的实例数，不能过期后的存在实例数比他小，因为可能会影响自我保护逻辑

* 过期的个数为Math.min(dxpiredLeases.size(),  evictionLimit)取小的

```java
int registrySize = (int) getLocalRegistrySize();
//阈值
int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
int evictionLimit = registrySize - registrySizeThreshold;

int toEvict = Math.min(expiredLeases.size(), evictionLimit);
```

AbstractInstanceRegistry#evict（long additionalLeaseMs） part3

* 使用公平洗牌法， 这里Random 传入System.currentTimeMillis()

* 将实例下线，参考下线逻辑

```java
if (toEvict > 0) {
            logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

            Random random = new Random(System.currentTimeMillis());
            for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
                int next = i + random.nextInt(expiredLeases.size() - i);
                Collections.swap(expiredLeases, i, next);
                Lease<InstanceInfo> lease = expiredLeases.get(i);

                String appName = lease.getHolder().getAppName();
                String id = lease.getHolder().getId();
                EXPIRED.increment();
                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                internalCancel(appName, id, false);
            }
        }

```