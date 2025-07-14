**思路介绍：**

​    记录操作日志的操作前数据是需要思考的重点。我们以修改场景来作为探讨。当我们要完全记录数据的流向的时候，我们必然要记录修改前的数据，而前台进行提交的时候，只有修改的数据，那么如何找到修改前的数据呢。有三个大的要素，我们需要知道修改前数据的表名，表的字段主键，表主键的值。这样通过这三个属性，我们可以很容易的拼出 select * from 表名 where 主键字段 = 主键值。我们就获得了修改前的数据，转换为json之后就可以存入到数据库中了。如何获取三个属性就是重中之重了。我们采取的方案是通过mybati拦截器拦截执行的sql，根据 Java 的反射取到值。



## 1.pom文件

​    本次是基于idea创建spring boot 项目，不会的同学请自行百度，本次就不在这里赘述。



```java
  <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.6</version>
        </dependency>

        <!--spring依赖-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.46</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjrt</artifactId>
            <version>1.9.2</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.9.2</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.69</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-collections4</artifactId>
            <version>4.1</version>
        </dependency>
```

# 2.mybatis拦截器获取sql



- 像springmvc一样，mybatis也提供了拦截器实现，只是他们拦截的对象不同。
- mybatis给Executor、StatementHandler、ResultSetHandler、ParameterHandler提供了拦截器功能，
  - Executor提供了增删改查的接口.
  - StatementHandler负责处理Mybatis与JDBC之间Statement的交互.
  - ResultSetHandler负责处理Statement执行后产生的结果集，生成结果列表.
  - ParameterHandler是Mybatis实现Sql入参设置的对象。



```java
import com.alibaba.fastjson.JSON;
import com.example.recordlog.tools.SqlTools;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.Configuration;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.Map;
import java.util.Properties;


@Component
@Intercepts({@Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})})
public class SqlInterceptor implements Interceptor {


    private static final Logger log = LoggerFactory.getLogger(SqlInterceptor.class);


    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        // 获取拦截方法的参数
        Object[] args = invocation.getArgs();
        MappedStatement mappedStatement = (MappedStatement) args[0];
        Configuration configuration = mappedStatement.getConfiguration();
        Object parameter = null;
        if (invocation.getArgs().length > 1) {
            parameter = invocation.getArgs()[1];
        }

        BoundSql boundSql = mappedStatement.getBoundSql(parameter);
 
        // 执行结果
        Object returnValue = invocation.proceed();

        // 获取sql语句
        String sql = SqlTools.showSql(configuration, boundSql);
        System.out.println("sql : " + sql);
      
        return returnValue;
    }


    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }

}
```



# **获取操作类型**

​    为什么要获取操作类型，因为只有在update或delete操作的时候才会有修改前的数据，所以要先获取操作类型，只有操作类型为update或delete操作的时候才会取查询修改前的数据。

```java
 /**
     * 判断sql操作类型 ：update和delete的时候才需要去查询修改前的值
     *
     * @param mappedStatement
     * @return java.lang.String
     */
    public static String SqlCommandType(MappedStatement mappedStatement) {

        if (SqlCommandType.SELECT.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.SELECT;
        }

        if (SqlCommandType.INSERT.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.INSERT;
        }

        if (SqlCommandType.UPDATE.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.UPDATE;
        }
        if (SqlCommandType.DELETE.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.DELETE;
        }
        return null;
    }
```

# 操作类型常量

```java
/**
 * @author liuminglin
 * @desc: 操作常量
 */
public class OperateTypeConstant {
    public static final String SELECT = "select";
    public static final String INSERT = "insert";
    public static final String UPDATE = "update";
    public static final String DELETE = "delete";
    public static final String FLUSH = "flush";

}
```

# 动态获取表名

​    为什么要动态获取表名? 我们需要知道修改前数据的表名,我这里是通过mybatis拦截器拿到sql,通过截取的方式获取表名，暂时没有更好方法获取表名，如有请告知

 截取规则 ：
   1.替换sql命令类型标签（select delete insert update）
   2.替换掉from标签，去除前后空格
   3.截取从0开始，\n结束

