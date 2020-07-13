eureka缓存

### 缓存接口 ResponseCache

public interface  ResponseCache {

    void invalidate(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress);

    AtomicLong getVersionDelta();

    AtomicLong getVersionDeltaWithRegions();

    String get(Key key);

    byte[] getGZIP(Key key);
}

### 缓存键  Key

public class Key {

    public enum KeyType {
        JSON, XML
    }

    /**
     * An enum to define the entity that is stored in this cache for this key.
     */
    public enum EntityType {
        Application, VIP, SVIP
    }

    private final String entityName;
    private final String[] regions;
    private final KeyType requestType;
    private final Version requestVersion;
    private final String hashKey;
    private final EntityType entityType;
    private final EurekaAccept eurekaAccept;

    public Key(EntityType entityType, String entityName, KeyType type, Version v, EurekaAccept eurekaAccept) {
        this(entityType, entityName, type, v, eurekaAccept, null);
    }

    public Key(EntityType entityType, String entityName, KeyType type, Version v, EurekaAccept eurekaAccept, @Nullable String[] regions) {
        this.regions = regions;
        this.entityType = entityType;
        this.entityName = entityName;
        this.requestType = type;
        this.requestVersion = v;
        this.eurekaAccept = eurekaAccept;
        hashKey = this.entityType + this.entityName + (null != this.regions ? Arrays.toString(this.regions) : "")
                + requestType.name() + requestVersion.name() + this.eurekaAccept.name();
    }

    public String getName() {
        return entityName;
    }

    public String getHashKey() {
        return hashKey;
    }

    public KeyType getType() {
        return requestType;
    }

    public Version getVersion() {
        return requestVersion;
    }

    public EurekaAccept getEurekaAccept() {
        return eurekaAccept;
    }

    public EntityType getEntityType() {
        return entityType;
    }

    public boolean hasRegions() {
        return null != regions && regions.length != 0;
    }

    public String[] getRegions() {
        return regions;
    }

    public Key cloneWithoutRegions() {
        return new Key(entityType, entityName, requestType, requestVersion, eurekaAccept);
    }

    @Override
    public int hashCode() {
        String hashKey = getHashKey();
        return hashKey.hashCode();
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Key) {
            return getHashKey().equals(((Key) other).getHashKey());
        } else {
            return false;
        }
    }

    public String toStringCompact() {
        StringBuilder sb = new StringBuilder();
        sb.append("{name=").append(entityName).append(", type=").append(entityType).append(", format=").append(requestType);
        if(regions != null) {
            sb.append(", regions=").append(Arrays.toString(regions));
        }
        sb.append('}');
        return sb.toString();
    }
}


### 缓存值 Value

public class Value {
    private final String payload;
    private byte[] gzipped;

    public Value(String payload) {
        this.payload = payload;
        if (!EMPTY_PAYLOAD.equals(payload)) {
            Stopwatch tracer = compressPayloadTimer.start();
            try {
                ByteArrayOutputStream bos = new ByteArrayOutputStream();
                GZIPOutputStream out = new GZIPOutputStream(bos);
                byte[] rawBytes = payload.getBytes();
                out.write(rawBytes);
                // Finish creation of gzip file
                out.finish();
                out.close();
                bos.close();
                gzipped = bos.toByteArray();
            } catch (IOException e) {
                gzipped = null;
            } finally {
                if (tracer != null) {
                    tracer.stop();
                }
            }
        } else {
            gzipped = null;
        }
    }

    public String getPayload() {
        return payload;
    }

    public byte[] getGzipped() {
        return gzipped;
    }
}



### 缓存使用场景，全量拉取

ApplicationsResource#getContainers

