- [apache-atlas-hbase-hook源码分析_风的心愿的博客-CSDN博客_apache atlas hbase](https://blog.csdn.net/windydreams/article/details/128013418)

# 元数据类型

Hbase元数据类型, 包括命令空间、表、列族、列

```java
public enum HBaseDataTypes {
    // Classes
    HBASE_NAMESPACE,
    HBASE_TABLE,
    HBASE_COLUMN_FAMILY,
    HBASE_COLUMN;

    public String getName() {
        return name().toLowerCase();
    }
}
```

Hbase元数据采集实现

1）批量采集HBaseBridge

2）实时变更 HBaseAtlasCoprocessor

虽然定义了HBASE_COLUMN，但是实际上是没有实现的，毕竟HBASE_COLUMN是动态添加的。

本文分析实时变更 HBaseAtlasCoprocessor。

# HBaseAtlasCoprocessor源码分析

Hbase的Hook是基于Coprocessor协处理器实现的，HBaseAtlasCoprocessor 源码如下：

```java
public class HBaseAtlasCoprocessor implements MasterCoprocessor, MasterObserver, RegionObserver, RegionServerObserver  {
    //...
    final HBaseAtlasHook hbaseAtlasHook;
    public HBaseAtlasCoprocessor() {
        hbaseAtlasHook = HBaseAtlasHook.getInstance();
    }
    @Override
    public void postCreateTable(ObserverContext<MasterCoprocessorEnvironment> observerContext, TableDescriptor tableDescriptor, RegionInfo[] hRegionInfos) throws IOException {
        hbaseAtlasHook.sendHBaseTableOperation(tableDescriptor, null, HBaseAtlasHook.OPERATION.CREATE_TABLE, observerContext);
       //...
    }
    @Override
    public void postDeleteTable(ObserverContext<MasterCoprocessorEnvironment> observerContext, TableName tableName) throws IOException {
        hbaseAtlasHook.sendHBaseTableOperation(null, tableName, HBaseAtlasHook.OPERATION.DELETE_TABLE, observerContext);
    }
//  其他Override的方法
//postModifyTable
//postCreateNamespace
//postDeleteNamespace
//postModifyNamespace
//postCloneSnapshot
//postRestoreSnapshot

}
```

HBaseAtlasCoprocessor 重写了MasterObserver 的8个方法， 这8个方法委托给hbaseAtlasHook实现， HbaseAtlasHook类图如下所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/73fbd651ff514d88a273f46b3fffeed8.png)

## hbaseAtlasHook#sendHBaseTableOperation

sendHBaseTableOperation是对外提供服务的接口，内部完成以下任务：

1. 消息封装 handleHBaseTableOperation
2. 消息发送 sendNotification

```java
public void sendHBaseTableOperation(TableDescriptor tableDescriptor, final TableName tableName, final OPERATION operation, ObserverContext<MasterCoprocessorEnvironment> ctx) {
   //...
    try {
        final UserGroupInformation ugi  = getUGI(ctx);
        final User user                 = getActiveUser(ctx);
        final String userName           = (user != null) ? user.getShortName() : null;
        HBaseOperationContext hbaseOperationContext = handleHBaseTableOperation(tableDescriptor, tableName, operation, ugi, userName);

        sendNotification(hbaseOperationContext);
    } catch (Throwable t) {
        LOG.error("<== HBaseAtlasHook.sendHBaseTableOperation(): failed to send notification", t);
    }
//...
}
```

### hbaseAtlasHook#handleHBaseTableOperation

```java
private HBaseOperationContext handleHBaseTableOperation(TableDescriptor tableDescriptor, TableName tableName, OPERATION operation, UserGroupInformation ugi, String userName) {
//...
    Map<String, String>  hbaseConf          = null;
    String               owner              = null;
    String               tableNameSpace     = null;
    TableName            hbaseTableName     = null;
    ColumnFamilyDescriptor[]  columnFamilyDescriptors = null;


    if (tableDescriptor != null) {
        owner = tableDescriptor.getOwnerString();
        hbaseConf = null;
        hbaseTableName = tableDescriptor.getTableName();
        if (hbaseTableName != null) {
            tableNameSpace = hbaseTableName.getNamespaceAsString();
            if (tableNameSpace == null) {
                tableNameSpace = hbaseTableName.getNameWithNamespaceInclAsString();
            }
        }
    }
//... 
    if (tableDescriptor != null) {
        columnFamilyDescriptors = tableDescriptor.getColumnFamilies();
    }
// 构建HBaseOperationContext
    HBaseOperationContext hbaseOperationContext = new HBaseOperationContext(tableNameSpace, tableDescriptor, tableName, columnFamilyDescriptors, operation, ugi, userName, owner, hbaseConf);
// 生成atlas-hook消息
    createAtlasInstances(hbaseOperationContext);
   //...
    return hbaseOperationContext;
}
```

### hbaseAtlasHook#createAtlasInstances

根据不同的OPERATION 生成消息， 消息封装在hbaseOperationContext

```java
public void createAtlasInstances(HBaseOperationContext hbaseOperationContext) {
    OPERATION operation = hbaseOperationContext.getOperation();
    switch (operation) {
        case CREATE_NAMESPACE:
        case ALTER_NAMESPACE:
            createOrUpdateNamespaceInstance(hbaseOperationContext);
            break;
        case DELETE_NAMESPACE:
            deleteNameSpaceInstance(hbaseOperationContext);
            break;
        case CREATE_TABLE:
        case ALTER_TABLE:
            createOrUpdateTableInstance(hbaseOperationContext);
            break;
        case DELETE_TABLE:
            deleteTableInstance(hbaseOperationContext);
            break;
        case CREATE_COLUMN_FAMILY:
        case ALTER_COLUMN_FAMILY:
            createOrUpdateColumnFamilyInstance(hbaseOperationContext);
            break;
        case DELETE_COLUMN_FAMILY:
            deleteColumnFamilyInstance(hbaseOperationContext);
            break;
    }
}
```

## hbaseAtlasHook#sendNotification

sendNotification处理UserGroupInformation 后，交给notifyEntities处理

```java
private void sendNotification(HBaseOperationContext hbaseOperationContext) {
    UserGroupInformation ugi = hbaseOperationContext.getUgi();
    if (ugi != null && ugi.getRealUser() != null) {
        ugi = ugi.getRealUser();
    }
    notifyEntities(hbaseOperationContext.getMessages(), ugi);
}
```

### hbaseAtlasHook#notifyEntities

notifyEntities 在《apache-[hive](https://so.csdn.net/so/search?q=hive&spm=1001.2101.3001.7020)-hook-源码分析》已分析，messages最终通过KafkaNotification发送到Atlas, 消息的消费后期分析

```java
protected void notifyEntities(List<HookNotification> messages, UserGroupInformation ugi) {
    notifyEntities(messages, ugi, notificationMaxRetries);
}
public static void notifyEntities(List<HookNotification> messages, UserGroupInformation ugi, int maxRetries) {
    if (executor == null) { // send synchronously
        notifyEntitiesInternal(messages, maxRetries, ugi, notificationInterface, logFailedMessages, failedMessagesLogger);
    } else {
        executor.submit(new Runnable() {
            @Override
            public void run() {
                notifyEntitiesInternal(messages, maxRetries, ugi, notificationInterface, logFailedMessages, failedMessagesLogger);
            }
        });
    }
}
```