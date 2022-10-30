# 一次insert操作过程
以保存一条记录到表中这个操作为例，就按这个例子来跟踪mybatis是如何执行sql语句的，要保存一个user记录到表中：
```java
sqlSession.insert("x.y.insertUser", user);
```
首先看看`SqlSession#insert`方法，到`DefaultSqlSession`这个实现类中查找具体的实现，下面两个代码片段是insert方法实现中的调用链：

```java
@Override
public int insert(String statement, Object parameter) {
    return update(statement, parameter);
}
```
```java
@Override
public int update(String statement, Object parameter) {
    try {
        dirty = true;
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.update(ms, wrapCollection(parameter));
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```
第一段代码中显示`insert`方法中调用到了`update`方法，通过`update`方法实现插入操作，另外`delete`方法实际上也是委托`update`方法完成删除操作的。

接着第二段代码，在update方法内部，首先获取了一个`MappedStatement`对象，这个对象实际上封装的就是mapper映射文件中的各个insert、delete、update的配置节点的信息，在mybatis初始化`Configuration`对象时就解析好了。

接着通过executor对象完成update操作，executor在创建DefaultSqlSession时就初始化了这个引用。

`Executor`是mybatis的底层核心组件，SqlSession提供了一套面向用户的操作接口，这些方法的底层都是通过`Executor`提供支持，`Executor`定义了一套和SqlSession相似的操作方法，有点像外观模式。

