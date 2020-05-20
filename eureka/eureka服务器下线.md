# eureka服务器下线

####1.客户端调用下方法声明

EurekaClientAutoConfiguration.RefreshableEurekaClientConfiguration#eurekaClient

```java
@Bean(destroyMethod = "shutdown")
        @ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
        @org.springframework.cloud.context.config.annotation.RefreshScope
        @Lazy
        public EurekaClient eurekaClient(ApplicationInfoManager manager,
                EurekaClientConfig config, EurekaInstanceConfig instance,
                @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
            // If we use the proxy of the ApplicationInfoManager we could run into a
            // problem
            // when shutdown is called on the CloudEurekaClient where the
            // ApplicationInfoManager bean is
            // requested but wont be allowed because we are shutting down. To avoid this
            // we use the
            // object directly.
            ApplicationInfoManager appManager;
            if (AopUtils.isAopProxy(manager)) {
                appManager = ProxyUtils.getTargetObject(manager);
            }
            else {
                appManager = manager;
            }
            CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager,
                    config, this.optionalArgs, this.context);
            cloudEurekaClient.registerHealthCheck(healthCheckHandler);
            return cloudEurekaClient;
        }
```


### 服务端接收下线请求

PeerAwareInstanceRegistryImpl#cancel

这里将expectedNumberOfClientsSendingRenews-- 并且调用updateRenewsPerMinThreshold（）更改threshold

```java
@Override
    public boolean cancel(final String appName, final String id,
                          final boolean isReplication) {
        if (super.cancel(appName, id, isReplication)) {
            replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to cancel it, reduce the number of clients to send renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews - 1;
                    updateRenewsPerMinThreshold();
                }
            }
            return true;
        }
        return false;
    }
```


AbstractInstanceRegistry#internalCancel

* 最核心的步骤，gmap.remove
*overriddenInstanceStatusMap 移除该实例
* leaseToCancel cancel 看下面
* 将这个删除的instanceinfo加入recentlyChangedQueue
* invalidateCache 失效缓存

```java

protected boolean internalCancel(String appName, String id, boolean isReplication) {
        try {
            read.lock();
            CANCEL.increment(isReplication);
            Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
            Lease<InstanceInfo> leaseToCancel = null;
            if (gMap != null) {
                leaseToCancel = gMap.remove(id);
            }
            synchronized (recentCanceledQueue) {
                recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
            }

            InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
            if (instanceStatus != null) {
                logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
            }
            if (leaseToCancel == null) {
                CANCEL_NOT_FOUND.increment(isReplication);
                logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
                return false;
            } else {
                leaseToCancel.cancel();
                InstanceInfo instanceInfo = leaseToCancel.getHolder();
                String vip = null;
                String svip = null;
                if (instanceInfo != null) {
                    instanceInfo.setActionType(ActionType.DELETED);
                    recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                    instanceInfo.setLastUpdatedTimestamp();
                    vip = instanceInfo.getVIPAddress();
                    svip = instanceInfo.getSecureVipAddress();
                }
                invalidateCache(appName, vip, svip);
                logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
                return true;
            }
        } finally {
            read.unlock();
        }
    }
```


Lease#cancel
```java
    public void cancel() {
        if (evictionTimestamp <= 0) {
            evictionTimestamp = System.currentTimeMillis();
        }
    }
```