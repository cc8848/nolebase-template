- [Druid SQL解析](https://java.springlearn.cn/learn/tools/druid/)
- [基于Druid的HiveSQL血缘解析](https://blog.csdn.net/master_hunter/article/details/127637747)
- [使用Druid的sql parser做一个表数据血缘分析工具](https://www.cnblogs.com/enhe/p/12141686.html)

# 一、Druid SQL Parser

Druid SQL Parser源码中主要的构成框架包括：Parser、AST和Visitor。

## 1.1 Parser

根据之前的研究我们清楚语法分析器(Parser)：将上一步得到的Token流转换为语法定义的树结构。对于HiveSQL的解析来讲，对于其定义的grammar语法文件来看，其各个不同的语法解析文件就是其SQL执行过程的支撑，自然需要先解析获取其对应的语法结构：
![img](https://img-blog.csdnimg.cn/df9e4318a742410cb544aaaafc562abc.png)

From的解析文件可以说是通用的，因此在parser并没有看到关于Hive的From文件，都统一由全局SQLParser获取。

![img](https://img-blog.csdnimg.cn/73c78b714bb94e5588763c89796262b0.png)

这些特定数据库的类都全部由通用parser继承而来，添加新方法。

## 1.2 AST

AST是abstract syntax tree的缩写，也就是抽象语法树。和所有的Parser一样，Druid Parser会生成一个抽象语法树。

之所以说语法是“抽象”的，是因为这里的语法并不会表示出真实语法中出现的每个细节。比如，嵌套括号被隐含在树的结构中，并没有以节点的形式呈现；而类似于if-condition-then这样的条件跳转语句，可以使用带有两个分支的节点来表示。

和抽象语法树相对的是具体语法树。一般的，在源代码的翻译和编译过程中，语法分析器创建出分析树。一旦AST被创建出来，在后续的处理过程中，比如语义分析阶段，会添加一些信息。

![img](https://img-blog.csdnimg.cn/527c80bab30e4afaa4ac9468b591267d.png)

```java
String sql_format=formatMysql(sql_4);
final DbType dbType = JdbcConstants.HIVE;
// SQLStatement就是AST
List<SQLStatement> stmtList = SQLUtils.parseStatements(sql_4, dbType);
System.out.println(stmtList)
```

在Druid中，AST节点类型主要包括SQLObject、SQLExpr、SQLStatement三种抽象类型。

```java
interface SQLObject {}
interface SQLExpr extends SQLObject {}
interface SQLStatement extends SQLObject {}

interface SQLTableSource extends SQLObject {}
class SQLSelect extends SQLObject {}
class SQLSelectQueryBlock extends SQLObject {}
```

## 1.3 Visitor

Visitor是遍历AST的手段，是处理AST最方便的模式，Visitor是一个接口。Druid内置提供了如下Visitor:

- OutputVisitor用来把AST输出为字符串
- WallVisitor 来分析SQL语意来防御SQL注入攻击
- ParameterizedOutputVisitor用来合并未参数化的SQL进行统计
- EvalVisitor 用来对SQL表达式求值
- ExportParameterVisitor用来提取SQL中的变量参数
- SchemaStatVisitor 用来统计SQL中使用的表、字段、过滤条件、排序表达式、分组表达式
- SQL格式化 Druid内置了基于语义的SQL格式化功能

Druid提供了多种默认实现的Visitor，可以满足基本需求，如果默认提供的不满足需求，可自行实现自定义Visitor。也就是利用该功能我们能够快速获取表与字段。

更多详细功能参阅官方对于Visitor的文档：[SQL_Parser_Demo_visitor](https://github.com/alibaba/druid/wiki/SQL_Parser_Demo_visitor)

# 二、语法树解析

## 2.1 核心类介绍

### 2.1.1 SQLStatemment DQL & DML顶级抽象

- DQL 数据查询语言 select
- DML 数据操纵语言 insert update delete

最常用的Statement当然是SELECT/UPDATE/DELETE/INSERT，他们分别是

| 核心类             | 说明     |
| :----------------- | :------- |
| SQLSelectStatement | 查询语句 |
| SQLUpdateStatement | 更新语句 |
| SQLDeleteStatement | 删除语句 |
| SQLInsertStatement | 新增语句 |

```java
@Test
public void statement() {
    // 以下全部 true
    System.out.println(SQLUtils.parseSingleMysqlStatement("select * from users") instanceof SQLSelectStatement);
    System.out.println(SQLUtils.parseSingleMysqlStatement("insert into users(id,name,age) values (1,'孙悟空',500)") instanceof SQLInsertStatement);
    System.out.println(SQLUtils.parseSingleMysqlStatement("update users set name = '唐僧' where id = 1 ") instanceof SQLUpdateStatement);
    System.out.println(SQLUtils.parseSingleMysqlStatement("delete from users where id = 1") instanceof SQLDeleteStatement);
}
```

### 2.1.2 SQLSelect SQL查询

SQLSelectStatement包含一个SQLSelect，SQLSelect包含一个SQLSelectQuery。SQLSelectQuery有主要的两个派生类， 分别是SQLSelectQueryBlock(单表sql查询)和SQLUnionQuery([union查询 (opens new window)](https://www.w3school.com.cn/sql/sql_union.asp))。

```java
/**
     * SQLSelectStatement包含一个SQLSelect，SQLSelect包含一个SQLSelectQuery。SQLSelectQuery有主要的两个派生类，
     * 分别是SQLSelectQueryBlock(单表sql查询)和SQLUnionQuery(联合查询)。
     */
@Test
public void SQLSelectQuery() {
    // true
    System.out.println(parseSQLSelectQuery("select * from users") instanceof SQLSelectQueryBlock);
    // true
    System.out.println(parseSQLSelectQuery("select name from users union select name from school") instanceof SQLUnionQuery);
}

public SQLSelectQuery parseSQLSelectQuery(String sql) {
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement(sql);
    SQLSelectStatement sqlSelectStatement = Utils.cast(sqlStatement, SQLSelectStatement.class);
    SQLSelect select = sqlSelectStatement.getSelect();
    return select.getQuery();
}
```

### 2.1.3 SQLExpr

SQLExpr 是有几个实现类的。

```sql
select id,name,age from users where id = 1 and name = '孙悟空';

select u.id, u.name from users as u where id = 1 and name = ?;
```

| 核心类            | 举例            | 说明                                                         | 适用范围              | 快速记忆       |
| :---------------- | :-------------- | :----------------------------------------------------------- | :-------------------- | :------------- |
| SQLIdentifierExpr | id,name,age     | SQLIdentifierExpr                                            | 查询字段或者where条件 | 唯一标记       |
| SQLPropertyExpr   | u.id,u.name     | 区别于SQLIdentifierExpr,适用于有别名的场景; SQLPropertyExpr.name = id, SQLPropertyExpr.owner = SQLIdentifierExpr = u） | 查询字段或者where条件 | 有别名就是它   |
| SQLBinaryOpExpr   | id = 1, id > 5  | SQLBinaryOpExpr(left = SQLIdentifierExpr = id ,right = SQLValuableExpr = 1) | where条件             | 有操作符就是它 |
| SQLVariantRefExpr | id = ?          | 变量                                                         | where条件             | 有变量符就是它 |
| SQLIntegerExpr    | id = 1          | 数字类型                                                     | 值类型                | -              |
| SQLCharExpr       | name = '孙悟空' | 字符类型                                                     | 值类型                | -              |

#### 2.1.3.1 SQLBinaryOpExpr

```java
/**
     * 操作符相关: SQLBinaryOpExpr
     */
@Test
public void SQLBinaryOpExpr() {
    String sql = "select * from users where id > 1 and age = 18";
    SQLSelectQuery sqlSelectQuery = Utils.parseSQLSelectQuery(sql);
    SQLSelectQueryBlock selectQueryBlock = Utils.cast(sqlSelectQuery, SQLSelectQueryBlock.class);
    SQLExpr where = selectQueryBlock.getWhere();
    List<SQLObject> conditions = where.getChildren();
    // [id > 1 , age = 18] 出现了操作符所以是SQLBinaryOpExpr
    for (SQLObject condition : conditions) {
        SQLBinaryOpExpr conditionExpr = Utils.cast(condition, SQLBinaryOpExpr.class);
        SQLBinaryOperator operator = conditionExpr.getOperator();
        SQLIdentifierExpr conditionColumn = Utils.cast(conditionExpr.getLeft(), SQLIdentifierExpr.class);
        SQLValuableExpr conditionColumnValue = Utils.cast(conditionExpr.getRight(), SQLValuableExpr.class);
        Utils.print("条件字段:{},操作符号:{},条件值:{}", conditionColumn.getName(), operator.name, conditionColumnValue);
    }
}
```

#### 2.1.3.2 SQLVariantRefExpr

```java
@Test
public void SQLVariantRefExpr() {
    String sql = "select * from users where id = ? and name = ?";
    SQLSelectQuery sqlSelectQuery = Utils.parseSQLSelectQuery(sql);
    SQLSelectQueryBlock selectQueryBlock = Utils.cast(sqlSelectQuery, SQLSelectQueryBlock.class);
    SQLExpr where = selectQueryBlock.getWhere();
    List<SQLObject> conditions = where.getChildren();
    // [id = ?] 出现了变量符,所以要用SQLVariantRefExpr
    for (SQLObject condition : conditions) {
        SQLBinaryOpExpr conditionExpr = Utils.cast(condition, SQLBinaryOpExpr.class);
        SQLBinaryOperator operator = conditionExpr.getOperator();
        SQLIdentifierExpr conditionColumn = Utils.cast(conditionExpr.getLeft(), SQLIdentifierExpr.class);
        SQLVariantRefExpr conditionColumnValue = Utils.cast(conditionExpr.getRight(), SQLVariantRefExpr.class);
        int index = conditionColumnValue.getIndex();
        Utils.print("条件字段:{},操作符号:{},索引位:{}", conditionColumn.getName(), operator.name, index);
    }
}
```

### 2.1.4 SQLTableSource

常见的SQLTableSource包括SQLExprTableSource、SQLJoinTableSource、SQLSubqueryTableSource、SQLWithSubqueryClause.Entry

| 核心类                 | 举例                                                         | 说明                                                         | 快速记忆      |
| :--------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- | :------------ |
| SQLExprTableSource     | select * from emp where i = 3                                | name = SQLIdentifierExpr = emp                               | 单表查询      |
| SQLJoinTableSource     | select * from emp e inner join org o on e.org_id = o.id      | left = SQLExprTableSource(emp e),right = SQLExprTableSource(org o), condition = SQLBinaryOpExpr(e.org_id = o.id) | join 查询使用 |
| SQLSubqueryTableSource | select * from (select * from temp) a                         | from(...)是一个SQLSubqueryTableSource                        | 子查询语句    |
| SQLWithSubqueryClause  | WITH RECURSIVE ancestors AS (SELECT * FROM org UNION SELECT f.* FROM org f, ancestors a WHERE f.id = a.parent_id ) SELECT * FROM ancestors; | ancestors AS (...) 是一个SQLWithSubqueryClause.Entry         | with          |

## 2.2 SQL语句解析示例

### 2.2.1 解析 Where

注意如果条件语句中只有一个条件,那么where就是一个 `SQLBinaryOpExpr`。 当条件大于2个,使用 `where.getChildren()`

```java
/**
     * 判断where要
     * 1. 注意是SQLBinaryOpExpr(id = 1) or (u.id = 1) 需要注意是否使用了别名<br>
     * 2. 注意如果只有一个查询添加 where本身就是一个SQLBinaryOpExpr，如果是多个就要用 where.getChildren()<br></>
     * 如果有别名: SQLPropertyExpr(name = id , ownerName = u)<br>
     * 如果没别名: SQLIdentifierExpr(name = id) <br></>
     * 值对象: SQLValuableExpr
     *
     * @param where 条件对象
     */
public static void parseWhere(SQLExpr where) {
    if (where instanceof SQLBinaryOpExpr) {
        parseSQLBinaryOpExpr(cast(where, SQLBinaryOpExpr.class));
    } else {
        List<SQLObject> childrenList = where.getChildren();
        for (SQLObject sqlObject : childrenList) {
            // 包含了 left 和 right
            SQLBinaryOpExpr conditionBinary = cast(sqlObject, SQLBinaryOpExpr.class);
            parseSQLBinaryOpExpr(conditionBinary);
        }
    }

}

public static void parseSQLBinaryOpExpr(SQLBinaryOpExpr conditionBinary) {
    SQLExpr conditionExpr = conditionBinary.getLeft();
    SQLExpr conditionValueExpr = conditionBinary.getRight();
    // 左边有别名所以是SQLPropertyExpr
    if (conditionExpr instanceof SQLPropertyExpr) {
        SQLPropertyExpr conditionColumnExpr = cast(conditionExpr, SQLPropertyExpr.class);
        // 右边根据类型进行转换 id是SQLIntegerExpr name是SQLCharExpr
        SQLValuableExpr conditionColumnValue = cast(conditionValueExpr, SQLValuableExpr.class);
        print("条件列名:{},条件别名:{},条件值:{}", conditionColumnExpr.getName(), conditionColumnExpr.getOwnernName(), conditionColumnValue);
    }
    // 如果没有别名
    if (conditionExpr instanceof SQLIdentifierExpr) {
        SQLIdentifierExpr conditionColumnExpr = cast(conditionExpr, SQLIdentifierExpr.class);
        SQLValuableExpr conditionColumnValue = cast(conditionValueExpr, SQLValuableExpr.class);
        print("条件列名:{},条件值:{}", conditionColumnExpr.getName(), conditionColumnValue);
    }
}
```

### 2.2.2 解析 SQLSelectItem

解析查询的列信息

```java
/**
     * 解析查询字段,注意是否使用了别名.u.id as userId, u.name as userName, u.age as userAge<br>
     * userId（sqlSelectItem.getAlias）<br>
     * 如果有别名: u.id( id = SQLPropertyExpr.getName,u = SQLPropertyExpr.getOwnernName)<br>
     * 如果没别名: id(id = SQLIdentifierExpr.name)
     *
     * @param selectColumnList 查询字段
     */
private void parseSQLSelectItem(List<SQLSelectItem> selectColumnList) {
    for (SQLSelectItem sqlSelectItem : selectColumnList) {
        // u.id as userId(selectColumnAlias)
        String selectColumnAlias = sqlSelectItem.getAlias();
        // u.id = SQLPropertyExpr
        SQLExpr expr = sqlSelectItem.getExpr();
        if (expr instanceof SQLPropertyExpr) {
            SQLPropertyExpr selectColumnExpr = cast(expr, SQLPropertyExpr.class);
            print("列名:{},别名:{},表别名:{}", selectColumnExpr.getName(), selectColumnAlias, selectColumnExpr.getOwnernName());
        }
        if (expr instanceof SQLIdentifierExpr) {
            SQLIdentifierExpr selectColumnExpr = cast(expr, SQLIdentifierExpr.class);
            print("列名:{},别名:{}", selectColumnExpr.getName(), selectColumnAlias);
        }
    }
}
```

### 2.2.3 解析 SQLUpdateSetItem

```java
@Test
public void SQLUpdateStatement() {
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement("update users u set u.name = '唐僧',age = 18 where u.id = 1 ");
    SQLUpdateStatement sqlUpdateStatement = Utils.cast(sqlStatement, SQLUpdateStatement.class);
    List<SQLUpdateSetItem> setItems = sqlUpdateStatement.getItems();
    for (SQLUpdateSetItem setItem : setItems) {
        SQLExpr column = setItem.getColumn();
        if (column instanceof SQLPropertyExpr) {
            SQLPropertyExpr sqlPropertyExpr = Utils.cast(column, SQLPropertyExpr.class);
            SQLExpr value = setItem.getValue();
            Utils.print("column:{},列owner:{},value:{}", sqlPropertyExpr.getName(), sqlPropertyExpr.getOwnernName(), value);
        }
        if (column instanceof SQLIdentifierExpr) {
            SQLExpr value = setItem.getValue();
            Utils.print("column:{},value:{}", column, value);
        }
    }
    SQLExpr where = sqlUpdateStatement.getWhere();
    Utils.startParse("解析where", Utils::parseWhere, where);
}
```

### 2.2.4 解析 SQLLimit

```java
/**
     * 偏移量,只有2个值
     *
     * @param limit 限制
     */
private void parseLimit(SQLLimit limit) {
    // 偏移量
    SQLExpr offset = limit.getOffset();
    // 便宜数量
    SQLExpr rowCount = limit.getRowCount();
    print("偏移量:{},偏移数量:{}", offset, rowCount);
}
```

### 2.2.5 解析 SQLSelectGroupBy

```java
@Test
public void groupBy() {
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement("select name,count(1) as count from users group by name,age having count > 2");
    SQLSelectStatement selectStatement = Utils.cast(sqlStatement, SQLSelectStatement.class);
    SQLSelect select = selectStatement.getSelect();
    SQLSelectQueryBlock query = Utils.cast(select.getQuery(), SQLSelectQueryBlock.class);
    SQLSelectGroupByClause groupBy = query.getGroupBy();
    List<SQLExpr> items = groupBy.getItems();
    for (SQLExpr item : items) {
        // group by name
        // group by age
        SQLIdentifierExpr groupByColumn = Utils.cast(item, SQLIdentifierExpr.class);
        Utils.print("group by {}", groupByColumn);
    }
}
```

### 2.2.6 解析 Having

```java
@Test
public void having() {
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement("select name,count(1) as count from users group by name,age having count > 2");
    SQLSelectStatement selectStatement = Utils.cast(sqlStatement, SQLSelectStatement.class);
    SQLSelect select = selectStatement.getSelect();
    SQLSelectQueryBlock query = Utils.cast(select.getQuery(), SQLSelectQueryBlock.class);
    SQLSelectGroupByClause groupBy = query.getGroupBy();
    SQLExpr having = groupBy.getHaving();
    // 因为只有一个条件,所以having就是SQLBinaryOpExpr
    SQLBinaryOpExpr havingExpr = Utils.cast(having, SQLBinaryOpExpr.class);
    // 没有使用别名,所以就是SQLIdentifierExpr
    SQLExpr left = havingExpr.getLeft();
    SQLIdentifierExpr leftExpr = Utils.cast(left, SQLIdentifierExpr.class);
    // 数字类型就是
    SQLExpr right = havingExpr.getRight();
    SQLValuableExpr rightValue = Utils.cast(right, SQLValuableExpr.class);
    SQLBinaryOperator operator = havingExpr.getOperator();
    // left:count, operator:>,right:2
    Utils.print("left:{}, operator:{},right:{}", leftExpr.getName(), operator.name, rightValue.getValue());
}
```

# 三、语法树生成

前面的内容如果都搞清楚了,那么我们就能对sql进行解析,通知可以修改sql解析后的语法树,同时再将修改后的语法树,重新转换成sql

## 3.1 修改语法树

### 3.1.1 增加一个条件

```java
@Test
public void SQLDeleteStatement(){
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement("delete from users where id = 1");
    SQLDeleteStatement sqlDeleteStatement = Utils.cast(sqlStatement, SQLDeleteStatement.class);
    sqlDeleteStatement.addCondition(SQLUtils.toSQLExpr("name = '孙悟空'"));
    //        DELETE FROM users
    //        WHERE id = 1
    //        AND name = '孙悟空'
    System.out.println(SQLUtils.toSQLString(sqlDeleteStatement));
}
```

### 3.1.2 修改一个条件值

将条件id = 1 修改成 id = 2

```java
@Test
public void SQLDeleteStatement2(){
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement("delete from users where id = 1");
    SQLDeleteStatement sqlDeleteStatement = Utils.cast(sqlStatement, SQLDeleteStatement.class);
    SQLExpr where = sqlDeleteStatement.getWhere();
    SQLBinaryOpExpr sqlBinaryOpExpr = Utils.cast(where, SQLBinaryOpExpr.class);
    //        DELETE FROM users
    //        WHERE id = 2
    sqlBinaryOpExpr.setRight(SQLUtils.toSQLExpr("2"));
    System.out.println(SQLUtils.toSQLString(sqlDeleteStatement));
}
```

# 四、Visitor模式

访问者模式

所有的AST节点都支持Visitor模式，需要自定义遍历逻辑，可以实现相应的ASTVisitorAdapter派生类

```java
public static class CustomerMySqlASTVisitorAdapter extends MySqlASTVisitorAdapter {

    private final Map<String, SQLTableSource> ALIAS_MAP = new HashMap<String, SQLTableSource>();

    private final Map<String, SQLExpr> ALIAS_COLUMN_MAP = new HashMap<String, SQLExpr>();

    public boolean visit(SQLExprTableSource x) {
        String alias = x.getAlias();
        ALIAS_MAP.put(alias, x);
        return true;
    }

    @Override
    public boolean visit(MySqlSelectQueryBlock x) {
        List<SQLSelectItem> selectList = x.getSelectList();
        for (SQLSelectItem sqlSelectItem : selectList) {
            String alias = sqlSelectItem.getAlias();
            SQLExpr expr = sqlSelectItem.getExpr();
            ALIAS_COLUMN_MAP.put(alias, expr);
        }
        return true;
    }

    public Map<String, SQLTableSource> getAliasMap() {
        return ALIAS_MAP;
    }

    public Map<String, SQLExpr> getAliasColumnMap() {
        return ALIAS_COLUMN_MAP;
    }
}

@Test
public void AliasVisitor() {
    String sql = "select u.id as userId, u.name as userName, age as userAge from users as u where u.id = 1 and u.name = '孙悟空' limit 2,10";
    // 解析SQL
    SQLStatement sqlStatement = SQLUtils.parseSingleMysqlStatement(sql);
    CustomerMySqlASTVisitorAdapter customerMySqlASTVisitorAdapter = new CustomerMySqlASTVisitorAdapter();
    sqlStatement.accept(customerMySqlASTVisitorAdapter);
    // 表别名:{u=users}
    System.out.println("表别名:" + customerMySqlASTVisitorAdapter.getAliasMap());
    // 列别名{userName=u.name, userId=u.id, userAge=age}
    System.out.println("列别名" + customerMySqlASTVisitorAdapter.getAliasColumnMap());
}
```

# 附录A

## A.1 Druid解析SQL实现血缘关系计算

- [Druid解析SQL实现血缘关系计算](https://www.cnblogs.com/changxy-codest/p/13535143.html)
- [druid的数据血缘的简单验证](https://blog.csdn.net/qq532026984/article/details/128274381)

```java
import com.alibaba.druid.sql.SQLUtils;
import com.alibaba.druid.sql.ast.SQLStatement;
import com.alibaba.druid.sql.visitor.SchemaStatVisitor;
import com.alibaba.druid.stat.TableStat;
import com.alibaba.druid.util.JdbcConstants;
import lombok.extern.slf4j.Slf4j;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Objects;

@Slf4j
public class Test {

    public static void main(String args[]) throws Exception {

        String sql = "insert overwrite table kf_zt.account2 SELECT swjg_dm,swjgmc FROM (SELECT swjg_dm AS swjg_dm, swjgmc AS swjgmc FROM (SELECT swjg_dm,swjgmc FROM ( SELECT swjg_dm,swjgmc FROM yuangongs) table_350 UNION SELECT swjg_dm,swjgmc FROM (SELECT swjg_dm2,swjgmc2 FROM yuangong ) table_295) union_013) udf_882";

        List<SQLStatement> sqlStatements = SQLUtils.parseStatements(sql.toLowerCase(), JdbcConstants.HIVE);

        for (SQLStatement sqlStatement : sqlStatements) {
            SchemaStatVisitor schemaStatVisitor = SQLUtils.createSchemaStatVisitor(JdbcConstants.HIVE);
            sqlStatement.accept(schemaStatVisitor);
            Map<TableStat.Name, TableStat> tables = schemaStatVisitor.getTables();
            Collection<TableStat.Column> columns = schemaStatVisitor.getColumns();
            if (Objects.nonNull(tables)) {
                tables.forEach(((name, tableStat) -> {
                    if (tableStat.getCreateCount() > 0 || tableStat.getInsertCount() > 0) {
                        log.info("to: table[{}]", name.getName().toLowerCase());
                        columns.stream().filter(column -> Objects.equals(column.getTable().toLowerCase(), name.getName().toLowerCase())).forEach(column -> {
                            log.info("to: table[{}] column[{}]", column.getTable().toLowerCase(), column.getName().toLowerCase());
                        });
                    } else  if (tableStat.getSelectCount() > 0) {
                        log.info("from: table[{}]", name.getName().toLowerCase());
                        columns.stream().filter(column -> Objects.equals(column.getTable().toLowerCase(), name.getName().toLowerCase())).forEach(column -> {
                            log.info("from: table[{}] column[{}]", column.getTable().toLowerCase(), column.getName().toLowerCase());
                        });
                    }
                }));
            }
        }
    }
}
```

## A.2 ParseUtils

- [ParseUtils（SQL解析）](https://juejin.cn/post/7071113139189710861)

ParseUtils：

```java
package com.example.util;

import com.alibaba.druid.sql.SQLUtils;
import com.alibaba.druid.sql.ast.SQLStatement;
import com.alibaba.druid.sql.visitor.SchemaStatVisitor;
import com.alibaba.druid.stat.TableStat;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.CollectionUtils;
import org.springframework.util.StringUtils;

import java.util.*;
import java.util.stream.Collectors;

import static com.alibaba.druid.util.JdbcConstants.*;

@Slf4j
public abstract class ParseUtil {

    /**
     * sql格式化
     *
     * @param sql    sql语句
     * @param dbType 数据库类型
     */
    public static void format(String sql, String dbType) {
        String sqlFormat = SQLUtils.format(sql, dbType);
        if (sql.equals(sqlFormat)) {
            throw new RuntimeException("SQL格式错误");
        }
    }

    /**
     * 校验结尾是否含有；
     *
     * @param sql sql语句
     */
    public static void checkSymbol(String sql) {
        Arrays.stream(sql.toUpperCase().split("DROP|CREATE|INSERT|UPDATE"))
                .filter(sqlStr -> !StringUtils.isEmpty(sqlStr))
                .filter(sqlStr -> !sqlStr.contains(";"))
                .forEach(sqlStr -> {
                    throw new RuntimeException("SQL缺失;");
                });
    }

    /**
     * sql解析
     *
     * @param sql    sql语句
     * @param dbType 数据库类型
     * @return 解析树
     */
    public static List<SQLStatement> parseStatements(String sql, String dbType) {
        try {
            return SQLUtils.parseStatements(sql, dbType);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            throw new RuntimeException("SQL格式错误");
        }
    }

    /**
     * 血缘关系解析
     *
     * @param sqlStatements sql语句
     * @param tableList     关系集合
     * @param dbType        数据库类型
     * @return k（table_target） - v（list<table_source>）
     */
    public static List<Map<String, List<String>>> bloodRelationship(List<SQLStatement> sqlStatements
            , List<Map<String, List<String>>> tableList, String dbType) {

        SchemaStatVisitor visitor;
        for (SQLStatement sqlStatement : sqlStatements) {

            //切入操作
            Map<String, List<String>> tableMap = new HashMap<>();
            visitor = SQLUtils.createSchemaStatVisitor(dbType);
            sqlStatement.accept(visitor);

            //过滤 DML
            visitor.getTables().forEach((key, value) -> {
                if (!"Select".equals(value.toString())) {
                    throw new RuntimeException("禁止进行其他DML操作!");
                }
            });

            //结果集封装
            List<String> tables = visitor.getTables()
                    .keySet()
                    .stream()
                    .map(TableStat.Name::getName)
                    .collect(Collectors.toList());

            if (!CollectionUtils.isEmpty(tables) && tables.size() > 1) {
                tableMap.put(tables.remove(0), tables);
                tableList.add(tableMap);
            }

        }
        return tableList;
    }


    static class OdpsParseUtil extends ParseUtil {

        /**
         * sql 解析(ODPS)
         *
         * @param sql sql语句
         * @return 血缘关系
         */
        public static List<Map<String, List<String>>> parse(String sql) {
            return bloodRelationship(checkSqlFormatByOdps(sql), new ArrayList<>(), ODPS);
        }

        public static List<SQLStatement> checkSqlFormatByOdps(String sql) {
            checkSymbol(sql);
            format(sql, ODPS);
            return parseStatements(sql, ODPS);
        }

    }

    static class HiveParseUtil extends ParseUtil {

        /**
         * sql 解析(HIVE)
         *
         * @param sql sql语句
         * @return 血缘关系
         */
        public static List<Map<String, List<String>>> parse(String sql) {
            return bloodRelationship(checkSqlFormatByHive(sql), new ArrayList<>(), HIVE);
        }

        public static List<SQLStatement> checkSqlFormatByHive(String sql) {
            checkSymbol(sql);
            format(sql, HIVE);
            return parseStatements(sql, HIVE);
        }

    }

    static class MysqlParseUtil extends ParseUtil {

        /**
         * sql 解析(MYSQL)
         *
         * @param sql sql语句
         * @return 血缘关系
         */
        public static List<Map<String, List<String>>> parse(String sql) {
            return bloodRelationship(checkSqlFormatByMysql(sql), new ArrayList<>(), MYSQL);
        }

        public static List<SQLStatement> checkSqlFormatByMysql(String sql) {
            checkSymbol(sql);
            format(sql, MYSQL);
            return parseStatements(sql, MYSQL);
        }

    }
}
```

源码分析：

```java
public class SchemaStatVisitor extends SQLASTVisitorAdapter {

    protected SchemaRepository repository;

    //SQL中所有的表
    protected final HashMap<TableStat.Name, TableStat> tableStats     = new LinkedHashMap<TableStat.Name, TableStat>();

    //SQL中所有的列
    protected final Map<Long, Column>                  columns        = new LinkedHashMap<Long, Column>();

    //SQL中条件
    protected final List<Condition>                    conditions     = new ArrayList<Condition>();

    //SQL中关系
    protected final Set<Relationship>                  relationships  = new LinkedHashSet<Relationship>();

    //SQL中排序
    protected final List<Column>                       orderByColumns = new ArrayList<Column>();

    //SQL中分组
    protected final Set<Column>                        groupByColumns = new LinkedHashSet<Column>();

     //SQL中函数
    protected final List<SQLAggregateExpr>             aggregateFunctions = new ArrayList<SQLAggregateExpr>();

    //SQL中方法
    protected final List<SQLMethodInvokeExpr>          functions          = new ArrayList<SQLMethodInvokeExpr>(2);
    ......
}
```

