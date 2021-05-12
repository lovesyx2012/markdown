[TOC]

#### 一 怎么优雅的对一个对象动态代理多次？

##### 粗鲁的方式

示例代码：

```java
/**
* 接口
*/
public interface HelloService{
    void sayHello();
}

/**
* 目标类实现接口
*/
static class HelloServiceImpl implements HelloService{

    @Override
    public void sayHello() {
        System.out.println("sayHello......");
    }
}

public class LogInvocationHandler implements InvocationHandler {
    /**
    * 目标对象
    */
    private Object target;

    public LogInvocationHandler(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("------插入Log前置通知代码-------------");
        //执行相应的目标方法
        Object rs = method.invoke(target,args);
        System.out.println("------插入Log后置通知代码-------------");
        return rs;
    }

    public static Object wrap(Object target) {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                                      target.getClass().getInterfaces(),new LogInvocationHandler(target));
    }
}
    

public class TransactionInvocationHandler implements InvocationHandler {
    /**
     * 目标对象
     */
    private Object target;

    public TransactionInvocationHandler(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("------插入Tx前置通知代码-------------");
        //执行相应的目标方法
        Object rs = method.invoke(target,args);
        System.out.println("------插入Tx后置通知代码-------------");
        return rs;
    }

    public static Object wrap(Object target) {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),new TransactionInvocationHandler(target));
    }
}

    public static void main(String[] args)  {
        HelloService proxyService = (HelloService) FirstInvocationHandler.wrap(new HelloServiceImpl());
        HelloService proxyService2 = (HelloService) SecondInvocationHandler.wrap(proxyService);
        proxyService2.sayHello();
    }
```

执行结果：

```
------插入Tx前置通知代码-------------
------插入Log前置通知代码-------------
sayHello......
------插入Log后置通知代码-------------
------插入Tx后置通知代码-------------
```

##### 优雅的方式

###### 第一步 想办法抽象增强逻辑

将增强逻辑通过Interceptor来抽象

```java
public interface Interceptor {
    /**
     * 具体拦截处理
     */
    void intercept();
}

public class LogInterceptor implements Interceptor {
    @Override
    public void intercept() {
        System.out.println("------插入Log前置通知代码-------------");
    }
}

public class TransactionInterceptor implements Interceptor {
    @Override
    public void intercept() {
        System.out.println("------插入Transaction前置通知代码-------------");
    }
}
```

调整代理类逻辑

```java
public class MyInvocationHandler implements InvocationHandler {
    private Object target;

    private List<Interceptor> interceptorList;

    public MyInvocationHandler(Object target,List<Interceptor> interceptorList) {
        this.target = target;
        this.interceptorList = interceptorList;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //处理多个拦截器
        for (Interceptor interceptor : interceptorList) {
            interceptor.intercept();
        }
        return method.invoke(target, args);
    }

    public static Object wrap(Object target,List<Interceptor> interceptorList) {
        MyInvocationHandler targetProxy = new MyInvocationHandler(target, interceptorList);
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),targetProxy);
    }
}
```

可以实现根据需要动态的添加拦截器了，在每次执行业务代码sayHello()之前都会拦截

```java
public class Test {
    public static void main(String[] args) {
        List<Interceptor> interceptorList = new ArrayList<>();
        interceptorList.add(new LogInterceptor());
        interceptorList.add(new TransactionInterceptor());

        HelloService target = new HelloServiceImpl();
        HelloService targetProxy = (HelloService) MyInvocationHandler.wrap(target,interceptorList);
        targetProxy.sayHello();
    }
}
```

执行结果：

```
------插入Log前置通知代码-------------
------插入Transaction前置通知代码-------------
sayHello......
```

###### 第二步 解决只能做到前置代理，无法做到前后代理

**把拦截对象信息进行封装，作为拦截器拦截方法的参数，把拦截目标对象真正的执行方法放到Interceptor中完成**，这样就可以实现前后拦截，并且还能对拦截

