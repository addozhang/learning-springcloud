# Archaius

## 类

### AbstractConfiguration

### ConcurrentCompositeConfiguration

### DynamicConfiguration
动态配置, 定时拉取

使用`调度器`通过`PolledConfigurationSource`获取配置, 并应用到本地配置.

### DynamicWatchedConfiguration
等待来自指定的配置源的观察事件. 如果底层配置源的配置发生变化, 相应的本地配置也会改变. 

不支持key或者value为null的配置.

具体实现:
通过`WatchedConfigurationSource`获取事件, 然后通过`DynamicPropertyUpdater`更新本地配置.

### PolledConfigurationSource (拉模式)
通过polling拉取远程配置

实现类JDBCConfigurationSource, URLConfigurationSource

```java
/**
 * The definition of configuration source that brings dynamic changes to the configuration via polling.
 */
public interface PolledConfigurationSource {

    /**
     * Poll the configuration source to get the latest content.
     * 
     * @param initial true if this operation is the first poll.
     * @param checkPoint Object that is used to determine the starting point if the result returned is incremental. 
     *          Null if there is no check point or the caller wishes to get the full content.
     * @return The content of the configuration which may be full or incremental.
     * @throws Exception If any exception occurs when fetching the configurations.
     */
    public PollResult poll(boolean initial, Object checkPoint) throws Exception;    
}
```

### WatchedConfigurationSource (推模式)

```java
/**
 * The definition of configuration source that brings dynamic changes to the configuration via watchers.
 */
public interface WatchedConfigurationSource {
    /**
     * Add {@link WatchedUpdateListener} listener
     * 
     * @param l
     */
    public void addUpdateListener(WatchedUpdateListener l);

    /**
     * Remove {@link WatchedUpdateListener} listener
     * 
     * @param l
     */
    public void removeUpdateListener(WatchedUpdateListener l);

    /**
     * Get a snapshot of the latest configuration data.<BR>
     * 
     * Note: The correctness of this data is only as good as the underlying config source's view of the data.
     */
    public Map<String, Object> getCurrentData() throws Exception;
}
```

## 其他实现

### archaius-etcd

### archaius-zookeeper
