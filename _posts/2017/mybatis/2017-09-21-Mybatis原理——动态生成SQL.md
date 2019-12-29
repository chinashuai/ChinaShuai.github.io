---
layout: post
title: Mybatis原理——动态生成SQL
category: mybatis
tags: [mybatis,java]
excerpt: Mybatis原理——动态生成SQL
keywords: mybatis,动态生成SQL,java,java高阶
---


本文将带你分析Mybatis是如何动态生成SQL。
首先，会根据源码分析框架初始化时xml文件的加载、解析、缓存过程。着重介绍 xml的解析过程 和 使用解析的结果，最后列举实例和对照源码`DeBug`分析：当`DAO`接口调用时标签的解析、参数的创建、SQL的生成过程，并总结整个流程。

* #####数据的处理
> Mybatis对数据的处理可以分为 **用入参动态的拼装sql** 和 **对sql执行的结果封装成 JavaBean**  

这里包括两个过程：**1.** 查询阶段我们要将java类型的数据，转换成jdbc类型的数据，通过 preparedStatement.setXXX() 来设值  **2.** 另一个就是对resultset查询结果集的jdbcType 数据转换成java 数据类型，本文只介绍第一个过程。

* #####根据传入的参数动态的拼装sql
在`Mybatis`中，需要根据xml标签的语法编写出动态SQL，在执行的时候会根据标签进行解析，这里使用的是 `Ognl` 来解析标签动态地构造SQL语句

* ######分析`parseDynamicTags` 的解析过程：

`Spring`与`Mybatis`整合的时候需要配置`SqlSessionFactoryBean`，该配置会加入数据源和Mybatis xml配置文件路径等信息
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation" value="classpath:mybatisConfig.xml"/>
    <property name="mapperLocations" value="classpath*:org/format/dao/*.xml"/>
</bean>
```
其中`SqlSessionFactoryBean`实现了`Spring的InitializingBean`接口，`InitializingBean`接口的`afterPropertiesSet`方法中会调用`buildSqlSessionFactory`方法 该方法内部会使用`XMLConfigBuilder`解析属性`configLocation`中配置的路径，还会使用`XMLMapperBuilder`属性解析`mapperLocations`属性中的各个xml文件，在启动的时候，会根据xml文件的配置路径来解析xml文件，下面我们就看看加载时候的部分源码，并做简单的分析，读者可重点关注加注解的部分代码：

**XMLMapperBuilder:**

```java
/* 读者可重点关注 加注解 的部分代码即可  */
public class XMLMapperBuilder extends BaseBuilder {
 
  public void parse() {
        if(!this.configuration.isResourceLoaded(this.resource)) {
            //根据xpath解析mapper节点 
            this.configurationElement(this.parser.evalNode("/mapper"));
            this.configuration.addLoadedResource(this.resource);
            this.bindMapperForNamespace();
        }

        this.parsePendingResultMaps();
        this.parsePendingChacheRefs();
        this.parsePendingStatements();
    }
	
	/**
	 * 根据xpath解析mapper节点
	 */
    private void configurationElement(XNode context) {
        try {
            String namespace = context.getStringAttribute("namespace");
            if(namespace.equals("")) {
                throw new BuilderException("Mapper's namespace cannot be empty");
            } else {
                //赋值当前处理的mapper的namespace
                this.builderAssistant.setCurrentNamespace(namespace);
                //处理二级缓存
                this.cacheRefElement(context.evalNode("cache-ref"));
                this.cacheElement(context.evalNode("cache"));
                //处理parameterMap节点
              this.parameterMapElement(context.evalNodes("/mapper/parameterMap"));
                //处理resultMap节点
                this.resultMapElements(context.evalNodes("/mapper/resultMap"));
                //处理sql节点
                this.sqlElement(context.evalNodes("/mapper/sql"));
                //处理select|insert|update|delete节点
                this.buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
            }
        } catch (Exception var3) {
            throw new BuilderException("Error parsing Mapper XML. Cause: " + var3, var3);
        }
    }

	/**
	 * 处理select|insert|update|delete节点
	 */
    private void buildStatementFromContext(List<XNode> list) {
        if(this.configuration.getDatabaseId() != null) {
            this.buildStatementFromContext(list, this.configuration.getDatabaseId());
        }

        this.buildStatementFromContext(list, (String)null);
    }