```java
 /**
     * 获取表名
     * 截取规则 ：
     * 1.替换sql命令类型标签（select delete insert update）
     * 2.替换掉from标签，去除前后空格
     * 3.截取从0开始，\n结束
     */
    public static String getTableName(BoundSql boundSql, String sqlType) {

        if (StringUtils.isBlank(sqlType) || StringUtils.isBlank(boundSql.getSql())) {
            return "";
        }


        //根据sqlType 去除update标签，delete标签，from标签，去除前后空格
        String tableStr = boundSql.getSql().replaceAll(sqlType, "").trim().replaceAll("from", "");
        String tableName = tableStr.substring(0, tableStr.indexOf("\n"));
        return tableName;
    }
```

![img](https://img-blog.csdnimg.cn/2021080216180188.png)



# 动态获取字段名

​    为什么要获取字段名? 我们需要知道修改前数据表的字段名（也就是主键名称）,我这里是通过mybatis拦截器拿到sql,通过截取的方式获取字段名，

截取规则 ：
   1 去掉  \n   ？ =
   2.从where索引位置开始截取, 到sqlStr最后一个位置 ,替换掉where操作符，去除空格
 

```java
 public static String getFieldName(BoundSql boundSql, String sqlType) {

        if (StringUtils.isBlank(sqlType) || StringUtils.isBlank(boundSql.getSql())) {
            return "";
        }
        log.info("getSql :{}", JSON.toJSONString(boundSql.getSql()));

        String sqlStr = boundSql.getSql().replaceAll("\n", "").replaceAll("\\?", "").replaceAll("=", "");
        String fieldName = sqlStr.substring(sqlStr.indexOf("where"), sqlStr.length()).replace("where", "").trim();


        return fieldName;
    }
```

![img](https://img-blog.csdnimg.cn/20210802164129592.png)

### 批量解析

思路，批量修改和删除会导致sql不一样，所以首先批量解析之前要先判断是否是批量，如果是参数批量就需要单独解析，如何判断是批量呢？

如果是批量参数，mapper接口会使用 @parm注解，参数是list，mapper.xml文件中要使用foreach标签进行遍历，所以我们要判断拦截的参数是否包含parm注解就可以了。

（1）用@Param注解标记list参数

```java
mapper接口@Param标记
Integer batchUpdateInfo( @Param("userInfoList") List<UserInfo> userInfoList);
```

（2）mapper.xml使用foreach标签遍历

```XML
<!-- mapper.xml使用foreach标签遍历   -->
    <update id="batchUpdateInfo" parameterType="java.util.List">
        <foreach collection="userInfoList" item="item" index="index" open="" close="" separator=";">
            update user_info
            <set>
                <if test="item.userId != null ">
                    user_Id = #{item.userId,jdbcType=VARCHAR},
                </if>
                <if test="item.userName != null">
                    user_name = #{item.userName,jdbcType=VARCHAR},
                </if>

            </set>
            where id = #{item.id,jdbcType=BIGINT}
        </foreach>
    </update>
```

（3）获取批量标记

```java
 /**
     * 获取批次标记
     *
     * @param parameter
     * @return boolen
     */
    private static boolean getbatchFlag(Object parameter) {

        if (null != parameter && parameter.toString().contains("param")) {
            return true;
        }
        return false;
    }
```

（4）批量解析：

​    批量解析只需要在原有的方法是判断是否批量，如果是批量的话进一步解析，batchFlag=true代表批量，需要从新解析（截取fieldName,从beginIndex=0开始截取，endIndex=;结束

```java
 /**
     * 获取sql获取FieldName
     * 截取规则 ：
     * （1）去掉\n ,？，=
     * （2）从where索引位置开始截取, 到sqlStr最后一个位置 ,替换掉where操作符，去除空格
     * (3) batchFlag=true代表批量，需要从新解析（截取fieldName,从beginIndex=0开始截取，endIndex=;结束
     */
    private static String getFieldName(BoundSql boundSql, String sqlType, boolean batchFlag) {

        if (StringUtils.isBlank(sqlType) || StringUtils.isBlank(boundSql.getSql())) {
            return "";
        }

        String sqlStr = boundSql.getSql().replaceAll("\n", "").replaceAll("\\?", "").replaceAll("=", "");
        String fieldName = sqlStr.substring(sqlStr.indexOf("where"), sqlStr.length()).replace("where", "").trim();

        //解析批量
        if (batchFlag) {
            fieldName = fieldName.substring(0, fieldName.indexOf(";")).trim();
        }
        return fieldName;
    }
```



# 反射动态获取属性值

**思路介绍：**

​    **第一种实现方式，调用对象的get方法获取**

​    **第二种实现方式，通过反射动态获取**

```java
 /**
     * 根据属性名获取属性值
     */
    public static Object getFieldValueByName(String fieldName, Object o) {

        if (StringUtils.isBlank(fieldName) || o == null) {
            return null;
        }

        try {
            String firstLetter = fieldName.substring(0, 1).toUpperCase();
            String getter = "get" + firstLetter + fieldName.substring(1);
            Method method = o.getClass().getMethod(getter, new Class[]{});
            Object value = method.invoke(o, new Object[]{});
            return value;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }

    /**
     * 根据属性名获取属性值
     *
     * @param fieldName
     * @param object
     * @return
     */
    public static Object getFieldValueByFieldName(String fieldName, Object object) {
        try {
            Field field = object.getClass().getDeclaredField(fieldName);
            //设置对象的访问权限，保证对private的属性的访问
            field.setAccessible(true);
            return  field.get(object);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }
```

![img](https://img-blog.csdnimg.cn/20210802172611275.png)

### 批量动态解析属性值

​    原来是单个参数，直接通过属性名获取属性值就可以了,批量是多个参数，需要单独解析

上面已经拿到批量标识，batchFlag=true走批量解析逻辑 ,batchFlag=false走单个参数解析，

```
判断field.getName()的值==fieldName值相等，获取主键值，这样就拿到了所有的主键。
/**
     * 反射获取  @Param注解标记list属性值
     *
     * @param primarykeyName 主键名称
     * @param object         对象
     * @param batchFlag      batchFlag=true批量 ,batchFlag=false代表不是批量操作
     * @return
     */
    private static List<Map> reflectGetListValue(String primarykeyName, Object object, boolean batchFlag) {

        List<Map> resultList = new ArrayList<>();
        if (null == object) {
            return resultList;
        }

        //单个参数解析
        Map argsValueMap = new HashMap();
        if (!batchFlag) {
            Map argsMaps = beanToMap(object);
            for (Object obj : argsMaps.keySet()) {
                if (obj.equals(primarykeyName)) {
                    argsValueMap.put("obj", argsMaps.get(obj));
                }
            }
            resultList.add(argsValueMap);
            return resultList;
        }

        //批量解析
        Map<String, Object> parameterMap = (Map) object;
        for (Map.Entry<String, Object> entery : parameterMap.entrySet()) {
            String key = entery.getKey();
            if (key.indexOf("param") < 0) {
                List<Object> objList = (List) entery.getValue();
                if (objList != null) {
                    for (Object obj : objList) {
                        Map<String, Object> listChild = new HashMap<>();
                        Class<?> clz = obj.getClass();
                        Field[] declaredFields = clz.getDeclaredFields();
                        for (int i = 0; i < declaredFields.length; i++) {
                            Field field = declaredFields[i];
                            field.setAccessible(true);
                            try {
                                //Object value = field.get(obj);
                                if (StringUtils.equals(primarykeyName, field.getName())) {
                                    Object value = field.get(obj);
                                    listChild.put(field.getName(), value);
                                }
                            } catch (IllegalAccessException e) {
                                e.printStackTrace();
                            }
                        }
                        resultList.add(listChild);
                    }
                }
            }
        }
        return resultList;
    }
```



# 查询修改前的数据

​    有了上面的4个属性，接下来的一步就是对修改前或删除前的数据进行拼接sql。之前我们提到过如果是修改和删除，我们才会进行数据的拼接获取，最终去查询修改前的数据。

通过 jdbcTemplate.queryForList方法查询修改前的数据

```java
jdbcTemplate.queryForList(sql, argsValue);
```

修改前数据

![img](https://img-blog.csdnimg.cn/20210802174248138.png)

![img](https://img-blog.csdnimg.cn/20210802174605450.png)

批量查询修改前的数据

​    批量怎么查询呢，上面我们已经拿到所有的主键，直接遍历主键列表，查询修改前的值

```java
List<Map> primarykeyList = SqlTools.reflectGetListValue(primarykeyName, parameter, batchFlag);
        //遍历主键列表
        for (Map map : primarykeyList) {
            for (Object obj : map.keySet()) {
                //获取主键值
                Object primarykeyValue = map.get(obj);
                queryData(jdbcTemplate, tableName, primarykeyName, primarykeyValue, resultList);
            }
        }
```



## 2.拦截器的完整实现代码

```java
import com.alibaba.fastjson.JSON;
import com.example.recordlog.tools.SqlTools;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Properties;


@Component
@Intercepts({@Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class}),
        @Signature(type = Executor.class, method = "query", args = {MappedStatement.class,
                Object.class, RowBounds.class, ResultHandler.class})})
public class MybatisInterceptor implements Interceptor {


    private static final Logger log = LoggerFactory.getLogger(MybatisInterceptor.class);

    @Autowired
    private JdbcTemplate jdbcTemplate;


    @Override
    public Object intercept(Invocation invocation) throws Throwable {

        // 获取拦截方法的参数
        Object[] args = invocation.getArgs();
        MappedStatement mappedStatement = (MappedStatement) args[0];
        Configuration configuration = mappedStatement.getConfiguration();
        Object parameter = null;
        if (invocation.getArgs().length > 1) {
            parameter = invocation.getArgs()[1];
        }

        // 获取到节点的id,即sql语句的id
//        String sqlId = mappedStatement.getId();
//        String namespace = sqlId.substring(sqlId.indexOf("c"), sqlId.lastIndexOf("."));
//        MappedStatement mappedStatementa = configuration.getMappedStatement(namespace + "." + "selectByPrimaryKey");
        
        BoundSql boundSql = mappedStatement.getBoundSql(parameter);
        //获取修改器前的值
        List resultMaps = SqlTools.getDataBeforeOperate(mappedStatement, boundSql, parameter, jdbcTemplate);
        // 执行结果
        Object returnValue = invocation.proceed();
        // 获取sql语句
        String sql = SqlTools.showSql(configuration, boundSql);
        log.info("Sql :{}", JSON.toJSONString(sql));
        log.info("修改前数据 :{}", JSON.toJSONString(resultMaps));
        log.info("修改后数据 :{}", JSON.toJSONString(parameter));
        return returnValue;
    }


    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
```

## 3.SqlTools工具类

```java
import com.example.recordlog.constant.OperateTypeConstant;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.type.TypeHandlerRegistry;
import org.springframework.cglib.beans.BeanMap;
import org.springframework.jdbc.core.JdbcTemplate;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.text.DateFormat;
import java.util.*;

@Slf4j
public class SqlTools {


    private SqlTools() {
    }


    /**
     * 解析BoundSql，生成不含占位符的SQL语句
     *
     * @param configuration
     * @param boundSql
     * @return java.lang.String
     */
    public static String showSql(Configuration configuration, BoundSql boundSql) {
        Object parameterObject = boundSql.getParameterObject();
        // 获取方法参数
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        // 格式化SQL
        String sql = boundSql.getSql().replaceAll("[\\s]+", " ");
        if (parameterMappings.size() > 0 && parameterObject != null) {
            // 类型解析器
            TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
            if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                sql = sql.replaceFirst("\\?", getParameterValue(parameterObject));
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                for (ParameterMapping parameterMapping : parameterMappings) {
                    String propertyName = parameterMapping.getProperty();
                    String[] s = metaObject.getObjectWrapper().getGetterNames();
                    Arrays.toString(s);
                    if (metaObject.hasGetter(propertyName)) {
                        Object obj = metaObject.getValue(propertyName);
                        sql = sql.replaceFirst("\\?", getParameterValue(obj));
                    } else if (boundSql.hasAdditionalParameter(propertyName)) {
                        Object obj = boundSql.getAdditionalParameter(propertyName);
                        sql = sql.replaceFirst("\\?", getParameterValue(obj));
                    }
                }
            }
        }
        return sql;
    }

    /**
     * 若为字符串或者日期类型，则在参数两边添加''
     *
     * @param obj
     * @return java.lang.String
     */
    private static String getParameterValue(Object obj) {
        String value;
        if (obj instanceof String) {
            value = "'" + obj.toString() + "'";
        } else if (obj instanceof Date) {
            DateFormat formatter = DateFormat.getDateTimeInstance(DateFormat.DEFAULT, DateFormat.DEFAULT, Locale.CHINA);
            value = "'" + formatter.format(new Date()) + "'";
        } else {
            if (obj != null) {
                value = obj.toString();
            } else {
                value = "";
            }
        }
        return value;
    }


    /**
     * @param operateType 操作类型
     * @param tableName   表名
     * @param fieldName   参数
     * @param argsValue   参数值
     * @return
     * @desc 操作前获取数据(修改或者删除的时候需要查询操作前的数据)
     */
    public static List getDataBeforeOperate(MappedStatement mappedStatement, BoundSql boundSql, Object parameter, JdbcTemplate jdbcTemplate) {

        List resultList = new ArrayList();

        //SELECT和INSERT操作不用记录修改前的值，直接返回
        if (parameter == null || SqlCommandType.SELECT.equals(mappedStatement.getSqlCommandType())
                || SqlCommandType.INSERT.equals(mappedStatement.getSqlCommandType())) {
            return resultList;

        }

        String operateType = SqlTools.SqlCommandType(mappedStatement);
        boolean batchFlag = SqlTools.getbatchFlag(parameter);
        String tableName = SqlTools.getTableName(boundSql, operateType);
        String primarykeyName = SqlTools.getFieldName(boundSql, operateType, batchFlag);
        // Object argsValue = SqlTools.getFieldValueByFieldName(fieldName, object);

        List<Map> primarykeyList = SqlTools.reflectGetListValue(primarykeyName, parameter, batchFlag);
        //遍历主键列表
        for (Map map : primarykeyList) {
            for (Object obj : map.keySet()) {
                //获取主键值
                Object primarykeyValue = map.get(obj);
                queryData(jdbcTemplate, tableName, primarykeyName, primarykeyValue, resultList);
            }
        }
        return resultList;
    }


    /**
     * 判断sql操作类型 ：update和delete的时候才需要去查询修改前的值
     *
     * @param mappedStatement
     * @return java.lang.String
     */
    private static String SqlCommandType(MappedStatement mappedStatement) {

        if (SqlCommandType.SELECT.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.SELECT;
        }

        if (SqlCommandType.INSERT.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.INSERT;
        }

        if (SqlCommandType.UPDATE.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.UPDATE;
        }
        if (SqlCommandType.DELETE.equals(mappedStatement.getSqlCommandType())) {
            return OperateTypeConstant.DELETE;
        }
        return null;
    }

    /**
     * 获取批次标记
     *
     * @param parameter
     * @return boolen
     */
    private static boolean getbatchFlag(Object parameter) {

        if (null != parameter && parameter.toString().contains("param")) {
            return true;
        }
        return false;
    }

    /**
     * 获取表名
     * 截取规则 ：
     * 1.替换sql命令类型标签（select delete insert update）
     * 2.替换掉from标签
     * 3.截取从0开始，\n结束
     */
    private static String getTableName(BoundSql boundSql, String sqlType) {

        if (StringUtils.isBlank(sqlType) || StringUtils.isBlank(boundSql.getSql())) {
            return "";
        }

        //根据sqlType 去除update标签，delete标签，from标签，去除前后空格
        String tableStr = boundSql.getSql().replaceAll(sqlType, "").trim().replaceAll("from", "");
        String tableName = tableStr.substring(0, tableStr.indexOf("\n"));

        return tableName;
    }


    /**
     * 获取sql获取FieldName
     * 截取规则 ：
     * （1）去掉\n ,？，=
     * （2）从where索引位置开始截取, 到sqlStr最后一个位置 ,替换掉where操作符，去除空格
     * (3) batchFlag=true代表批量，需要从新解析（截取fieldName,从beginIndex=0开始截取，endIndex=;结束
     */
    private static String getFieldName(BoundSql boundSql, String sqlType, boolean batchFlag) {

        if (StringUtils.isBlank(sqlType) || StringUtils.isBlank(boundSql.getSql())) {
            return "";
        }

        String sqlStr = boundSql.getSql().replaceAll("\n", "").replaceAll("\\?", "").replaceAll("=", "");
        String fieldName = sqlStr.substring(sqlStr.indexOf("where"), sqlStr.length()).replace("where", "").trim();

        //解析批量
        if (batchFlag) {
            fieldName = fieldName.substring(0, fieldName.indexOf(";")).trim();
        }
        return fieldName;
    }

    /**
     * 根据属性名获取属性值
     */
    private static Object getFieldValueByName(String fieldName, Object o) {

        if (StringUtils.isBlank(fieldName) || o == null) {
            return null;
        }

        try {
            String firstLetter = fieldName.substring(0, 1).toUpperCase();
            String getter = "get" + firstLetter + fieldName.substring(1);
            Method method = o.getClass().getMethod(getter, new Class[]{});
            Object value = method.invoke(o, new Object[]{});
            return value;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }

    /**
     * 根据属性名获取属性值
     *
     * @param fieldName
     * @param object
     * @return
     */
    private static Object getFieldValueByFieldName(String fieldName, Object object) {
        try {
            Field field = object.getClass().getDeclaredField(fieldName);
            //设置对象的访问权限，保证对private的属性的访问
            field.setAccessible(true);
            return field.get(object);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return null;
        }
    }


    /**
     * 反射获取  @Param注解标记list属性值
     *
     * @param primarykeyName 主键名称
     * @param object         对象
     * @param batchFlag      batchFlag=true批量 ,batchFlag=false代表不是批量操作
     * @return
     */
    private static List<Map> reflectGetListValue(String primarykeyName, Object object, boolean batchFlag) {

        List<Map> resultList = new ArrayList<>();
        if (null == object) {
            return resultList;
        }

        //单个参数解析
        Map argsValueMap = new HashMap();
        if (!batchFlag) {
            Map argsMaps = beanToMap(object);
            for (Object obj : argsMaps.keySet()) {
                if (obj.equals(primarykeyName)) {
                    argsValueMap.put("obj", argsMaps.get(obj));
                }
            }
            resultList.add(argsValueMap);
            return resultList;
        }

        //批量解析
        Map<String, Object> parameterMap = (Map) object;
        for (Map.Entry<String, Object> entery : parameterMap.entrySet()) {
            String key = entery.getKey();
            if (key.indexOf("param") < 0) {
                List<Object> objList = (List) entery.getValue();
                if (objList != null) {
                    for (Object obj : objList) {
                        Map<String, Object> listChild = new HashMap<>();
                        Class<?> clz = obj.getClass();
                        Field[] declaredFields = clz.getDeclaredFields();
                        for (int i = 0; i < declaredFields.length; i++) {
                            Field field = declaredFields[i];
                            field.setAccessible(true);
                            try {
                                //Object value = field.get(obj);
                                if (StringUtils.equals(primarykeyName, field.getName())) {
                                    Object value = field.get(obj);
                                    listChild.put(field.getName(), value);
                                }
                            } catch (IllegalAccessException e) {
                                e.printStackTrace();
                            }
                        }
                        resultList.add(listChild);
                    }
                }
            }
        }
        return resultList;
    }


    /**
     * bean转map
     *
     * @param bean
     * @return map
     */
    private static <T> Map<String, Object> beanToMap(T bean) {

        Map<String, Object> map = new HashMap<>();
        if (bean != null) {
            BeanMap beanMap = BeanMap.create(bean);
            for (Object key : beanMap.keySet()) {
                map.put(String.valueOf(key), beanMap.get(key));
            }
        }
        return map;
    }

    /**
     * @param jdbcTemplate    操作类型
     * @param tableName       表名
     * @param primarykeyName  主键名称
     * @param primarykeyValue 主键值
     * @param resultList      查询结果集
     * @return
     * @desc 操作前获取数据(修改或者删除的时候需要查询操作前的数据)
     */
    private static void queryData(JdbcTemplate jdbcTemplate, String tableName, String primarykeyName, Object primarykeyValue, List resultList) {

        if (StringUtils.isNotEmpty(tableName) && StringUtils.isNotEmpty(primarykeyName) && primarykeyValue != null) {
            StringBuffer sb = new StringBuffer();
            sb.append(" select * from ");
            sb.append(tableName);
            sb.append(" where ");
            sb.append(primarykeyName);
            sb.append(" = ? ");
            String sql = sb.toString();
            List<Map<String, Object>> resultMaps = jdbcTemplate.queryForList(sql, primarykeyValue);
            resultList.add(resultMaps);
        }
    }
}
```



## 4.SpringContextUtil工具类

```java
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import javax.servlet.http.HttpServletRequest;
import java.util.Objects;

/**
 * @author liuminglin
 * @date 2021/6/22 10:48
 * @description: TODO
 */

@Component
public class SpringContextUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    /**
     * 实现ApplicationContextAware接口的回调方法，设置上下文环境
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        SpringContextUtil.applicationContext = applicationContext;
    }

    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    /**
     * 获取对象
     *
     * @return Object 一个以所给名字注册的bean的实例 (service注解方式，自动生成以首字母小写的类名为bean name)
     */
    public static Object getBean(String name) {
        return applicationContext.getBean(name);
    }

    /**
     * 获取对象通过Class
     *
     * @param cls
     * @return Object
     * @throws BeansException
     */
    public static <C> Object getBean(Class<C> cls) {
        return applicationContext.getBean(cls);
    }

    //获取HttpServletRequest请求
    public static HttpServletRequest getHttpServletRequest() {
        return ((ServletRequestAttributes) Objects.requireNonNull(RequestContextHolder.getRequestAttributes())).getRequest();
    }

}
```



## 5.RestResponse工具类（REST接口统一返回数据工具类封装）



```java
import java.io.Serializable;

/**
 * REST接口统一返回数据工具类封装RestResponse
 */
public class RestResponse<T> implements Serializable {

    private static final long serialVersionUID = 3728877563912075885L;

    private int code;
    private String msg;
    private T data;


    public int getCode() {
        return code;
    }

    public String getMsg() {
        return msg;
    }


    public T getData() {
        return data;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public void setData(T data) {
        this.data = data;
    }


    public RestResponse() {

    }

    public RestResponse(int code, String message, T data) {
        this.code = code;
        this.setMsg(message);
        this.data = data;
    }

    public RestResponse(int code, T data) {
        this.code = code;
        this.data = data;
    }

    public RestResponse(int code, String message) {
        this.code = code;
        this.setMsg(message);
    }

    /**
     * 成功时-返回data
     *
     * @param <T>
     * @return
     */
    public static <T> RestResponse<T> success(T data) {
        return new RestResponse<T>(200, null, data);
    }

    /**
     * 成功-不返回data
     *
     * @param <T>
     * @return
     */
    public static <T> RestResponse<T> success(String msg) {
        return new RestResponse<T>(200, msg);
    }

    /**
     * 成功-返回data+msg
     *
     * @param <T>
     * @return
     */
    public static <T> RestResponse<T> success(String msg, T data) {
        return new RestResponse<T>(200, msg, data);
    }

    /**
     * 失败
     *
     * @param <T>
     * @return
     */
    public static <T> RestResponse<T> fail(String msg) {
        return new RestResponse<T>(500, msg, null);
    }

    /**
     * 失败-code
     *
     * @param <T>
     * @return
     */
    public static <T> RestResponse<T> fail(int code, String msg) {
        return new RestResponse<T>(code, msg, null);
    }


    @Override
    public String toString() {
        return "RestResponse{" + "code=" + code + ", msg='" + msg + '\'' + ", data=" + data + '}';
    }


}
```

## 6.Controller调用

```java
import com.example.recordlog.aspect.OperateLog;
import com.example.recordlog.bean.Goods;
import com.example.recordlog.bean.UserInfo;
import com.example.recordlog.constant.OperateTypeConstant;
import com.example.recordlog.mapper.GoodsMapper;
import com.example.recordlog.service.UserInfoService;
import com.example.recordlog.tools.RestResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api")
public class OperateController {

    @Autowired
    private UserInfoService userInfoService;


    @RequestMapping(value = "/updateInfo", produces = "application/json;charset=utf-8", method = {RequestMethod.POST})
    public RestResponse updateInfo(@RequestBody UserInfo info) {
       int count = userInfoService.updateByPrimaryKeySelective(info);
        RestResponse restResponse = RestResponse.success(count);
        return restResponse;
    }


@RequestMapping(value = "/batchUpdateInfo", produces = "application/json;charset=utf-8", method = {RequestMethod.POST})
    public RestResponse batchUpdateInfo(@RequestBody List<UserInfo> userInfoList) {
        int count = userInfoService.batchUpdateInfo(userInfoList);
        RestResponse restResponse = RestResponse.success(count);
        return restResponse;
    }


    @RequestMapping(value = "/removeInfo", produces = "application/json;charset=utf-8", method = {RequestMethod.POST})
    public RestResponse removeInfo(@RequestBody Long id) {
        RestResponse restResponse = RestResponse.success(userInfoService.deleteByPrimaryKey(id));
        return restResponse;
    }
```