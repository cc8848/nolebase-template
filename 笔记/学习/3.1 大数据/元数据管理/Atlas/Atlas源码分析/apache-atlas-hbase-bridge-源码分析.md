- [apache-atlas-hbase-bridge-源码分析_风的心愿的博客-CSDN博客](https://blog.csdn.net/windydreams/article/details/127978562)

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

# 执行流程

HBaseBridge 执行流程如下图所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/de3891146f554d12b1a28e1cc7c51d7e.png#pic_center)

# 源码分析

## HBaseBridge #main

```java
public class HBaseBridge {
   
//...
    private final String        metadataNamespace;
    private final AtlasClientV2 atlasClientV2;
    private final Admin         hbaseAdmin;

    public static void main(String[] args) {
        int exitCode = EXIT_CODE_FAILED;
        AtlasClientV2 atlasClientV2  =null;

        try {
            Options options = new Options();
            options.addOption("n","namespace", true, "namespace");
            options.addOption("t", "table", true, "tablename");
            options.addOption("f", "filename", true, "filename");

            CommandLineParser parser            = new BasicParser();
            CommandLine       cmd               = parser.parse(options, args);
            String            namespaceToImport = cmd.getOptionValue("n");
            String            tableToImport     = cmd.getOptionValue("t");
            String            fileToImport      = cmd.getOptionValue("f");
            Configuration     atlasConf         = ApplicationProperties.get();
            String[]          urls              = atlasConf.getStringArray(ATLAS_ENDPOINT);

           //...

            if (!AuthenticationUtil.isKerberosAuthenticationEnabled()) {
                String[] basicAuthUsernamePassword = AuthenticationUtil.getBasicAuthenticationInput();
                atlasClientV2 = new AtlasClientV2(urls, basicAuthUsernamePassword);
            } else {
                UserGroupInformation ugi = UserGroupInformation.getCurrentUser();
                atlasClientV2 = new AtlasClientV2(ugi, ugi.getShortUserName(), urls);
            }
//...
            HBaseBridge importer = new HBaseBridge(atlasConf, atlasClientV2);
            if (StringUtils.isNotEmpty(fileToImport)) {
                File f = new File(fileToImport);
                if (f.exists() && f.canRead()) {
                    BufferedReader br   = new BufferedReader(new FileReader(f));
                    String         line = null;

                    while((line = br.readLine()) != null) {
                        String val[] = line.split(":");
                        if (ArrayUtils.isNotEmpty(val)) {
                            //...
                            importer.importHBaseEntities(namespaceToImport, tableToImport);
                        }
                    }
                    exitCode = EXIT_CODE_SUCCESS;
                } else {
                    LOG.error("Failed to read the file");
                }
            } else {
                importer.importHBaseEntities(namespaceToImport, tableToImport);
                exitCode = EXIT_CODE_SUCCESS;
            }
        } catch(ParseException e) {
           //...
        } catch(Exception e) {
            //...
        }finally {
            //...
        }
    }
```

## HBaseBridge#importHBaseEntities

importHBaseEntities 只要负责处理namespaceToImport和tableToImport参数，然后执行相应的流程

```java
private boolean importHBaseEntities(String namespaceToImport, String tableToImport) throws Exception {
    boolean ret = false;

    if (StringUtils.isEmpty(namespaceToImport) && StringUtils.isEmpty(tableToImport)) {
        // when both NameSpace and Table options are not present
        importNameSpaceAndTable();
        ret = true;
    } else if (StringUtils.isNotEmpty(namespaceToImport)) {
        // When Namespace option is present or both namespace and table options are present
        importNameSpaceWithTable(namespaceToImport, tableToImport);
        ret = true;
    } else  if (StringUtils.isNotEmpty(tableToImport)) {
        importTable(tableToImport);
        ret = true;
    }

    return ret;
}
```

## 导入所有的命名空间和表

namespaceToImport和tableToImport均为空，导入所有的namespace和table

```java
private void importNameSpaceAndTable() throws Exception {

    NamespaceDescriptor[] namespaceDescriptors = hbaseAdmin.listNamespaceDescriptors();

    if (ArrayUtils.isNotEmpty(namespaceDescriptors)) {
        for (NamespaceDescriptor namespaceDescriptor : namespaceDescriptors) {
            String namespace = namespaceDescriptor.getName();
            importNameSpace(namespace);
        }
    }

    TableDescriptor[] htds = hbaseAdmin.listTables();
    if (ArrayUtils.isNotEmpty(htds)) {
        for (TableDescriptor htd : htds) {
            String tableName = htd.getTableName().getNameAsString();
            importTable(tableName);
        }
    }
}
```

## 导入指定的命名空间

namespaceToImport不为空，导入指定的namespace和namespace下的table

```java
private void importNameSpaceWithTable(String namespaceToImport, String tableToImport) throws Exception {
    importNameSpace(namespaceToImport);

    List<TableDescriptor> hTableDescriptors = new ArrayList<>();

    if (StringUtils.isEmpty(tableToImport)) {
// 导入指定namespace
        List<NamespaceDescriptor> matchingNameSpaceDescriptors = getMatchingNameSpaces(namespaceToImport);
        if (CollectionUtils.isNotEmpty(matchingNameSpaceDescriptors)) {
            hTableDescriptors = getTableDescriptors(matchingNameSpaceDescriptors);
        }
    } else {
        tableToImport = namespaceToImport +":" + tableToImport;
        TableDescriptor[] htds = hbaseAdmin.listTables(Pattern.compile(tableToImport));
        hTableDescriptors.addAll(Arrays.asList(htds));
    }

    if (CollectionUtils.isNotEmpty(hTableDescriptors)) {
        for (TableDescriptor htd : hTableDescriptors) {
            String tblName = htd.getTableName().getNameAsString();
            importTable(tblName);
        }
    }
}
```

## 导入指定的表

tableToImport不为空，导入指定的table和table的命名空间。importTable会处理表、列族的实体，没有处理列

```java
public void importTable(final String tableName) throws Exception {
    String            tableNameStr = null;
    TableDescriptor[] htds         = hbaseAdmin.listTables(Pattern.compile(tableName));

    if (ArrayUtils.isNotEmpty(htds)) {
        for (TableDescriptor htd : htds) {
            String tblNameWithNameSpace    = htd.getTableName().getNameWithNamespaceInclAsString();
            String tblNameWithOutNameSpace = htd.getTableName().getNameAsString();

            if (tableName.equals(tblNameWithNameSpace)) {
                tableNameStr = tblNameWithNameSpace;
            } else if (tableName.equals(tblNameWithOutNameSpace)) {
                tableNameStr = tblNameWithOutNameSpace;
            } else {
                // when wild cards are used in table name
                if (tblNameWithNameSpace != null) {
                    tableNameStr = tblNameWithNameSpace;
                } else if (tblNameWithOutNameSpace != null) {
                    tableNameStr = tblNameWithOutNameSpace;
                }
            }

            byte[]                 nsByte       = htd.getTableName().getNamespace();
            String                 nsName       = new String(nsByte);
            NamespaceDescriptor    nsDescriptor = hbaseAdmin.getNamespaceDescriptor(nsName);
            AtlasEntityWithExtInfo entity       = createOrUpdateNameSpace(nsDescriptor);
            ColumnFamilyDescriptor[]    hcdts        = htd.getColumnFamilies();
// 处理表、列族，没有处理列
            createOrUpdateTable(nsName, tableNameStr, entity.getEntity(), htd, hcdts);
        }
    } else {
        throw new AtlasHookException("No Table found for the given criteria. Table = " + tableName);
    }
}
```

createOrUpdateTable处理表、列族列实体，比较简单这里就不详细描述

### 导入命名空间

```java
public void importNameSpace(final String nameSpace) throws Exception {
    List<NamespaceDescriptor> matchingNameSpaceDescriptors = getMatchingNameSpaces(nameSpace);

    if (CollectionUtils.isNotEmpty(matchingNameSpaceDescriptors)) {
        for (NamespaceDescriptor namespaceDescriptor : matchingNameSpaceDescriptors) {
            createOrUpdateNameSpace(namespaceDescriptor);
        }
    } else {
        throw new AtlasHookException("No NameSpace found for the given criteria. NameSpace = " + nameSpace);
    }
}
```

createOrUpdateNameSpace处理命名空间实体，比较简单这里就不详细描述