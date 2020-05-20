# eureka客户端全量获取



####1.客户端轮询拉取


DiscoveryClient#initScheduledTasks()

* expBackOffBound = 10 说明每次超时，会将超时时间乘以2，最多给他乘到10为止

* registryFetchIntervalSeconds 为30 ，说明30s 轮询一次


    private final ScheduledExecutorService scheduler;

    scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

schedule.schedule 其实仅仅会执行一次timedsupervisor类的run方法，所以，所有猫腻都在TimedSupervisorTask里面
这个类传入scheduler， executor， task， run方法自己执行executor.submit(task)， 等待时间可以指数级增长，
单凭sechedule是做不到的，在finally再调用scheduler.schedule(this)

```java
    private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
```

TimedSupervisorTask#run

再次一睹timedsupervisortask芳容

```java
@Override
    public void run() {
        Future<?> future = null;
        try {
            future = executor.submit(task);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            future.get(timeoutMillis, TimeUnit.MILLISECONDS);  // block until done or timeout
            delay.set(timeoutMillis);
            threadPoolLevelGauge.set((long) executor.getActiveCount());
            successCounter.increment();
        } catch (TimeoutException e) {
            logger.warn("task supervisor timed out", e);
            timeoutCounter.increment();

            long currentDelay = delay.get();
            long newDelay = Math.min(maxDelay, currentDelay * 2);
            delay.compareAndSet(currentDelay, newDelay);

        } catch (RejectedExecutionException e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, reject the task", e);
            } else {
                logger.warn("task supervisor rejected the task", e);
            }

            rejectedCounter.increment();
        } catch (Throwable e) {
            if (executor.isShutdown() || scheduler.isShutdown()) {
                logger.warn("task supervisor shutting down, can't accept the task");
            } else {
                logger.warn("task supervisor threw an exception", e);
            }

            throwableCounter.increment();
        } finally {
            if (future != null) {
                future.cancel(true);
            }

            if (!scheduler.isShutdown()) {
                scheduler.schedule(this, delay.get(), TimeUnit.MILLISECONDS);
            }
        }
    }
```


CacheRefreshThread

任务task类

```java
    class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }
```

```java

DiscoveryClient#refreshRegistry

不要看代码一大坨，boolean success = fetchRegistry(remoteRegionsModified)。 只有这一行是重点。。。



@VisibleForTesting
    void refreshRegistry() {
        try {
            boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

            boolean remoteRegionsModified = false;
            // This makes sure that a dynamic change to remote regions to fetch is honored.
            String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
            if (null != latestRemoteRegions) {
                String currentRemoteRegions = remoteRegionsToFetch.get();
                if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                    // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                    synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                        if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                            String[] remoteRegions = latestRemoteRegions.split(",");
                            remoteRegionsRef.set(remoteRegions);
                            instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                            remoteRegionsModified = true;
                        } else {
                            logger.info("Remote regions to fetch modified concurrently," +
                                    " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                        }
                    }
                } else {
                    // Just refresh mapping to reflect any DNS/Property change
                    instanceRegionChecker.getAzToRegionMapper().refreshMapping();
                }
            }

            boolean success = fetchRegistry(remoteRegionsModified);
            if (success) {
                registrySize = localRegionApps.get().size();
                lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
            }

            if (logger.isDebugEnabled()) {
                StringBuilder allAppsHashCodes = new StringBuilder();
                allAppsHashCodes.append("Local region apps hashcode: ");
                allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
                allAppsHashCodes.append(", is fetching remote regions? ");
                allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
                for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                    allAppsHashCodes.append(", Remote region: ");
                    allAppsHashCodes.append(entry.getKey());
                    allAppsHashCodes.append(" , apps hashcode: ");
                    allAppsHashCodes.append(entry.getValue().getAppsHashCode());
                }
                logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                        allAppsHashCodes);
            }
        } catch (Throwable e) {
            logger.error("Cannot fetch registry from server", e);
        }
    }
```


DiscoveryClient#fetchRegistry

* 如果启动第一次进来， getApplications()调用获取localRegionApps.get()，拿到的Applications调用.getRegisteredApplications().size()为0， 所以会需要执行getAndStoreFullRegistry()

* 反之， 执行getAndUpdateDelta(applications)

* 最终， 更新下hashcoe码

```java
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                getAndStoreFullRegistry();
            } else {
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
```

DiscoveryClient#getAndStoreFullRegistry()

