---
title: 【数据库】数据库杂记(一)之JDBC
date: 2023-6-7
tags: [数据库,MySQL,Java]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/2022_06_01_10.webp
mathjax: true
---
# 数据库杂记(一)之JDBC

## 1. JDBC

### 1.1 JDBC QuickStart

使用JDBC操作数据库的步骤

1. 注册驱动

```java
Class.forName("com.mysql.jdbc.Driver")
```

2. 获取连接

```java
Connection conn = DriverManager.getConnection(url,username,password)
```

3. 定义SQL语句

```java
String sql = "update...";
```

4. 获取执行SQL对象

```java
Statement stmt = conn.createStatement();
```

5. 执行SQL

```java
stmt.executeUpdate(sql);
```

6. 处理返回结果
7. 释放资源

### 1.2 JDBC API

#### 1.2.1 DriverManage(驱动管理类)作用:
+ 注册驱动


```java
//1. 注册驱动
//在加载类的时候会调用静态方法 
Class.forName("com.mysql.jdbc.Driver")
// 向 DriverManager 注册给定驱动程序。新加载的驱动程序类应该调用 registerDriver 方法让 DriverManager 知道自己。
public static void registerDriver(Driver driver) 
    

```
+ 获取数据库链接
```java
//2. 获取数据库连接
String url = "jdbc:mysql://localhost:3306/sql_store?useUnicode=true&characterEncoding=UTF-8&userSSL=false&serverTimezone=GMT%2B8";
String username = "root";
String password = "2154477";
Connection conn = DriverManager.getConnection(url, username, password);
```

#### 1.2.2 Connection(数据库连接对象)作用

+ 获取执行SQL的对象

```java
//1. 获取执行SQL的对象
// 普通执行SQL对象
Statement createStatement()
//预编译SQL的执行SQL对象:防止SQL注入
PreparedStatement preparedStatement(sql)
//执行存储过程的对象
CallableStatement prepareCall(sql)
```

+ 事务管理

MySQL事务管理

```sql
开启事务:BEGIN;
提交事务:COMMIT;
回滚事务:ROLLBACK;

MySQL默认自动提交事务
```

JDBC事务管理:Connection 接口中定义了3个对应的方法

```java
开启事务:setAutoCommit(boolean autoCommit);true 为自动提交事务;false为手动提交事务
提交事务:commit();
回滚事务:rollback();
```

样例代码：

```java
    public static void main(String[] args) throws Exception {
        //1. 注册驱动
//        Class.forName("com.mysql.cj.jdbc.Driver");
        //2. 获取连接
        String url = "jdbc:mysql://localhost:3306/sql_store?useUnicode=true&characterEncoding=UTF-8&userSSL=false&serverTimezone=GMT%2B8";
        String username = "root";
        String password = "2154477";
        Connection conn = DriverManager.getConnection(url, username, password);

        //3. 定义sql
        String sql1 = "UPDATE shippers_copy SET name = 'a' WHERE shipper_id = 5";
        String sql2 = "UPDATE shippers_copy SET name = 'b' WHERE shipper_id = 4";

        //4. 获取执行sql的对象
        Statement stmt = conn.createStatement();



        try {
            //开启事务
            conn.setAutoCommit(false);
            //5. 执行sql,返回受影响的行数
            int cnt1 = stmt.executeUpdate(sql1);
            //6. 处理结果
            System.out.println(cnt1);

            //int i = 3/0;

            //5. 执行sql,返回受影响的行数
            int cnt2 = stmt.executeUpdate(sql2);
            //6. 处理结果
            System.out.println(cnt2);
            //提交事务
            conn.commit();
        } catch (Exception throwables) {
            //回滚事务
            conn.rollback();
            throwables.printStackTrace();
        }



        //7. 释放资源
        stmt.close();
        conn.close();
    }
```



#### 1.2.3 Statement 作用

执行SQL语句

```java
int executeUpdate(sql):执行DML、DDL语句
    返回值:(1)DML语句影响的行数 (2)DDL语句执行后，执行成功也可能返回0
        
ResultSet executeQuery(sql):执行DQL语句
    返回值:ResultSet结果集对象
```

#### 1.2.4 ResultSet(结果集对象)

封装了DQL查询语句的结果

```java
ResultSet stmt.executeQuery(sql):执行DQL语句，返回ResultSet对象
```

