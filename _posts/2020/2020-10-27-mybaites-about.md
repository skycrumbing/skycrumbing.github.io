---
layout: post
title: 关于mybaties的一些知识点整理  
tags:
- mybaties  
- spring
- AOP
categories: framework
description: 关于mybaties的一些知识点整理   
---

## mybaties   
这段时间总算抽空将mybaties的一些知识点进行了整理    
<!-- more -->

## 代码执行过程  
### 获取SqlSession     
```
//读取配置
reader = Resources.getResourceAsReader(resource);
SqlSessionFactory factory=new SqlSessionFactoryBuilder().build(reader);
//获取SqlSession
SqlSession session=factory.openSession();
```    

我们跟进factory.openSession()方法看做了什么    
```
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;

        DefaultSqlSession var8;
        try {
            //根据配置获取事务，其中事务的自动提交默认是false，当连续的操作数据中间出现错误时，修改的内容不会提交到数据库，需要手动commit.事务隔离级别默认是null
            Environment environment = this.configuration.getEnvironment();
            TransactionFactory transactionFactory = this.getTransactionFactoryFromEnvironment(environment);
            tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
            //重点是执行器，mybaites中有四种执行器。此时execType是默认执行器类型：SIMPLE，我们下一步进入该方法看看怎么获取
            Executor executor = this.configuration.newExecutor(tx, execType);
            //最后创建sqlsession
            var8 = new DefaultSqlSession(this.configuration, executor, autoCommit);
        } catch (Exception var12) {
            this.closeTransaction(tx);
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + var12, var12);
        } finally {
            ErrorContext.instance().reset();
        }

        return var8;
    }

```  

执行器是如何被创建的：Executor executor = this.configuration.newExecutor(tx, execType)  
```
    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? this.defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Object executor;
        if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
        } else {
            //先获取SimpleExecutor执行器
            executor = new SimpleExecutor(this, transaction);
        }
        //在Configuration的构造方法中cacheEnabled默认为true，此时SimpleExecutor作为CachingExecutor的一个属性被装饰（装饰器模式）
        //它提供了一个二级缓存的功能，如何提供的呢，我们稍后往下看
        if (this.cacheEnabled) {
            executor = new CachingExecutor((Executor)executor);
        }
        //为开发人员提供定制开发插件（拦截器）的功能
        Executor executor = (Executor)this.interceptorChain.pluginAll(executor);
        return executor;
    }

```  
### 获取mapper  
使用过mybaties的同学都知道，我们只需要提供mapper接口，然后通过注解或者XML的方式注入sql就可以直接进行对数据库进行操作，既然是接口，没有实现类是不可能直接调用mapper的方法的，显然这里mybaties使用了代理对象真正执行了数据库操作。  
```
//获取mapper
 StockMapper stockMapper = session.getMapper(StockMapper.class);

```  

根据debuge最终我们定位到关键代码在MapperProxy类中的一个内部类PlainMethodInvoker   
```  
	private static class PlainMethodInvoker implements MapperProxy.MapperMethodInvoker {
        private final MapperMethod mapperMethod;

        public PlainMethodInvoker(MapperMethod mapperMethod) {
            this.mapperMethod = mapperMethod;
        }

        public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
        	//当执行mapper的方法时实际执行的是该方法  
            return this.mapperMethod.execute(sqlSession, args);
        }
    }
```  

### sql执行  
```
//执行sql
 Stock stock = stockMapper.selectByDeviceName("TL20061715844");
```  

