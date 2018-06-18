---
layout:     post
title:      MyBatis
subtitle:   MyBatis深入研究
catalog: true
tags:
    - MyBatis
---

### resultMap配置
研究目的：resultMap的javaType是否需要配置
#### id,result
```java
public class Student {
    private Long id;
    private Long sid;
    private String name; // 姓名
    private Integer age; // 年龄

    public Student(){}

    public Student(Long id, String name) {
        this.id = id;
        this.name = name+"(constructor)";
    }
}
```

```xml
    <resultMap id="resultMap" type="model.Student" >
        <id property="id" column="id"/>
        <!-- name故意配置了错误的类型，本应为string -->
        <result property="name" column="name" javaType="long"/>
        <result property="sid" column="sid" />
        <result property="age" column="age"/>
    </resultMap>
```

以上就是此次研究创建的po以及相应的resultMap，我特意选用String强转为Long，通过调试看到一下结果
![mybatis-resultMaps-debug](/img/mybatis/mybatis-resultMaps-debug.png)
可以看到sid没有设置JavaType，但Mybatis正确地识别出类型，而name属性设置了错误JavaType，将会是这样的结果
![mybatis-resultMaps-SQLException](/img/mybatis/mybatis-resultMaps-SQLException.png)

#### constructor
```xml
    <resultMap id="resultMap" type="model.Student" >
        <constructor>
            <idArg column="id" javaType="long"/>
            <arg column="name" javaType="string"/>
        </constructor>
        <id property="id" column="id"/>
        <result property="name" column="name" javaType="long"/>
        <result property="sid" column="sid" />
        <result property="age" column="age"/>
    </resultMap>
```

要注意的是，constructor里面必须要配置javaType，否则MyBatis会将这两个参数解析为Object，最终结果就是找不到构造函数。

#### 综合
id,result的javaType是不是就没用了呢？有用的。比如po里面是String类型，但数据库是Long类型，这时就需要配置，虽然有点奇怪，

**结论：id,result可不设置JavaType，constructor必须设置JavaType**

### XML parameterType
源于疑问，parameterType通过Java就能获取，为何在配置文件中要让程序员配置呢？
```java
    public Student queryByID1(Long id);

    public Student queryByID2(Long id);

    public Student queryByID3(Long id);
```
```xml
    <select id="queryByID1" resultMap="resultMap" parameterType="long">
      SELECT * FROM t_student WHERE sid=#{sid}
    </select>
    <select id="queryByID2" resultMap="resultMap">
        SELECT * FROM t_student WHERE sid=#{sid}
    </select>
    <select id="queryByID3" resultMap="resultMap" parameterType="string">
        SELECT * FROM t_student WHERE sid=#{sid}
    </select>
```
以上就是我的DAO和配置文件，通过调试发现，决定参数类型的其实是DAO的形参，所以parameterType无需配置，以上三种配置都能成功。

### 数据类型
#### 类型对比
- 此次对比以下三个类
  - java.sql.Types
  - java.sql.JDBCType
  - org.apache.ibatis.type.JdbcType

- 特性1：<br>
**针对BIG、SMALL、TINY这三个字段，JDBCType、JdbcType都加了后缀INT**<br>
*BIG->BIGINT、SMALL->SMALLINT、TINY->TINYINT*

- 特性2：
  - JDBCType一词不落的全都使用枚举表示了Types
  - JdbcType增加了UNDEFINED，而且没有表示以下几个值
    - DATALINK
    - DISTINCT
    - JAVA_OBJECT
    - LONGNVARCHAR
    - NCHAR
    - NCLOB
    - NVARCHAR
    - REF
    - REF_CURSOR
    - ROWID
    - SQLXML
    - TIMESTAMP_WITH_TIMEZONE
    - TIME_WITH_TIMEZONE

#### 类型映射
我列举出了部分常用的JdbcType与JavaType的映射关系

|JdbcType|JavaType|
|---|---|
|BIGINT|Long|
|INTEGER|Integer|
|TINYINT|Short|
|SMALLINT|Integer|
|NUMERIC|BigDecimal|
|DECIMAL|BigDecimal|
|DOUBLE|Double|
|FLOAT|Float|
|CHAR|String|
|VARCHAR|String|
|TIMESTAMP|TimeStamp|
|DATE|Date|
|TIME|Time|

### MyBatis部分类的职能

|类名|列名/函数名|职能|
|---|---|---|
|FastResultSetHandler|mappedStatement|存储resultMap|
|BaseTypeHandler|getResult|根据列名从结果集中取出数据|
|java.sql.ResultSet|-|MyBatis根据类型调用对应的get方法取数据|
|FastResultSetHandler|applyPropertyMappings|将ResultSet的值存储到对象|