获取查询结果

```java
boolean next()
    (1)将光标从当前位置向前移动一行，默认初始指向数据行的上一行
    (2)判断当前行是否为有效行
    返回值:true 有效行 false 无效行
        
xxx getXxx(参数):获取数据
        xxx:数据类型:int getInt(参数);String getString(参数)
        参数:int :列的编号，从1开始
            String :列的名称
```

样例代码:

```java

public static void main(String[] args) throws Exception {

    //1.获取数据库连接
    String url = "jdbc:mysql://127.0.0.1:3306/learn_jdbc";
    String user = "root";
    String password = "100228";
    Connection conn = DriverManager.getConnection(url, user, password);

    //2.定义Sql语句
    String sql = "SELECT * from user;";

    //3.获取执行SQL的对象statement
    Statement stmt = conn.createStatement();

    //4.执行sql语句,获的res对象
    ResultSet res = stmt.executeQuery(sql);

    //5.处理结果，遍历res中的所有数据
    //5.1光标下移一行，并判断当前行是否有效
    while (res.next()){
        //5.2获取数据
        int id = res.getInt(1);
        String name = res.getString(2);
        int age = res.getInt(3);

        System.out.println(id);
        System.out.println(name);
        System.out.println(age);
        System.out.println("***************");

    }
    //6.释放资源
    res.close();
    stmt.close();
    conn.close();
}
```

#### 1.2.5 PreparedStatement

预编译SQL语句并执行，预防SQL注入问题

SQL注入DEMO

```java
   @Test
    public void testLogin_Inject() throws Exception {
        //1. 注册驱动
//        Class.forName("com.mysql.cj.jdbc.Driver");
        //2. 获取连接
        String url = "jdbc:mysql://localhost:3306/db1?useUnicode=true&characterEncoding=UTF-8&userSSL=false&serverTimezone=GMT%2B8";
        String username = "root";
        String password = "2154477";
        Connection conn = DriverManager.getConnection(url, username, password);

        //接收用户输入 用户名和密码
        String name = "szhgfewbhf";
        String pwd = "' or '1' = '1";

        String sql = "select * from tb_user where username = '" + name + "' and password = '" + pwd + "'";
        System.out.println(sql);
		//select * from tb_user where username = 'szhgfewbhf' and password = '' or '1' = '1'
        Statement stmt = conn.createStatement();

        ResultSet rs = stmt.executeQuery(sql);

        //判断登录是否成功,如果拿到数据就登陆成功
        if (rs.next()) {
            System.out.println("Login Success!");
        } else {
            System.out.println("Login failed!");
        }
        //7. 释放资源
        rs.close();
        stmt.close();
        conn.close();
    }
```

获取PreparedStatement对象

```java
//SQL语句中的参数值,使用?占位符代替
String sql = "select * from user where username = ? and password = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
```

PreparedStatement是将特殊字符进行了转义，转义的SQL如下：

```sql
select * from tb_user where username = 'szhgfewbhf' and password = '\'or \'1\' = \'1'
```

设置参数值

```java
PreparedStatement对象: setXxx(参数1,参数2):给?赋值
    Xxx:数据类型;如setInt(参数1,参数2)
    参数:
		参数1:?的位置编号,从1开始
        参数2:?的值
```

执行SQL不需要再传递sql

```java
executeUpdate();
executeQuery();
```

#### 1.2.6 PreparedStatement简要原理

PreparedStatement 好处：

* 预编译SQL，性能更高
* 防止SQL注入：==将敏感字符进行转义==

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/image-20210725195756848.webp)

Java代码操作数据库流程如图所示：

* 将sql语句发送到MySQL服务器端

* MySQL服务端会对sql语句进行如下操作

  * 检查SQL语句

    检查SQL语句的语法是否正确。

  * 编译SQL语句。将SQL语句编译成可执行的函数。

    检查SQL和编译SQL花费的时间比执行SQL的时间还要长。如果我们只是重新设置参数，那么检查SQL语句和编译SQL语句将不需要重复执行。这样就提高了性能。

  * 执行SQL语句

* 开启预编译功能

  在代码中编写url时需要加上以下参数。而我们之前根本就没有开启预编译功能，只是解决了SQL注入漏洞。

  ```sql
  useServerPrepStmts=true
  ```

