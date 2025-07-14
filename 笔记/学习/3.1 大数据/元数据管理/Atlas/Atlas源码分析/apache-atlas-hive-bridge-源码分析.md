- [apache-atlas-hive-bridge-源码分析_风的心愿的博客-CSDN博客_apache atlas 源码](https://blog.csdn.net/windydreams/article/details/127843534)

# Hive元数据类型

```java
public enum HiveDataTypes {
    // Enums 枚举
    HIVE_OBJECT_TYPE,
    HIVE_PRINCIPAL_TYPE,
    HIVE_RESOURCE_TYPE,

    // Structs 结构体
    HIVE_SERDE,
    HIVE_ORDER,
    HIVE_RESOURCEURI,

    // Classes 类
    HIVE_DB,
    HIVE_STORAGEDESC,
    HIVE_TABLE,
    HIVE_COLUMN,
    HIVE_PARTITION,
    HIVE_INDEX,
    HIVE_ROLE,
    HIVE_TYPE,
    HIVE_PROCESS,
    HIVE_COLUMN_LINEAGE,
    HIVE_PROCESS_EXECUTION,
    // HIVE_VIEW,
    ;
    public String getName() {
        return name().toLowerCase();
    }
}
```

# 元数据采集

Hive 的元数据采集分为2个部分：

1、 批量采集 HiveMetaStoreBridge

2 、实时变更采集 HiveHook

本文先分析HiveMetaStoreBridge ，批量采集 HiveMetaStoreBridge 流程图如下所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/32636125a56b401dbb947e9b1c637cdf.png)
注册tableEntity步骤中，会处理存储描述器sd、hive列、分区键等信息，在注册tableEntity的时候，会同时注册存储描述器sd、hive列、分区键实体等

## HiveMetaStoreBridge构造函数

```java
public HiveMetaStoreBridge(Configuration atlasProperties, HiveConf hiveConf, AtlasClientV2 atlasClientV2) throws Exception {
// 元数据命名空间
    this.metadataNamespace          = getMetadataNamespace(atlasProperties);
// hive客户端
    this.hiveClient                 = Hive.get(hiveConf);
//atlas 客户端
    this.atlasClientV2              = atlasClientV2;
// 将tHdfsPath转成小写
    this.convertHdfsPathToLowerCase = atlasProperties.getBoolean(HDFS_PATH_CONVERT_TO_LOWER_CASE, false);
}
```

## HiveMetaStoreBridge#main 方法

