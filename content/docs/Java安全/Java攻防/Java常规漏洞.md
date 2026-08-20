---
title: Java常规漏洞
date: 2026-08-19
---

# Java 常规漏洞

## SQL注入

### JDBC

#### 漏洞场景

**原生 SQL 语句拼接**

```java
// 原生sql语句动态拼接 参数未进行任何处理
public R vul1(String type,String id,String username,String password) {
    //注册数据库驱动类
    Class.forName("com.mysql.cj.jdbc.Driver");

    //调用DriverManager.getConnection()方法创建Connection连接到数据库
    Connection conn = DriverManager.getConnection(dbUrl, dbUser, dbPass);

    //调用Connection的createStatement()或prepareStatement()方法 创建Statement对象
    Statement stmt = conn.createStatement();
    switch (type) {
        case "add":
            //这里没有标识id id自增长
            sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
            //通过Statement对象执行SQL语句，得到ResultSet对象-查询结果集
            // 这里注意一下 insert、update、delete 语句应使用executeUpdate()
            rowsAffected = stmt.executeUpdate(sql);
            //关闭ResultSet结果集 Statement对象 以及数据库Connection对象 释放资源
            stmt.close();
            conn.close();
            return R.ok(message);
        case "delete":
            sql = "DELETE FROM sqli WHERE id = '" + id + "'";
            rowsAffected = stmt.executeUpdate(sql);
            ...
        case "update":
            sql = "UPDATE sqli SET password = '" + password + "', username = '" + username + "' WHERE id = '" + id + "'";
            rowsAffected = stmt.executeUpdate(sql);
            ...
        case "select":
            sql = "SELECT * FROM sqli WHERE id  = " + id;
            ResultSet rs = stmt.executeQuery(sql);
            ...
        }
}
```

**JDBC 伪预编译**

```java
// 虽然使用了conn.prepareStatement(sql)创建了一个PreparedStatement对象，但在执行 stmt.executeUpdate(sql)时，却是传递了完整的SQL语句作为参数，而不是使用了预编译的功能
public R vul2(String type,String id,String username,String password) {
    Class.forName("com.mysql.cj.jdbc.Driver");
    Connection conn = DriverManager.getConnection(dbUrl, dbUser, dbPass);
    PreparedStatement stmt;
    switch (type) {
        case "add":
            sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
            stmt = conn.prepareStatement(sql);
            rowsAffected = stmt.executeUpdate(sql);
            ...
        case "delete":
            sql = "DELETE FROM sqli WHERE id = '" + id + "'";
            stmt = conn.prepareStatement(sql);
            rowsAffected = stmt.executeUpdate(sql);
            ...
        case "update":
            sql = "UPDATE sqli SET username = '" + username + "', password = '" + password + "' WHERE id = '" + id + "'";
            stmt = conn.prepareStatement(sql);
            rowsAffected = stmt.executeUpdate(sql);
            ...
        case "select":
            sql = "SELECT * FROM sqli WHERE id  = " + id;
            stmt = conn.prepareStatement(sql);
            ResultSet rs = stmt.executeQuery(sql);
            ...
    }
}
```

**JdbcTemplate SQL 语句拼接**

```java
// JDBCTemplate是Spring对JDBC的封装，底层实现实际上还是JDBC
public R vul3(String type,String id,String username,String password) {
    DriverManagerDataSource dataSource = new DriverManagerDataSource();
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
    dataSource.setUrl(dbUrl);
    dataSource.setUsername(dbUser);
    dataSource.setPassword(dbPass);
    JdbcTemplate jdbctemplate = new JdbcTemplate(dataSource);
    switch (type) {
        case "add":
            sql = "INSERT INTO sqli (username, password) VALUES ('" + username + "', '" + password + "')";
            //Spring的JdbcTemplate会自动管理连接的获取和释放，不需要手动关闭连接
            rowsAffected = jdbctemplate.update(sql);
            ...
        case "delete":
            sql = "DELETE FROM sqli WHERE id = '" + id + "'";
            rowsAffected = jdbctemplate.update(sql);
            ...
        case "update":
            sql = "UPDATE sqli SET username = '" + username + "', password = '" + password + "' WHERE id = '" + id + "'";
            rowsAffected = jdbctemplate.update(sql);
            ...
        case "select":
            sql = "SELECT * FROM sqli WHERE id  = " + id;
            resultList = jdbctemplate.queryForList(sql);
            ...
    }
}
```