	/**
	 * 处理select|insert|update|delete节点
	 */
    private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
        Iterator i$ = list.iterator();

        while(i$.hasNext()) {
            XNode context = (XNode)i$.next();
          	//对每个节点都用XMLStatementBuilder进行解析
            XMLStatementBuilder statementParser = new XMLStatementBuilder(this.configuration, this.builderAssistant, context, requiredDatabaseId);
            try {
                //解析每个节点
                statementParser.parseStatementNode();
            } catch (IncompleteElementException var7) {
                this.configuration.addIncompleteStatement(statementParser);
            }
        }
    }
}
```
**XMLStatementBuilder :**
```java
public class XMLStatementBuilder extends BaseBuilder {

   /**
    * 解析节点
    */
  public void parseStatementNode() {
        String id = this.context.getStringAttribute("id");
        String databaseId = this.context.getStringAttribute("databaseId");
        if(this.databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
            Integer fetchSize = this.context.getIntAttribute("fetchSize");
            Integer timeout = this.context.getIntAttribute("timeout");
            String parameterMap = this.context.getStringAttribute("parameterMap");
            String parameterType = this.context.getStringAttribute("parameterType");
            Class<?> parameterTypeClass = this.resolveClass(parameterType);
            String resultMap = this.context.getStringAttribute("resultMap");
            String resultType = this.context.getStringAttribute("resultType");
            String lang = this.context.getStringAttribute("lang");
          
          	//使用LanguageDriver进行解析SQL
          	LanguageDriver langDriver = this.getLanguageDriver(lang);
          
            Class<?> resultTypeClass = this.resolveClass(resultType);
            String resultSetType = this.context.getStringAttribute("resultSetType");
            StatementType statementType = StatementType.valueOf(this.context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
            ResultSetType resultSetTypeEnum = this.resolveResultSetType(resultSetType);
            String nodeName = this.context.getNode().getNodeName();
            SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
            boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
            boolean flushCache = this.context.getBooleanAttribute("flushCache", Boolean.valueOf(!isSelect)).booleanValue();
            boolean useCache = this.context.getBooleanAttribute("useCache", Boolean.valueOf(isSelect)).booleanValue();
            boolean resultOrdered = this.context.getBooleanAttribute("resultOrdered", Boolean.valueOf(false)).booleanValue();
            XMLIncludeTransformer includeParser = new XMLIncludeTransformer(this.configuration, this.builderAssistant);
            includeParser.applyIncludes(this.context.getNode());
            this.processSelectKeyNodes(id, parameterTypeClass, langDriver);
          
          	//解析创建SQL
            SqlSource sqlSource = langDriver.createSqlSource(this.configuration, this.context, parameterTypeClass);
          
            //此处省略其他代码
    }  
}
```
默认会使用XMLLanguageDriver创建SqlSource（Configuration构造函数中设置）。

**XMLLanguageDriver** 创建`SqlSource`：
```java
public class XMLLanguageDriver implements LanguageDriver {
    public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
        XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
        //使用XMLScriptBuilder的parseScriptNode方法解析节点的SQL部分
        return builder.parseScriptNode();
    }
}
```

**XMLScriptBuilder解析:**

```java
public class XMLScriptBuilder extends BaseBuilder {
    public SqlSource parseScriptNode() {
        //解析节点，若有子节点，就会递归的调用解析
        List<SqlNode> contents = this.parseDynamicTags(this.context);
        MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
        SqlSource sqlSource = null;
        if(this.isDynamic) {
            sqlSource = new DynamicSqlSource(this.configuration, rootSqlNode);
        } else {
            sqlSource = new RawSqlSource(this.configuration, rootSqlNode, this.parameterType);
        }

        return (SqlSource)sqlSource;
    }
}
```

**XMLScriptBuilder 递归的解析所有节点:**

```java
public class XMLScriptBuilder extends BaseBuilder {
    private XNode context;
    private boolean isDynamic;
    private Class<?> parameterType;
    private Map<String, XMLScriptBuilder.NodeHandler> nodeHandlers;