```java
public static void main(String[] args) {
//...
    try {
// 1、准备
// 读取命令行参数
        Options options = new Options();
        options.addOption("d", "database", true, "Database name");
        options.addOption("t", "table", true, "Table name");
        options.addOption("f", "filename", true, "Filename");
        options.addOption("failOnError", false, "failOnError");

        CommandLine   cmd              = new BasicParser().parse(options, args);
        boolean       failOnError      = cmd.hasOption("failOnError");
// 指定数据库
        String        databaseToImport = cmd.getOptionValue("d");
// 指定数据库表b
        String        tableToImport    = cmd.getOptionValue("t");
// 从文件中读取
        String        fileToImport     = cmd.getOptionValue("f");
// 获取配置文件
        Configuration atlasConf        = ApplicationProperties.get();
        String[]      atlasEndpoint    = atlasConf.getStringArray(ATLAS_ENDPOINT);

        if (atlasEndpoint == null || atlasEndpoint.length == 0) {
            atlasEndpoint = new String[] { DEFAULT_ATLAS_URL };
        }

//  初始化AtlasClientV2客户端
        if (!AuthenticationUtil.isKerberosAuthenticationEnabled()) {
            String[] basicAuthUsernamePassword = AuthenticationUtil.getBasicAuthenticationInput();
            atlasClientV2 = new AtlasClientV2(atlasEndpoint, basicAuthUsernamePassword);
        } else {
            UserGroupInformation ugi = UserGroupInformation.getCurrentUser();
            atlasClientV2 = new AtlasClientV2(ugi, ugi.getShortUserName(), atlasEndpoint);
        }

// 2、创建HiveMetaStoreBridge
        HiveMetaStoreBridge hiveMetaStoreBridge = new HiveMetaStoreBridge(atlasConf, new HiveConf(), atlasClientV2);
//3、导入
        if (StringUtils.isNotEmpty(fileToImport)) {

            File f = new File(fileToImport);

            if (f.exists() && f.canRead()) {
                BufferedReader br   = new BufferedReader(new FileReader(f));
                String         line = null;

                while((line = br.readLine()) != null) {
                    String val[] = line.split(":");
                    if (ArrayUtils.isNotEmpty(val)) {
                        databaseToImport = val[0];
                        if (val.length > 1) {
                            tableToImport = val[1];
                        } else {
                            tableToImport = "";
                        }
// 导入文件的库表
                        hiveMetaStoreBridge.importHiveMetadata(databaseToImport, tableToImport, failOnError);
                    }
                }


                exitCode = EXIT_CODE_SUCCESS;
            } else {
                LOG.error("Failed to read the input file: " + fileToImport);
            }
        } else {
//  导入命令行的库表
            hiveMetaStoreBridge.importHiveMetadata(databaseToImport, tableToImport, failOnError);
        }

        exitCode = EXIT_CODE_SUCCESS;
    } catch(ParseException e) {
       //...
    } catch(Exception e) {
        LOG.error("Import failed", e);
    } finally {
       //...
    }
    System.exit(exitCode);
}
```

## HiveMetaStoreBridge#importDatabases

importDatabases是提供给外部调用的接口，完成导入Hive库表等元数据信息

```java
private void importDatabases(boolean failOnError, String databaseToImport, String tableToImport) throws Exception {
    List<String> databaseNames = null;

// 获取数据库名
    if (StringUtils.isEmpty(databaseToImport) && StringUtils.isEmpty(tableToImport)) {
        //when both database and table to import are empty, import all
        databaseNames = hiveClient.getAllDatabases();
    } else if (StringUtils.isEmpty(databaseToImport) && StringUtils.isNotEmpty(tableToImport)) {
        //when database is empty and table is not, then check table has database name in it and import that db and table
        if (isTableWithDatabaseName(tableToImport)) {
            String val[] = tableToImport.split("\\.");
            if (val.length > 1) {
                databaseToImport = val[0];
                tableToImport = val[1];
            }
            databaseNames = hiveClient.getDatabasesByPattern(databaseToImport);
        } else {
            databaseNames = hiveClient.getAllDatabases();
        }
    } else {
        //when database to import has some value then, import that db and all table under it.
        databaseNames = hiveClient.getDatabasesByPattern(databaseToImport);
    }

    if(!CollectionUtils.isEmpty(databaseNames)) {
        LOG.info("Found {} databases", databaseNames.size());
// 依次导入库表
        for (String databaseName : databaseNames) {

// 注册数据库实体
            AtlasEntityWithExtInfo dbEntity = registerDatabase(databaseName);
//  导入表
            if (dbEntity != null) {
                importTables(dbEntity.getEntity(), databaseName, tableToImport, failOnError);
            }
        }
    } else {
        LOG.info("No database found");
    }
}
```

### HiveMetaStoreBridge# registerDatabase

registerDatabase方法主要注册数据库信息， toDbEntity主要是构建数据库实体对象

