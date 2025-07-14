- [apache-atlas-hive-hook-源码分析_风的心愿的博客-CSDN博客_atlas hive hook](https://blog.csdn.net/windydreams/article/details/127893478)

# Atlas Hook类图

Hive 元数据变更有2种实现：
1)基于 Hook 函数实现，实现类为 HiveHook
2)基于MetaStoreEventListener 实现， 实现类为HiveMetastoreHookImpl
所以提供2 种配置，即配置钩子函数或监听器，我们目前采用的是第二种方案。Atlas hook的类图如下所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/2a1f328d63064ad88285de8e6758a631.png#pic_center)Hive hook 开发的步骤
1）创建类实现ExecuteWithHookContext
2）打包上传到HIVE_HOME/lib
3）配置，可在hive client中临时配置，或者设置hive-site.xml配置

# Hive Hook[源码](https://so.csdn.net/so/search?q=源码&spm=1001.2101.3001.7020)分析

HiveHook重写了AtlasHook的run方法，根据hive的操作类型，生成事件，通过AtlasHook#notifyEntities发送事件消息。

## HiveHook#run

```java
public class HiveHook extends AtlasHook implements ExecuteWithHookContext {
@Override
public void run(HookContext hookContext) throws Exception {
//...
    try {
// 从hookContext中获取HiveOperation
        HiveOperation        oper    = OPERATION_MAP.get(hookContext.getOperationName());
        AtlasHiveHookContext context = new AtlasHiveHookContext(this, oper, hookContext, getKnownObjects());
        BaseHiveEvent        event   = null;
// 匹配不同的HiveOperation, 创建事件        
        switch (oper) {
            case CREATEDATABASE:
                event = new CreateDatabase(context);
            break;
            case DROPDATABASE:
                event = new DropDatabase(context);
            break;
//...
            case ALTERTABLE_RENAME:
            case ALTERVIEW_RENAME:
                event = new AlterTableRename(context);
            break;
            case ALTERTABLE_RENAMECOL:
                event = new AlterTableRenameCol(context);
            break;
            default:
                //...
            break;
        }
        if (event != null) {
// 用户组信息
            final UserGroupInformation ugi = hookContext.getUgi() == null ? Utils.getUGI() : hookContext.getUgi();
// 委托给AtlasHook#notifyEntities
            super.notifyEntities(event.getNotificationMessages(), ugi);
        }
    } catch (Throwable t) {
        LOG.error("HiveHook.run(): failed to process operation {}", hookContext.getOperationName(), t);
    }
//...
}
}
```