当我们执行接口方法时，程序进入MapperMethod中的execute方法  
```
	 Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        Object param;
        switch(this.command.getType()) {
        case INSERT:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.insert(this.command.getName(), param));
            break;
        case UPDATE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.update(this.command.getName(), param));
            break;
        case DELETE:
            param = this.method.convertArgsToSqlCommandParam(args);
            result = this.rowCountResult(sqlSession.delete(this.command.getName(), param));
            break;
        case SELECT:
            if (this.method.returnsVoid() && this.method.hasResultHandler()) {
                this.executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (this.method.returnsMany()) {
                result = this.executeForMany(sqlSession, args);
            } else if (this.method.returnsMap()) {
                result = this.executeForMap(sqlSession, args);
            } else if (this.method.returnsCursor()) {
                result = this.executeForCursor(sqlSession, args);
            } else {
            	//由于方法返回的是单个对象，所以进入这个条件
                param = this.method.convertArgsToSqlCommandParam(args);
                //进入selectOne方法看看
                result = sqlSession.selectOne(this.command.getName(), param);
                if (this.method.returnsOptional() && (result == null || !this.method.getReturnType().equals(result.getClass()))) {
                    result = Optional.ofNullable(result);
                }
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + this.command.getName());
        }

        if (result == null && this.method.getReturnType().isPrimitive() && !this.method.returnsVoid()) {
            throw new BindingException("Mapper method '" + this.command.getName() + " attempted to return null from a method with a primitive return type (" + this.method.getReturnType() + ").");
        } else {
            return result;
        }
    }
```  

进入DefaultSqlSession的selectOne方法  
```
    public <T> T selectOne(String statement, Object parameter) {
    	//最终执行的还是selectList
        List<T> list = this.selectList(statement, parameter);
        if (list.size() == 1) {
            return list.get(0);
        } else if (list.size() > 1) {
            throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
        } else {
            return null;
        }
    }

```

经过debug，代码进入重载的selectList方法
```
	public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        List var5;
        try {
            MappedStatement ms = this.configuration.getMappedStatement(statement);
            //重点，这是进入查询的核心代码，这里 Executor.NO_RESULT_HANDLER是null
            var5 = this.executor.query(ms, this.wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception var9) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + var9, var9);
        } finally {
            ErrorContext.instance().reset();
        }

        return var5;
    }
```  

根据开始时获取Sqlsession的初始化代码可知，这里的executor是CachingExecutor，所以我们进入CachingExecutor的query方法  
```
	public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
        BoundSql boundSql = ms.getBoundSql(parameterObject);
        //创建缓存key对象，这个CacheKey和一级缓存（本地缓存，sqlsession级别）有关，为什么说是sqlsession级别呢？我们稍后会看到关于它的用法
        CacheKey key = this.createCacheKey(ms, parameterObject, rowBounds, boundSql);
        //进入重组的query方法
        return this.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

```  

进入CachingExecutor重组的query方法  
```
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    	//获取二级缓存。MappedStatement中的cache属性，即mapper级别的缓存
    	//TransactionalCacheManager类管理TransactionalCache类，而TransactionalCache修饰了实现Cache接口的类，并且每个TransactionalCache对应一个Cache的实现类，当commit时会调用每个TransactionalCache的commit方法，内部则会调用对应Cache的实现类的clear方法。Cache可由开发者自己实现，使用一些缓存中间件（可以看TransactionalCacheManager源码，本期不在流程内，暂不进行讨论）
        Cache cache = ms.getCache();
        if (cache != null) {
        	//如果存在二级缓存，并且mapper的<select>标签中的flushCache属性设置为true，则会清空二级缓存（直接查询数据库）
            this.flushCacheIfRequired(ms);
            //如果开启二级缓存，即mapper的<select>标签中的useCache属性设置为true
            if (ms.isUseCache() && resultHandler == null) {
                this.ensureNoOutParams(ms, boundSql);
                //先从二级缓存中获取，如果不存在则去查数据库，并且将其放入二级缓存中
                List<E> list = (List)this.tcm.getObject(cache, key);
                if (list == null) {
                    //this.delegate指向被装饰的SimpleExecutor，SimpleExecutor又指向它的抽象类BaseExecutor
                    list = this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                    //将结果放入二级缓存
                    this.tcm.putObject(cache, key, list);
                }

                return list;
            }
        }
        //如果mapper没有启用二级缓存，同上直接调用BaseExecutor的query
        return this.delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

```  