    public XMLScriptBuilder(Configuration configuration, XNode context) {
        this(configuration, context, (Class)null);
    }

    public XMLScriptBuilder(Configuration configuration, XNode context, Class<?> parameterType) {
        super(configuration);
        this.nodeHandlers = new HashMap<String, XMLScriptBuilder.NodeHandler>() {
            private static final long serialVersionUID = 7123056019193266281L;

            {
                //不同的标签有不同的解析类
                this.put("trim", XMLScriptBuilder.this.new TrimHandler(null));
                this.put("where", XMLScriptBuilder.this.new WhereHandler(null));
                this.put("set", XMLScriptBuilder.this.new SetHandler(null));
                this.put("foreach", XMLScriptBuilder.this.new ForEachHandler(null));
                this.put("if", XMLScriptBuilder.this.new IfHandler(null));
                this.put("choose", XMLScriptBuilder.this.new ChooseHandler(null));
                this.put("when", XMLScriptBuilder.this.new IfHandler(null));
                this.put("otherwise", XMLScriptBuilder.this.new OtherwiseHandler(null));
                this.put("bind", XMLScriptBuilder.this.new BindHandler(null));
            }
        };
        this.context = context;
        this.parameterType = parameterType;
    }
      
	/**
	 * 递归的解析所有节点
	 */
	private List<SqlNode> parseDynamicTags(XNode node) {
        List<SqlNode> contents = new ArrayList();
        NodeList children = node.getNode().getChildNodes();

        for(int i = 0; i < children.getLength(); ++i) {
            XNode child = node.newXNode(children.item(i));
            String nodeName;
            if(child.getNode().getNodeType() != 4 && child.getNode().getNodeType() != 3) {
                if(child.getNode().getNodeType() == 1) {
                    nodeName = child.getNode().getNodeName();
                    //根据不同的标签使用不同的解析类
                    XMLScriptBuilder.NodeHandler handler = (XMLScriptBuilder.NodeHandler)this.nodeHandlers.get(nodeName);
                    if(handler == null) {
                        throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
                    }
                    //解析
                    handler.handleNode(child, contents);
                    this.isDynamic = true;
                }
            } else {
                nodeName = child.getStringBody("");
                TextSqlNode textSqlNode = new TextSqlNode(nodeName);
                if(textSqlNode.isDynamic()) {
                    contents.add(textSqlNode);
                    this.isDynamic = true;
                } else {
                    contents.add(new StaticTextSqlNode(nodeName));
                }
            }
        }

        return contents;
    }

    /**
     * 内部类IfHandler的实现
     */
    private class IfHandler implements XMLScriptBuilder.NodeHandler {
        private IfHandler() {
        }