```java
@GET
public Response getContainers(@PathParam("version") String version,
                              @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                              @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
                              @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
                              @Context UriInfo uriInfo,
                              @Nullable @QueryParam("regions") String regionsStr) {

    boolean isRemoteRegionRequested = null != regionsStr && !regionsStr.isEmpty();
    String[] regions = null;
    if (!isRemoteRegionRequested) {
        EurekaMonitors.GET_ALL.increment();
    } else {
        regions = regionsStr.toLowerCase().split(",");
        Arrays.sort(regions); // So we don't have different caches for same regions queried in different order.
        EurekaMonitors.GET_ALL_WITH_REMOTE_REGIONS.increment();
    }

    // Check if the server allows the access to the registry. The server can
    // restrict access if it is not
    // ready to serve traffic depending on various reasons.
    if (!registry.shouldAllowAccess(isRemoteRegionRequested)) {
        return Response.status(Status.FORBIDDEN).build();
    }
    CurrentRequestVersion.set(Version.toEnum(version));
    KeyType keyType = Key.KeyType.JSON;
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }

    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS,
            keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
    );

    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        response = Response.ok(responseCache.getGZIP(cacheKey))
                .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
                .header(HEADER_CONTENT_TYPE, returnMediaType)
                .build();
    } else {
    response = Response.ok(responseCache.get(cacheKey))
            .build();
    }
    return response;
}

```

### 查询缓存

ResponseCacheImpl#get

```java
public String get(final Key key) {
    return get(key, shouldUseReadOnlyResponseCache);
}
```

ResponseCacheImpl#get(final Key key, boolean useReadOnlyCache)

```java
String get(final Key key, boolean useReadOnlyCache) {
    Value payload = getValue(key, useReadOnlyCache);
    if (payload == null || payload.getPayload().equals(EMPTY_PAYLOAD)) {
        return null;
    } else {
        return payload.getPayload();
    }
}
```


ResponseCacheImpl#getValue

private final ConcurrentMap<Key, Value> readOnlyCacheMap = new ConcurrentHashMap<Key, Value>();

```java
Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    try {
        //$1 假设允许只读缓存，那么先从只读缓存中获取
        if (useReadOnlyCache) {
            final Value currentPayload = readOnlyCacheMap.get(key);
            if (currentPayload != null) {
                payload = currentPayload;
            //$2 如果只读缓存中找不到，那么从读写缓存中找
            } else {
                //$3 从读写缓存中拿到Value，随后塞到读缓存
                payload = readWriteCacheMap.get(key);
                readOnlyCacheMap.put(key, payload);
            }
        } else {
            payload = readWriteCacheMap.get(key);
        }
    } catch (Throwable t) {
        logger.error("Cannot get value for key : {}", key, t);
    }
    return payload;
}
```


------------------------
读写缓存

ResponseCacheImpl@readWriteCacheMap

```java
private final LoadingCache<Key, Value> readWriteCacheMap;

//$1 初始化缓存的容量，1000
this.readWriteCacheMap = CacheBuilder.newBuilder().initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
//$2 180s自动过期
.expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
//$3 移除监听回调
.removalListener(new RemovalListener<Key, Value>() {
    @Override
    public void onRemoval(RemovalNotification<Key, Value> notification) {
        Key removedKey = notification.getKey();
        if (removedKey.hasRegions()) {
            Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
            regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
        }
    }
})
//$4 真正的get方法，如果缓存拉取不到，他会走load方法去拉，然后放回缓存
.build(new CacheLoader<Key, Value>() {
    @Override
    public Value load(Key key) throws Exception {
        if (key.hasRegions()) {
            Key cloneWithNoRegions = key.cloneWithoutRegions();
            regionSpecificKeys.put(cloneWithNoRegions, key);
        }
        //$5 真正的get方法 @
        Value value = generatePayload(key);
        return value;
    }
});


```

--------------------------------------------------

5、缓存真正的get方法

ResponseCacheImpl#generatePayload

* 5.1 通过register获取到applications，里面根据当前所有apps计算一致性hashcode  @
* 5.2 私有方法getpayload将applications进行json或者xml的序列化，生成字符串 @
* 5.3 通过payload生成Value @

