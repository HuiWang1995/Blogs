# calcite入门01-连接mysql

> apache calcite 可以连接多种数据源，mysql,redis,elasticsearch等等，并且封装了jdbc协议进行查询。另外它也易于扩展，可以查询自定义的数据源。

## MAVEN依赖
```xml
        <!--calcite核心包-->
        <dependency>
            <groupId>org.apache.calcite</groupId>
            <artifactId>calcite-core</artifactId>
            <version>1.22.0</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.10.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.79</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.google.guava/guava -->
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>24.1-jre</version>
        </dependency>

        <!-- mysql -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.25</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.26</version>
        </dependency>

```

## 准备数据源

首先准备一个可以运行的Mysql服务器，其次，创建schema以及table，当然，也插入一些数据，这边准备了`init.sql`

```sql
create schema test;

use test;
create table student
(
    pk_id varchar(32)  not null
        primary key,
    name  varchar(128) null
);

insert into test.student (pk_id, name)
values  ('1', 'wang');

commit;
```

## DEMO代码

```java
/**
 * test connect mysql by calcite
 *
 * to run this test normally please run init.sql on your mysql server first
 *
 * schema: test
 * table: test.student
 */
public class Test01 {

    // single schema query
    public static void main(String[] args) throws Exception {
        // check driver exist
        Class.forName("org.apache.calcite.jdbc.Driver");
        Class.forName("com.mysql.jdbc.Driver");

        // the properties for calcite connection
        Properties info = new Properties();
        info.setProperty("lex", "JAVA");
        info.setProperty("remarks","true");
        // SqlParserImpl can analysis sql dialect for sql parse
        info.setProperty("parserFactory","org.apache.calcite.sql.parser.impl.SqlParserImpl#FACTORY");

        // create calcite connection and schema
        Connection connection = DriverManager.getConnection("jdbc:calcite:", info);
        CalciteConnection calciteConnection = connection.unwrap(CalciteConnection.class);
        System.out.println(calciteConnection.getProperties());
        SchemaPlus rootSchema = calciteConnection.getRootSchema();

        // code for mysql datasource
        MysqlDataSource dataSource = new MysqlDataSource();
        // please change host and port maybe like "jdbc:mysql://127.0.0.1:3306/test"
        dataSource.setUrl("jdbc:mysql://aaa.club/test");
        dataSource.setUser("root");
        dataSource.setPassword("123456");
        // mysql schema, the sub schema for rootSchema, "test" is a schema in mysql
        Schema schema = JdbcSchema.create(rootSchema, "test", dataSource, null, "test");
        rootSchema.add("test", schema);

        // run sql query
        Statement statement = calciteConnection.createStatement();
        ResultSet resultSet = statement.executeQuery("select * from test.student");
        while (resultSet.next()) {
            System.out.println(resultSet.getObject(1) + "__" + resultSet.getObject(2));
        }

        statement.close();
        connection.close();
    }
}

```

- 保证驱动包存在
- 创建calcite的连接对象connection以及rootSchema
- 创建mysql的datasource
- 创建mysql的schema
- 将mysql的schema添加到rootSchema中
- 执行SQL
- 打印SQL结果

## github地址

[calcite-mysql](https://github.com/HuiWang1995/calcite-mysql)