进入BaseExecutor的query方法  
```  
 	public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (this.closed) {
            throw new ExecutorException("Executor was closed.");
        } else {
        	mapper的<select>标签中的flushCache属性设置为true，会清空一级缓存
            if (this.queryStack == 0 && ms.isFlushCacheRequired()) {
                this.clearLocalCache();
            }

            List list;
            try {
                ++this.queryStack;
                //resultHandler默认传过来的是null,所以会先获取一级缓存的数据，如果一级缓存没有，再去数据库查询
                list = resultHandler == null ? (List)this.localCache.getObject(key) : null;
                if (list != null) {
                    //处理缓存数据，这里我们不深入查看
                    this.handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
                } else {
                    //从数据库获取数据,我们下面进入该方法  
                    list = this.queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
                }
            } finally {
                --this.queryStack;
            }

            if (this.queryStack == 0) {
                Iterator var8 = this.deferredLoads.iterator();

                while(var8.hasNext()) {
                    BaseExecutor.DeferredLoad deferredLoad = (BaseExecutor.DeferredLoad)var8.next();
                    deferredLoad.load();
                }

                this.deferredLoads.clear();
                //一级缓存级别是否是STATEMENT(默认值是SESSION),是的话则会直接清空缓存
                if (this.configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
                    this.clearLocalCache();
                }
            }

            return list;
        }
    }


```  

我们接着进入BaseExecutor的queryFromDatabase方法  
```
 	private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        this.localCache.putObject(key, ExecutionPlaceholder.EXECUTION_PLACEHOLDER);

        List list;
        try {
            //查询数据库，具体的查询逻辑就是jdbc的查询逻辑，然后将结果进行属性映射
            list = this.doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
            this.localCache.removeObject(key);
        }
        //重新放入一级缓存
        this.localCache.putObject(key, list);
        //这边是放入另外一个缓存对象中，作用未知
        if (ms.getStatementType() == StatementType.CALLABLE) {
            this.localOutputParameterCache.putObject(key, parameter);
        }

        return list;
    }

```  

这里为什么说一级缓存是sqlSession级别的呢，这是因为localCache是BaseExecutor的属性，而Executor是由sqlSession创建，并且set到sqlSession的executor属性，如果，是另外的sqlSession，则指向的则是另外一个执行器，同样指向的localCache自然不是同一个。在进行commit操作后，会将缓存清空，这里就不在进行代码梳理。  

## Spring mybaties
接下来我们可以看看mybaites是怎么和spring进行整合的，顺便巩固一下spring的知识  
### spring如何获取mapper  
通过上面的学习我们可以知道mybaties通过sqlsession的getMapper方法获取到mapper的代理对象，然后由代理对象执行真正的查询。那spring是怎么获取到代理对象的呢。  
首先我们知道在springboot中可以通过在启动类上加@MapperScan注解就可以由IOC容器生成mapper的代理对象。我们先进入MapperScan这个注解类  
```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
//重要的是这个@import,它将MapperScannerRegistrar注册到ioc容器
@Import({MapperScannerRegistrar.class})
public @interface MapperScan {
    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

    Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

    Class<? extends Annotation> annotationClass() default Annotation.class;

    Class<?> markerInterface() default Class.class;

    String sqlSessionTemplateRef() default "";

    String sqlSessionFactoryRef() default "";
    //重点关注一下 MapperFactoryBean
    Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

    String[] properties() default {};

    String mapperHelperRef() default "";
}
```  

我们进入MapperScannerRegistrar类中  