对象的参数等做修改。设计一个`Invocation 对象`。

```java
public class Invocation {
    /**
     * 目标对象
     */
    private Object target;
    /**
     * 执行的方法
     */
    private Method method;
    /**
     * 方法的参数
     */
    private Object[] args;

    //省略getter setter
    public Invocation(Object target, Method method, Object[] args) {
        this.target = target;
        this.method = method;
        this.args = args;
    }

    /**
     * 执行目标对象的方法
     */
    public Object process() throws Exception{
        return method.invoke(target,args);
    }
}
```

调整Interceptor拦截接口，Invocation 类就是被代理对象的封装

```java
public interface Interceptor {
    /**
     * 具体拦截处理
     */
    Object intercept(Invocation invocation) throws Exception;

    /**
     *  插入目标类
     */
    Object plugin(Object target);
}
```

调整拦截接口实现类

```java
public class TransactionInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Exception {
        System.out.println("------插入前置通知代码-------------");
        Object result = invocation.process();
        System.out.println("------插入后置处理代码-------------");
        return result;
    }

    @Override
    public Object plugin(Object target) {
        return null;
    }
}
```

调整InvocationHandler

```java
public class TargetProxy implements InvocationHandler {
    private Object target;

    private Interceptor interceptor;

    public TargetProxy(Object target,Interceptor interceptor) {
        this.target = target;
        this.interceptor = interceptor;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Invocation invocation = new Invocation(target,method,args);
        return interceptor.intercept(invocation);
    }

    public static Object wrap(Object target,Interceptor interceptor) {
        TargetProxy targetProxy = new TargetProxy(target, interceptor);
        Object object = Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),targetProxy);
        return object;
    }
}
```

测试类

```java
public class Test {
    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        Interceptor transactionInterceptor = new TransactionInterceptor();
        HelloService targetProxy = (Target) TargetProxy.wrap(target,transactionInterceptor);
        targetProxy.sayHello();
    }
}
```

运行结果

```
------插入前置通知代码-------------
sayHello......
------插入后置通知代码-------------
```

###### 第三步 解决多个拦截器的处理

定义多个拦截器

```java
public class LogInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Exception {
        System.out.println("------插入Log前置通知代码-------------");
        Object result = invocation.process();
        System.out.println("------插入Log后置处理代码-------------");
        return result;
    }

    @Override
    public Object plugin(Object target) {
        return TargetProxy.wrap(target,this);
    }
}

public class TransactionInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Exception{
        System.out.println("------插入Transaction前置通知代码-------------");
        Object result = invocation.process();
        System.out.println("------插入Transaction后置处理代码-------------");
        return result;
    }

    @Override
    public Object plugin(Object target) {
        return TargetProxy.wrap(target,this);
    }
}
```

测试类

```java
public class Test {
    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        //把事务拦截器插入到目标类中
        Interceptor transactionInterceptor = new TransactionInterceptor();
        target = (HelloService) transactionInterceptor.plugin(target);
        //把日志拦截器插入到目标类中
        LogInterceptor logInterceptor = new LogInterceptor();
        target = (HelloService)logInterceptor.plugin(target);
        target.sayHello();
    }
}
```

运行结果

```
------插入Log前置通知代码-------------
------插入Transaction前置通知代码-------------
sayHello......
------插入Transaction后置处理代码-------------
------插入Log后置处理代码-------------
```

##### 第四步 让添加多个拦截器的代码更丝滑一点

其实上面已经实现的没问题了，只是还差那么一点点，添加多个拦截器的时候不太美观，让我们再次利用面向对象思想封装一下。我们设计一个`InterceptorChain 拦截器链类`