```java
private AtlasEntityWithExtInfo registerDatabase(String databaseName) throws Exception {

    AtlasEntityWithExtInfo ret = null;
    Database               db  = hiveClient.getDatabase(databaseName);
    if (db != null) {
        ret = findDatabase(metadataNamespace, databaseName);

        if (ret == null) {
// toDbEntity 创建数据库实体并设置相关属性Attribute ，registerInstance注册数据库实例
            ret = registerInstance(new AtlasEntityWithExtInfo(toDbEntity(db)));
        } else {
            LOG.info("Database {} is already registered - id={}. Updating it.", databaseName, ret.getEntity().getGuid());

            ret.setEntity(toDbEntity(db, ret.getEntity()));
//updateInstance 注册数据库实例
            updateInstance(ret);
        }
    }
    return ret;
}
```

### HiveMetaStoreBridge#importTable

获取表名并委托给importTables导入

```java
private int importTables(AtlasEntity dbEntity, String databaseName, String tblName, final boolean failOnError) throws Exception {
    int tablesImported = 0;
    final List<String> tableNames;

// 获取表名列表
    if (StringUtils.isEmpty(tblName)) {
        tableNames = hiveClient.getAllTables(databaseName);
    } else {
        tableNames = hiveClient.getTablesByPattern(databaseName, tblName);
    }

    if(!CollectionUtils.isEmpty(tableNames)) {
        LOG.info("Found {} tables to import in database {}", tableNames.size(), databaseName);


        try {
            for (String tableName : tableNames) {
// 导入表，委托给importTable
                int imported = importTable(dbEntity, databaseName, tableName, failOnError);
                tablesImported += imported;
            }
        } finally {
            if (tablesImported == tableNames.size()) {
                LOG.info("Successfully imported {} tables from database {}", tablesImported, databaseName);
            } else {
                LOG.error("Imported {} of {} tables from database {}. Please check logs for errors during import", tablesImported, tableNames.size(), databaseName);
            }
        }
    } else {
        LOG.info("No tables to import in database {}", databaseName);
    }

    return tablesImported;
}
```

### HiveMetaStoreBridge#importTable

importTable方法主要负责注册表实体，已经处理外部表的逻辑

```java
@VisibleForTesting
public int importTable(AtlasEntity dbEntity, String databaseName, String tableName, final boolean failOnError) throws Exception {

    try {
        Table                  table       = hiveClient.getTable(databaseName, tableName);

// 注册表实体，会调用 registerInstance
        AtlasEntityWithExtInfo tableEntity = registerTable(dbEntity, table);

        if (table.getTableType() == TableType.EXTERNAL_TABLE) {
// 外部表 额外逻辑

// getTableProcessQualifiedName 会考虑临时表的情况
            String                 processQualifiedName = getTableProcessQualifiedName(metadataNamespace, table);
            AtlasEntityWithExtInfo processEntity        = findProcessEntity(processQualifiedName);

            if (processEntity == null) {
// processEntity 不存在
                String      tableLocation = isConvertHdfsPathToLowerCase() ? lower(table.getDataLocation().toString()) : table.getDataLocation().toString();
                String      query         = getCreateTableString(table, tableLocation);
// 表hdfs路径实体
                AtlasEntity pathInst      = toHdfsPathEntity(tableLocation);
                AtlasEntity tableInst     = tableEntity.getEntity();
// HIVE_PROCESS 实体
                AtlasEntity processInst   = new AtlasEntity(HiveDataTypes.HIVE_PROCESS.getName());
                long        now           = System.currentTimeMillis();
// 设置processInst的属性
                processInst.setAttribute(ATTRIBUTE_QUALIFIED_NAME, processQualifiedName);
                processInst.setAttribute(ATTRIBUTE_NAME, query);
                processInst.setAttribute(ATTRIBUTE_CLUSTER_NAME, metadataNamespace);
// ATTRIBUTE_INPUTS
                processInst.setRelationshipAttribute(ATTRIBUTE_INPUTS, Collections.singletonList(AtlasTypeUtil.getAtlasRelatedObjectId(pathInst, RELATIONSHIP_DATASET_PROCESS_INPUTS)));
// ATTRIBUTE_OUTPUTS
                processInst.setRelationshipAttribute(ATTRIBUTE_OUTPUTS, Collections.singletonList(AtlasTypeUtil.getAtlasRelatedObjectId(tableInst, RELATIONSHIP_PROCESS_DATASET_OUTPUTS)));
                processInst.setAttribute(ATTRIBUTE_USER_NAME, table.getOwner());
                processInst.setAttribute(ATTRIBUTE_START_TIME, now);
                processInst.setAttribute(ATTRIBUTE_END_TIME, now);
                processInst.setAttribute(ATTRIBUTE_OPERATION_TYPE, "CREATETABLE");
                processInst.setAttribute(ATTRIBUTE_QUERY_TEXT, query);
                processInst.setAttribute(ATTRIBUTE_QUERY_ID, query);
                processInst.setAttribute(ATTRIBUTE_QUERY_PLAN, "{}");
                processInst.setAttribute(ATTRIBUTE_RECENT_QUERIES, Collections.singletonList(query));

                AtlasEntitiesWithExtInfo createTableProcess = new AtlasEntitiesWithExtInfo();
// createTableProcess 实体
                createTableProcess.addEntity(processInst);
                createTableProcess.addEntity(pathInst);
// 注册createTableProcess例子
                registerInstances(createTableProcess);

            } else {
                LOG.info("Process {} is already registered", processQualifiedName);
            }
        }

        return 1;
    } catch (Exception e) {
        LOG.error("Import failed for hive_table {}", tableName, e);
        if (failOnError) {
            throw e;
        }
        return 0;
    }
}
```