```java
private Value generatePayload(Key key) {
    Stopwatch tracer = null;
    try {
        String payload;
        switch (key.getEntityType()) {
            case Application:
                boolean isRemoteRegionRequested = key.hasRegions();

                if (ALL_APPS.equals(key.getName())) {
                    if (isRemoteRegionRequested) {
                        tracer = serializeAllAppsWithRemoteRegionTimer.start();
                        payload = getPayLoad(key, registry.getApplicationsFromMultipleRegions(key.getRegions()));
                    } else {
                        tracer = serializeAllAppsTimer.start();
                        //$1 通过register获取到applications，里面根据当前所有apps计算一致性hashcode
                        //$2 私有方法getpayload将applications进行json或者xml的序列化，生成字符串
                        payload = getPayLoad(key, registry.getApplications());
                    }
                } else if (ALL_APPS_DELTA.equals(key.getName())) {
                    if (isRemoteRegionRequested) {
                        tracer = serializeDeltaAppsWithRemoteRegionTimer.start();
                        versionDeltaWithRegions.incrementAndGet();
                        versionDeltaWithRegionsLegacy.incrementAndGet();
                        payload = getPayLoad(key,
                                registry.getApplicationDeltasFromMultipleRegions(key.getRegions()));
                    } else {
                        tracer = serializeDeltaAppsTimer.start();
                        versionDelta.incrementAndGet();
                        versionDeltaLegacy.incrementAndGet();
                        payload = getPayLoad(key, registry.getApplicationDeltas());
                    }
                } else {
                    tracer = serializeOneApptimer.start();
                    payload = getPayLoad(key, registry.getApplication(key.getName()));
                }
                break;
            case VIP:
            case SVIP:
                tracer = serializeViptimer.start();
                payload = getPayLoad(key, getApplicationsForVip(key, registry));
                break;
            default:
                logger.error("Unidentified entity type: {} found in the cache key.", key.getEntityType());
                payload = "";
                break;
        }
        //$3 通过payload生成Value @
        return new Value(payload);
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }
}
```

---------------------------------

5.1、通过register获取到applications，里面根据当前所有apps计算一致性hashcode


AbstractInstanceRegistry#getApplicationsFromMultipleRegions


```java
public Applications getApplicationsFromMultipleRegions(String[] remoteRegions) {

    boolean includeRemoteRegion = null != remoteRegions && remoteRegions.length != 0;

    logger.debug("Fetching applications registry with remote regions: {}, Regions argument {}",
            includeRemoteRegion, remoteRegions);

    if (includeRemoteRegion) {
        GET_ALL_WITH_REMOTE_REGIONS_CACHE_MISS.increment();
    } else {
        GET_ALL_CACHE_MISS.increment();
    }
    Applications apps = new Applications();
    apps.setVersion(1L);
    for (Entry<String, Map<String, Lease<InstanceInfo>>> entry : registry.entrySet()) {
        Application app = null;

        if (entry.getValue() != null) {
            for (Entry<String, Lease<InstanceInfo>> stringLeaseEntry : entry.getValue().entrySet()) {
                Lease<InstanceInfo> lease = stringLeaseEntry.getValue();
                if (app == null) {
                    app = new Application(lease.getHolder().getAppName());
                }
                app.addInstance(decorateInstanceInfo(lease));
            }
        }
        if (app != null) {
            apps.addApplication(app);
        }
    }
    if (includeRemoteRegion) {
        for (String remoteRegion : remoteRegions) {
            RemoteRegionRegistry remoteRegistry = regionNameVSRemoteRegistry.get(remoteRegion);
            if (null != remoteRegistry) {
                Applications remoteApps = remoteRegistry.getApplications();
                for (Application application : remoteApps.getRegisteredApplications()) {
                    if (shouldFetchFromRemoteRegistry(application.getName(), remoteRegion)) {
                        logger.info("Application {}  fetched from the remote region {}",
                                application.getName(), remoteRegion);

                        Application appInstanceTillNow = apps.getRegisteredApplications(application.getName());
                        if (appInstanceTillNow == null) {
                            appInstanceTillNow = new Application(application.getName());
                            apps.addApplication(appInstanceTillNow);
                        }
                        for (InstanceInfo instanceInfo : application.getInstances()) {
                            appInstanceTillNow.addInstance(instanceInfo);
                        }
                    } else {
                        logger.debug("Application {} not fetched from the remote region {} as there exists a "
                                        + "whitelist and this app is not in the whitelist.",
                                application.getName(), remoteRegion);
                    }
                }
            } else {
                logger.warn("No remote registry available for the remote region {}", remoteRegion);
            }
        }
    }
    apps.setAppsHashCode(apps.getReconcileHashCode());
    return apps;
}
```

5.2、私有方法getpayload将applications进行json或者xml的序列化，生成字符串

ResponseCacheImpl#getPayLoad

根据keytype是json还是xml，和compact还是full，获取encoderWrapper，通过wrapper进行编码，生成字符串

