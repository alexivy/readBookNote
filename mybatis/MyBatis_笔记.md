- [Mybatis的使用](#mybatis的使用)
  - [工具类](#工具类)
    - [SQL](#sql)
    - [MetaObject](#metaobject)
    - [ObjectFactory](#objectfactory)
    - [ProxyFactory](#proxyfactory)
  - [核心组件](#核心组件)
    - [Configuration](#configuration)
  - [配置文件](#配置文件)
    - [Mapper配置文件](#mapper配置文件)
  - [映射](#映射)
    - [一对多关联映射的二种方法](#一对多关联映射的二种方法)
  - [一对一关联映射的二种方法](#一对一关联映射的二种方法)
- [MyBatis的原理](#mybatis的原理)
  - [MyBatis涉及的类型转换](#mybatis涉及的类型转换)
    - [MyBaits如何为SQL语句设置参数](#mybaits如何为sql语句设置参数)
    - [MyBaits如何处理SQL语句查询结果](#mybaits如何处理sql语句查询结果)
  - [SqlSession的创建过程](#sqlsession的创建过程)
  - [MyBatis如何执行查询](#mybatis如何执行查询)
    - [查询流程及对应源码](#查询流程及对应源码)
- [参考书籍](#参考书籍)

# Mybatis的使用  

## 工具类  

### SQL  

在Java代码中动态构建SQL语句。

    String sql=new SQL().INSERT_INTO("PERSON")
                .VALUES("ID,FIRST_NAME","#{id},#{firstName}")
                .VALUES("LAST_NAME","#{lastName}").toString();

### MetaObject  

MetaObject是MyBatis中的反射工具类，使用MetaObject工具类，我们可以方便地获取和设置对象的属性值。  

<details>
    <summary>点击查看示例代码</summary>

    static class User{
        List<Order> orders;
        String name;

        public User(List<Order> orders, String name) {
            this.orders = orders;
            this.name = name;
        }
    }

    static class Order{
        String orderNo;
        String goodsName;

        public Order(String orderNo, String goodsName) {
            this.orderNo = orderNo;
            this.goodsName = goodsName;
        }
    }

    public static void main(String[] args) {
        List<Order> orders=new ArrayList<Order>(){{
            add(new Order("13241421","phone"));
            add(new Order("12314314","house"));
        }};
        User user=new User(orders,"people");
        MetaObject metaObject= SystemMetaObject.forObject(user);
        System.out.println(metaObject.getValue("orders[0].goodsName"));
    }

</details>

### ObjectFactory  

ObjectFactory是MyBatis中的对象工厂，MyBatis每次创建Mapper映射结果对象的新实例时，都会使用一个对象工厂（ObjectFactory）实例来完成。ObjectFactory接口只有一个默认的实现，即DefaultObjectFactory，默认的对象工厂需要做的仅仅是实例化目标类，要么通过默认构造方法，要么在参数映射存在的时候通过参数构造方法来实例化。  

### ProxyFactory  

ProxyFactory是MyBatis中的代理工厂，主要用于创建动态代理对象，ProxyFactory接口有两个不同的实现，分别为CglibProxyFactory和JavassistProxyFactory。从实现类的名称可以看出，MyBatis支持两种动态代理策略，分别为Cglib和Javassist动态代理。ProxyFactory主要用于实现MyBatis的懒加载功能。  

## 核心组件

![MyBatis核心组件](/mybatis/MyBatisP001.PNG)  
MyBatis核心组件( 图源自[参考书籍1](#ref001) )  

MyBatis使用StatementHandler组件控制JDBC的Statement对象，进而对数据库进行操作。[完整流程](#mybatis如何执行查询)  
MyBatis通过ResultSetHandler组件从Statement对象中获取ResultSet对象，然后将ResultSet对象转换为Java对象。[转换为Java对象的详细过程](#mybaits如何处理sql语句查询结果)  

各组件的作用如下：  
<details>
    <summary> 点击查看各组件 </summary>

Configuration：用于描述MyBatis的主配置信息，其他组件需要获取配置信息时，直接通过Configuration对象获取。除此之外，MyBatis在应用启动时，将Mapper配置信息、类型别名、TypeHandler等注册到Configuration组件中，其他组件需要这些信息时，也可以从Configuration对象中获取。  

MappedStatement：MappedStatement用于描述Mapper中的SQL配置信息，是对Mapper XML配置文件中<select|update|delete|insert>等标签或者@Select/@Update等注解配置信息的封装。  

SqlSession：SqlSession是MyBatis提供的面向用户的API，表示和数据库交互时的会话对象，用于完成数据库的增删改查功能。SqlSession是Executor组件的外观，目的是对外提供易于理解和使用的数据库操作接口。  

Executor：Executor是MyBatis的SQL执行器，MyBatis中对数据库所有的增删改查操作都是由Executor组件完成的。  

StatementHandler：StatementHandler封装了对JDBCStatement对象的操作，比如为Statement对象设置参数，调用Statement接口提供的方法与数据库交互，等等。  

ParameterHandler：当MyBatis框架使用的Statement类型为CallableStatement和PreparedStatement时，ParameterHandler用于为Statement对象参数占位符设置值。  

[ResultSetHandler](#handleresultsets)：ResultSetHandler封装了对JDBC中的ResultSet对象操作，当执行SQL类型为SELECT语句时，ResultSetHandler用于将查询结果转换成Java对象。  

TypeHandler：TypeHandler是MyBatis中的类型处理器，用于处理Java类型与JDBC类型之间的映射。它的作用主要体现在能够根据Java类型调用PreparedStatement或CallableStatement对象对应的setXXX()方法为Statement对象设置值，而且能够根据Java类型调用ResultSet对象对应的getXXX()获取SQL执行结果。  

</details>

### Configuration  

在MyBatis框架启动时，会对所有的配置信息（TypeHandler（类型处理器）、TypeAlias（类型别名）、Mapper接口及Mapper SQL的配置等等）进行解析，然后将解析后的内容注册到Configuration对象的这些属性中。除此之外，Configuration组件还作为Executor、StatementHandler、ResultSetHandler、ParameterHandler组件的工厂类，用于创建这些组件的实例。  

*** Executor ***  

MyBatis提供了3种不同的Executor，分别为SimpleExecutor、ReuseExecutor、BatchExecutor，这些Executor都继承至BaseExecutor。  
Executor与数据库交互需要Mapper配置信息，MyBatis通过MappedStatement对象描述Mapper的配置信息，Configuration.getMappedStatement()方法可获取对应的对象。  

*** MappedStatement ***  

MyBatis通过MappedStatement描述\<select\|update\|insert\|delete\>或者@Select、@Update等注解配置的SQL信息。  


## 配置文件  
  
### Mapper配置文件  

\<select\>标签中的属性  

    <select id="getUserById"
    parameterType="int"  //可选
    parameterMap="Deprecated"  //Deprecated
    resultType="hashmap"  //如果返回结果是集合类型，则resultType属性应该指定集合中可以包含的类型，而不是集合本身。
    resultMap="userResultMap"  //resultMap和resultType属性不能同时使用
    flushCache="false"  
    useCache="true"  //是否使用二级缓存,对select标签，该属性的默认值为true。
    timeout="10000"  
    fetchSize="256"  //指定SQL执行后返回的最大行数。
    statementType="PREPARED"  //与数据库交互的Statement
    resultSetType="FORWARD_ONLY"
    >

## 映射  

### 一对多关联映射的二种方法  

MyBatis的Mapper配置中提供了一个\<collection\>标签，用于建立实体间一对多的关系。MyBatis在执行getUserByIdFull查询时先后执行了两条查询语句。第一条语句查询用户信息，第二条（即使resultMap.collection.select属性指定的查询）语句查询用户关联的订单信息。  

    <resultMap id="detailMap" type="com.blog4java.mybatis.example.entity.User">
        <collection property="orders"
            ofType="com.blog4java.mybatis.example.entity.Order"
            select="com.blog4java.mybatis.example.mapper.OrderMapper.listOrdersByUserId"
            javaType="java.util.ArrayList"
            column="id">
        </collection>
    </resultMap>

    对应的查询语句：  
    <select id="getUserByIdFull" resultMap="detailMap">
        select * from user where id = #{userId}
    </select>

    关联的查询：  
    <select id="listOrdersByUserId" resultType="com.blog4java.mybatis.example.entity.Order">
       select * from "order" where userId = #{userId}
    </select>

join子句实现一对多查询，只用执行一次查询语句：

    <resultMap autoMapping="true" id="detailMapForJoin" type="com.blog4java.mybatis.example.entity.User">
        <collection property="orders" ofType="com.blog4java.mybatis.example.entity.Order">
            <id column="id" property="id"></id>
            <result column="createTime" property="createTime"></result>
            <result column="userId" property="userId"></result>
            <result column="amount" property="amount"></result>
            <result column="orderNo" property="orderNo"></result>
            <result column="address" property="address"></result>
        </collection>
    </resultMap>

    对应的查询语句：  
    <select id="getUserByIdForJoin" resultMap="detailMapForJoin">
        select u.*,o.* from user u left join "order" o on (o.userId = u.id) where u.id = #{userId}
    </select>

## 一对一关联映射的二种方法  

MyBatis一对一关联映射的配置方式与一对多映射类似，不同的是我们在定义ResultMap时需要使用< association >标签。  

    <resultMap id="detailMap" type="com.blog4java.mybatis.example.entity.Order">
        <association property="user" javaType="com.blog4java.mybatis.example.entity.User"
                     select="com.blog4java.mybatis.example.mapper.UserMapper.getUserById" column="userId">
        </association>
    </resultMap>

    对应的查询语句：  
    <select id="getOrderByNo" resultMap="detailMap">
       select * from "order" where orderNo = #{orderNo}
    </select>

    关联的查询：  
    <select id="getUserById" resultType="com.blog4java.mybatis.example.entity.User">
        select * from user where id = #{userId}
    </select>

join子句实现：

    <resultMap  id="detailNestMap" type="com.blog4java.mybatis.example.entity.Order">
        <id column="id" property="id"></id>
        <result column="createTime" property="createTime"></result>
        <result column="userId" property="userId"></result>
        <result column="amount" property="amount"></result>
        <result column="orderNo" property="orderNo"></result>
        <result column="address" property="address"></result>
        <association property="user"  javaType="com.blog4java.mybatis.example.entity.User" >
            <id column="userId" property="id"></id>
            <result column="name" property="name"></result>
            <result column="createTime" property="createTime"></result>
            <result column="password" property="password"></result>
            <result column="phone" property="phone"></result>
            <result column="nickName" property="nickName"></result>
        </association>
    </resultMap>

    对应的查询语句：  
    <select id="getOrderByNoWithJoin" resultMap="detailNestMap">
       select o.*,u.* from "order" o left join user u on (u.id = o.userId) where orderNo = #{orderNo}
    </select>

# MyBatis的原理  

## MyBatis涉及的类型转换  

MyBatis涉及Java类型和JDBC类型转换的两种情况如下：  
1. PreparedStatement对象为参数占位符设置值时，需要调用PreparedStatement接口中提供的一系列的setXXX()方法，将Java类型转换为对应的JDBC类型并为参数占位符赋值。  
2. 执行SQL语句获取ResultSet对象后，需要调用ResultSet对象的getXXX()方法获取字段值，此时会将JDBC类型转换为Java类型。  

MyBatis中使用TypeHandler解决上面两种情况，MyBatis中的BaseTypeHandler类实现了TypeHandler接口。  

### MyBaits如何为SQL语句设置参数  

即类型转换第一种情况对应的场景：  

ParameterHandler的作用是为SQL语句中的参数占位符设置值。ParameterHandler接口只有一个默认的实现类，即DefaultParameterHandler，在DefaultParameterHandler类的setParameters()方法中，首先获取Mapper配置中的参数映射，然后对所有参数映射信息进行遍历，接着根据参数名称获取对应的参数值，调用对应的TypeHandler对象的setParameter()方法为Statement对象中的参数占位符设置值。  


<p id="handleresultsets"></p>  

### MyBaits如何处理SQL语句查询结果  

即类型转换第二种情况对应的场景：  

ResultSetHandler用于在StatementHandler对象执行完查询操作或存储过程后，对结果集或存储过程的执行结果进行处理。ResultSetHandler接口只有一个默认的实现，即DefaultResultHandler。  

DefaultResultSetHandler类的handleResultSets()方法具体逻辑如下：
1. 首先从Statement对象中获取ResultSet对象，然后将ResultSet包装为ResultSetWrapper对象，通过ResultSetWrapper对象能够更方便地获取数据库字段名称以及字段对应的TypeHandler信息。  
2. 获取Mapper SQL配置中通过resultMap属性指定的ResultMap信息，一条SQL Mapper配置一般只对应一个ResultMap。  
3. 调用handleResultSet()方法对ResultSetWrapper对象进行处理，将结果集转换为Java实体对象，然后将生成的实体对象存放在multipleResults列表中。  
4. 调用collapseSingleResultList()方法对multipleResults进行处理，如果只有一个结果集，就返回结果集中的元素，否则返回多个结果集。  

<details>
    <summary>点击查看esultSetHandler的handleResultSets方法
    </summary>
    
```
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
    ResultMap resultMap = resultMaps.get(resultSetCount);
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResulSets();
    if (resultSets != null) {
    while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
        String nestedResultMapId = parentMapping.getNestedResultMapId();
        ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
        handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }
    }

    return collapseSingleResultList(multipleResults);
}
```

</details>

## SqlSession的创建过程  

框架在启动时会解析MyBatis主配置文件及所有Mapper文件，将配置信息转换为Configuration对象。在创建SqlSession对象之前，需要先创建SqlSessionFactory对象（SqlSessionFactory.build()方法返回一个此对象）。SqlSessionFactory对象中持有Configuration对象的引用。有了SqlSessionFactory对象后，调用SqlSessionFactory对象的openSession()方法即可创建SqlSession对象。  





## MyBatis如何执行查询  

在创建SqlSession实例后，需要调用SqlSession的getMapper()方法获取一个UserMapper的引用，然后通过该引用调用Mapper接口中定义的方法。SqlSession对象的getMapper()方法返回的是一个动态代理对象。MyBatis中通过MapperProxy类实现动态代理。MapperProxy使用的是JDK内置的动态代理。  

MyBatis执行查询的大致流程：写代码时定义的是Mapper接口及其中的方法，MyBatis会生成Mapper对应的动态代理对象，在通过代理对象调用Mapper接口中定义的方法时，会执行代理对象中的拦截逻辑，将Mapper方法的调用转换为调用SqlSession提供的API方法。在SqlSession的API方法中通过Mapper的Id找到对应的MappedStatement对象，获取对应的SQL信息，通过StatementHandler操作JDBC API的Statement对象完成与数据库的交互，然后通过[ResultSetHandler](#handleresultsets)处理结果集，将结果返回给调用者。  

### 查询流程及对应源码  

MyBatis版本3.2.5。  

***首先获得动态代理对象***  

以下是SqlSession中的getMapper方法的调用流程，以DefaulfSqlSession为例。对应源码如下：  

    DefaulfSqlSession中的getMapper方法源码
    public <T> T getMapper(Class<T> type) {
        return configuration.<T>getMapper(type, this);
    }

    接着调用Configuration中的getMapper方法
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
    }

接着是MapperRegistry中的getMapper方法。
在这里通过knownMappers得到一个type Class对应的动态代理对象工厂mapperProxyFactory。

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null)
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        try {
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }

接着是代理对象工厂MapperProxyFactory中的两个newInstance方法。  
调用后返回一个type Class的动态代理对象。  
在这里调用Proxy.newProxyInstance方法时传入的InvocationHandler参数为mapperProxy。
mapperProxy中定义了调用动态代理对象中方法时的逻辑。

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }

***调用动态代理对象中的方法***  
MyBatis中的MapperProxy使用的是JDK内置的动态代理。  

MapperProxy类中，invoke方法中定义了调用动态代理对象中方法的逻辑。
对从Object类继承的方法不做任何处理，对Mapper接口中定义的方法，调用cachedMapperMethod()方法获取一个MapperMethod对象。  
对应的源码：

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
        }
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }

MapperProxy类中，cachedMapperMethod方法获取MapperMethod对象。  
先尝试从methodCache中获取method对应的MapperMethod，若未获取到，则new一个相应Mappermethod，并将其存入methodCache。  
new MapperMethod对象时会从sqlSession中获取相应的配置，在MapperMethod构造方法中将其处理成execute中使用到的属性。  
对应的源码：

    private MapperMethod cachedMapperMethod(Method method) {
        MapperMethod mapperMethod = methodCache.get(method);
        if (mapperMethod == null) {
        mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
        methodCache.put(method, mapperMethod);
        }
        return mapperMethod;
    }

获取到mapperMethod后，执行其execute方法。  
首先获取方法执行的SQL语句类型，再处理传入参数，最后执行sqlSession中的对应方法，取得并处理返回值。  
若是Insert、Update、Delete三种返回影响记录数量的SQL类型，会用rowCountResult方法处理返回值。  
对Select类型的SQL语句处理较为复杂，会根据方法的返回类型调用相应方法。  
源码如下：  

    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        if (SqlCommandType.INSERT == command.getType()) {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
        } else if (SqlCommandType.UPDATE == command.getType()) {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
        } else if (SqlCommandType.DELETE == command.getType()) {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
        } else if (SqlCommandType.SELECT == command.getType()) {
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
            }
        } else {
            throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
            throw new BindingException("Mapper method '" + command.getName() 
            + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
    }

SqlSession中的对应方法中会根据Mapper的Id从Configuration对象中获取对应的MappedStatement对象，然后以MappedStatement对象作为参数，调用Executor实例的对应方法完成查询。  
selectList方法的源码：  

    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
            MappedStatement ms = configuration.getMappedStatement(statement);
            List<E> result = executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
            return result;
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }

BaseExecutor中的query方法  
在query中，若不要求刷新缓存，会首先从MyBatis一级缓存中获取查询结果，如果缓存中没有，则调用BaseExecutor类的queryFromDatabase()方法从数据库中查询。  
对应源码：  
<details>
    <summary>点击查看（过长折叠）</summary>

    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameter);
        CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
        return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }

    @SuppressWarnings("unchecked")
    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) throw new ExecutorException("Executor was closed.");
        if (queryStack == 0 && ms.isFlushCacheRequired()) {
            clearLocalCache();
        }
        List<E> list;
        try {
            queryStack++;
            list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
            if (list != null) {
                handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
            } else {
                list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
            }
        } finally {
            queryStack--;
        }
        if (queryStack == 0) {
            for (DeferredLoad deferredLoad : deferredLoads) {
                deferredLoad.load();
            }
            deferredLoads.clear(); // issue #601
            if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
                clearLocalCache(); // issue #482
            }
        }
        return list;
    }

</details>

BaseExecutor中的queryFromDatabase方法中调用doQuery方法，doQuery方法由BaseExecutor的子类实现。  
对应源码：

    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        List<E> list;
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        try {
            list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
            localCache.removeObject(key);
        }
        localCache.putObject(key, list);
        if (ms.getStatementType() == StatementType.CALLABLE) {
            localOutputParameterCache.putObject(key, parameter);
        }
        return list;
    }
   

BaseExecutor的子类SimpleExecutor实现的doQuery方法中，会先调用prepareStatement方法，再调用StatementHandler的query方法。  
SimpleExecutor.doQuery源码如下：

    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, rowBounds, resultHandler, boundSql);
            stmt = prepareStatement(handler, ms.getStatementLog());
            return handler.<E>query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);
        }
    }

prepareStatement方法首先获取JDBC中的Connection对象，然后调用StatementHandler对象的prepare()方法创建Statement对象，接着调用StatementHandler对象的parameterize()方法（parameterize()方法中会使用ParameterHandler为Statement对象设置参数）。  

<details>
    <summary>点击查看prepareStatement方法</summary>
    <pre><blockcode> 
    private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
        Statement stmt;
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection);
        handler.parameterize(stmt);
        return stmt;
    }
    </blockcode></pre>
</details>

MyBatis默认情况下会使用PreparedStatementHandler与数据库交互。在PreparedStatementHandler的query()方法中，首先调用PreparedStatement对象的execute()方法（注：JDBC API）执行SQL语句，然后调用ResultSetHandler的[handleResultSets](#handleresultsets)方法处理结果集。  
对应源码：

    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        return resultSetHandler.<E> handleResultSets(ps);
    }






<!-- ## MyBatis启动时进行了哪些工作   -->






<!-- 
调用Mapper接口中定义的方法的具体流程：  
1. 首先需要调用SqlSession对象的getMapper()方法获取一个MapperProxy类的动态代理对象，然后通过代理对象调用方法。  
2. 在MapperProxy类的invoke()方法中，对从Object类继承的方法不做任何处理，对Mapper接口中定义的方法，调用cachedMapperMethod()方法获取一个MapperMethod对象。  
3. MapperMethod提供了一个execute()方法，用于执行SQL命令。MapperProxy类的invoke()方法中获取MapperMethod对象后，最终会调用MapperMethod类的execute()。  
4. 在execute()方法中，首先根据SqlCommand对象获取SQL语句的类型，然后根据SQL语句的类型调用SqlSession对象对应的方法。  
5. SqlSession接口只有一个默认的实现，即DefaultSqlSession。在DefaultSqlSession的selectList()方法中，首先根据Mapper的Id从Configuration对象中获取对应的MappedStatement对象
6. 然后以MappedStatement对象作为参数，调用Executor实例的query()方法完成查询操作。  
7. 在BaseExecutor类的query()方法中，首先从MappedStatement对象中获取BoundSql对象，BoundSql类中封装了经过解析后的SQL语句及参数映射信息。然后创建CacheKey对象，该对象用于缓存的Key值。接着调用重载的query()方法。  
8. 在重载的query()方法中，首先从MyBatis一级缓存中获取查询结果，如果缓存中没有，则调用BaseExecutor类的queryFromDatabase()方法从数据库中查询。  
9. 在queryFromDatabase()方法中，调用doQuery()方法进行查询，然后将查询结果进行缓存，doQuery()是一个模板方法，由BaseExecutor子类实现。  
10. 在SimpleExecutor类的prepareStatement()方法中，首先获取JDBC中的Connection对象，然后调用StatementHandler对象的prepare()方法创建Statement对象，接着调用StatementHandler对象的parameterize()方法（parameterize()方法中会使用ParameterHandler为Statement对象设置参数）。  
11. MyBatis默认情况下会使用PreparedStatementHandler与数据库交互。在PreparedStatementHandler的query()方法中，首先调用PreparedStatement对象的execute()方法执行SQL语句，然后调用ResultSetHandler的handleResultSets()方法处理结果集。  
12. ResultSetHandler只有一个默认的实现，即DefaultResultSetHandler类。DefaultResultSetHandler处理结果集的逻辑见[handleResultSets](#handleresultsets)。  

cachedMapperMethod()方法中对MapperMethod对象做了缓存，首先从缓存中获取，如果获取不到，则创建MapperMethod对象，然后添加到缓存中。MapperMethod类是对Mapper方法相关信息的封装，通过MapperMethod能够很方便地获取SQL语句的类型、方法的签名信息等。MapperMethod中有SqlCommand对象和MethodSignature对象。SqlCommand对象用于获取SQL语句的类型、Mapper的Id等信息；MethodSignature对象用于获取方法的签名信息，例如Mapper方法的参数名、参数注解等信息。

MethodSignature构造方法完成的任务：
1. 获取Mapper方法的返回值类型，通过boolean类型的属性进行标记。  
2. 记录RowBounds参数位置，用于处理后续的分页查询，同时记录ResultHandler参数位置，用于处理从数据库中检索的每一行数据。  
3. 创建ParamNameResolver对象。ParamNameResolver对象用于解析Mapper方法中的参数名称及参数注解信息。

MyBatis框架在应用启动时会解析所有的Mapper接口，然后调用MapperRegistry对象的addMapper()方法将Mapper接口信息和对应的MapperProxyFactory对象注册到MapperRegistry对象中。   -->


# 参考书籍  
<p id="ref001"></p>
[1] 《MyBatis 3源码深度解析》-江荣波  