MyBatis内部提供了多种Executor具体实现，下面是它的体系结构图：
![mybatis- exectuor扩展结构](https://oscimg.oschina.net/oscnet/up-cca172f840f1e49693eb50e4fab198a580c.JPEG)

默认MyBatis使用`SimpleExecutor`，如何使用其他类型的Executor之后会提到。因此要看一下SimpleExector中update的实现，在BaseExecutor中有一个基本实现，然后通过`doUpdate`这个模板方法实现到具体子类的调用：
```java
// BaseExecutor:
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache(); // 清理缓存结果
    return doUpdate(ms, parameter); // 通过doUpdate抽象方法调用具体子类的实现
}
```
```java
// SimpleExecutor:
@Override
public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
        stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.update(stmt);
    } finally {
        closeStatement(stmt);
    }
}
```
如上所示，在SimpleExecutor中，通过configuration对象创建了一个`StatementHandler`，这里`StatementHandler`是MyBatis内部另一个重要核心对象，它是一个java接口，事实上对于数据库的操作是在StatementHandler组件中完成的，而Executor则是一个分发器，把不同种类操作分发给StatementHandler对象处理，StatementHandler基本上实现了全部的操作过程：从sql语句创建Statement对象、替换sql参数占位符、查询、更新以及处理结果集等等，从下面贴出的StatementHandler接口定义可以看到这些操作：
```java
public interface StatementHandler {
    // 创建jdbc Statement对象
    Statement prepare(Connection connection, Integer transactionTimeout)
        throws SQLException;

    // 执行参数化sql的参数填充
    void parameterize(Statement statement)
        throws SQLException;

    // 批量update操作，对应jdbc的addBatch操作
    void batch(Statement statement)
        throws SQLException;

    int update(Statement statement)
        throws SQLException;

    <E> List<E> query(Statement statement, ResultHandler resultHandler)
        throws SQLException;

    <E> Cursor<E> queryCursor(Statement statement)
        throws SQLException;

    // 获取绑定的sql语句对象
    BoundSql getBoundSql();

    ParameterHandler getParameterHandler();
}
```
针对PreparedStatement、CallableStatement以及Statement类型的sql，StatementHandler分别提供了具体的实现类，它的类扩展结构图如下：
![](https://oscimg.oschina.net/oscnet/up-944b324d7e68066383ae95d5f2be98b9c42.JPEG)

图中的`RoutingStatementHandler`和其他三种实现不同，它是一个具有分发功能的实现，把具体的请求转发到另外三种具体的StatementHandler，它里面持有一个其他特定类型的StatementHandler引用`delegate`，根据不同的sql种类，把请求委托给具体的StatementHandler，看看RoutingStatementHandlerde的代码就清楚了：

```java
// RoutingStatementHandler:
public class RoutingStatementHandler implements StatementHandler {
	// 委托对象
    private final StatementHandler delegate;

    // 根据不同种类的sql，把delegate对象初始化为某种具体的StatementHandler
    public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

        switch (ms.getStatementType()) {
            case STATEMENT:
                delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case PREPARED:
                delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            case CALLABLE:
                delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            default:
                throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
        }
    }

    @Override
    public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
        return delegate.prepare(connection, transactionTimeout);
    }

    @Override
    public void parameterize(Statement statement) throws SQLException {
        delegate.parameterize(statement);
    }
    
    // ... 省略其他StatementHandler的实现方法
}
```

现在再回到上面标注”SimpleExecutor:“处的代码片段：

首先，里面通过configuration创建的StatementHandler就是一个`RoutingStatementHandler`；

第二步，通过prepareStatement方法获得一个Statement对象：

```java
// SimpleExecutor#prepareStatement:

private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 在方法的底层从数据源获取Connection对象，并设置了事务提交属性和事务隔离级别
    Connection connection = getConnection(statementLog);
    // 通过刚才说的RoutingStatementHandler得prepare方法创建一个Statement对象；
    // RoutingStatementHandler里面则委托给具体的StatementHandler，
    // 如果sql mapper中没有特殊配置，那么应该是PreparedStatementHandler
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 最后填充参数占位符
    handler.parameterize(stmt);
    return stmt;
}
```
此处是替换参数占位符的方法调用，PreparedStatementHandler中是通过一个`ParameterHandler`实现的，PreparedStatementHandler还有一个`ResultHandler`，这是用来处理查询结果集的。可以看到Executor通过StatementHandler、ParameterHandler、ResultHandler几个对象合作来完成整个数据库的操作。

第三步，调用了`handler.update(stmt)`，根据上面的解释知道，这句调用委托到了`PreparedStatementHandler#update`方法，看看最终的update实现：
```java
// PreparedStatementHandler#update:

@Override
public int update(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    // 执行语句
    ps.execute();
    // 获取影响行数
    int rows = ps.getUpdateCount();
    // 下面获取产生的key
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
    return rows;
}
```
到这里整个insert数据库操作过程完成了。

# Executor的创建
前面说过DefaultSqlSession中的executor引用在初始化时设置引用值，这是通过`DefaultSqlSessionFactory`创建的，以我们通常创建一个SqlSession对象的例子为例：

```java
SqlSessionFactory sqlSessionFactory = SqlSessionFactoryBuilder.build(stream);
SqlSession sqlSession = sqlSessionFactory.open();
```
SqlSessionFactoryBuilder.build()方法里面创建的实际就是一个DefaultSqlSessionFactory类型的实例，下面我们跟踪DefaultSqlSessionFactory里的流程：
```java
// DefaultSqlSessionFactory#openSession:
public SqlSession openSession() {
    // 获取了默认的executor type
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```
```java
// DefaultSqlSessionFactory#openSessionFromDataSource:
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 创建一个executor对象
        final Executor executor = configuration.newExecutor(tx, execType);
        // 创建DefaultSqlSession对象，把executor对象传递给它
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

下面看看Executor实例的创建代码：
```java
// Configuration#newExecutor:
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```
newExecutor方法中根据参数executorType来创建具体类型的Executor实例，在openSession方法中executorType的值设置Configuration对象中的默认值，Configuration中默认设置的就是`SIMPLE`：
```java
protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
```

因此，默认我们使用`openSession()`或者`openSession(autoCommit)`方法时内部得到的是`SimpleExecutor`实例，当然我们可以修改默认值，Configuration类有defaultExecutorType的setter方法，对应到配置文件中就是设置`defaultExecutorType`属性值为`SIMPLE`或者`REUSE`或者`BATCH`即可，另一种方式是直接使用SqlSession的其他重载形式：
```java
sqlSession.openSession(ExecutorType.REUSE);
// 或者：
sqlSession.openSession(ExecutorType.REUSE, true);
```

# 几个重要类的依赖关系
下面是mybatis中几个重要类的依赖关系图，通过此图有助于理解mybatis是如何实现事物控制的。
![](https://oscimg.oschina.net/oscnet/up-2ef845299fc1660a575f65e2337b918ea7d.JPEG)