```java
private String getPayLoad(Key key, Applications apps) {
    EncoderWrapper encoderWrapper = serverCodecs.getEncoder(key.getType(), key.getEurekaAccept());
    String result;
    try {
        result = encoderWrapper.encode(apps);
    } catch (Exception e) {
        logger.error("Failed to encode the payload for all apps", e);
        return "";
    }
    if(logger.isDebugEnabled()) {
        logger.debug("New application cache entry {} with apps hashcode {}", key.toStringCompact(), apps.getAppsHashCode());
    }
    return result;
}

```

5.3、 通过payload生成Value

Value#Value(String payload)

Value赋值payload，并且生成gzip字节数组

```java
public Value(String payload) {
    this.payload = payload;
    if (!EMPTY_PAYLOAD.equals(payload)) {
        Stopwatch tracer = compressPayloadTimer.start();
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            GZIPOutputStream out = new GZIPOutputStream(bos);
            byte[] rawBytes = payload.getBytes();
            out.write(rawBytes);
            // Finish creation of gzip file
            out.finish();
            out.close();
            bos.close();
            gzipped = bos.toByteArray();
        } catch (IOException e) {
            gzipped = null;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }
    } else {
        gzipped = null;
    }
}
```


### 主动过期读写缓存

AbstractInstanceRegistry#invalidateCache

```java
private void invalidateCache(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress) {
    // invalidate cache
    responseCache.invalidate(appName, vipAddress, secureVipAddress);
}
```

ResponseCacheImpl#invalidate

```java
@Override
public void invalidate(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress) {
    //$1 过期所有有关的缓存，版本:v1/v2，keytype：json/xml
    for (Key.KeyType type : Key.KeyType.values()) {
        for (Version v : Version.values()) {
            invalidate(
                    new Key(Key.EntityType.Application, appName, type, v, EurekaAccept.full),
                    new Key(Key.EntityType.Application, appName, type, v, EurekaAccept.compact),
                    new Key(Key.EntityType.Application, ALL_APPS, type, v, EurekaAccept.full),
                    new Key(Key.EntityType.Application, ALL_APPS, type, v, EurekaAccept.compact),
                    new Key(Key.EntityType.Application, ALL_APPS_DELTA, type, v, EurekaAccept.full),
                    new Key(Key.EntityType.Application, ALL_APPS_DELTA, type, v, EurekaAccept.compact)
            );
            if (null != vipAddress) {
                invalidate(new Key(Key.EntityType.VIP, vipAddress, type, v, EurekaAccept.full));
            }
            if (null != secureVipAddress) {
                invalidate(new Key(Key.EntityType.SVIP, secureVipAddress, type, v, EurekaAccept.full));
            }
        }
    }
}
```

ResponseCacheImpl#invalidate

写缓存invalid这个Key对应的value

```java
public void invalidate(Key... keys) {
    for (Key key : keys) {
        logger.debug("Invalidating the response cache key : {} {} {} {}, {}",
                key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept());

        readWriteCacheMap.invalidate(key);
        Collection<Key> keysWithRegions = regionSpecificKeys.get(key);
        if (null != keysWithRegions && !keysWithRegions.isEmpty()) {
            for (Key keysWithRegion : keysWithRegions) {
                logger.debug("Invalidating the response cache key : {} {} {} {} {}",
                        key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept());
                readWriteCacheMap.invalidate(keysWithRegion);
            }
        }
    }
}
```

### 被动过期写缓存

CacheBuilder生成的读写缓存
expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)


### 定时刷新只读缓存

1、启动

ResponseCacheImpl#ResponseCacheImpl

```java

private final java.util.Timer timer = new java.util.Timer("Eureka-CacheFillTimer", true);


if (shouldUseReadOnlyResponseCache) {
    
    //$1 30s 定时任务， 同步只读缓存与读写缓存
    timer.schedule(getCacheUpdateTask(),
            new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                    + responseCacheUpdateIntervalMs),
            responseCacheUpdateIntervalMs);
}
```

2、同步更新

ResponseCacheImpl#getCacheUpdateTask

* 2.1 轮训只读缓存的key，分别从读写与只读缓存中拿出对应的value，由于只读缓存更新只有2个途径，从读写缓存get然后put到只读缓存，还有就是这里定期同步，所以他们地址是一样的   important tim