```java
public class InterceptorChain {
    private List<Interceptor> interceptorList = new ArrayList<>();

    /**
     * 插入所有拦截器
     */
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptorList) {
            target = interceptor.plugin(target);
        }
        return target;
    }

    public void addInterceptor(Interceptor interceptor) {
        interceptorList.add(interceptor);
    }
    /**
     * 返回一个不可修改集合，只能通过addInterceptor方法添加
     * 这样控制权就在自己手里
     */
    public List<Interceptor> getInterceptorList() {
        return Collections.unmodifiableList(interceptorList);
    }
}
```

测试类

```java
public class Test {

    public static void main(String[] args) {
        HelloService target = new HelloServiceImpl();
        Interceptor transactionInterceptor = new TransactionInterceptor();
        LogInterceptor logInterceptor = new LogInterceptor();
        InterceptorChain interceptorChain = new InterceptorChain();
        interceptorChain.addInterceptor(transactionInterceptor);
        interceptorChain.addInterceptor(logInterceptor);
        target = (Target) interceptorChain.pluginAll(target);
        target.sayHello();
    }
}
```

运行结果

```
------插入Log前置通知代码-------------
------插入Transaction前置通知代码-------------
sayHello......
------插入Transaction后置处理代码-------------
------插入Log后置处理代码-------------
```



#### 二、JDK动态代理+责任链设计模式

###### 2.1 代理类本质（俄罗斯套娃）

