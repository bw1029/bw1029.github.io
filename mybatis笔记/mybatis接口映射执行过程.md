用户代码：  
```java
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    userMapper.saveUser(user);
} finally {
    sqlSession.close();
}
```

以上代码片段是使用mybatis接口映射方式进行保存记录，`UserMapper`是一个java接口，mybatis让我们直接调用一个接口方法就可以执行crud操作，这背后机制其实是代理，语句`sqlSession.getMapper(UserMapper.class)`实际上是用动态代理机制返回一个实现了UserMapper接口的代理对象，可以打印`userMapper.getClass().getName()`的结果确认一下，利用IDE调试工具也可以直接看到是一个代理对象。

那到底是如何获得一个代理对象的呢，`SqlSession`接口的默认实现是`DefaultSqlSession`，我们`DefaultSqlSession#getMapper`方法的实现：  
```java
// DefaultSqlSession#getMapper:
public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
}
```

跟踪Configuration#getMapper方法实现：  
```java
// Configuration#getMapper:
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 直接调用了mapperRegistry的getMapper()
    // mapperRegistry是Configuration中的mapper注册容器
    return mapperRegistry.getMapper(type, sqlSession);
}
```

```java
// MapperRegistry#getMapper:
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 从knownMappers容器中读取与mapper类型匹配的MapperProxyFactory对象
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        // throw e
    }
    try {
        // 调用工厂方法产生mapper的代理实例
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        // throw e
    }
}
```

参考上面代码中添加的注释，程序流程从**knownMappers**容器中获取一个与mapper类型匹配的`MapperProxyFactory`实例，这是一个可以创建代理对象的工厂对象，工厂方法`newInstance`获得的就是一个mapper类型的代理实例。这里**knownMappers**是`MapperRegistry`中维护的一个map结构的容器字段：  
```java
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();
```

**knownMappers**容器中缓存了mapper类型和产生mapper类型实例的工厂，缓存内容是在mybatis初始化`Configuration`对象的过程中设置的，我们可以在MapperRegistry中找到添加缓存条目的方法：  

```java
// MapperProxy#addMapper:

public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
            // throw e
        }
        boolean loadCompleted = false;
        try {
            // 在缓存中增加一个条目mapperType -> MapperProxyFactory
            knownMappers.put(type, new MapperProxyFactory<T>(type));
            // ...
        } finally {
            // ...
        }
    }
}
```

现在我们来看看`MapperProxyFactory`这个工厂，如下所示是它的全部实现代码，用注释标注了代码的含义：  
```java
// MapperProxyFactroy:
public class MapperProxyFactory<T> {
    // mapperInterface字段维护着目标mapper类型
    private final Class<T> mapperInterface;
    // 缓存着已经构建好的mapper方法和和其对应的MapperMethod对象；
    // MapperMethod类似一个包含mapper方法的执行上下文信息的封装对象。
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();

    // 构造方法，在前一个代码片段MapperProxy#addMapper里有调用体现
    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
    }

    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
        // 动态代理创建代理实例
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }

    // 对外暴露的工厂方法
    public T newInstance(SqlSession sqlSession) {
        // 首先创建一个MapperProxy实例
        // MapperProxy事实上就是一个InvocationHandler实例
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        // 调用内部重载的newInstance方法返回mapper实例
        return newInstance(mapperProxy);
    }

}
```

注释里基本已经说明了MapperProxyFactory的整体结构和代码流程，主要就是两个重载的newInstance方法体现了利用动态代理来创建mapper代理实例，public newInstance方法内部先实例化了一个`MapperProxy`对象，MapperProxy其实就是一个`InvocationHandler`，通过动态代理创建的代理对象内部都持有一个InvocationHandler的引用，最终的数据库的操作就实现在InvocationHandler的invoke方法中，在创建MapperProxy对象时构造方法接受了几个参数，这些参数在invoke方法中操作数据库时都需要依赖，我们看看它声明的这几个字段以及构造方法：  
```java
// MapperProxy:
private final SqlSession sqlSession;
private final Class<T> mapperInterface;
private final Map<Method, MapperMethod> methodCache;

public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
}
```

**methodCache**字段就是前面说过的methodCache缓存，**sqlSession**指向的就是最开头的用户代码中使用的sqlSession对象，把sqlSession对象传递这么远，因为最终还是通过sqlSession对象的insert、update、select系列方法来实现数据库操作的。

看看`invoke`方法是怎么实现的吧：  
```java
// MapperProxy#invoke:

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {// 如果不是接口
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {// 如果是default方法，java 8支持默认方法
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    // 这里才表示是接口代理
    // 从前面所说的methodCache中取MapperMethod，
    // 缓存中没有就创建一个，可以查看cachedMapperMethod方法的实现
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 通过MapperMethod对象执行实际操作
    return mapperMethod.execute(sqlSession, args);
}

private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
        mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
        methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
}
```

可以看到最终是调用`mapperMethod.execute(sqlSession, args)`执行数据库操作的，前面说过，MapperMethod其实是一个上下文信息对象，它封装的是正在执行的mapper方法的细节，还有就是方法执行数据库操作所需要的环境依赖信息等。

看看它的构造方法和字段：  
```java
// MapperMethod:

private final SqlCommand command;
private final MethodSignature method;
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
}
```

然后我们跟踪它的execute方法实现细节：  
```java
// MapperMethod#execute:

public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        case INSERT: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        throw new BindingException("Mapper method '" + command.getName() 
                                   + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
}
```

execute方法中检测当前sql命令的形式，根据类型是select、insert、upadte、delete或者flush分别采取不同的执行过程，以insert为例，看看insert这个case分支里有：  
```java
result = rowCountResult(sqlSession.insert(command.getName(), param));
```

最终还是进入了sqlSession的insert方法过程，接口映射可以理解为就是利用动态代理包装了SqlSession实例，我们得到的是代理对象，在代理对象上进行crud操作，然后又把请求委托给SqlSession处理。