```java
 /**
   * PreparedStatement原理
   * @throws Exception
   */
@Test
public void testPreparedStatement2() throws  Exception {

    //2. 获取连接：如果连接的是本机mysql并且端口是默认的 3306 可以简化书写
    // useServerPrepStmts=true 参数开启预编译功能
    String url = "jdbc:mysql:///db1?useSSL=false&useServerPrepStmts=true";
    String username = "root";
    String password = "1234";
    Connection conn = DriverManager.getConnection(url, username, password);

    // 接收用户输入 用户名和密码
    String name = "zhangsan";
    String pwd = "' or '1' = '1";

    // 定义sql
    String sql = "select * from tb_user where username = ? and password = ?";

    // 获取pstmt对象
    PreparedStatement pstmt = conn.prepareStatement(sql);

    Thread.sleep(10000);
    // 设置？的值
    pstmt.setString(1,name);
    pstmt.setString(2,pwd);
    ResultSet rs = null;
    // 执行sql
    rs = pstmt.executeQuery();

    // 设置？的值
    pstmt.setString(1,"aaa");
    pstmt.setString(2,"bbb");
    // 执行sql
    rs = pstmt.executeQuery();

    // 判断登录是否成功
    if(rs.next()){
        System.out.println("登录成功~");
    }else{
        System.out.println("登录失败~");
    }

    //7. 释放资源
    rs.close();
    pstmt.close();
    conn.close();
}
```

### 1.3 数据库连接池

#### 1.3.1  数据库连接池简介

> * 数据库连接池是个容器，负责分配、管理数据库连接(Connection)
>
> * 它允许应用程序重复使用一个现有的数据库连接，而不是再重新建立一个；
>
> * 释放空闲时间超过最大空闲时间的数据库连接来避免因为没有释放数据库连接而引起的数据库连接遗漏
> * 好处
>   * 资源重用
>   * 提升系统响应速度
>   * 避免数据库连接遗漏

之前我们代码中使用连接是没有使用都创建一个Connection对象，使用完毕就会将其销毁。这样重复创建销毁的过程是特别耗费计算机的性能的及消耗时间的。

而数据库使用了数据库连接池后，就能达到Connection对象的复用，如下图

<img src="https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/image-20210725210432985.webp" alt="image-20210725210432985" style="zoom:80%;" />

连接池是在一开始就创建好了一些连接（Connection）对象存储起来。用户需要连接数据库时，不需要自己创建连接，而只需要从连接池中获取一个连接进行使用，使用完毕后再将连接对象归还给连接池；这样就可以起到资源重用，也节省了频繁创建连接销毁连接所花费的时间，从而提升了系统响应的速度。

#### 1.3.2  数据库连接池实现

* 标准接口：==DataSource==

  官方(SUN) 提供的数据库连接池标准接口，由第三方组织实现此接口。该接口提供了获取连接的功能：

  ```java
  Connection getConnection()
  ```

  那么以后就不需要通过 `DriverManager` 对象获取 `Connection` 对象，而是通过连接池（DataSource）获取 `Connection` 对象。

* 常见的数据库连接池

  * DBCP
  * C3P0
  * Druid

  我们现在使用更多的是Druid，它的性能比其他两个会好一些。

* Druid（德鲁伊）

  * Druid连接池是阿里巴巴开源的数据库连接池项目 

  * 功能强大，性能优秀，是Java语言最好的数据库连接池之一

一般通过创建配置文件来保存数据库连接池的初始化参数,创建druid.properties
```properties
driverClassName=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost:3306/db1?useServerPrepStmts=true&useUnicode=true&characterEncoding=UTF-8&userSSL=false&serverTimezone=GMT%2B8
username=root
password=2154477
# 初始化连接数量
initialSize=5
# 最大连接数
maxActive=10
# 最大等待时间
maxWait=3000
```

样例demo

```java
    public static void main(String[] args) throws Exception {
        //1. 导入jar包
        //2. 定义配置文件

        //3. 加载配置文件
        Properties prop = new Properties();
        //4. 获取连接池对象

        prop.load(new FileInputStream("src\\main\\resources\\druid.properties"));

        DataSource dataSource = DruidDataSourceFactory.createDataSource(prop);
        //5. 获取数据库连接
        Connection connection = dataSource.getConnection();

        System.out.println(connection);
    }
```