hive事件类图
![在这里插入图片描述](https://img-blog.csdnimg.cn/1c626fb46a2a4d4bb9aff2436aefa908.png#pic_center)BaseHiveEvent事件主要分为以下几类
1）Hive类型 HIVE_TYPE_
2）Hbase类型 HBASE_TYPE_
3）Hive属性 ATTRIBUTE_
4）Hbase属性 HBASE_
5）Hive关系属性 RELATIONSHIP_
6）其他

## AtlasHook#notifyEntities

notifyEntities消息通知有同步和异步两种策略，notifyEntities委托给notifyEntitiesInternal

```java
public static void notifyEntities(List<HookNotification> messages, UserGroupInformation ugi, int maxRetries) {
    if (executor == null) { // send synchronously
// 同步通知
        notifyEntitiesInternal(messages, maxRetries, ugi, notificationInterface, logFailedMessages, failedMessagesLogger);
    } else {
// 异步通知
        executor.submit(new Runnable() {
            @Override
            public void run() {
                notifyEntitiesInternal(messages, maxRetries, ugi, notificationInterface, logFailedMessages, failedMessagesLogger);
            }
        });
    }
}
```

## AtlasHook#notifyEntitiesInternal

notifyEntitiesInternal实现了重试策略， 掉用notificationInterface.send方法发送消息

```java
@VisibleForTesting
static void notifyEntitiesInternal(List<HookNotification> messages, int maxRetries, UserGroupInformation ugi,
                                   NotificationInterface notificationInterface,
                                   boolean shouldLogFailedMessages, FailedMessagesLogger logger) {
// ...
    final int maxAttempts         = maxRetries < 1 ? 1 : maxRetries;
    Exception notificationFailure = null;
// 多次尝试
    for (int numAttempt = 1; numAttempt <= maxAttempts; numAttempt++) {
        if (numAttempt > 1) { // retry attempt
// 重试，线程sleep
            try {
                LOG.debug("Sleeping for {} ms before retry", notificationRetryInterval);
                Thread.sleep(notificationRetryInterval);
            } catch (InterruptedException ie) {
                LOG.error("Notification hook thread sleep interrupted");
                break;
            }
        }
        try {
//  发送通知，什么地方接收通知
            if (ugi == null) {
                notificationInterface.send(NotificationInterface.NotificationType.HOOK, messages);
            } else {
                PrivilegedExceptionAction<Object> privilegedNotify = new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                        notificationInterface.send(NotificationInterface.NotificationType.HOOK, messages);
                        return messages;
                    }
                };
                ugi.doAs(privilegedNotify);
            }
            notificationFailure = null; // notification sent successfully, reset error
            break;
        } catch (Exception e) {
            notificationFailure = e;
            LOG.error("Failed to send notification - attempt #{}; error={}", numAttempt, e.getMessage());
        }
    }

// 通知失败处理
// ...
}
```

通知消息最终由KafkaNotification将消息发送到kafka， KafkaNotification在AtlasHook静态方法种由NotificationProvider创建

## notificationInterface的初始化

notificationInterface的初始化在AtlasHook 的static方法中初始化，notificationInterface的实现类为KafkaNotification，消息将发送到kafka中，Atlas服务端对消息进行消费，然后入库。

```java
public abstract class AtlasHook {
//...
    protected static Configuration         atlasProperties;
    protected static NotificationInterface notificationInterface;
//...
    private static       ExecutorService      executor = null;

    static {
        try {
            atlasProperties = ApplicationProperties.get();
        } catch (Exception e) {
            LOG.info("Failed to load application properties", e);
        }
//...
        metadataNamespace         = getMetadataNamespace(atlasProperties);
        placeCode                 = getPlaceCode(atlasProperties);
        notificationMaxRetries    = atlasProperties.getInt(ATLAS_NOTIFICATION_MAX_RETRIES, 3);
        notificationRetryInterval = atlasProperties.getInt(ATLAS_NOTIFICATION_RETRY_INTERVAL, 1000);
// 消息通知接口实现
        notificationInterface     = NotificationProvider.get();
//...
}
```

# HiveMetastoreHookImpl源码分析

## HiveMetastoreHookImpl

HiveMetastoreHookImpl 继承了MetaStoreEventListener 类，重写了 onCreateDatabase、onDropDatabase等方法，并委托给HiveMetastoreHook#handleEvent处理

```java
public class HiveMetastoreHookImpl extends MetaStoreEventListener {
    private static final Logger            LOG = LoggerFactory.getLogger(HiveMetastoreHookImpl.class);
    private        final HiveHook          hiveHook;
    private        final HiveMetastoreHook hook;

    public HiveMetastoreHookImpl(Configuration config) {
        super(config);
        this.hiveHook = new HiveHook();
        this.hook     = new HiveMetastoreHook();
    }
//...
    @Override
    public void onCreateDatabase(CreateDatabaseEvent dbEvent) {
        HiveOperationContext context = new HiveOperationContext(CREATEDATABASE, dbEvent);
    // 委托给HiveMetastoreHook执行
        hook.handleEvent(context);
    }
    @Override
    public void onDropDatabase(DropDatabaseEvent dbEvent) {
        HiveOperationContext context = new HiveOperationContext(DROPDATABASE, dbEvent);
        hook.handleEvent(context);
}
//...
}
```

## HiveMetastoreHookImpl#handleEvent

handleEvent将变更消息通过AtlasHook#notifyEntities，消息将发送到kafka中，Atlas服务端对消息进行消费，然后入库。

```java
public void handleEvent(HiveOperationContext operContext) {
        ListenerEvent listenerEvent = operContext.getEvent();
//...
        try {
            HiveOperation        oper    = operContext.getOperation();
            AtlasHiveHookContext context = new AtlasHiveHookContext(hiveHook, oper, hiveHook.getKnownObjects(), this, listenerEvent);
            BaseHiveEvent        event   = null;
            switch (oper) {
                case CREATEDATABASE:
                    event = new CreateDatabase(context);
                    break;
               //...
                case ALTERTABLE_RENAMECOL:
                    FieldSchema columnOld = operContext.getColumnOld();
                    FieldSchema columnNew = operContext.getColumnNew();
                    event = new AlterTableRenameCol(columnOld, columnNew, context);
                    break;
                default:
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("HiveMetastoreHook.handleEvent({}): operation ignored.", listenerEvent);
                    }
                    break;
            }
            if (event != null) {
                final UserGroupInformation ugi = SecurityUtils.getUGI() == null ? Utils.getUGI() : SecurityUtils.getUGI();
// 消息通知,AtlasHook#notifyEntities
                super.notifyEntities(event.getNotificationMessages(), ugi);
            }
        } catch (Throwable t) {
            LOG.error("HiveMetastoreHook.handleEvent({}): failed to process operation {}", listenerEvent, t);
        }
    }
}
```