        public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
            List<SqlNode> contents = XMLScriptBuilder.this.parseDynamicTags(nodeToHandle);
            MixedSqlNode mixedSqlNode = new MixedSqlNode(contents);
            String test = nodeToHandle.getStringAttribute("test");
            IfSqlNode ifSqlNode = new IfSqlNode(mixedSqlNode, test);
            targetContents.add(ifSqlNode);
        }
    }
  //其他标签类型的handler就不一一举例了，感兴趣的可以看看 XMLScriptBuilder 的源码实现
}
```
>* **`XMLConfigBuilder`**：解析mybatis中configLocation属性中的全局xml文件，内部会使用 XMLMapperBuilder 解析各个xml文件。
>* **`XMLMapperBuilder`**：遍历mybatis中mapperLocations属性中的xml文件中每个节点的Builder，比如user.xml，内部会使用 XMLStatementBuilder 处理xml中的每个节点。
>* **`XMLStatementBuilder`**：解析xml文件中各个节点，比如`select,insert,update,delete`节点，内部会使用 XMLScriptBuilder 处理节点的sql部分，遍历产生的数据会丢到Configuration的mappedStatements中。
>* **`XMLScriptBuilder`**：解析xml中各个节点sql部分的Builder。

至此，mapper.xml文件就已经解析加载完成了并得到`SqlSource`，`SqlSource`将会放到`Configuration`中，有了`SqlSource`，在执行的时候会根据`SqlSource`获取`BoundSql`从而得到需要的`SQL`，`Configuration`可以看做巨大的资源库，Mybatis框架执行时需要的数据都可以`Configuration`中获取，`Configuration`的源码为:

```java
public class Configuration {
    protected Environment environment;
    protected boolean safeRowBoundsEnabled;
    protected boolean safeResultHandlerEnabled;
    protected boolean mapUnderscoreToCamelCase;
    protected boolean aggressiveLazyLoading;
    protected boolean multipleResultSetsEnabled;
    protected boolean useGeneratedKeys;
    protected boolean useColumnLabel;
    protected boolean cacheEnabled;
    protected boolean callSettersOnNulls;
    protected String logPrefix;
    protected Class<? extends Log> logImpl;
    protected LocalCacheScope localCacheScope;
    protected JdbcType jdbcTypeForNull;
    protected Set<String> lazyLoadTriggerMethods;
    protected Integer defaultStatementTimeout;
    protected ExecutorType defaultExecutorType;
    protected AutoMappingBehavior autoMappingBehavior;
    protected Properties variables;
    protected ObjectFactory objectFactory;
    protected ObjectWrapperFactory objectWrapperFactory;
    protected MapperRegistry mapperRegistry;
    protected boolean lazyLoadingEnabled;
    protected ProxyFactory proxyFactory;
    protected String databaseId;
    protected Class<?> configurationFactory;
    protected final InterceptorChain interceptorChain;
    protected final TypeHandlerRegistry typeHandlerRegistry;
    protected final TypeAliasRegistry typeAliasRegistry;
    protected final LanguageDriverRegistry languageRegistry;
    protected final Map<String, MappedStatement> mappedStatements;
    protected final Map<String, Cache> caches;
    protected final Map<String, ResultMap> resultMaps;
    protected final Map<String, ParameterMap> parameterMaps;
    protected final Map<String, KeyGenerator> keyGenerators;
    protected final Set<String> loadedResources;
    protected final Map<String, XNode> sqlFragments;
    protected final Collection<XMLStatementBuilder> incompleteStatements;
    protected final Collection<CacheRefResolver> incompleteCacheRefs;
    protected final Collection<ResultMapResolver> incompleteResultMaps;
    protected final Collection<MethodResolver> incompleteMethods;
    protected final Map<String, String> cacheRefMap;
  
  //SomeMethod...
}
```

下面举例来说明：
实例中我们使用：
```java
//mapper接口的方法
schoolCustomerDao.selectBySome(1l,  "2017-09-17","120706049");
```
```xml
此SQL使用了一个 if 标签
<select id="selectBySome"  resultMap="BaseResultMap">
        SELECT
          id,student_number
        FROM
          school_customer
        WHERE
          id = #{id}
        <if test="studentNumber!=null">
            AND 
              student_number = #{studentNumber}
        </if>
        AND 
          create_time = #{createTime}
    </select>
```
在执行`schoolCustomerDao.selectBySome(1l,  "2017-09-17","120706049");`时，
mapper的代理类会先判断是否在缓存中存在此方法，若不存在则需要加载，若已存在则直接调用，然后会根据 `select|insert|update|delete` 的类型调用不同的`SqlSession`方法，在调用之前会根据入参`(1l,  "2017-09-17","120706049")`封装参数，封装参数的源码如下：
```java
//MapperMethod类会执行这个方法进行参数的拼装
param = this.method.convertArgsToSqlCommandParam(args);
/**
 * 拼装入参
 */
public Object convertArgsToSqlCommandParam(Object[] args) {
            int paramCount = this.params.size();
            if(args != null && paramCount != 0) {
                if(!this.hasNamedParameters && paramCount == 1) {
                    return args[((Integer)this.params.keySet().iterator().next()).intValue()];
                } else {
                    Map<String, Object> param = new MapperMethod.ParamMap();
                    int i = 0;
                    //将入参拼装成key value 形式，并用param+数字作为key对value按照入参顺序排序
                    for(Iterator i$ = this.params.entrySet().iterator(); i$.hasNext(); ++i) {
                        Entry<Integer, String> entry = (Entry)i$.next();
                        param.put(entry.getValue(), args[((Integer)entry.getKey()).intValue()]);
                        String genericParamName = "param" + String.valueOf(i + 1);
                        if(!param.containsKey(genericParamName)) {
                            param.put(genericParamName, args[((Integer)entry.getKey()).intValue()]);
                        }
                    }
                    return param;
                }
            } else {
                return null;
            }
        }
