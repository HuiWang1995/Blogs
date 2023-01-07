# 巧用Hibernate 完成多数据库的DDL脚本创建

> spring boot jpa 默认的orm框架就是Hibernate。 由hibernate完成数据库的读写也是主流的方式之一。但是不同数据库之间，**建表、建索引**的方言语句都有较多差别，很难做到一套SQL在所有数据库上进行执行。
>
> 那么Hibernate可以做到DML等语句通用，DDL可以吗？ **当然，是可以的。**
>
> Hibernate兼容全靠`org.hibernate.dialect.Dialect`下的具体实现类。
>
> 下面我们来简单演示如何基于hibernate的注解，来生成DDL语句。



## 定义一个Entity

使用hibernate的注解描述一个Entity，我们定义了其表名，id，name属性，并指定id为主键列。

```java
// hibernate 6.x 开始，注解移动到了jakarta.persistence包下
import jakarta.persistence.*;

@Entity
@Table(name = "my_entity")
public class MyEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id", nullable = false)
    private Long id;

    @Column(name = "name", nullable = true, length = 256)
    private String name;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

## 期待生成的SQL

我们在建表的时候，还要创建序列，指定执行引擎。要知道，MYSQL如果使用MyISAM将无法使用会话。

### oracle

```sql
create sequence my_entity_SEQ start with 1 increment by 50;
create table my_entity (id number(19,0) not null, name varchar2(256 char), primary key (id));
```

### mysql

```sql
create table my_entity (id bigint not null, name varchar(256), primary key (id)) engine=InnoDB;
create table my_entity_SEQ (next_val bigint) engine=InnoDB;
insert into my_entity_SEQ values ( 1 );
```

## 具体实现

具体实现并不难，主要是调用`org.hibernate.tool.hbm2ddl.SchemaExport`进行导出，主要方法如下：

```java
     /**
     * 导出
     *
     * @param classes    导出的类
     * @param dialect    数据库方言
     * @param action     导出动作
     * @param targetType 导出方式
     * @param scriptPath 脚本位置
     */
	public static void export0(List<Class> classes,
        Class<? extends Dialect> dialect,
        SchemaExport.Action action,
        TargetType targetType,
        String scriptPath) {

        Properties p = new Properties();
        // 数据库方言，最终输出的方言
        p.put("hibernate.dialect", dialect.getName());
        // 自动执行的动作
        p.put("hibernate.hbm2dll.auto", action.name());
        // 分隔符，默认为空
        p.put("hibernate.hbm2ddl.delimiter", ";");
        // 是否展示SQL
        p.put("show_sql", true);
        // 是否使用默认的jdbc元数据，默认为true，读取项目自身的元数据
        p.put("hibernate.temp.use_jdbc_metadata_defaults", false);
        ConfigurationHelper.resolvePlaceHolders(p);
        ServiceRegistry registry = new StandardServiceRegistryBuilder()
            .applySettings(p)
            .build();


        // org.dark.migration.demo
        MetadataSources metadataSources = new MetadataSources(registry);
        classes.forEach(metadataSources::addAnnotatedClass);
        Metadata metadata = metadataSources.buildMetadata();
        SchemaExport export = new SchemaExport();
        export.setOverrideOutputFileContent();
        export.setOutputFile(scriptPath);
        export.execute(EnumSet.of(targetType), action, metadata);
    }
```

调用方法如下：

```java
    public static void main(String[] args) {
        List<Class> classes = new ArrayList<>();
        classes.add(MyEntity.class);
//        export0(classes, Oracle12cDialect.class,
        export0(classes, MySQL8Dialect.class,
            SchemaExport.Action.CREATE,
            TargetType.SCRIPT, "test-output2.txt");
    }
```

最终就会在`"test-output2.txt"`中生成我们期待的SQL。

## 支持的数据库方言

通过查看`Dialect`的子类，可以看到除了主流的mysql、Oracle、postgre、DB2等，像TIDB、CockroachDB等等数据库等均有方言支持。

![image-20221229210437049](/home/dark/GitHub/Blogs/springboot/03/pic/01.png)



## 总结

利用好hibernate的多数据源方言兼容性，我们仅维护一套注解即可创建多数据源DDL。文章代码库位置：[https://gitee.com/wanglhup/migration]( https://gitee.com/wanglhup/migration)。

