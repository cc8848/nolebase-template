- [mybatis连接Hive和MySQL双源操作](https://blog.csdn.net/xiaoxaoyu/article/details/115985145)
- [springboot集成hive实战](https://blog.csdn.net/Winter_chen001/article/details/110449266)

# 一、基础配置

## 1.1 pom.xml

```xml
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version>2.1.0</version>
</dependency>
```

## 1.2 配置

```yaml
# MyBatis-Plus 多数据源
spring:
  datasource:
    dynamic:
      primary: mysql #设置默认的数据源或者数据源组,默认值即为master
      strict: false #严格匹配数据源,默认false. true未匹配到指定数据源时抛异常,false使用默认数据源
      datasource:
        mysql:
          url: jdbc:mysql://your_mysql_ip:port/your_database_name?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=false
          username: your_username
          password: your_password
          driver-class-name: com.mysql.cj.jdbc.Driver
        hive:
          url: jdbc:hive2://your_hive_ip:port/your_database_name
          username: your_username
          password: your_password
          driver-class-name: org.apache.hive.jdbc.HiveDriver
```

# 二、示例

## 2.1 hive

```java
import com.baomidou.dynamic.datasource.annotation.DS;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/hive")
public class HiveController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

	// @DS 注解要标在执行 SQL 语句用到的数据源为非主要数据源的方法上，value 为 配置文件中的（此例为 hive）
    @DS("hive")
    @RequestMapping("/select")
    public Object select() {
    	// 要执行的 SQL
        String sql = "SELECT * FROM xxxxxx";
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
        log.info(rows.toString());
        return rows;
    }
}
```

## 2.2 demo2

- [springboot集成hive实战](https://blog.csdn.net/Winter_chen001/article/details/110449266)