* 这里有点意思，没有用互斥，使用了CAS的comparedAndSet，值得学习下，用这种方法，类似db的无锁操作， 操作完对比下version，如果
version改变，就放弃。如果version一致，进去以后set其实可以不必考虑原子性问题，确实有可能两个线程同时执行到localRegionApps.set(this.filterAndShuffle(apps))，

* filterandShuffle filter UP状态并且shuffle，下面详解

```java
    private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {
            logger.error("The application is null for some reason. Not storing this information");
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
```


Application#_shuffleAndStoreInstances  

* synchronized (instances) 这里为啥要用锁呢， instances虽然共享变量，但是只是读操作啊。。。因为new ArrayList()构造方法存在中间状态，可能期间instances改变。

* iterator.remove()虽然线程不安全，但是这里instanceInfoList是局部变量。。

* Collections.shuffle(instanceInfoList, shuffleRandom);  洗牌， shuffleRandom 是new Random()， 里面可能最好传入System.currentTimeMills()； 或者使用 Collections.swap(Collection list, index1, index2)

* instanceInfoList 传入shuffledInstance.

```java
private void _shuffleAndStoreInstances(boolean filterUpInstances, boolean indexByRemoteRegions,
                                           @Nullable Map<String, Applications> remoteRegionsRegistry,
                                           @Nullable EurekaClientConfig clientConfig,
                                           @Nullable InstanceRegionChecker instanceRegionChecker) {
        List<InstanceInfo> instanceInfoList;
        synchronized (instances) {
            instanceInfoList = new ArrayList<InstanceInfo>(instances);
        }
        boolean remoteIndexingActive = indexByRemoteRegions && null != instanceRegionChecker && null != clientConfig
                && null != remoteRegionsRegistry;
        if (remoteIndexingActive || filterUpInstances) {
            Iterator<InstanceInfo> it = instanceInfoList.iterator();
            while (it.hasNext()) {
                InstanceInfo instanceInfo = it.next();
                if (filterUpInstances && InstanceStatus.UP != instanceInfo.getStatus()) {
                    it.remove();
                } else if (remoteIndexingActive) {
                    String instanceRegion = instanceRegionChecker.getInstanceRegion(instanceInfo);
                    if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                        Applications appsForRemoteRegion = remoteRegionsRegistry.get(instanceRegion);
                        if (null == appsForRemoteRegion) {
                            appsForRemoteRegion = new Applications();
                            remoteRegionsRegistry.put(instanceRegion, appsForRemoteRegion);
                        }

                        Application remoteApp =
                                appsForRemoteRegion.getRegisteredApplications(instanceInfo.getAppName());
                        if (null == remoteApp) {
                            remoteApp = new Application(instanceInfo.getAppName());
                            appsForRemoteRegion.addApplication(remoteApp);
                        }

                        remoteApp.addInstance(instanceInfo);
                        this.removeInstance(instanceInfo, false);
                        it.remove();
                    }
                }
            }

        }
        Collections.shuffle(instanceInfoList, shuffleRandom);
        this.shuffledInstances.set(instanceInfoList);
    }

```


Applications#getReconcileHashCode

获取一致性hash码

TreeMaps是有序的, 并不是存入顺序有序，而是根据key自己排序

    public String getReconcileHashCode() {
        TreeMap<String, AtomicInteger> instanceCountMap = new TreeMap<String, AtomicInteger>();
        populateInstanceCountMap(instanceCountMap);
        return getReconcileHashCode(instanceCountMap);
    }


Applications#populateInstanceCountMap

* 构建instanceCountMap

* computeIfAbsent， 假设key 不存在，构建value， 返回该value， 可以操作，如果key存在，返回value，操作

* putifabsent， 如果key存在，返回对应value，不放入，如果key不存在，放入，返回null

```java

public void populateInstanceCountMap(Map<String, AtomicInteger> instanceCountMap) {
        for (Application app : this.getRegisteredApplications()) {
            for (InstanceInfo info : app.getInstancesAsIsFromEureka()) {
                AtomicInteger instanceCount = instanceCountMap.computeIfAbsent(info.getStatus().name(),
                        k -> new AtomicInteger(0));
                instanceCount.incrementAndGet();
            }
        }
    }
```

Applications#getReconcileHashCode

通过stringbuilder生成concilehashcode   UP_N_， 因为第一次全量拉取，这里应该状态只有up