#### 安全场景

**JDBC 预编译**

```java
// 采用预编译的方法，使用?占位，也叫参数化的SQL
public R safe1(String type,String id,String username,String password) {
    Class.forName("com.mysql.cj.jdbc.Driver");
    Connection conn = DriverManager.getConnection(dbUrl, dbUser, dbPass);
    PreparedStatement stmt;
    switch (type) {
        case "add":
            // 这里可以看到使用了?占位符 sql语句和参数进行分离
            sql = "INSERT INTO sqli (username, password) VALUES (?, ?)"; 
            stmt = conn.prepareStatement(sql);
            // 参数化处理
            stmt.setString(1, username); 
            stmt.setString(2, password);
            // 使用预编译时 不需要传递sql语句
            rowsAffected = stmt.executeUpdate();
        case "delete":
            sql = "DELETE FROM sqli WHERE id = ?";
            stmt = conn.prepareStatement(sql);
            stmt.setString(1, id);
            rowsAffected = stmt.executeUpdate();
            ...
        case "update":
            sql = "UPDATE sqli SET username = ?, password = ? WHERE id = ?";
            stmt = conn.prepareStatement(sql);
            stmt.setString(1, username);  
            stmt.setString(2, password);
            stmt.setString(3, id);
            rowsAffected = stmt.executeUpdate();
            ...
        case "select":
            sql = "SELECT * FROM sqli WHERE id  = ?";
            stmt = conn.prepareStatement(sql);
            stmt.setString(1, id);
            ResultSet rs = stmt.executeQuery();
            ...
   }
}
```

**JdbcTemplate 参数绑定**

```java
// JDBCTemplate预编译 此时在常规DML场景有效的防止了SQL注入攻击的发生
public R safe2(String type,String id,String username,String password) {
    DriverManagerDataSource dataSource = new DriverManagerDataSource();
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
    dataSource.setUrl(dbUrl);
    dataSource.setUsername(dbUser);
    dataSource.setPassword(dbPass);
    JdbcTemplate jdbctemplate = new JdbcTemplate(dataSource);
    switch (type) {
        case "add":
            sql = "INSERT INTO sqli (username, password) VALUES (?,?)";
            rowsAffected = jdbctemplate.update(sql, username, password);
            ...
        case "delete":
            sql = "DELETE FROM sqli WHERE id = ?";
            rowsAffected = jdbctemplate.update(sql, id);
            ...
        case "update":
            sql = "UPDATE sqli SET username = ?, password = ? WHERE id = ?";
            rowsAffected = jdbctemplate.update(sql, username, password, id);
            ...
        case "select":
            sql = "SELECT * FROM sqli WHERE id  = ?";
            stringObjectMap = jdbctemplate.queryForMap(sql, id);
            ...
    }
}
```

### MyBatis

MyBatis 支持两种参数符号，分别是 `#` 和 `$`，`#` 使用预编译，`$`使用 SQL 拼接。

#### 安全场景

**MyBatis 内置方法**

```java
// 这里以增加功能为例
// Controller层
public R safe1(
switch (type) {
    case "add":
        rowsAffected = sqliService.nativeInsert(new Sqli(id, username, password));
        message = (rowsAffected > 0) ? "数据插入成功 username:" + username + " password:" + password : "数据插入失败";
        return R.ok(message);
        ...
}
// Service层
@Override
public int nativeInsert(Sqli user) {
    return sqliMapper.insert(user);
}

// Mapper层
int insert(T entity); 

```