```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package tk.mybatis.spring.annotation;

import java.lang.annotation.Annotation;
import java.util.ArrayList;
import java.util.List;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanNameGenerator;
import org.springframework.context.EnvironmentAware;
import org.springframework.context.ResourceLoaderAware;
import org.springframework.context.annotation.ImportBeanDefinitionRegistrar;
import org.springframework.core.annotation.AnnotationAttributes;
import org.springframework.core.env.Environment;
import org.springframework.core.io.ResourceLoader;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.util.ClassUtils;
import org.springframework.util.StringUtils;
import tk.mybatis.spring.mapper.ClassPathMapperScanner;
import tk.mybatis.spring.mapper.MapperFactoryBean;

//MapperScannerRegistrar实现了ImportBeanDefinitionRegistrar方法，所以@import该类时会调用registerBeanDefinitions方法将其中要注册的类注册成bean
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    public static final Logger LOGGER = LoggerFactory.getLogger(MapperScannerRegistrar.class);
    private ResourceLoader resourceLoader;
    private Environment environment;

    public MapperScannerRegistrar() {
    }

    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    	//下面都是获取MapperScan的配置和相关包下面的所有mapper接口
        AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
        if (this.resourceLoader != null) {
            scanner.setResourceLoader(this.resourceLoader);
        }

        Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
        if (!Annotation.class.equals(annotationClass)) {
            scanner.setAnnotationClass(annotationClass);
        }

        Class<?> markerInterface = annoAttrs.getClass("markerInterface");
        if (!Class.class.equals(markerInterface)) {
            scanner.setMarkerInterface(markerInterface);
        }

        Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
        if (!BeanNameGenerator.class.equals(generatorClass)) {
            scanner.setBeanNameGenerator((BeanNameGenerator)BeanUtils.instantiateClass(generatorClass));
        }
        //重点关注一下 MapperFactoryBean
        Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
        if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
            scanner.setMapperFactoryBean((MapperFactoryBean)BeanUtils.instantiateClass(mapperFactoryBeanClass));
        }

        scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
        scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));
        List<String> basePackages = new ArrayList();
        String[] var10 = annoAttrs.getStringArray("value");
        int var11 = var10.length;

        int var12;
        String pkg;
        for(var12 = 0; var12 < var11; ++var12) {
            pkg = var10[var12];
            if (StringUtils.hasText(pkg)) {
                basePackages.add(pkg);
            }
        }

        var10 = annoAttrs.getStringArray("basePackages");
        var11 = var10.length;

        for(var12 = 0; var12 < var11; ++var12) {
            pkg = var10[var12];
            if (StringUtils.hasText(pkg)) {
                basePackages.add(pkg);
            }
        }

        Class[] var15 = annoAttrs.getClassArray("basePackageClasses");
        var11 = var15.length;

        for(var12 = 0; var12 < var11; ++var12) {
            Class<?> clazz = var15[var12];
            basePackages.add(ClassUtils.getPackageName(clazz));
        }

        String mapperHelperRef = annoAttrs.getString("mapperHelperRef");
        String[] properties = annoAttrs.getStringArray("properties");
        if (StringUtils.hasText(mapperHelperRef)) {
            scanner.setMapperHelperBeanName(mapperHelperRef);
        } else if (properties != null && properties.length > 0) {
            scanner.setMapperProperties(properties);
        } else {
            try {
                scanner.setMapperProperties(this.environment);
            } catch (Exception var14) {
                LOGGER.warn("只有 Spring Boot 环境中可以通过 Environment(配置文件,环境变量,运行参数等方式) 配置通用 Mapper，其他环境请通过 @MapperScan 注解中的 mapperHelperRef 或 properties 参数进行配置!如果你使用 tk.mybatis.mapper.session.Configuration 配置的通用 Mapper，你可以忽略该错误!", var14);
            }
        }

        scanner.registerFilters();
        //进入doScan方法查看逻辑
        scanner.doScan(StringUtils.toStringArray(basePackages));
    }

    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }
}
```  
现在进入ClassPathMapperScanner的doScan方法  
```
    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);
        if (beanDefinitions.isEmpty()) {
            this.logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
        } else {
            //进入该方法
            this.processBeanDefinitions(beanDefinitions);
        }

        return beanDefinitions;
    }

    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        Iterator var3 = beanDefinitions.iterator();

        while(var3.hasNext()) {
            BeanDefinitionHolder holder = (BeanDefinitionHolder)var3.next();
            GenericBeanDefinition definition = (GenericBeanDefinition)holder.getBeanDefinition();
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + definition.getBeanClassName() + "' mapperInterface");
            }

            definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName());
            //非常重要，这是我们前面重点关注的MapperFactoryBean 最终实际注册的对象就是MapperFactoryBean
            definition.setBeanClass(this.mapperFactoryBean.getClass());
            if (StringUtils.hasText(this.mapperHelperBeanName)) {
                definition.getPropertyValues().add("mapperHelper", new RuntimeBeanReference(this.mapperHelperBeanName));
            } else {
                if (this.mapperHelper == null) {
                    this.mapperHelper = new MapperHelper();
                }

                definition.getPropertyValues().add("mapperHelper", this.mapperHelper);
            }

            definition.getPropertyValues().add("addToConfig", this.addToConfig);
            boolean explicitFactoryUsed = false;
            if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
                definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
                explicitFactoryUsed = true;
            } else if (this.sqlSessionFactory != null) {
                definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
                explicitFactoryUsed = true;
            }

            if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
                if (explicitFactoryUsed) {
                    this.logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
                }

                definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
                explicitFactoryUsed = true;
            } else if (this.sqlSessionTemplate != null) {
                if (explicitFactoryUsed) {
                    this.logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
                }
                //sqlSessionTemplate也非常重要，暂时不做讨论
                definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
                explicitFactoryUsed = true;
            }

            if (!explicitFactoryUsed) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
                }

                definition.setAutowireMode(2);
            }
        }

    }
```