```java
    public static String getReconcileHashCode(Map<String, AtomicInteger> instanceCountMap) {
        StringBuilder reconcileHashCode = new StringBuilder(75);
        for (Map.Entry<String, AtomicInteger> mapEntry : instanceCountMap.entrySet()) {
            reconcileHashCode.append(mapEntry.getKey()).append(STATUS_DELIMITER).append(mapEntry.getValue().get())
                    .append(STATUS_DELIMITER);
        }
        return reconcileHashCode.toString();
    }

```

DiscoverClient#onCacheRefreshed()

事件监听器模式

    protected void onCacheRefreshed() {
        fireEvent(new CacheRefreshedEvent());
    }


DiscoverClient#fireEvent()

监听器listeners队列，每个调用onEvent()，这里listeners是copyonwritearrayset通过构造函数传参AbstractDiscoveryClientOptionalArgs传入的， 但是默认这个abstract类中setEventListeners不会被调用到
后续解决吧

```java

@Inject(optional = true)
    public void setEventListeners(Set<EurekaEventListener> listeners) {
        if (eventListeners == null) {
            eventListeners = new HashSet<>();
        }
        eventListeners.addAll(listeners);
    }
```

监听队列轮询调用onevent

```java
    protected void fireEvent(final EurekaEvent event) {
        for (EurekaEventListener listener : eventListeners) {
            try {
                listener.onEvent(event);
            } catch (Exception e) {
                logger.info("Event {} throw an exception for listener {}", event, listener, e.getMessage());
            }
        }
    }
```

DiscoverClient#updateInstanceRemoteStatus

通过拿到的最新apps，找到当前instance对应的status，假设与本地不符，发出一个event，然后本地覆盖

```java
private synchronized void updateInstanceRemoteStatus() {
        // Determine this instance's status for this app and set to UNKNOWN if not found
        InstanceInfo.InstanceStatus currentRemoteInstanceStatus = null;
        if (instanceInfo.getAppName() != null) {
            Application app = getApplication(instanceInfo.getAppName());
            if (app != null) {
                InstanceInfo remoteInstanceInfo = app.getByInstanceId(instanceInfo.getId());
                if (remoteInstanceInfo != null) {
                    currentRemoteInstanceStatus = remoteInstanceInfo.getStatus();
                }
            }
        }
        if (currentRemoteInstanceStatus == null) {
            currentRemoteInstanceStatus = InstanceInfo.InstanceStatus.UNKNOWN;
        }

        // Notify if status changed
        if (lastRemoteInstanceStatus != currentRemoteInstanceStatus) {
            onRemoteStatusChanged(lastRemoteInstanceStatus, currentRemoteInstanceStatus);
            lastRemoteInstanceStatus = currentRemoteInstanceStatus;
        }
    }
```


####1.客户端启动拉取

DiscoverClient 的构造方法

启动执行fetchRegistry(false)

```java
if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }
```

DiscoverClient#fetchRegistryFromBackup()

这个方法其实没啥用，就是一个降级机制，配合之前的onCacheRefreshed方法，onCacheRefreshed本地写入文件，当远程拉取失败，
用这种backup机制获取本地文件

```java
private void fetchRegistryFromBackup() {
        try {
            @SuppressWarnings("deprecation")
            BackupRegistry backupRegistryInstance = newBackupRegistryInstance();
            if (null == backupRegistryInstance) { // backward compatibility with the old protected method, in case it is being used.
                backupRegistryInstance = backupRegistryProvider.get();
            }

            if (null != backupRegistryInstance) {
                Applications apps = null;
                if (isFetchingRemoteRegionRegistries()) {
                    String remoteRegionsStr = remoteRegionsToFetch.get();
                    if (null != remoteRegionsStr) {
                        apps = backupRegistryInstance.fetchRegistry(remoteRegionsStr.split(","));
                    }
                } else {
                    apps = backupRegistryInstance.fetchRegistry();
                }
                if (apps != null) {
                    final Applications applications = this.filterAndShuffle(apps);
                    applications.setAppsHashCode(applications.getReconcileHashCode());
                    localRegionApps.set(applications);
                    logTotalInstances();
                    logger.info("Fetched registry successfully from the backup");
                }
            } else {
                logger.warn("No backup registry instance defined & unable to find any discovery servers.");
            }
        } catch (Throwable e) {
            logger.warn("Cannot fetch applications from apps although backup registry was specified", e);
        }
    }
```