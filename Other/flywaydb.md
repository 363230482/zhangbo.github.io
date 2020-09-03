# flywaydb自动管理数据库升级脚本

## 1.添加依赖包
springboot会有自动配置`org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration`.但要注意版本兼容性
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>5.2.4</version>
</dependency>
```

## 2.配置文件
```yaml
spring:
  application:
    name: zb-springboot-demo
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
  flyway:
    enabled: true
    baseline-on-migrate: true #如果是项目中后期才开始使用，则配置该选项
```

## 3.添加升级脚本
在classpath下，新建目录：`db/migration`，然后新建脚本，命名规则为：`V1.0.1__desc.sql`.如：
```sql
-- V1.0.1__20200903.sql
alter table t_user add column sex varchar(1) comment '性别';
```

## 4.启动项目，测试
最后启动项目，发现`flywaydb`自动在数据库中新建了`flyway_schema_history`的表，表中有每次升级执行SQL脚本的记录。
同时，我们的升级脚本也一并执行了，对应的表应该也添加了sex字段。