我们不继续深入，只需要知道这里会注册一个很重要的类MapperFactoryBean   
```
package tk.mybatis.spring.mapper;

import org.apache.ibatis.executor.ErrorContext;
import org.apache.ibatis.session.Configuration;
import org.mybatis.spring.support.SqlSessionDaoSupport;
import org.springframework.beans.factory.FactoryBean;
import org.springframework.util.Assert;
import tk.mybatis.mapper.mapperhelper.MapperHelper;
//实现了FactoryBean接口，所以在MapperFactoryBean被实例化时，返回的是getObject方法返回的对象  
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    private Class<T> mapperInterface;
    private boolean addToConfig = true;
    private MapperHelper mapperHelper;

    public MapperFactoryBean() {
    }

    public MapperFactoryBean(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    protected void checkDaoConfig() {
        super.checkDaoConfig();
        Assert.notNull(this.mapperInterface, "Property 'mapperInterface' is required");
        Configuration configuration = this.getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception var6) {
                this.logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", var6);
                throw new IllegalArgumentException(var6);
            } finally {
                ErrorContext.instance().reset();
            }
        }

        if (configuration.hasMapper(this.mapperInterface) && this.mapperHelper != null && this.mapperHelper.isExtendCommonMapper(this.mapperInterface)) {
            this.mapperHelper.processConfiguration(this.getSqlSession().getConfiguration(), this.mapperInterface);
        }

    }

    public Class<T> getMapperInterface() {
        return this.mapperInterface;
    }

    public void setMapperInterface(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public T getObject() throws Exception {
    	//返回的依旧是其代理对象
        return this.getSqlSession().getMapper(this.mapperInterface);
    }

    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    public boolean isAddToConfig() {
        return this.addToConfig;
    }

    public void setAddToConfig(boolean addToConfig) {
        this.addToConfig = addToConfig;
    }

    public void setMapperHelper(MapperHelper mapperHelper) {
        this.mapperHelper = mapperHelper;
    }

    public boolean isSingleton() {
        return true;
    }
}

```  
总结，spring利用了MapperFactoryBean，最终还是调用了sqlsession的getMapper返回了mapper的代理对象



 