![](https://www.yunsom.com/storage/api/file/200014202102252004480-0b61126403)

###### 2.2 调用的本质（俄罗斯套娃的抽丝剥茧）

![](https://www.yunsom.com/storage/api/file/200014202102252000550-8c2f7d045d)

#### 三、手动实现一个统计慢查询的Mybatis插件

###### 3.1  实现Interceptor接口

```java
@Intercepts({
        @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
        @Signature(type = StatementHandler.class, method = "update", args = {Statement.class}),
        @Signature(type = StatementHandler.class, method = "batch", args = {Statement.class})
})
public class SlowSqlPlugin implements Interceptor {
    private Integer limitSecond;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long beginTimeMillis = System.currentTimeMillis();
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        try {
            return invocation.proceed();
        } finally {
            long endTimeMillis = System.currentTimeMillis();
            long costTimeMills = endTimeMillis - beginTimeMillis;
            if (costTimeMills > limitSecond * 1000) {
                BoundSql boundSql = statementHandler.getBoundSql();
                String sql = getFormattedSql(boundSql);
                System.out.println("SQL语句[" + sql + "]，执行耗时：" + costTimeMills + "ms");
            }
        }
    }

    private String getFormattedSql(BoundSql boundSql) {
        String sql = boundSql.getSql();
        Object parameterObject = boundSql.getParameterObject();
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        sql = beautifySql(sql);
        if (parameterObject == null || parameterMappings == null || parameterMappings.isEmpty()) {
            return sql;
        }
        String sqlWithoutReplacePlaceholder = sql;
        try {
            if (parameterMappings != null) {
                Class<?> parameterObjectClass = parameterObject.getClass();
                if (isStrictMap(parameterObjectClass)) {
                    DefaultSqlSession.StrictMap<Collection<?>> strictMap = (DefaultSqlSession.StrictMap<Collection<?>>) parameterObject;
                    if (isList(strictMap.get("list").getClass())) {
                        sql = handleListParameter(sql, strictMap.get("list"));
                    }
                } else if (isParamMap(parameterObjectClass)) {
                    MapperMethod.ParamMap<Collection<?>> paramMap = (MapperMethod.ParamMap<Collection<?>>) parameterObject;
                    if (!paramMap.isEmpty()) {
                        Set<String> keys = paramMap.keySet();
                        for (String key : keys) {
                            if (isList(paramMap.get(key).getClass())) {
                                sql = handleListParameter(sql, paramMap.get(key));
                            }
                            break;
                        }
                    }
                    System.out.println(paramMap);
                } else if (isMap(parameterObjectClass)) {
                    Map<?, ?> paramMap = (Map<?, ?>) parameterObject;
                    sql = handleMapParameter(sql, paramMap, parameterMappings);
                } else {
                    sql = handleCommonParameter(sql, parameterMappings, parameterObjectClass, parameterObject);
                }
            }
        } catch (Exception e) {
            return sqlWithoutReplacePlaceholder;
        }
        return sql;

    }

    @Override
    public Object plugin(Object o) {
        return Plugin.wrap(o, this);
    }

    @Override
    public void setProperties(Properties properties) {
        this.limitSecond = Integer.parseInt(properties.getProperty("limitSecond"));
    }

    private boolean isStrictMap(Class<?> parameterObjectClass) {
        return parameterObjectClass.isAssignableFrom(DefaultSqlSession.StrictMap.class);
    }

    private boolean isParamMap(Class<?> parameterObjectClass) {
        return parameterObjectClass.isAssignableFrom(MapperMethod.ParamMap.class);
    }

    private String handleListParameter(String sql, Collection<?> col) {
        if (col != null && col.size() != 0) {
            for (Object obj : col) {
                String value = null;
                Class<?> objClass = obj.getClass();
                if (isPrimitiveOrPrimitiveWrapper(objClass)) {
                    value = obj.toString();
                } else if (objClass.isAssignableFrom(String.class)) {
                    value = "\"" + obj.toString() + "\"";
                }
                sql = sql.replaceFirst("\\?", value);
            }
        }
        return sql;

    }

    private String beautifySql(String sql) {
        sql = sql.replace("\n", "")
                .replace("\t", "")
                .replace("  ", " ")
                .replace("( ", "(")
                .replace(" )", ")")
                .replace(" ,", ",");
        return sql;

    }

    private String handleMapParameter(String sql, Map<?, ?> paramMap, List<ParameterMapping> parameterMappingList) {
        for (ParameterMapping parameterMapping : parameterMappingList) {
            Object propertyName = parameterMapping.getProperty();
            Object propertyValue = paramMap.get(propertyName);
            if (propertyValue != null) {
                if (propertyValue.getClass().isAssignableFrom(String.class)) {
                    propertyValue = "\"" + propertyValue + "\"";
                }
                sql = sql.replaceFirst("\\?", propertyValue.toString());
            }
        }
        return sql;

    }


    private String handleCommonParameter(String sql, List<ParameterMapping> parameterMappingList,
                                         Class<?> parameterObjectClass,
                                         Object parameterObject) throws Exception {
        for (ParameterMapping parameterMapping : parameterMappingList) {
            String propertyValue = null;
            if (isPrimitiveOrPrimitiveWrapper(parameterObjectClass)) {
                propertyValue = parameterObject.toString();
            } else {
                String propertyName = parameterMapping.getProperty();
                Field field = parameterObjectClass.getDeclaredField(propertyName);
                field.setAccessible(true);
                propertyValue = String.valueOf(field.get(parameterObject));
                if (parameterMapping.getJavaType().isAssignableFrom(String.class)) {
                    propertyValue = "\"" + propertyValue + "\"";
                }
            }
            sql = sql.replaceFirst("\\?", propertyValue);
        }
        return sql;
    }


    private boolean isPrimitiveOrPrimitiveWrapper(Class<?> parameterObjectClass) {
        return parameterObjectClass.isPrimitive() ||
                (parameterObjectClass.isAssignableFrom(Byte.class)
                        || parameterObjectClass.isAssignableFrom(Short.class)
                        || parameterObjectClass.isAssignableFrom(Integer.class)
                        || parameterObjectClass.isAssignableFrom(Long.class)
                        || parameterObjectClass.isAssignableFrom(Double.class)
                        || parameterObjectClass.isAssignableFrom(Float.class)
                        || parameterObjectClass.isAssignableFrom(Character.class)
                        || parameterObjectClass.isAssignableFrom(Boolean.class));
    }

    private boolean isList(Class<?> clazz) {
        Class<?>[] interfaceClasses = clazz.getInterfaces();
        for (Class<?> interfaceClass : interfaceClasses) {
            if (interfaceClass.isAssignableFrom(List.class)) {
                return true;
            }
        }
        return false;
    }


    private boolean isMap(Class<?> parameterObjectClass) {
        Class<?>[] interfaceClasses = parameterObjectClass.getInterfaces();
        for (Class<?> interfaceClass : interfaceClasses) {
            if (interfaceClass.isAssignableFrom(Map.class)) {
                return true;
            }
        }
        return false;
    }
}
```

###### 3.2 配置文件中增加插件配置

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
    <configuration>

    <properties resource="db.properties"></properties>

    <plugins>
        <plugin interceptor="com.zisuye.mybatis.plugin.SlowSqlPlugin">
            <property name="limitSecond" value="0"/>
        </plugin>
    </plugins>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${driver}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/BlogMapper.xml"/>
    </mappers>


</configuration>
```



#### 四、Mybatis 插件原理

Mybatis的插件其实就是个**拦截器功能**。它利用`JDK动态代理和责任链设计模式的综合运用`。采用责任链模式，通过动态代理组织多个拦截器,通过这些拦截器你可以做一些你想做的事。

![](https://img2018.cnblogs.com/blog/1090617/201908/1090617-20190821194055006-760822200.png)

通过在对Executor，StatementHandler，ResultSetHandler，ParameterHandler对象的拦截处理，动态插入我们的增强逻辑

![](https://img2018.cnblogs.com/blog/1090617/201908/1090617-20190821194103639-1139103584.png)

###### 4.1 在初始化配置文件把所有的拦截器添加到拦截器链中

```java
public class Configuration {

    protected final InterceptorChain interceptorChain = new InterceptorChain();
    //创建参数处理器
  public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    //创建ParameterHandler
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    //插件在这里插入
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }

  //创建结果集处理器
  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    //创建DefaultResultSetHandler
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    //插件在这里插入
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }

  //创建语句处理器
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //创建路由选择语句处理器
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    //插件在这里插入
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

  public Executor newExecutor(Transaction transaction) {
    return newExecutor(transaction, defaultExecutorType);
  }

  //产生执行器
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    //这句再做一下保护,囧,防止粗心大意的人将defaultExecutorType设成null?
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //然后就是简单的3个分支，产生3种执行器BatchExecutor/ReuseExecutor/SimpleExecutor
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //如果要求缓存，生成另一种CachingExecutor(默认就是有缓存),装饰者模式,所以默认都是返回CachingExecutor
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    //此处调用插件,通过插件可以改变Executor行为
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
}
```

###### 4.2 代理逻辑分析

```java
public class Plugin implements InvocationHandler {

    public static Object wrap(Object target, Interceptor interceptor) {
    //从拦截器的注解中获取拦截的类名和方法信息
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    //取得要改变行为的类(ParameterHandler|ResultSetHandler|StatementHandler|Executor)
    Class<?> type = target.getClass();
    //取得接口
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    //产生代理，是Interceptor注解的接口的实现类才会产生代理
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
    
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //获取需要拦截的方法
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      //是Interceptor实现类注解的方法才会拦截处理
      if (methods != null && methods.contains(method)) {
        //调用Interceptor.intercept，也即插入了我们自己的逻辑
        return interceptor.intercept(new Invocation(target, method, args));
      }
      //最后还是执行原来逻辑
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
    
    //取得签名Map,就是获取Interceptor实现类上面的注解，要拦截的是那个类（Executor，ParameterHandler，   ResultSetHandler，StatementHandler）的那个方法
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    //取Intercepts注解，例子可参见ExamplePlugin.java
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    //必须得有Intercepts注解，没有报错
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    //value是数组型，Signature的数组
    Signature[] sigs = interceptsAnnotation.value();
    //每个class里有多个Method需要被拦截,所以这么定义
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }
    
    //取得接口
  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        //拦截其他的无效
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
}
```

