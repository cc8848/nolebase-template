- [Apache-atlas-kafka-hook-源码分析_风的心愿的博客-CSDN博客_apache atlas 源码](https://blog.csdn.net/windydreams/article/details/127813483)

# 流程图

KafkaBridge导入kafka比较简单， 主要是导入kafka的topic名称，流程图大致如下：
![流程图](https://img-blog.csdnimg.cn/39e34d662d9c4d9aa3aa95cc75a9558d.png#pic_center)

# 源码梳理

## KafkaBridge.main方法

```java
在这里插入代码片
public static void main(String[] args) {

    int exitCode = EXIT_CODE_FAILED;
    AtlasClientV2 atlasClientV2 = null;

    try {

// 获取命令行参数
        Options options = new Options();
// 指定topic
        options.addOption("t","topic", true, "topic");
// 指定文件（包含主题名，每行一个主题）
        options.addOption("f", "filename", true, "filename");

        CommandLineParser parser        = new BasicParser();
        CommandLine       cmd           = parser.parse(options, args);
        String            topicToImport = cmd.getOptionValue("t");
        String            fileToImport  = cmd.getOptionValue("f");                 // 导入文件？
// 获取atlas-application.properties文件
        Configuration     atlasConf     = ApplicationProperties.get();           
 
        String[]          urls          = atlasConf.getStringArray(ATLAS_ENDPOINT);// ATLAS API接口地址
        if (urls == null || urls.length == 0) {
            urls = new String[] { DEFAULT_ATLAS_URL };
        }

        if (!AuthenticationUtil.isKerberosAuthenticationEnabled()) {
 // 接收控制台的输入
            String[] basicAuthUsernamePassword = AuthenticationUtil.getBasicAuthenticationInput();
            atlasClientV2 = new AtlasClientV2(urls, basicAuthUsernamePassword);
        } else {
            UserGroupInformation ugi = UserGroupInformation.getCurrentUser();


            atlasClientV2 = new AtlasClientV2(ugi, ugi.getShortUserName(), urls);
        }

// 执行导入
        KafkaBridge importer = new KafkaBridge(atlasConf, atlasClientV2);

        if (StringUtils.isNotEmpty(fileToImport)) {
            File f = new File(fileToImport);

            if (f.exists() && f.canRead()) {

                BufferedReader br   = new BufferedReader(new FileReader(f));
                String         line = null;

                while((line = br.readLine()) != null) {
                    topicToImport = line.trim();
                    // 导入命令行指定文件包含的topic 按行进行分割
                    importer.importTopic(topicToImport);
                }
                exitCode = EXIT_CODE_SUCCESS;

            } else {
                LOG.error("Failed to read the file");
            }
        } else {
            // 导入命令行指定的topic
            importer.importTopic(topicToImport);
            exitCode = EXIT_CODE_SUCCESS;
        }

    } catch(ParseException e) {
        LOG.error("Failed to parse arguments. Error: ", e.getMessage());
        printUsage();
    } catch(Exception e) {
        System.out.println("ImportKafkaEntities failed. Please check the log file for the detailed error message");
        e.printStackTrace();
        LOG.error("ImportKafkaEntities failed", e);
    } finally {
        if (atlasClientV2 != null) {
            atlasClientV2.close();
        }
    }

    System.exit(exitCode);
}
```

## KafkaBridge构造函数， 进行初始化

```java
public KafkaBridge(Configuration atlasConf, AtlasClientV2 atlasClientV2) throws Exception {

    String   zookeeperConnect    = getZKConnection(atlasConf);
    int      sessionTimeOutMs    = atlasConf.getInt(ZOOKEEPER_SESSION_TIMEOUT_MS, DEFAULT_ZOOKEEPER_SESSION_TIMEOUT_MS) ;
    int      connectionTimeOutMs = atlasConf.getInt(ZOOKEEPER_CONNECTION_TIMEOUT_MS, DEFAULT_ZOOKEEPER_CONNECTION_TIMEOUT_MS);
// 创建zk 从zk中获取元数据
    ZkClient zkClient            = new ZkClient(zookeeperConnect, sessionTimeOutMs, connectionTimeOutMs, ZKStringSerializer$.MODULE$);

    this.atlasClientV2     = atlasClientV2;
    this.metadataNamespace = getMetadataNamespace(atlasConf);
    this.zkUtils           = new ZkUtils(zkClient, new ZkConnection(zookeeperConnect), JaasUtils.isZkSecurityEnabled());
// 获取所有的主题
    this.availableTopics   = scala.collection.JavaConversions.seqAsJavaList(zkUtils.getAllTopics());
}
```

## KafkaBridge.importTopic

```java
public void importTopic(String topicToImport) throws Exception {
    List<String> topics = availableTopics;

// topicToImport 不为空， 导入知道的topic, 否则全部导入
    if (StringUtils.isNotEmpty(topicToImport)) {
        List<String> topics_subset = new ArrayList<>();
        for(String topic : topics) {
            if (Pattern.compile(topicToImport).matcher(topic).matches()) {
                topics_subset.add(topic);
            }
        }
        topics = topics_subset;
    }

// 依次导入元数据
    if (CollectionUtils.isNotEmpty(topics)) {
        for(String topic : topics) {
            createOrUpdateTopic(topic);
        }
    }
}
```

## KafkaBridge.createOrUpdateTopic

```java
AtlasEntityWithExtInfo createOrUpdateTopic(String topic) throws Exception {

//主题限定名（唯一） 
    String                 topicQualifiedName = getTopicQualifiedName(metadataNamespace, topic);

// 查询
    AtlasEntityWithExtInfo topicEntity        = findTopicEntityInAtlas(topicQualifiedName);

    if (topicEntity == null) {
// 新增
        System.out.println("Adding Kafka topic " + topic);
        LOG.info("Importing Kafka topic: {}", topicQualifiedName);

        AtlasEntity entity = getTopicEntity(topic, null);
        topicEntity = createEntityInAtlas(new AtlasEntityWithExtInfo(entity));
    } else {
// 修改
        System.out.println("Updating Kafka topic "  + topic);
        LOG.info("Kafka topic {} already exists in Atlas. Updating it..", topicQualifiedName);

        AtlasEntity entity = getTopicEntity(topic, topicEntity.getEntity());
        topicEntity.setEntity(entity);

        topicEntity = updateEntityInAtlas(topicEntity);
    }

    return topicEntity;
}
```

kafka 元数据采集进入后，可以对元数据进行治理， 比如分类和关联术语
![分类、关联术语](https://img-blog.csdnimg.cn/801ed9fff43b4a418e3767f89d1c6390.png)

# 思考

kafka 元数据采集的几点思考

1）考虑到业务系统割裂的情况，不同的系统可能使用不同的kafka集群，因此需要采集不同集群的数据

2）目前atlas提供采集的元数据是主题名称，是否可以将主题消息的定义、主题的生产端、消费端、集群等纳入元数据的范畴？比如定义如下主题：

kafka_topic（已有)

kafka_topic_def

kafka_zk(zookeeper 集群 另外一种数据源)

kafka_borker

kafka_producer

kafka_consumer