**MyBatis #{} 参数绑定**

```java
// 这里以增加功能为例
// Controller层
public R safe2( 
switch (type) {
    case "add":
        //这里插入数据使用MyBatiX插件生成的方法
        rowsAffected = sqliService.customInsert(new Sqli(id, username, password));
        message = (rowsAffected > 0) ? "数据插入成功 username:" + username + " password:" + password : "数据插入失败";
        return R.ok(message);
        ...
}
// Service层
//自定义SQL-使用#{}
@Override
public int customInsert(Sqli user) {
    return sqliMapper.customInsert(user);
}

// Mapper层
<insert id="customInsert">
    insert into sqli (id,username,password) values (#{id,jdbcType=INTEGER},#{username,jdbcType=VARCHAR},#{password,jdbcType=VARCHAR})
</insert>
```

#### 特殊场景

##### order by 注入

由于使用`#{}`会将对象转化为字符串，形成`order by "user" desc`造成错误，隐藏很多开发会采用`${}`来解决，从而造成了注入。

```java
// Controller层
public R special1OrderBy() {
  List<Sqli> sqlis = new ArrayList<>();
  switch (type) {
      case "raw":
          sqlis = sqliService.orderByVul(field);
          break;
      case "prepareStatement":
          sqlis = sqliService.orderByPrepareStatement(field);
          break;
      case "writeList":
          if (!checkUserInput.checkSqlWhiteList(field)) {
              return R.error("field字段不合法！");
          }
          sqlis = sqliService.orderByWriteList(field);
      ...
// Service层
//自定义SQL-使用#{}
@Override
public List<Sqli> orderByVul(String field) {
    return sqliMapper.orderByVul(field);
}
@Override
public List<Sqli> orderByPrepareStatement(String field) {
    return sqliMapper.orderByPrepareStatement(field);
}
@Override
public List<Sqli> orderByWriteList(String field) {
    return sqliMapper.orderByWriteList(field);
}
// Mapper层
<!--    Order by下的${}拼接注入问题：${}只能用于白名单枚举后的受控SQL结构-->
<select id="orderByVul" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli
    <if test="field != null and field != ''">
        ORDER BY ${field}
    </if>
</select>
<!--    Order by下的#{}写法：#{}只能绑定值，不能绑定列名，所以排序不生效-->
<select id="orderByPrepareStatement" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli
    <if test="field != null and field != ''">
        ORDER BY #{field}
    </if>
</select>
<!--    Order by下的安全写法：列名先做白名单枚举，再进入${}拼接受控SQL结构-->
<select id="orderByWriteList" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli
    <if test="field != null and field != ''">
        <choose>
            <!-- 排序列名白名单 -->
            <when test="field == 'id' or field == 'username' or field == 'password'">
                ORDER BY ${field}
            </when>
            <otherwise>
                <!-- 默认使用id进行排序 -->
                ORDER BY id
            </otherwise>
        </choose>
    </if>
</select>
```

##### like 注入

模糊搜索时，直接使用`%#{}%`会报错，部分开发为了方便直接改成`%${}%`从而造成注入。

```java
// Controller层
public R special1OrderBy() {
@PostMapping("/special2-Like")
public R special2Like(String type,String keyword) {
    List<Sqli> sqlis = new ArrayList<>();
    switch (type) {
        case "raw":
            sqlis = sqliService.likeVul(keyword);
            break;
        case "prepareStatement":
            sqlis = sqliService.likePrepareStatement(keyword);
            break;
    ...
// Service层
@Override
public List<Sqli> orderByWriteList(String field) {
    return sqliMapper.orderByWriteList(field);
}
@Override
public List<Sqli> likeVul(String keyword) {
    return sqliMapper.likeVul(keyword);
}
// Mapper层
<!--  模糊查询-->
<select id="likeVul" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli WHERE username LIKE '%${keyword}%'
</select>
<select id="likePrepareStatement" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli WHERE username LIKE CONCAT('%', #{keyword}, '%')
</select>
```