### HiveMetaStoreBridge#registerTable

注册表

```java
private AtlasEntityWithExtInfo registerTable(AtlasEntity dbEntity, Table table) throws AtlasHookException {
    try {
        AtlasEntityWithExtInfo ret;
        AtlasEntityWithExtInfo tableEntity = findTableEntity(table);


        if (tableEntity == null) {
            tableEntity = toTableEntity(dbEntity, table);


            ret = registerInstance(tableEntity);
        } else {
            LOG.info("Table {}.{} is already registered with id {}. Updating entity.", table.getDbName(), table.getTableName(), tableEntity.getEntity().getGuid());


            ret = toTableEntity(dbEntity, table, tableEntity);


            updateInstance(ret);
        }


        return ret;
    } catch (Exception e) {
        throw new AtlasHookException("HiveMetaStoreBridge.registerTable() failed.", e);
    }
}
```

### HiveMetaStoreBridge#toTableEntity

toTableEntity方法主要负责组装表实体，主要设置表的属性、与数据库的关系属性，处理取存储描述器、分区键、列质之间的关联关系，返回AtlasEntityWithExtInfo对象

```java
private AtlasEntityWithExtInfo toTableEntity(AtlasEntity database, final Table hiveTable, AtlasEntityWithExtInfo table) throws AtlasHookException {
    if (table == null) {
        table = new AtlasEntityWithExtInfo(new AtlasEntity(HiveDataTypes.HIVE_TABLE.getName()));
    }

    AtlasEntity tableEntity        = table.getEntity();
    String      tableQualifiedName = getTableQualifiedName(metadataNamespace, hiveTable);
    long        createTime         = BaseHiveEvent.getTableCreateTime(hiveTable);
    long        lastAccessTime     = hiveTable.getLastAccessTime() > 0 ? hiveTable.getLastAccessTime() : createTime;

// 设置tableEntity与dbEntity的关系属性
    tableEntity.setRelationshipAttribute(ATTRIBUTE_DB, AtlasTypeUtil.getAtlasRelatedObjectId(database, RELATIONSHIP_HIVE_TABLE_DB));
// 设置tableEntity其他属性
    tableEntity.setAttribute(ATTRIBUTE_QUALIFIED_NAME, tableQualifiedName);
    tableEntity.setAttribute(ATTRIBUTE_NAME, hiveTable.getTableName().toLowerCase());
```