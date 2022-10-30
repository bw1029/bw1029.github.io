mybatis是一个sql持久化框架，主要封装了java对象和数据库表两者之间的映射过程，支持动态SQL、存储过程等。

# 创建SqlSessionFactory

使用mybatis首先需要构造一个`SqlSessionFactory`对象，`SqlSessionFactory`是mybatis的核心对象，可以通过加载一段xml配置来构建一个SqlSessionFactory对象，可以从任何地方加载配置信息，但推荐从classpath中加载配置文件，下面一个加载xml配置的示例： 

```java 
String resource = "mybatis-config.xml"; 
// 使用mybatis提供的工具类Resources加载资源 
InputStream stream = Resources.getResourceAsStream(resource); 
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(stream); 
```

加载xml配置来创建`SqlSessionFactory`对象是最方便和推荐的方式。

mybatis也支持编程的方式创建`SqlSessionFactory`，这需要我们首先创建一个`Configuration`对象，`Configuration`对象封装所有的核心配置信息，下面是通过创建`Configuration`对象的方式构建`SqlSessionFactory`： 

```java 
DataSource dataSource = ...
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment env = new Environment("develop", transactionFactory, dataSource);
Configuration configuration = new Configuration(env);
configuration.addMapper(UserMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

# 获取SqlSession 

`SqlSession`是mybatis提供暴露给用户操作数据库的入口对象，`SqlSession`可以执行sql命令、运行事务、获取Mapper，`SqlSession`由`SqlSessionFactory`生成。 

``` java
SqlSession sqlSession = sqlSessionFactory.openSession();
```

没有参数的*openSession()*方法创建SqlSession的行为是：

* 采用SimpleExecutor
* 事务自动提交设置为false

SqlSessionFactory还有另外几个重载的openSession方法，可以让我指定executor类型、指定事务隔离级别以及自动提交属性。 

# 核心对象的生命周期 

不正确的管理SqlSessionFactory、SqlSession等对象的生命周期范围，很可能会导致线程并发问题。 

## SqlSessionFactoryBuilder

SqlSessionFactoryBuilder这个对象唯一作用就是创建SqlSessionFactory，一般只会使用一次，当创建SqlSessionFactory对象完成后即可丢弃它，因此SqlSessionFactoryBuilder的范围最好是方法局部的。 

## SqlSessionFactory

SqlSessionFactory被创建后在整个应用程序周期内都需要存在，基本上没有理由需要释放一个SqlSessionFactory对象并重新创建新的对象，而且这样做是不好的使用习惯，创建SqlSessionFactory是个耗时的操作，因此，对SqlSessionFactory的范围管理应该是application级别的。

最好是使用单例模式来创建SqlSessionFactory对象，然后一直存在。 

## SqlSession的范围 

每个线程都应该创建一个SqlSession对象，SqlSession不是线程安全的，因此SqlSession的范围管理应该是线程级别或者方法级别的，不要把一个SqlSession变量声明为静态字段或者实例字段。如果是在web环境中，不要把SqlSession的引用添加到HttpSession中，它应该是和HttpRequest关联的，当接收到一个HttpRequest，就创建一个SqlSession，当响应HttpResponse完成后，SqlSession就可以释放了。 

注意，正确关闭SqlSession对象是必要的步骤，我们应该总是在finally中关闭它： 

```java 
SqlSession sqlSession = sqlSessionFactory.openSession(); 
try { 
    // do some work with sqlSession instance 
} finally { 
    sqlSession.close(); 
} 
```

## Mapper对象的范围 

Mapper实例是用来绑定sql语句的，它由SqlSession创建，因此理论上Mapper的存在周期可以和SqlSession相同，但是最佳实践是在方法中管理Mapper对象，当方法执行结束后Mapper实例就销毁了。 

Mapper实例不需要我们明确的关闭。 

# 配置项 

* defaultExecutorType：指定默认的执行器类型，mybatis有三种Executor：SIMPLE，REUSE和BATCH，mybatis默认设置的是SIMPLE类型。 

# 日志处理

MyBatis的日志是通过自己内部的Log对象来打印的，Log由LogFactory的静态工厂方法获得，LogFactory会自动在类路径下按照一定的顺序查找具体的日志实现库，Log对象封装了这些日志实现产品，会把日志请求委托到具体的日志库处理。mybatis支持多种常见的日志库，LogFactory运行时按照如下顺序搜索日志实现产品： 

1. SLF4J 

2. Apache Commons Logging 

3. Log4j2 

4. Log4j 

5. JDK logging 
6. 不打印日志

LogFactory最终使用第一个找到的实现产品，如果没有找到任何实现类，则日志被丢弃。 

需要注意，只要在类路径下存在Commons Logging组件，则mybatis就会使用Commons logging来处理日志，但是如果仍然期望使用其它日志实现组件来处理日志，则需要在mybatis配置文件中明确指定，下面是配置片段例子： 

```xml 
<settings> 
    <setting name="logImpl" value="LOG4J2"/> 
</settings> 
```

value属性可以写成这些串值：SLF4J、LOG4J、LOG4J2、JDK_LOGGING、COMMONS_LOGGING、STDOUT_LOGGING、NO_LOGGING，除了这些还可以直接写成`org.apache.ibatis.logging.Log`接口的实现类的全限定名。 