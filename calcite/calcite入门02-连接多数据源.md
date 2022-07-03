# calcite入门02-连接多数据源

> calcite可以连接多个数据源，多个库，并且将查询结果在内存中使用Linq进行计算合并。那么在01的基础上，我们尝试再新建一个库并且在一个SQL中查两个库的表。

## SQL

在01的基础上，增量执行以下SQL

```sql
create schema test2;

use test2;
create table student2
(
    pk_id varchar(32)  not null
        primary key,
    name  varchar(128) null
);

insert into test2.student2 (pk_id, name)
values ('2', 'lu');

insert into test2.student2 (pk_id, name)
values ('1', 'wang');

commit;

```

那么正常SQL，肯定不支持test.student1与tese2.student2进行连表查询的。 那么如何支持以下SQL呢?

```sql
select *
from test.student
         left join test2.student2
                   on test.student.pk_id = test2.student2.pk_id
```

## calcite demo代码

```java
/**
 * test connect mysql by calcite
 *
 * to run this test normally please run init2.sql on your mysql server first
 *
 * schema: test & test2
 * table: test.student & test2.student2
 */
public class Test02 {

    // two schema query for mysql
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


        // code for mysql datasource2
        MysqlDataSource dataSource2 = new MysqlDataSource();
        dataSource2.setUrl("jdbc:mysql://aaa.club/test2");
        dataSource2.setUser("root");
        dataSource2.setPassword("123456");
        // mysql schema, the sub schema for rootSchema, "test2" is a schema in mysql
        Schema schema2 = JdbcSchema.create(rootSchema, "test2", dataSource2, null, "test2");
        rootSchema.add("test2", schema2);


        // run sql query
        Statement statement = calciteConnection.createStatement();
        ResultSet resultSet = statement.executeQuery("select * from test.student " +
                "left join test2.student2 on test.student.pk_id = test2.student2.pk_id");
        while (resultSet.next()) {
            System.out.println(resultSet.getObject(1) + "__" + resultSet.getObject(2));
        }

        statement.close();
        connection.close();
    }

}
```

与01相比，我们基本就是添加了一份与schema基本相同的schema2，仅仅库名不同~

那么暂且不去关心calcite如何进行合并的，上述代码就可以做到将SQL进行跨库连接查询了~


## github地址

[calcite-mysql](https://github.com/HuiWang1995/calcite-mysql)