```java
private TimerTask getCacheUpdateTask() {
    return new TimerTask() {
        @Override
        public void run() {
            logger.debug("Updating the client cache from response cache");
            //$1 轮训只读缓存的key，分别从读写与只读缓存中拿出对应的value，由于只读缓存更新只有2个途径
            //从读写缓存get然后put到只读缓存，还有就是这里定期同步，所以他们地址是一样的
            for (Key key : readOnlyCacheMap.keySet()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Updating the client cache from response cache for key : {} {} {} {}",
                            key.getEntityType(), key.getName(), key.getVersion(), key.getType());
                }
                try {
                    CurrentRequestVersion.set(key.getVersion());
                    Value cacheValue = readWriteCacheMap.get(key);
                    Value currentCacheValue = readOnlyCacheMap.get(key);
                    if (cacheValue != currentCacheValue) {
                        readOnlyCacheMap.put(key, cacheValue);
                    }
                } catch (Throwable th) {
                    logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), th);
                }
            }
        }
    };
}

```

### 增量拉取

AbstractInstanceRegistry#getApplicationDeltas

* 1 这里为啥是写锁，我觉得是为了避免这种场景，假设注册，register中添加了新的实例，但是rencentQueue还没有变动
那么计算出的hashcode是包含新注册的实例，但是recentqueue中没有，客户端获取到的结果加入到原先存量后计算得到的hashcode
一定和服务端返回是不同的；所以写锁不一定是用在写，读锁不一定用在读。


```java
@Deprecated
public Applications getApplicationDeltas() {
    GET_ALL_CACHE_MISS_DELTA.increment();
    Applications apps = new Applications();
    apps.setVersion(responseCache.getVersionDelta().get());
    Map<String, Application> applicationInstancesMap = new HashMap<String, Application>();
    try {
        //$0 这里为啥是写锁  @ important tim
        write.lock();
        //$1 使用iterator循环获取最近变更queue中的成员变量，组装为Applications
        Iterator<RecentlyChangedItem> iter = this.recentlyChangedQueue.iterator();
        logger.debug("The number of elements in the delta queue is : {}",
                this.recentlyChangedQueue.size());
        while (iter.hasNext()) {
            Lease<InstanceInfo> lease = iter.next().getLeaseInfo();
            InstanceInfo instanceInfo = lease.getHolder();
            logger.debug(
                    "The instance id {} is found with status {} and actiontype {}",
                    instanceInfo.getId(), instanceInfo.getStatus().name(), instanceInfo.getActionType().name());
            Application app = applicationInstancesMap.get(instanceInfo
                    .getAppName());
            if (app == null) {
                app = new Application(instanceInfo.getAppName());
                applicationInstancesMap.put(instanceInfo.getAppName(), app);
                apps.addApplication(app);
            }
            app.addInstance(new InstanceInfo(decorateInstanceInfo(lease)));
        }

        boolean disableTransparentFallback = serverConfig.disableTransparentFallbackToOtherRegion();

        if (!disableTransparentFallback) {
            Applications allAppsInLocalRegion = getApplications(false);

            for (RemoteRegionRegistry remoteRegistry : this.regionNameVSRemoteRegistry.values()) {
                Applications applications = remoteRegistry.getApplicationDeltas();
                for (Application application : applications.getRegisteredApplications()) {
                    Application appInLocalRegistry =
                            allAppsInLocalRegion.getRegisteredApplications(application.getName());
                    if (appInLocalRegistry == null) {
                        apps.addApplication(application);
                    }
                }
            }
        }
        //$2  从register中获得所有applications，计算hashcode，然后将code塞到通过最近变更apps里面再返回
        Applications allApps = getApplications(!disableTransparentFallback);
        apps.setAppsHashCode(allApps.getReconcileHashCode());
        return apps;
    } finally {
        write.unlock();
    }
}
```

rencentQueue的过期逻辑

180s过期，根据最后更新时间与当前时间与180s相减进行比较

```java
private TimerTask getDeltaRetentionTask() {
    return new TimerTask() {

        @Override
        public void run() {
            Iterator<RecentlyChangedItem> it = recentlyChangedQueue.iterator();
            while (it.hasNext()) {
                if (it.next().getLastUpdateTime() <
                        System.currentTimeMillis() - serverConfig.getRetentionTimeInMSInDeltaQueue()) {
                    it.remove();
                } else {
                    break;
                }
            }
        }

    };
}

```