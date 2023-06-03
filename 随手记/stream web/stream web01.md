# Stream Web：简化后端开发的利器

## 什么是 Stream Web

Stream Web 受到 Stream Query 项目的启发，它通过定义一个数据库持久层对象（PO），自动生成相应的基础 RESTful 接口，使得简单的增删改查操作变得更加便捷。

已经支持以下接口实现：
```java
    @GetMapping("/{entity}/{id}")
    public Object getById(@PathVariable("entity") String entity, @PathVariable("id") String id);
    @DeleteMapping("/{entity}/{id}")
    public void deleteById(@PathVariable("entity") String entity, @PathVariable("id") String id);
    @PostMapping("/{entity}")
    public int create(@PathVariable("entity") String entity, @RequestBody Object entityObject);
    @PutMapping("/{entity}")
    public int updateById(@PathVariable("entity") String entity, @RequestBody Object entityObject);
    @GetMapping("/{entity}/all")
    public List<Object> getAll(@PathVariable("entity") String entity);

```
其实现可见：org.dark.web.AutoGenApi

## 用起来真的方便吗？

Stream Web 基于 Stream Query 进行开发，只需要依赖 `stream-web-core` 并配置需要扫描的 PO 包名，即可轻松实现功能。您可以通过以下 Maven 依赖配置将 `stream-web-core` 引入到您的项目中：

```xml
<dependency>
    <groupId>org.dark</groupId>
    <artifactId>stream-web-core</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

同时，您还需要在您的应用程序配置中添加以下配置项，以指定需要扫描的 PO 包名：

```yaml
stream-web:
  base-packages: org.dark.streamweb.demo.po
```

请将上述配置项添加到您的应用程序的配置文件中（比如 application.yaml 或 application.properties）。将 org.dark.streamweb.demo.po 替换为您实际的 PO 包名，以确保 Stream Web 正确扫描并生成相应的 RESTful 接口。请将上述配置项添加到您的应用程序的配置文件中（比如 application.yaml 或 application.properties）。将 org.dark.streamweb.demo.po 替换为您实际的 PO 包名，以确保 Stream Web 正确扫描并生成相应的 RESTful 接口。

### DEMO

#### 定义了一个PO

```java
@Data
@TableName("users")
public class UserPO {

    @TableId(value = "pk_id", type = IdType.AUTO)
    private String id;
    @TableField("name")
    private String name;
    @TableField("email")
    private String email;
}

```
这是它对应的建表语句：
```sql
CREATE TABLE users (
    pk_id INT NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (pk_id)
);

-- 初始化三条数据
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES ('Bob', 'bob@example.com');
INSERT INTO users (name, email) VALUES ('Charlie', 'charlie@example.com');
commit;

```

### 启动项目并访问

浏览器访问，或者发送一个curl请求：
```bash
curl http://localhost:8080/auto-gen/users/2
```

就能得到一个json相应：
```json
{"id":"2","name":"Bob","email":"bob@example.com"}
```


## 那它有什么问题吗？

然而，目前的 Stream Web 只是一个玩具工程，它将所有接口都暴露给前端，没有进行鉴权的情况下，可能会导致被人拖库和删库等安全问题。因此，在将 Stream Web 用于实际项目中时，我们需要注意加强安全措施，包括但不限于鉴权机制和访问权限的限制，以确保数据的安全性。


尽管 Stream Web 目前还存在一些问题，但它作为一个简化后端开发的工具，依然具有巨大的潜力。随着项目的不断演进和改进，相信 Stream Web 能够成为开发人员的得力助手，提升开发效率，减少重复工作，从而更好地服务于开发者和用户的需求。