````
封装参数时，不仅将入参封装进去，还会根据入参顺序，用`param` + 数字 作为key，入参作为value 放入 Map 中，如下所示：
![封装参数后的结果](http://upload-images.jianshu.io/upload_images/2710833-b480c8ed254d74d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


封装完参数，就会将 方法的全限名也是`StatementId`(本例中是：`com.school.dao.SchoolCustomerDao.selectBySome`)和封装好的参数传入：
```java
//调用查询 this.command.getName() 为 com.school.dao.SchoolCustomerDao.selectBySome
result = sqlSession.selectOne(this.command.getName(), param);
```
执行到这一步就准备开始解析并生成SQL了，
先取出`Configuration `中的 `MappedStatement`，根据入参进行拼装`SQL`再执行
```java
//从configuration中获取MappedStatement
//statement 为 com.school.dao.SchoolCustomerDao.selectBySome
MappedStatement ms = this.configuration.getMappedStatement(statement);
//调用查询
List<E> result = this.executor.query(ms, this.wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);

/**
 * 调用查询
 */
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        //此处拼装生成BoundSql
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        CacheKey key = this.createCacheKey(ms, parameterObject, rowBounds, boundSql);
        //执行查询
        return this.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }
```




`getBoundSql`拼装的SQL代码为：
```java
/**
 * BoundSql boundSql = ms.getBoundSql(parameter);
 * 调用下面方法类中的方法，获取 BoundSql 
 */
public BoundSql getBoundSql(Object parameterObject) {
        //会根据不同的sqlSource类型执行不同的解析，
       //此处会调用DynamicSqlSource的解析方法，并返回解析好的BoundSql，和已经排好序，需要替换的参数如下图，在下文中会详细解释
        BoundSql boundSql = this.sqlSource.getBoundSql(parameterObject);
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        if(parameterMappings == null || parameterMappings.size() <= 0) {
            boundSql = new BoundSql(this.configuration, boundSql.getSql(), this.parameterMap.getParameterMappings(), parameterObject);
        }
        Iterator i$ = boundSql.getParameterMappings().iterator();

        while(i$.hasNext()) {
            ParameterMapping pm = (ParameterMapping)i$.next();
            String rmId = pm.getResultMapId();
            if(rmId != null) {
                ResultMap rm = this.configuration.getResultMap(rmId);
                if(rm != null) {
                    this.hasNestedResultMaps |= rm.hasNestedResultMaps();
                }
            }
        }

        return boundSql;
    }

```

![替换SQL中对应的参数](http://upload-images.jianshu.io/upload_images/2710833-808dff4a0bdbd5e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```java
//DynamicSqlSource类
public class DynamicSqlSource implements SqlSource {
    private Configuration configuration;
    private SqlNode rootSqlNode;

    public DynamicSqlSource(Configuration configuration, SqlNode rootSqlNode) {
        this.configuration = configuration;
        this.rootSqlNode = rootSqlNode;
    }
    /**
     * 解析SQL
     */
    public BoundSql getBoundSql(Object parameterObject) {
        DynamicContext context = new DynamicContext(this.configuration, parameterObject);
        //会根据 rootSqlNode 中每个节点的内容，会调用 MixedSqlNode 的 apply 方法，解析合并SQL
        this.rootSqlNode.apply(context);
        //用 SqlSourceBuilder 将SQL中的 #{} 换成 ？
        SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(this.configuration);
        Class<?> parameterType = parameterObject == null?Object.class:parameterObject.getClass();
        SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
        BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
        Iterator i$ = context.getBindings().entrySet().iterator();

        while(i$.hasNext()) {
            Entry<String, Object> entry = (Entry)i$.next();
            boundSql.setAdditionalParameter((String)entry.getKey(), entry.getValue());
        }
        return boundSql;
    }
}
```
![debug一些参数的信息](http://upload-images.jianshu.io/upload_images/2710833-eb04fd56e0cfc6f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 ```java
/**
 *  this.rootSqlNode.apply(context);
 *  会调用MixedSqlNode类的方法解析拼装SQL
 */
public class MixedSqlNode implements SqlNode {
    private List<SqlNode> contents;

    public MixedSqlNode(List<SqlNode> contents) {
        this.contents = contents;
    }

    public boolean apply(DynamicContext context) {
        Iterator i$ = this.contents.iterator();

        while(i$.hasNext()) {
            SqlNode sqlNode = (SqlNode)i$.next();
            //进行解析并拼装，此处调用下文的 IfSqlNode 的 apply 方法
            sqlNode.apply(context);
        }

        return true;
    }
}

public class IfSqlNode implements SqlNode {
    private ExpressionEvaluator evaluator;
    private String test;
    private SqlNode contents;

    public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
    }
    //被调用的 IfSqlNode 的 apply 方法 
    public boolean apply(DynamicContext context) {
        //调用 ExpressionEvaluator 的 evaluateBoolean 方法
        //判断执行结果是否为true
        if(this.evaluator.evaluateBoolean(this.test, context.getBindings())) {
            this.contents.apply(context);
            return true;
        } else {
            return false;
        }
    }
}

public class ExpressionEvaluator {
    public ExpressionEvaluator() {
    }
     //被调用的 ExpressionEvaluator 的 evaluateBoolean 方法
    public boolean evaluateBoolean(String expression, Object parameterObject) {
        //调用 Ognl 进行解析，实现细节不再细梳，感兴趣的读者可以DeBug查看
        Object value = OgnlCache.getValue(expression, parameterObject);
        return value instanceof Boolean?((Boolean)value).booleanValue():(value instanceof Number?!(new BigDecimal(String.valueOf(value))).equals(BigDecimal.ZERO):value != null);
    }
}
```

![sqlNode.apply(context);方法的调用在此处使用Ognl进行解析](http://upload-images.jianshu.io/upload_images/2710833-13cafccc8d16d6df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解析完成后，会生成一个已经将参数替换为 ？ 的SQL，在执行的时候只需要调用preparedStatement.setXXX()将List<ParameterMapping> parameterMappings 中的参数按照顺序替换，就可以生成一个SQL。至此，动态解析Mybatis标签生成SQL，已经完成。

> **总结：**
> * Mybatis中`mapper.xml`文件会再加载的时候全部解析为`rootSqlNode`节点
> * 调用mapper的DAO接口时会代理其方法，封装参数，并传入StatementId （方法全限名）
> * 根据StatementId获取到`rootSqlNode`节点，循环调用 `MixedSqlNode `方法，使用Ognl 解析的结果进行拼装返回
>* 使用`SqlSourceBuilder`将#{} 中的内容解析，生成已经排序的参数`List<ParameterMapping> parameterMappings`并将 #{} 替换成 ?
> * 最终调用JDBC中的`preparedStatement.setXXX()`方法，将已经排序的参数`List<ParameterMapping> parameterMappings` 中的参数按照顺序将?替换，便得到一个完整的SQL


`以上就是《Mybatis原理--动态生成SQL》的全部内容，如有不正确的地方，请读者指正，互相学习，共同进步，谢谢。`

<br/>
<br/>

> ### Java高阶-Mybatis原理系列全部文章：

* [Mybatis原理——总纲](http://www.chinaxieshuai.com/mybatis/2017/09/20/Mybatis原理-总纲.html)

* [Mybatis原理——动态生成SQL](http://www.chinaxieshuai.com/mybatis/2017/09/21/Mybatis原理-动态生成SQL.html)

* [Mybatis原理——事务管理](http://www.chinaxieshuai.com/mybatis/2017/09/22/Mybatis原理-事务管理.html)

* [Mybatis原理——数据源和连接池](http://www.chinaxieshuai.com/mybatis/2017/09/23/Mybatis原理-数据源和连接池.html)

* [Mybatis原理——缓存机制(一级缓存)](http://www.chinaxieshuai.com/mybatis/2017/09/24/Mybatis原理-缓存机制(一级缓存).html)

* [Mybatis原理——缓存机制(二级缓存)](http://www.chinaxieshuai.com/mybatis/2017/09/25/Mybatis原理-缓存机制(二级缓存).html)