##### in 注入

in 之后多个 id 查询时使用`#`同样会报错，从而造成注入。

```java
// Controller层
public R special3In(String type,String scope) {
  switch (type) {
      case "raw":
          sqlis = sqliService.inVul(scope);
          break;
      case "prepareStatement":
          sqlis = sqliService.inPrepareStatement(scope);
          break;
      case "Foreach":
          List<Integer> idList = parseInputToList(scope);
          if (idList.isEmpty()) {
              return R.error("scope中没有合法整数ID!");
          }
          sqlis = sqliService.inSafeForeach(idList);
          break;
  ...
// Service层
@Override
public List<Sqli> inVul(String scope) {
    return sqliMapper.inVul(scope);
}
@Override
public List<Sqli> inPrepareStatement(String scope) {
    return sqliMapper.inPrepareStatement(scope);
}
@Override
public List<Sqli> inSafeForeach(List<Integer> scope) {
    return sqliMapper.inSafeForeach(scope);
}
// Mapper层
<select id="inVul" resultType="top.whgojp.modules.sqli.entity.Sqli">
    select * from sqli where id in (${id})
</select>

<select id="inPrepareStatement" resultType="top.whgojp.modules.sqli.entity.Sqli">
    select * from sqli where id in (#{id})
</select>
<select id="inSafeForeach" resultType="top.whgojp.modules.sqli.entity.Sqli">
    SELECT * FROM sqli WHERE id IN
    <foreach collection="scope" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

### Hibernate

Hibernate 需要同时关注 HQL 和原生 SQL：参数值用setParameter绑定，动态列名、排序方向等SQL结构用枚举映射。测试时建议关注SQL执行结果和后端日志，并区分HQL语法与原生SQL语法差异。

#### 漏洞场景

##### 原生 SQL 注入

```java
public R vul1(@RequestParam String username) {
    try {
        String sql = "SELECT * FROM sqli WHERE username = '" + username + "'";
        Object[] result = (Object[]) hibernateTemplate.execute(session ->
                session.createNativeQuery(sql).uniqueResult()
        );
        message = "查询成功，用户名：" + result[1] + " 密码：" +result[2];
        return R.ok(message);
    } catch (Exception e) {
        log.error("查询失败", e);
        return R.error(e.getMessage());
    }
}
```

##### HQL 注入

```java
public R vul2(@RequestParam String username) {
    try {
        String hql = "FROM Sqli WHERE username = '" + username + "'";
        Sqli result = (Sqli) hibernateTemplate.execute(session ->
                session.createQuery(hql).uniqueResult()
        );
        message = "查询成功，用户名：" +result.getUsername()+ " 密码：" +result.getPassword();
        return R.ok(message);
    } catch (Exception e) {
        log.error("查询失败", e);
        return R.error(e.getMessage());
    }
}
```

#### 安全场景

```java
public R safe(@RequestParam String username) {
    try {
        String hql = "FROM Sqli WHERE username = :username";
        Sqli result = hibernateTemplate.execute(session ->
                (Sqli) session.createQuery(hql)
                        .setParameter("username", username)
                        .uniqueResult()
        );
        message = "查询成功，用户名：" +result.getUsername()+ " 密码：" +result.getPassword();
        return R.ok(message);
    } catch (Exception e) {
        log.error("查询失败", e);
        return R.error(e.getMessage());
    }
}
```

### JPA

#### 漏洞场景

##### JPQL 注入

```java
public R vul1(@RequestParam String username) {
    try {
        String jpql = "SELECT s FROM Sqli s WHERE s.username = '" + username + "'";
        Query query = entityManager.createQuery(jpql);
        List<Sqli> results = query.getResultList();
        if (results == null || results.isEmpty()) {
            return R.error("未找到记录");
        }
        StringBuilder sb = new StringBuilder();
        sb.append("查询成功，找到 ").append(results.size()).append(" 条记录\n");
        message = sb.toString();
        log.info(message);
        return R.ok(message);
    } catch (Exception e) {
        String errorMsg = e.getMessage();
        log.error("查询失败: {}", errorMsg, e);
        return R.error(errorMsg);
    }
}
```

##### 动态排序注入

```java
public R vul2(@RequestParam String orderBy) {
    try {
        String jpql = "SELECT s FROM Sqli s ORDER BY s." + orderBy;
        Query query = entityManager.createQuery(jpql);
        List<Sqli> results = query.getResultList();
        return R.ok(formatResults(results));
    } catch (Exception e) {
        String errorMsg = e.getMessage();
        log.error("查询失败: {}", errorMsg, e);
        return R.error(errorMsg);
    }
}
```

#### 安全场景

##### JPA 参数化查询

```java
public R safe(@RequestParam String username) {
    try {
        String jpql = "SELECT s FROM Sqli s WHERE s.username = :username";
        Query query = entityManager.createQuery(jpql)
                .setParameter("username", username);
        List<Sqli> results = query.getResultList();
        if (results == null || results.isEmpty()) {
            return R.error("未找到记录");
        }
        StringBuilder sb = new StringBuilder();
        sb.append("查询成功，找到 ").append(results.size()).append(" 条记录\n");
        message = sb.toString();
        log.info(message);
        return R.ok(message);
    } catch (Exception e) {
        String errorMsg = e.getMessage();
        log.error("查询失败: {}", errorMsg, e);
        return R.error(errorMsg);
    }
}
```

##### 动态排序白名单

```java
public R safeOrder(@RequestParam String orderBy) {
    try {
        Map<String, String> orderByMap = new HashMap<>();
        orderByMap.put("id", "id");
        orderByMap.put("username", "username");
        orderByMap.put("password", "password");

        String safeOrderBy = orderByMap.get(orderBy);
        if (safeOrderBy == null) {
            return R.error("排序字段不合法");
        }

        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Sqli> cq = cb.createQuery(Sqli.class);
        Root<Sqli> root = cq.from(Sqli.class);
        cq.select(root).orderBy(cb.asc(root.get(safeOrderBy)));

        List<Sqli> results = entityManager.createQuery(cq).getResultList();
        return R.ok(formatResults(results));
    } catch (Exception e) {
        String errorMsg = e.getMessage();
        log.error("查询失败: {}", errorMsg, e);
        return R.error(errorMsg);
    }
}
```

## XXE
代码审计 SINK 点：

1. XMLReader
2. SAXReader
3. DocumentBuilder
4. XMLStreamReader
5. SAXBuilder
6. SAXParser
7. SAXSource
8. TransformerFactory
9. SAXTransformerFactory
10. SchemaFactory
11. Unmarshaller
12. XPathExpression

### 漏洞场景

**XMLReader**

```java
public String vul1(String payload) {
    try {
        XMLReader xmlReader = XMLReaderFactory.createXMLReader();
        StringWriter stringWriter = new StringWriter();
        xmlReader.setContentHandler(new DefaultHandler() {
            public void characters(char[] ch, int start, int length) {
                for (int i = start; i < start + length; i++) {
                    if (ch[i] == '\n') {
                        stringWriter.write("<br/>");
                    } else {
                        stringWriter.write(ch[i]);
                    }
                }
            }
        });
        xmlReader.parse(new InputSource(new StringReader(payload)));
        return stringWriter.toString();
    } catch (Exception e) {
        return e.getMessage();
    }
}
```

**SAXParser**

```java
public String vul2(String payload) {
    try {
        SAXParserFactory factory = SAXParserFactory.newInstance();
        SAXParser parser = factory.newSAXParser();
        ...
        parser.parse(new InputSource(new StringReader(payload)), handler);
        return stringWriter.toString();
    } catch (Exception e) {
        return e.toString();
    }
}
```

**DocumentBuilder**

```java
public String vul3(String payload) {
    try {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder builder = factory.newDocumentBuilder();
        Document document = builder.parse(new InputSource(new StringReader(payload)));
        return document.getDocumentElement().getTextContent();
    } catch (Exception e) {
        return e.toString();
    }
}
```

### 安全场景
安全编码建议：
1. 解析不可信 XML 时显式禁用 DOCTYPE、外部通用实体、外部参数实体和外部 DTD 加载。
2. 配置 EntityResolver 或同类机制，确保解析器不会访问本地文件、内网地址或远程 DTD。
3. 限制 XML 大小、解析深度和实体展开，避免 Billion Laughs 等拒绝服务攻击。
4. 黑名单只能作为辅助检测，不应替代解析器安全配置。

**禁用外部实体引用**

```java
public String safe1(String payload) {
    try {
        XMLReader xmlReader = XMLReaderFactory.createXMLReader();
        // 禁用外部实体引用，防止XXE攻击
        xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
        xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        xmlReader.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        xmlReader.setEntityResolver((publicId, systemId) -> new InputSource(new StringReader("")));
         ...
        xmlReader.parse(new InputSource(new StringReader(payload)));
        return stringWriter.toString();
    } catch (Exception e) {
        return e.getMessage();
    }
}
```

**DocumentBuilder 安全配置**

```java
public String safe3(String payload) {
    try {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        factory.setXIncludeAware(false);
        factory.setExpandEntityReferences(false);
        setAttributeIfSupported(factory, XMLConstants.ACCESS_EXTERNAL_DTD, "");
        setAttributeIfSupported(factory, XMLConstants.ACCESS_EXTERNAL_SCHEMA, "");
        DocumentBuilder builder = factory.newDocumentBuilder();
        builder.setEntityResolver((publicId, systemId) -> new InputSource(new StringReader("")));
        ...
    } catch (Exception e) {
        return e.toString();
    }
}
private void setAttributeIfSupported(DocumentBuilderFactory factory, String name, String value) {
    try {
        factory.setAttribute(name, value);
    } catch (IllegalArgumentException ignored) {
    }
}
```

## RCE

### 命令注入

#### ProcessBuilder

```java
public R vul1(String payload) throws IOException {
    String[] command = {"sh", "-c",payload};

    ProcessBuilder pb = new ProcessBuilder(command);
    pb.redirectErrorStream(true);
    Process process = pb.start();
    InputStream inputStream = process.getInputStream();
    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
    String line;
    StringBuilder output = new StringBuilder();
    while ((line = reader.readLine()) != null) {
        output.append(line).append("\n");
    }
    return R.ok(output.toString());
}
```

`ProcessBuilder`在调用列表参数的构造器时，会依次把参数传递给操作系统，但是第一个参数不会被 shell 解析，而是会去寻找第一个参数对应的可执行程序，并以这个程序开启一个进程，后面的参数就会被传入这个进程。如果第一个参数是`sh`，系统就会去调用`/bin/sh`这个可执行程序，后续的参数就被传入`sh`中解析。

#### Runtime.getRuntime.exec()

```java
public R vul2(String payload) throws IOException {
    StringBuilder sb = new StringBuilder();
    String line;
    Process proc = Runtime.getRuntime().exec(payload);
    InputStream inputStream = proc.getInputStream();
    InputStreamReader isr = new InputStreamReader(inputStream);
    BufferedReader br = new BufferedReader(isr);
    while ((line = br.readLine()) != null) {
        sb.append(line);
    }
    return R.ok(sb.toString());
}
```

`Runtime.exec()`内部其实就是用`ProcessBuilder`来实现的：

```java
Runtime.getRuntime().exec(String command);
// 等价于：
new ProcessBuilder(command.split(" ")).start();  // 按空格拆分！
```

**关键差异**：

- `ProcessBuilder` 让你**显式传递参数列表**（安全）
- `Runtime.exec()` 让你**传递一个字符串**，JVM帮你按空格拆分（容易出错）

| 方法签名                                 | 参数形式          | 安全性         | 说明                           |
| :--------------------------------------- | :---------------- | :------------- | :----------------------------- |
| `exec(String command)`                   | 单个字符串        | ❌ **高危**     | JVM按空格拆分，容易注入        |
| `exec(String[] cmdarray)`                | 字符串数组        | ⚠️ **相对安全** | 和`ProcessBuilder`列表模式类似 |
| `exec(String command, String[] envp)`    | 字符串 + 环境变量 | ❌ **高危**     | 同上                           |
| `exec(String[] cmdarray, String[] envp)` | 数组 + 环境变量   | ⚠️ **相对安全** | 同上                           |

#### ProcessImpl

其实`ProcessImpl`才是所有命令执行方法的底层执行者，上面两种最终都是依赖于`ProcessImpl`实现命令执行的。

```java
public R vul3(String payload) throws Exception {
    // 获取 ProcessImpl 类对象
    Class<?> clazz = Class.forName("java.lang.ProcessImpl");

    // 获取 start 方法
    Method method = clazz.getDeclaredMethod("start", String[].class, Map.class, String.class, ProcessBuilder.Redirect[].class, boolean.class);
    method.setAccessible(true);

    Process process = (Process) method.invoke(null, new String[]{payload}, null, null, null, false);
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
        StringBuilder output = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            output.append(line).append("\n");
        }
        return R.ok(output.toString());
    }
}
```

### 代码注入

#### Groovy 代码注入

Groovy 是一种基于 JVM 的动态语言，语法简洁，支持闭包、动态类型和 Java 互操作性，常用于脚本开发和自动化任务。

```java
public R vulGroovy(String payload) {
    try {
        GroovyShell shell = new GroovyShell();
        Object result = shell.evaluate(payload); 
        if (result instanceof Process) {
            Process process = (Process) result;
            String output = getProcessOutput(process);
            return R.ok("[+] Groovy代码执行，结果：" + output);
        } else {
            return R.ok("[+] Groovy代码执行，结果：" + result.toString());
        }
    } catch (Exception e) {
        return R.error(e.getMessage());
    }
}
private String getProcessOutput(Process process) {
    StringBuilder output = new StringBuilder();
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()))) {
        String line;
        while ((line = reader.readLine()) != null) {
            output.append(line).append("\n");
        }
    } catch (Exception e) {
        return "读取输出失败: " + e.getMessage();
    }
    return output.toString();
}
```

payload：`'calc'.execute()`。

## SSRF
代码审计 SINK 点：
1. URL
2. URLConnection
3. HttpURLConnection
4. HttpClient
5. OkHttp
6. RestTemplate
7. WebClient
8. Socket
9. ImageIO
10. JNDI
11. DriverManager.getConnection

```java
@GetMapping("/internal/metadata")
public String internalMetadata() {
    return "instance-id: i-javaseclab-ssrf ...";
}

@GetMapping("/redirect")
public void redirect(String target, HttpServletResponse response) throws IOException {
    response.sendRedirect(target);
}

public String vul(String url) {
    try {
        URL u = new URL(url);
        // URLConnection默认可请求file/http等协议，HTTP请求还可能自动跟随跳转
        URLConnection conn = u.openConnection();
        BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()));
        String content;
        StringBuilder html = new StringBuilder();
        html.append("<pre>");
        while ((content = reader.readLine()) != null) {
            html.append(content).append("\n");
        }
        html.append("</pre>");
        reader.close();
        return html.toString();
    } catch (Exception e) {
        return e.getMessage();
    }
}
```

































