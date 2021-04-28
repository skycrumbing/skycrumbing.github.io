---
layout: post
title: spirng-rabbit和它的重试机制  
tags:
- spirng-rabbit  
- spring
- spring-retry
categories: framework
description: 探索spring-rabbit的执行流程和重试机制   
---

## spirng-rabbit   
因为项目用到rabbit的地方比较多，所以逐步debug了spring中整合rabbit的执行流程方便以后更好的使用，并且加深对spring的了解。    
<!-- more -->
测试版本  
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.2.13.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

## 消费失败的情况以及处理  
在项目中用到消息队列最多的地方是异步通知和存储数据。异步通知的时候可能由于网络等外部因素失败，这种情况下就应该重复执行异步通知的方法，并且如果每次执行都失败，需要设置最多执行次数和每次执行的时间间隔。 
org.springframework.amqp.rabbit.listener.BlockingQueueConsumer#rollbackOnExceptionIfNecessary 
在测试项目中，按照最基本的配置（即只配置rabbit的连接信息）来进行消费测试，如果消费失败，代码会执行channel.basicNack(long deliveryTag, boolean multiple, boolean requeue)，默认requeue是true，所以会重新将消息放入队列，等着被重新消费。这样如果一直消费失败将会阻塞队列，造成生产事故。  
如果想按照预期的目的（消费失败重试，并且限制次数和时间间隔），有两种实现方式：  
1. 根据rabbit死信交换机(DLX dead-letter-exchange)的特性。  
    将队列和死信交换机绑定，设置消息超时时间，当消息超时或者消息堆积超过指定大小时会将队列的消息丢弃到死信交换机，再通过指定队列的方式将死信交换机的消息分发到新的指定队列，然后消费新的指定队列的消息。如果消费失败，将其再发送到最开始的和死信交换机绑定的队列所在交换机，并在消息头部记录失败次数。消费的时候查看失败次数，如果失败次数达到上限，则将消息丢弃，完成消费。
2. 使用spring-rabbit自带的retry功能  
    spring-retry是一个重试框架，他通过代理方式，捕捉异常，然后根据重试机制，比如固定间隔，前一次时间间隔的倍数，通过Thread.sleep的方式，重新执行消费的方法。  
    spring-retry有几个重要的类  
    BackOffPolicy：重试的回退策略，指以何种方式进行下一次重试（第一次重试后什么时候进行第二次重试），比如过了15秒后重试，随机时间重试。  
    RetryPolicy：重试策略或条件，可以指定超时重试，一直重试，简单重试等。  
    MessageRecoverer：消息回收类，当所有的重试次数都失败后，就会调用该类的recover方法,在rabbit默认是org.springframework.amqp.rabbit.retry.RejectAndDontRequeueRecoverer。即失败了也不会再冲洗放入队列，直接channel.basicAck
    RetryTemplate：组合了BackOffPolicy，RetryPolicy，RetryListener，执行重试步骤的具体类。  
    RetryOperationsInterceptor：方法执行失败的拦截器类，拦截失败后交给RertyTemplate去执行重试。  
    SimpleMessageListenerContainer：用于管理消费者。  
    RetryListener：重试过程的监听器，第一次重试调用该类的open，每次重试不成功调用onError，最后一次重试调用close。  
相比前一种，使用retry更加简单，只需要添加相应的配置信息即可，代码的侵入性很低。 我们现在就来看看该方式代码的执行逻辑。  

## 测试项目rabbit的相关配置     
```
spring:
  rabbitmq:
    host: localhost
    username: guest
    password: guest
    port: 5672
    listener:
      simple:
        retry:
          enabled: true #是否开启消费者重试（为false时关闭消费者重试，这时消费端代码异常会一直重复收到消息）
          max-attempts: 5 #最大重试次数
          initial-interval: 1000 #重试间隔时间（单位毫秒）
          max-interval: 1200000 #重试最大时间间隔（单位毫秒）
          multiplier: 2 #应用于前一重试间隔的乘
```    
## 代码执行逻辑      

### 首先先看看项目初始化做了什么  
在springboot中，是通过spring-boot-autoconfigure实现自动装配的。即上面的配置信息是通过这个模块来进行相关bean的初始化的。  
springboot通过/META-INF/spring.factories文件去加载的第三方jar包的bean。在spring-boot-autoconfigure中的/META-INF/spring.factories可以看到这些信息  
```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudServiceConnectorsAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\

```  
这边我只复制了部分代码，只需要知道这边加载了org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration。 
我们进入RabbitAutoConfiguration看看它实现了哪些方法。  
```
/**
 * 这个类大致就是通过看有没有引入spring-rabbit依赖来决定是否将一些重要组件注入IOC容器。其中：   
 * 注册了rabbitTemplate：管具体的收发消息
 */
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({RabbitTemplate.class, Channel.class})
//RabbitProperties是我们的配置类，里面的属性就是我们上边配置的信息。  
//因为RabbitProperties类只配置@ConfigurationProperties注解，而没有使用@Component，  
//那么在IOC容器中是获取不到properties 配置文件转化的bean。通过@EnableConfigurationProperties把RabbitProperties的属性又注入了一次。
@EnableConfigurationProperties({RabbitProperties.class})
//将RabbitAnnotationDrivenConfiguration类导入IOC容器中，我们稍后看看这个配置类做了什么。
@Import({RabbitAnnotationDrivenConfiguration.class})
public class RabbitAutoConfiguration {
    public RabbitAutoConfiguration() {
    }

    @Configuration(
        proxyBeanMethods = false
    )
    @ConditionalOnClass({RabbitMessagingTemplate.class})
    @ConditionalOnMissingBean({RabbitMessagingTemplate.class})
    @Import({RabbitAutoConfiguration.RabbitTemplateConfiguration.class})
    protected static class MessagingTemplateConfiguration {
        protected MessagingTemplateConfiguration() {
        }

        @Bean
        @ConditionalOnSingleCandidate(RabbitTemplate.class)
        public RabbitMessagingTemplate rabbitMessagingTemplate(RabbitTemplate rabbitTemplate) {
            return new RabbitMessagingTemplate(rabbitTemplate);
        }
    }

    @Configuration(
        proxyBeanMethods = false
    )
    @Import({RabbitAutoConfiguration.RabbitConnectionFactoryCreator.class})
    protected static class RabbitTemplateConfiguration {
        protected RabbitTemplateConfiguration() {
        }

        /**
         *重点关注
         */
        @Bean
        @ConditionalOnSingleCandidate(ConnectionFactory.class)
        @ConditionalOnMissingBean({RabbitOperations.class})
        public RabbitTemplate rabbitTemplate(RabbitProperties properties, ObjectProvider<MessageConverter> messageConverter, ObjectProvider<RabbitRetryTemplateCustomizer> retryTemplateCustomizers, ConnectionFactory connectionFactory) {
            PropertyMapper map = PropertyMapper.get();
            RabbitTemplate template = new RabbitTemplate(connectionFactory);
            messageConverter.ifUnique(template::setMessageConverter);
            template.setMandatory(this.determineMandatoryFlag(properties));
            Template templateProperties = properties.getTemplate();
            if (templateProperties.getRetry().isEnabled()) {
                template.setRetryTemplate((new RetryTemplateFactory((List)retryTemplateCustomizers.orderedStream().collect(Collectors.toList()))).createRetryTemplate(templateProperties.getRetry(), Target.SENDER));
            }

            templateProperties.getClass();
            map.from(templateProperties::getReceiveTimeout).whenNonNull().as(Duration::toMillis).to(template::setReceiveTimeout);
            templateProperties.getClass();
            map.from(templateProperties::getReplyTimeout).whenNonNull().as(Duration::toMillis).to(template::setReplyTimeout);
            templateProperties.getClass();
            map.from(templateProperties::getExchange).to(template::setExchange);
            templateProperties.getClass();
            map.from(templateProperties::getRoutingKey).to(template::setRoutingKey);
            templateProperties.getClass();
            map.from(templateProperties::getDefaultReceiveQueue).whenNonNull().to(template::setDefaultReceiveQueue);
            return template;
        }

        private boolean determineMandatoryFlag(RabbitProperties properties) {
            Boolean mandatory = properties.getTemplate().getMandatory();
            return mandatory != null ? mandatory : properties.isPublisherReturns();
        }

        @Bean
        @ConditionalOnSingleCandidate(ConnectionFactory.class)
        @ConditionalOnProperty(
            prefix = "spring.rabbitmq",
            name = {"dynamic"},
            matchIfMissing = true
        )
        @ConditionalOnMissingBean
        public AmqpAdmin amqpAdmin(ConnectionFactory connectionFactory) {
            return new RabbitAdmin(connectionFactory);
        }
    }

    @Configuration(
        proxyBeanMethods = false
    )
    @ConditionalOnMissingBean({ConnectionFactory.class})
    protected static class RabbitConnectionFactoryCreator {
        protected RabbitConnectionFactoryCreator() {
        }

        @Bean
        public CachingConnectionFactory rabbitConnectionFactory(RabbitProperties properties, ObjectProvider<ConnectionNameStrategy> connectionNameStrategy) throws Exception {
            PropertyMapper map = PropertyMapper.get();
            CachingConnectionFactory factory = new CachingConnectionFactory((com.rabbitmq.client.ConnectionFactory)this.getRabbitConnectionFactoryBean(properties).getObject());
            properties.getClass();
            map.from(properties::determineAddresses).to(factory::setAddresses);
            properties.getClass();
            map.from(properties::isPublisherReturns).to(factory::setPublisherReturns);
            properties.getClass();
            map.from(properties::getPublisherConfirmType).whenNonNull().to(factory::setPublisherConfirmType);
            org.springframework.boot.autoconfigure.amqp.RabbitProperties.Cache.Channel channel = properties.getCache().getChannel();
            channel.getClass();
            map.from(channel::getSize).whenNonNull().to(factory::setChannelCacheSize);
            channel.getClass();
            map.from(channel::getCheckoutTimeout).whenNonNull().as(Duration::toMillis).to(factory::setChannelCheckoutTimeout);
            Connection connection = properties.getCache().getConnection();
            connection.getClass();
            map.from(connection::getMode).whenNonNull().to(factory::setCacheMode);
            connection.getClass();
            map.from(connection::getSize).whenNonNull().to(factory::setConnectionCacheSize);
            connectionNameStrategy.getClass();
            map.from(connectionNameStrategy::getIfUnique).whenNonNull().to(factory::setConnectionNameStrategy);
            return factory;
        }

        private RabbitConnectionFactoryBean getRabbitConnectionFactoryBean(RabbitProperties properties) throws Exception {
            PropertyMapper map = PropertyMapper.get();
            RabbitConnectionFactoryBean factory = new RabbitConnectionFactoryBean();
            properties.getClass();
            map.from(properties::determineHost).whenNonNull().to(factory::setHost);
            properties.getClass();
            map.from(properties::determinePort).to(factory::setPort);
            properties.getClass();
            map.from(properties::determineUsername).whenNonNull().to(factory::setUsername);
            properties.getClass();
            map.from(properties::determinePassword).whenNonNull().to(factory::setPassword);
            properties.getClass();
            map.from(properties::determineVirtualHost).whenNonNull().to(factory::setVirtualHost);
            properties.getClass();
            map.from(properties::getRequestedHeartbeat).whenNonNull().asInt(Duration::getSeconds).to(factory::setRequestedHeartbeat);
            Ssl ssl = properties.getSsl();
            if (ssl.determineEnabled()) {
                factory.setUseSSL(true);
                ssl.getClass();
                map.from(ssl::getAlgorithm).whenNonNull().to(factory::setSslAlgorithm);
                ssl.getClass();
                map.from(ssl::getKeyStoreType).to(factory::setKeyStoreType);
                ssl.getClass();
                map.from(ssl::getKeyStore).to(factory::setKeyStore);
                ssl.getClass();
                map.from(ssl::getKeyStorePassword).to(factory::setKeyStorePassphrase);
                ssl.getClass();
                map.from(ssl::getTrustStoreType).to(factory::setTrustStoreType);
                ssl.getClass();
                map.from(ssl::getTrustStore).to(factory::setTrustStore);
                ssl.getClass();
                map.from(ssl::getTrustStorePassword).to(factory::setTrustStorePassphrase);
                ssl.getClass();
                map.from(ssl::isValidateServerCertificate).to((validate) -> {
                    factory.setSkipServerCertificateValidation(!validate);
                });
                ssl.getClass();
                map.from(ssl::getVerifyHostname).to(factory::setEnableHostnameVerification);
            }

            properties.getClass();
            map.from(properties::getConnectionTimeout).whenNonNull().asInt(Duration::toMillis).to(factory::setConnectionTimeout);
            factory.afterPropertiesSet();
            return factory;
        }
    }
}

```

现在我们看看通过@Import导入IOC容器的org.springframework.boot.autoconfigure.amqp.RabbitAnnotationDrivenConfiguration  
```
/**
 * 这个类和RabbitAutoConfiguration一样，通过判断有没有引入spring-rabbit相关依赖决定是否实例化一些重要类，我们需要重点关注：   
 * SimpleRabbitListenerContainerFactory：这个是SimpleRabbitListenerContainer创建工厂  
 * SimpleRabbitListenerContainer作用是循环监听队列是否产生消息，如果有则获取进行处理  
 * 我们下一步进入SimpleRabbitListenerContainerFactoryConfigurer.config()方法看看为SimpleRabbitListenerContainerFactory配置了哪些东西
 */
@Confi
@Configuration(
    proxyBeanMethods = false
)
//这个类是否创建取决于是否存在EnableRabbit这个类，这个类同样存在于spring-rabbit，如果没有引入该依赖，这个类无法实例化
@ConditionalOnClass({EnableRabbit.class})
class RabbitAnnotationDrivenConfiguration {
    private final ObjectProvider<MessageConverter> messageConverter;
    private final ObjectProvider<MessageRecoverer> messageRecoverer;
    private final ObjectProvider<RabbitRetryTemplateCustomizer> retryTemplateCustomizers;
    private final RabbitProperties properties;

    RabbitAnnotationDrivenConfiguration(ObjectProvider<MessageConverter> messageConverter, ObjectProvider<MessageRecoverer> messageRecoverer, ObjectProvider<RabbitRetryTemplateCustomizer> retryTemplateCustomizers, RabbitProperties properties) {
        this.messageConverter = messageConverter;
        this.messageRecoverer = messageRecoverer;
        this.retryTemplateCustomizers = retryTemplateCustomizers;
        this.properties = properties;
    }

    @Bean
    @ConditionalOnMissingBean
    SimpleRabbitListenerContainerFactoryConfigurer simpleRabbitListenerContainerFactoryConfigurer() {
        SimpleRabbitListenerContainerFactoryConfigurer configurer = new SimpleRabbitListenerContainerFactoryConfigurer();
        configurer.setMessageConverter((MessageConverter)this.messageConverter.getIfUnique());
        configurer.setMessageRecoverer((MessageRecoverer)this.messageRecoverer.getIfUnique());
        configurer.setRetryTemplateCustomizers((List)this.retryTemplateCustomizers.orderedStream().collect(Collectors.toList()));
        configurer.setRabbitProperties(this.properties);
        return configurer;
    }

    /**
     *重点关注
     */
    @Bean(
        name = {"rabbitListenerContainerFactory"}
    )
    @ConditionalOnMissingBean(
        name = {"rabbitListenerContainerFactory"}
    )
    @ConditionalOnProperty(
        prefix = "spring.rabbitmq.listener",
        name = {"type"},
        havingValue = "simple",
        matchIfMissing = true
    )
    SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory(SimpleRabbitListenerContainerFactoryConfigurer configurer, ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        configurer.configure(factory, connectionFactory);
        return factory;
    }

    @Bean
    @ConditionalOnMissingBean
    DirectRabbitListenerContainerFactoryConfigurer directRabbitListenerContainerFactoryConfigurer() {
        DirectRabbitListenerContainerFactoryConfigurer configurer = new DirectRabbitListenerContainerFactoryConfigurer();
        configurer.setMessageConverter((MessageConverter)this.messageConverter.getIfUnique());
        configurer.setMessageRecoverer((MessageRecoverer)this.messageRecoverer.getIfUnique());
        configurer.setRetryTemplateCustomizers((List)this.retryTemplateCustomizers.orderedStream().collect(Collectors.toList()));
        configurer.setRabbitProperties(this.properties);
        return configurer;
    }

    @Bean(
        name = {"rabbitListenerContainerFactory"}
    )
    @ConditionalOnMissingBean(
        name = {"rabbitListenerContainerFactory"}
    )
    @ConditionalOnProperty(
        prefix = "spring.rabbitmq.listener",
        name = {"type"},
        havingValue = "direct"
    )
    DirectRabbitListenerContainerFactory directRabbitListenerContainerFactory(DirectRabbitListenerContainerFactoryConfigurer configurer, ConnectionFactory connectionFactory) {
        DirectRabbitListenerContainerFactory factory = new DirectRabbitListenerContainerFactory();
        configurer.configure(factory, connectionFactory);
        return factory;
    }

    @Configuration(
        proxyBeanMethods = false
    )
    @EnableRabbit
    @ConditionalOnMissingBean(
        name = {"org.springframework.amqp.rabbit.config.internalRabbitListenerAnnotationProcessor"}
    )
    static class EnableRabbitConfiguration {
        EnableRabbitConfiguration() {
        }
    }
}

```
进入org.springframework.boot.autoconfigure.amqp.SimpleRabbitListenerContainerFactoryConfigurer#configure方法  
```
/**
 * 这里的factory指的是SimpleRabbitListenerContainerFactory
 */
protected void configure(T factory, ConnectionFactory connectionFactory, AmqpContainer configuration) {
        public void configure(SimpleRabbitListenerContainerFactory factory, ConnectionFactory connectionFactory) {
        PropertyMapper map = PropertyMapper.get();
        SimpleContainer config = this.getRabbitProperties().getListener().getSimple();
        //进入父类AbstractRabbitListenerContainerFactoryConfigurer的configure方法
        this.configure(factory, connectionFactory, config);
        config.getClass();
        map.from(config::getConcurrency).whenNonNull().to(factory::setConcurrentConsumers);
        config.getClass();
        map.from(config::getMaxConcurrency).whenNonNull().to(factory::setMaxConcurrentConsumers);
        config.getClass();
        map.from(config::getBatchSize).whenNonNull().to(factory::setBatchSize);
    }
}
```
进入org.springframework.boot.autoconfigure.amqp.AbstractRabbitListenerContainerFactoryConfigurer#configure(T, org.springframework.amqp.rabbit.connection.ConnectionFactory, org.springframework.boot.autoconfigure.amqp.RabbitProperties.AmqpContainer)方法  
```
    /**
     * 这里的factory指的是SimpleRabbitListenerContainerFactory
     * 重点是设置了retry相关信息
     * 这里可以注意一下设置了默认的MessageRecoverer：RejectAndDontRequeueRecoverer。当所有的重试次数都失败后，就会调用该类的recover方法  
     * 该类的recover方法是直接对消息进行ack确认，然后抛弃，不再重新入队。但是源码中recover方法并没有相关操作，这里先不做了解。
     */
    protected void configure(T factory, ConnectionFactory connectionFactory, AmqpContainer configuration) {
        Assert.notNull(factory, "Factory must not be null");
        Assert.notNull(connectionFactory, "ConnectionFactory must not be null");
        Assert.notNull(configuration, "Configuration must not be null");
        factory.setConnectionFactory(connectionFactory);
        if (this.messageConverter != null) {
            factory.setMessageConverter(this.messageConverter);
        }

        factory.setAutoStartup(configuration.isAutoStartup());
        if (configuration.getAcknowledgeMode() != null) {
            factory.setAcknowledgeMode(configuration.getAcknowledgeMode());
        }

        if (configuration.getPrefetch() != null) {
            factory.setPrefetchCount(configuration.getPrefetch());
        }

        if (configuration.getDefaultRequeueRejected() != null) {
            factory.setDefaultRequeueRejected(configuration.getDefaultRequeueRejected());
        }

        if (configuration.getIdleEventInterval() != null) {
            factory.setIdleEventInterval(configuration.getIdleEventInterval().toMillis());
        }

        factory.setMissingQueuesFatal(configuration.isMissingQueuesFatal());
        ListenerRetry retryConfig = configuration.getRetry();
        //如果配置信息开了listern的重试机制，则将RetryOperationsInterceptor配置到工厂类，这是一个方法拦截器，创建SimpleRabbitListenerContainer时会使用到相关代理
        if (retryConfig.isEnabled()) {
            RetryInterceptorBuilder<?, ?> builder = retryConfig.isStateless() ? RetryInterceptorBuilder.stateless() : RetryInterceptorBuilder.stateful();
            RetryTemplate retryTemplate = (new RetryTemplateFactory(this.retryTemplateCustomizers)).createRetryTemplate(retryConfig, Target.LISTENER);
            ((RetryInterceptorBuilder)builder).retryOperations(retryTemplate);
            MessageRecoverer recoverer = this.messageRecoverer != null ? this.messageRecoverer : new RejectAndDontRequeueRecoverer();
            ((RetryInterceptorBuilder)builder).recoverer((MessageRecoverer)recoverer);
            factory.setAdviceChain(new Advice[]{((RetryInterceptorBuilder)builder).build()});
        }

    }
```  
初始化的部分先看到这里，接下来具体看发送和接收的执行流程。主要设计spring-rabbit组件。  

 ### 消息发送流程  
 在测试项目中，发送消息的代码如下  
 ```
@Component
public class RabbitProducer {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void sendExposureinfo(String requestVo) {
        //参数分别是交换机名称，路由规则，消息内容
        this.rabbitTemplate.convertAndSend(RabbitConfig.AD_EXCHANGE, RabbitConfig.AD_SAVE_EXPOSUREINFO_ROUTING_KEY, requestVo);
    }
}
```
我们直接跟进具体实现的方法看看里面具体的逻辑  
该方法是org.springframework.amqp.rabbit.core.RabbitTemplate#convertAndSend(java.lang.String, java.lang.String, java.lang.Object)  
```
//这边CorrelationData是和消息绑定关联的，方便之后进行重发报警
    public void convertAndSend(String exchange, String routingKey, Object object) throws AmqpException {
        this.convertAndSend(exchange, routingKey, object, (CorrelationData)null);
    }
```   
继续跟进org.springframework.amqp.rabbit.core.RabbitTemplate#convertAndSend(java.lang.String, java.lang.String, java.lang.Object, org.springframework.amqp.rabbit.connection.CorrelationData)    
```
//这边主要将消息转化为Message类型的对象
    public void convertAndSend(String exchange, String routingKey, Object object, @Nullable CorrelationData correlationData) throws AmqpException {
        this.send(exchange, routingKey, this.convertMessageIfNecessary(object), correlationData);
    }
```  
跟进org.springframework.amqp.rabbit.core.RabbitTemplate#send(java.lang.String, java.lang.String, org.springframework.amqp.core.Message, org.springframework.amqp.rabbit.connection.CorrelationData)  
```
//这里真正开始执行发送操作，这里调用了org.springframework.amqp.rabbit.core.RabbitTemplate#execute(org.springframework.amqp.rabbit.core.ChannelCallback<T>, org.springframework.amqp.rabbit.connection.ConnectionFactory)  
//传入了一个利用lambda实现了ChannelCallback接口的对象，该对象实现了org.springframework.amqp.rabbit.core.ChannelCallback#doInRabbit方法，下面代码中该方法调用了doSend发送消息，我们接下来看这个发送消息的逻辑什么时候执行。还传入了一个连接工厂获取一个rabbit连接
    public void send(String exchange, String routingKey, Message message, @Nullable CorrelationData correlationData) throws AmqpException {
        this.execute((channel) -> {
            this.doSend(channel, exchange, routingKey, message, (this.returnCallback != null || correlationData != null && StringUtils.hasText(correlationData.getId())) && (Boolean)this.mandatoryExpression.getValue(this.evaluationContext, message, Boolean.class), correlationData);
            return null;
        }, this.obtainTargetConnectionFactory(this.sendConnectionFactorySelectorExpression, message));
    }
```  
跟进org.springframework.amqp.rabbit.core.RabbitTemplate#execute(org.springframework.amqp.rabbit.core.ChannelCallback<T>, org.springframework.amqp.rabbit.connection.ConnectionFactory)  
```
    @Nullable
    private <T> T execute(ChannelCallback<T> action, ConnectionFactory connectionFactory) {
        if (this.retryTemplate != null) {
            try {
                return this.retryTemplate.execute((context) -> {
                    return this.doExecute(action, connectionFactory);
                }, this.recoveryCallback);
            } catch (RuntimeException var4) {
                throw var4;
            } catch (Exception var5) {
                throw RabbitExceptionTranslator.convertRabbitAccessException(var5);
            }
        } else {
            //因为没有配置retryTemplate，所以执行到这里
            return this.doExecute(action, connectionFactory);
        }
    }
```
跟进org.springframework.amqp.rabbit.core.RabbitTemplate#doExecute  
```
    @Nullable
    private <T> T doExecute(ChannelCallback<T> action, ConnectionFactory connectionFactory) {
        Assert.notNull(action, "Callback object must not be null");
        Channel channel = null;
        boolean invokeScope = false;
        if (this.activeTemplateCallbacks.get() > 0) {
            channel = (Channel)this.dedicatedChannels.get();
        }

        RabbitResourceHolder resourceHolder = null;
        Connection connection = null;
        if (channel == null) {
            if (this.isChannelTransacted()) {
                resourceHolder = ConnectionFactoryUtils.getTransactionalResourceHolder(connectionFactory, true, this.usePublisherConnection);
                channel = resourceHolder.getChannel();
                if (channel == null) {
                    ConnectionFactoryUtils.releaseResources(resourceHolder);
                    throw new IllegalStateException("Resource holder returned a null channel");
                }
            } else {
                //这里获取连接，具体的实现是org.springframework.amqp.rabbit.connection.CachingConnectionFactory#createConnection  
                //该连接第一次创建之后就一直缓存着，之后就不会再创建，而是直接获取，第一次创建就是在项目启动的时候初始化申明交换机，队列和路由规则。因为连接只有一条，可能涉及到加锁抢连接的操作
                connection = ConnectionFactoryUtils.createConnection(connectionFactory, this.usePublisherConnection);
                if (connection == null) {
                    throw new IllegalStateException("Connection factory returned a null connection");
                }

                try {
                    //创建channel,并且不开启事务
                    channel = connection.createChannel(false);
                    if (channel == null) {
                        throw new IllegalStateException("Connection returned a null channel");
                    }
                } catch (RuntimeException var12) {
                    RabbitUtils.closeConnection(connection);
                    throw var12;
                }
            }
        } else {
            invokeScope = true;
        }

        Object var7;
        try {
            //该方法就会开始执行ChannelCallback的doInRabbit方法，我们跟进具体看看
            var7 = this.invokeAction(action, connectionFactory, channel);
        } catch (Exception var13) {
            if (this.isChannelLocallyTransacted(channel)) {
                resourceHolder.rollbackAll();
            }

            throw this.convertRabbitAccessException(var13);
        } finally {
            this.cleanUpAfterAction(channel, invokeScope, resourceHolder, connection);
        }

        return var7;
    }
```  
跟进org.springframework.amqp.rabbit.core.RabbitTemplate#invokeAction  
```
    @Nullable
    private <T> T invokeAction(ChannelCallback<T> action, ConnectionFactory connectionFactory, Channel channel) throws Exception {
        if (this.confirmsOrReturnsCapable == null) {
            this.determineConfirmsReturnsCapability(connectionFactory);
        }

        if (this.confirmsOrReturnsCapable) {
            this.addListener(channel);
        }

        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Executing callback " + action.getClass().getSimpleName() + " on RabbitMQ Channel: " + channel);
        }
        //具体执行方法，这里会执行刚刚实现的doSend方法发送信息，具体的发送逻辑这里不去深入了解，最终会调用com.rabbitmq.client.Channel#basicPublish(java.lang.String, java.lang.String, boolean, com.rabbitmq.client.AMQP.BasicProperties, byte[])发送消息
        return action.doInRabbit(channel);
    }
```  
 ### 消息接收流程  
 测试项目中是通过 @RabbitListener来接收处理rabbit队列中的消息  
 ```
  @RabbitListener(queues = {RabbitConfig.ADQUEUE_SAVEEXPOSUREINFO})
    public void saveexposureinfo(String json) {
        log.error("接受失败");
        throw new CustomException("失败");
    }
 ``` 
 显然这里应该有个RabbitListener注解的处理器将所有有该注解的方法通过反射获取，然后当消息接收到时再执行对应的方法。  
 这里这个处理器就是org.springframework.amqp.rabbit.annotation.RabbitListenerAnnotationBeanPostProcessor   
 它实现了BeanPostProcessor和SmartInitializingSingleton接口，当所有bean初始化完成后，会先执行BeanPostProcessor.postProcessAfterInitialization()这个方法，再执行SmartInitializingSingleton.afterSingletonsInstantiated()方法。     
 ```  
 public class RabbitListenerAnnotationBeanPostProcessor implements BeanPostProcessor, Ordered, BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, SmartInitializingSingleton {
   .....
}
 ```
 我们先看看postProcessBeforeInitialization方法   
 ```
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        Class<?> targetClass = AopUtils.getTargetClass(bean);
        //通过该方法找出所有注解RabbitListener的方法，具体方法是buildMetadata()
        RabbitListenerAnnotationBeanPostProcessor.TypeMetadata metadata = (RabbitListenerAnnotationBeanPostProcessor.TypeMetadata)this.typeCache.computeIfAbsent(targetClass, this::buildMetadata);
        RabbitListenerAnnotationBeanPostProcessor.ListenerMethod[] var5 = metadata.listenerMethods;
        int var6 = var5.length;

        for(int var7 = 0; var7 < var6; ++var7) {
            RabbitListenerAnnotationBeanPostProcessor.ListenerMethod lm = var5[var7];
            RabbitListener[] var9 = lm.annotations;
            int var10 = var9.length;

            for(int var11 = 0; var11 < var10; ++var11) {
                RabbitListener rabbitListener = var9[var11];
                //该方法处理方法上的rabbitListener,我们稍后看看里面的执行逻辑
                this.processAmqpListener(rabbitListener, lm.method, bean, beanName);
            }
        }

        if (metadata.handlerMethods.length > 0) {
            //该方法处理方法上的rabbitHandler
            this.processMultiMethodListeners(metadata.classAnnotations, metadata.handlerMethods, bean, beanName);
        }

        return bean;
    }
 ``` 
跟进org.springframework.amqp.rabbit.annotation.RabbitListenerAnnotationBeanPostProcessor#buildMetadata查看具体获取注解  
```
    private RabbitListenerAnnotationBeanPostProcessor.TypeMetadata buildMetadata(Class<?> targetClass) {
        //获取类上RabbitListener注解
        Collection<RabbitListener> classLevelListeners = this.findListenerAnnotations(targetClass);
        boolean hasClassLevelListeners = classLevelListeners.size() > 0;
        List<RabbitListenerAnnotationBeanPostProcessor.ListenerMethod> methods = new ArrayList();
        List<Method> multiMethods = new ArrayList();
        ReflectionUtils.doWithMethods(targetClass, (method) -> {
            //获取方法上RabbitListener注解
            Collection<RabbitListener> listenerAnnotations = this.findListenerAnnotations(method);
            //将方法上注解RabbitListener的方法放入methods中
            if (listenerAnnotations.size() > 0) {
                methods.add(new RabbitListenerAnnotationBeanPostProcessor.ListenerMethod(method, (RabbitListener[])listenerAnnotations.toArray(new RabbitListener[listenerAnnotations.size()])));
            }
            //如果RabbitListener是注解在类上，再获取方法上的RabbitHandler注解
            //将方法上注解RabbitHandler的方法放入multiMethods
            if (hasClassLevelListeners) {
                RabbitHandler rabbitHandler = (RabbitHandler)AnnotationUtils.findAnnotation(method, RabbitHandler.class);
                if (rabbitHandler != null) {
                    multiMethods.add(method);
                }
            }

        }, ReflectionUtils.USER_DECLARED_METHODS);
        //构建TypeMetadata，方法上注解RabbitListener的放入listenerMethods属性，方法上注解RabbitListener的放入handlerMethods属性
        //类上注解RabbitListener放入classAnnotations  
        return methods.isEmpty() && multiMethods.isEmpty() ? RabbitListenerAnnotationBeanPostProcessor.TypeMetadata.EMPTY : new RabbitListenerAnnotationBeanPostProcessor.TypeMetadata((RabbitListenerAnnotationBeanPostProcessor.ListenerMethod[])methods.toArray(new RabbitListenerAnnotationBeanPostProcessor.ListenerMethod[methods.size()]), (Method[])multiMethods.toArray(new Method[multiMethods.size()]), (RabbitListener[])classLevelListeners.toArray(new RabbitListener[classLevelListeners.size()]));
}
```  
我们现在看看方法上有RabbitListener注解的处理方法：org.springframework.amqp.rabbit.annotation.RabbitListenerAnnotationBeanPostProcessor#processAmqpListener  
```
    /**
     *新建了一个MethodRabbitListenerEndpoint，将我们测试项目写的消费方法放入MethodRabbitListenerEndpoint的method属性，然后调用了processListener方法
     */
    protected void processAmqpListener(RabbitListener rabbitListener, Method method, Object bean, String beanName) {
        Method methodToUse = this.checkProxy(method, bean);
        MethodRabbitListenerEndpoint endpoint = new MethodRabbitListenerEndpoint();
        endpoint.setMethod(methodToUse);
        this.processListener(endpoint, rabbitListener, bean, methodToUse, beanName);
    }
```  
跟进org.springframework.amqp.rabbit.annotation.RabbitListenerAnnotationBeanPostProcessor#processListener方法  
```
    /**
     *将注解相关配置放入MethodRabbitListenerEndpoint，再调用RabbitListenerEndpointRegistrar.registerEndpoint方法
     */
    protected void processListener(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object bean, Object target, String beanName) {
        endpoint.setBean(bean);
        endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);
        endpoint.setId(this.getEndpointId(rabbitListener));
        endpoint.setQueueNames(this.resolveQueues(rabbitListener));
        endpoint.setConcurrency(this.resolveExpressionAsStringOrInteger(rabbitListener.concurrency(), "concurrency"));
        endpoint.setBeanFactory(this.beanFactory);
        endpoint.setReturnExceptions(this.resolveExpressionAsBoolean(rabbitListener.returnExceptions()));
        Object errorHandler = this.resolveExpression(rabbitListener.errorHandler());
        String errorHandlerBeanName;
        if (errorHandler instanceof RabbitListenerErrorHandler) {
            endpoint.setErrorHandler((RabbitListenerErrorHandler)errorHandler);
        } else {
            if (!(errorHandler instanceof String)) {
                throw new IllegalStateException("error handler mut be a bean name or RabbitListenerErrorHandler, not a " + errorHandler.getClass().toString());
            }

            errorHandlerBeanName = (String)errorHandler;
            if (StringUtils.hasText(errorHandlerBeanName)) {
                endpoint.setErrorHandler((RabbitListenerErrorHandler)this.beanFactory.getBean(errorHandlerBeanName, RabbitListenerErrorHandler.class));
            }
        }

        errorHandlerBeanName = rabbitListener.group();
        if (StringUtils.hasText(errorHandlerBeanName)) {
            Object resolvedGroup = this.resolveExpression(errorHandlerBeanName);
            if (resolvedGroup instanceof String) {
                endpoint.setGroup((String)resolvedGroup);
            }
        }

        String autoStartup = rabbitListener.autoStartup();
        if (StringUtils.hasText(autoStartup)) {
            endpoint.setAutoStartup(this.resolveExpressionAsBoolean(autoStartup));
        }

        endpoint.setExclusive(rabbitListener.exclusive());
        String priority = this.resolveExpressionAsString(rabbitListener.priority(), "priority");
        if (StringUtils.hasText(priority)) {
            try {
                endpoint.setPriority(Integer.valueOf(priority));
            } catch (NumberFormatException var11) {
                throw new BeanInitializationException("Invalid priority value for " + rabbitListener + " (must be an integer)", var11);
            }
        }

        this.resolveExecutor(endpoint, rabbitListener, target, beanName);
        this.resolveAdmin(endpoint, rabbitListener, target);
        this.resolveAckMode(endpoint, rabbitListener);
        this.resolvePostProcessor(endpoint, rabbitListener, target, beanName);
        RabbitListenerContainerFactory<?> factory = this.resolveContainerFactory(rabbitListener, target, beanName);
        this.registrar.registerEndpoint(endpoint, factory);
    }
```  
跟进org.springframework.amqp.rabbit.listener.RabbitListenerEndpointRegistrar#registerEndpoint(org.springframework.amqp.rabbit.listener.RabbitListenerEndpoint, org.springframework.amqp.rabbit.listener.RabbitListenerContainerFactory<?>)方法
```
    public void registerEndpoint(RabbitListenerEndpoint endpoint, @Nullable RabbitListenerContainerFactory<?> factory) {
        Assert.notNull(endpoint, "Endpoint must be set");
        Assert.hasText(endpoint.getId(), "Endpoint id must be set");
        Assert.state(!this.startImmediately || this.endpointRegistry != null, "No registry available");
        RabbitListenerEndpointRegistrar.AmqpListenerEndpointDescriptor descriptor = new RabbitListenerEndpointRegistrar.AmqpListenerEndpointDescriptor(endpoint, factory);
        synchronized(this.endpointDescriptors) {
            //关键代码，将MethodRabbitListenerEndpoint注册到ListenerContainer。
            //根据startImmediately参数判断是否一个个注册，还是先添加到endpointDescriptors，最后一起注册
            //我们只需要看看registerListenerContainer方法即可，一起注册应该逻辑差不多
            if (this.startImmediately) {
                this.endpointRegistry.registerListenerContainer(descriptor.endpoint, this.resolveContainerFactory(descriptor), true);
            } else {
                this.endpointDescriptors.add(descriptor);
            }

        }
    }
```
根据org.springframework.amqp.rabbit.listener.RabbitListenerEndpointRegistry#registerListenerContainer(org.springframework.amqp.rabbit.listener.RabbitListenerEndpoint, org.springframework.amqp.rabbit.listener.RabbitListenerContainerFactory<?>, boolean)  
```
    public void registerListenerContainer(RabbitListenerEndpoint endpoint, RabbitListenerContainerFactory<?> factory, boolean startImmediately) {
        Assert.notNull(endpoint, "Endpoint must not be null");
        Assert.notNull(factory, "Factory must not be null");
        String id = endpoint.getId();
        Assert.hasText(id, "Endpoint id must not be empty");
        synchronized(this.listenerContainers) {
            Assert.state(!this.listenerContainers.containsKey(id), "Another endpoint is already registered with id '" + id + "'");
            //每一个MethodRabbitListenerEndpoint都创建一个MessageListenerContainer，这里我们从前面自动配置知道这里的Comtainer是SimpleMessageListenerContainer
            MessageListenerContainer container = this.createListenerContainer(endpoint, factory);
            this.listenerContainers.put(id, container);
            if (StringUtils.hasText(endpoint.getGroup()) && this.applicationContext != null) {
                Object containerGroup;
                if (this.applicationContext.containsBean(endpoint.getGroup())) {
                    containerGroup = (List)this.applicationContext.getBean(endpoint.getGroup(), List.class);
                } else {
                    containerGroup = new ArrayList();
                    this.applicationContext.getBeanFactory().registerSingleton(endpoint.getGroup(), containerGroup);
                }

                ((List)containerGroup).add(container);
            }

            if (this.contextRefreshed) {
                container.lazyLoad();
            }
            //同样这边是启动Comtainer是SimpleMessageListenerContainer
            if (startImmediately) {
                this.startIfNecessary(container);
            }

        }
    }
```
跟进org.springframework.amqp.rabbit.listener.RabbitListenerEndpointRegistry#startIfNecessary  
```
    private void startIfNecessary(MessageListenerContainer listenerContainer) {
        if (this.contextRefreshed || listenerContainer.isAutoStartup()) {
            //调了MessageListenerContainer的start方法
            listenerContainer.start();
        }

    }
```
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#start  
```
    public void start() {
        if (!this.isRunning()) {
            if (!this.initialized) {
                synchronized(this.lifecycleMonitor) {
                    if (!this.initialized) {
                        this.afterPropertiesSet();
                    }
                }
            }

            try {
                this.logger.debug("Starting Rabbit listener container.");
                this.configureAdminIfNeeded();
                this.checkMismatchedQueues();
                //这边调用的是Comtainer是SimpleMessageListenerContainer的doStart方法
                this.doStart();
            } catch (Exception var7) {
                throw this.convertRabbitAccessException(var7);
            } finally {
                this.lazyLoad = false;
            }

        }
    }
```  
跟进org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer#doStart  
```
    protected void doStart() {
        Assert.state(!this.consumerBatchEnabled || this.getMessageListener() instanceof BatchMessageListener || this.getMessageListener() instanceof ChannelAwareBatchMessageListener, "When setting 'consumerBatchEnabled' to true, the listener must support batching");
        this.checkListenerContainerAware();
        super.doStart();
        synchronized(this.consumersMonitor) {
            if (this.consumers != null) {
                throw new IllegalStateException("A stopped container should not have consumers");
            } else {
                //这边创建BlockingQueueConsumer进行消费，默认BlockingQueueConsumer只创建一个
                int newConsumers = this.initializeConsumers();
                if (this.consumers == null) {
                    this.logger.info("Consumers were initialized and then cleared (presumably the container was stopped concurrently)");
                } else if (newConsumers <= 0) {
                    if (this.logger.isInfoEnabled()) {
                        this.logger.info("Consumers are already running");
                    }

                } else {
                    Set<SimpleMessageListenerContainer.AsyncMessageProcessingConsumer> processors = new HashSet();
                    Iterator var4 = this.consumers.iterator();
                    //每个BlockingQueueConsumer创建一个线程
                    while(var4.hasNext()) {
                        BlockingQueueConsumer consumer = (BlockingQueueConsumer)var4.next();
                        //AsyncMessageProcessingConsumer显然是实现了Runnable接口的类，我们看看它的run方法就可以知道监听器的运作
                        SimpleMessageListenerContainer.AsyncMessageProcessingConsumer processor = new SimpleMessageListenerContainer.AsyncMessageProcessingConsumer(consumer);
                        processors.add(processor);
                        this.getTaskExecutor().execute(processor);
                        if (this.getApplicationEventPublisher() != null) {
                            this.getApplicationEventPublisher().publishEvent(new AsyncConsumerStartedEvent(this, consumer));
                        }
                    }

                    this.waitForConsumersToStart(processors);
                }
            }
        }
    }
```  
跟进org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.AsyncMessageProcessingConsumer#run  
```
        public void run() {
            if (SimpleMessageListenerContainer.this.isActive()) {
                boolean aborted = false;
                this.consumer.setLocallyTransacted(SimpleMessageListenerContainer.this.isChannelLocallyTransacted());
                String routingLookupKey = SimpleMessageListenerContainer.this.getRoutingLookupKey();
                if (routingLookupKey != null) {
                    SimpleResourceHolder.bind(SimpleMessageListenerContainer.this.getRoutingConnectionFactory(), routingLookupKey);
                }

                if (this.consumer.getQueueCount() < 1) {
                    if (SimpleMessageListenerContainer.this.logger.isDebugEnabled()) {
                        SimpleMessageListenerContainer.this.logger.debug("Consumer stopping; no queues for " + this.consumer);
                    }

                    SimpleMessageListenerContainer.this.cancellationLock.release(this.consumer);
                    if (SimpleMessageListenerContainer.this.getApplicationEventPublisher() != null) {
                        SimpleMessageListenerContainer.this.getApplicationEventPublisher().publishEvent(new AsyncConsumerStoppedEvent(SimpleMessageListenerContainer.this, this.consumer));
                    }

                    this.start.countDown();
                } else {
                    try {
                        //这边是一些初始化操作，获取channel
                        this.initialize();
                        //this.consumer.hasDelivery()方法只要queues不为空就是true  
                        while(SimpleMessageListenerContainer.this.isActive(this.consumer) || this.consumer.hasDelivery() || !this.consumer.cancelled()) {
                            //重点是这个,由于外层的while正常情况下一直是true，所以这个方法会一直执行
                            this.mainLoop();
                        }
                    } catch (InterruptedException var15) {
                        SimpleMessageListenerContainer.this.logger.debug("Consumer thread interrupted, processing stopped.");
                        Thread.currentThread().interrupt();
                        aborted = true;
                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer thread interrupted, processing stopped", true, var15);
                    } catch (QueuesNotAvailableException var16) {
                        SimpleMessageListenerContainer.this.logger.error("Consumer threw missing queues exception, fatal=" + SimpleMessageListenerContainer.this.isMissingQueuesFatal(), var16);
                        if (SimpleMessageListenerContainer.this.isMissingQueuesFatal()) {
                            this.startupException = var16;
                            aborted = true;
                        }

                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer queue(s) not available", aborted, var16);
                    } catch (FatalListenerStartupException var17) {
                        SimpleMessageListenerContainer.this.logger.error("Consumer received fatal exception on startup", var17);
                        this.startupException = var17;
                        aborted = true;
                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer received fatal exception on startup", true, var17);
                    } catch (FatalListenerExecutionException var18) {
                        SimpleMessageListenerContainer.this.logger.error("Consumer received fatal exception during processing", var18);
                        aborted = true;
                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer received fatal exception during processing", true, var18);
                    } catch (PossibleAuthenticationFailureException var19) {
                        SimpleMessageListenerContainer.this.logger.error("Consumer received fatal=" + SimpleMessageListenerContainer.this.isPossibleAuthenticationFailureFatal() + " exception during processing", var19);
                        if (SimpleMessageListenerContainer.this.isPossibleAuthenticationFailureFatal()) {
                            this.startupException = new FatalListenerStartupException("Authentication failure", new AmqpAuthenticationException(var19));
                            aborted = true;
                        }

                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer received PossibleAuthenticationFailure during startup", aborted, var19);
                    } catch (ShutdownSignalException var20) {
                        if (RabbitUtils.isNormalShutdown(var20)) {
                            if (SimpleMessageListenerContainer.this.logger.isDebugEnabled()) {
                                SimpleMessageListenerContainer.this.logger.debug("Consumer received Shutdown Signal, processing stopped: " + var20.getMessage());
                            }
                        } else {
                            this.logConsumerException(var20);
                        }
                    } catch (AmqpIOException var21) {
                        if (var21.getCause() instanceof IOException && var21.getCause().getCause() instanceof ShutdownSignalException && var21.getCause().getCause().getMessage().contains("in exclusive use")) {
                            SimpleMessageListenerContainer.this.getExclusiveConsumerExceptionLogger().log(SimpleMessageListenerContainer.this.logger, "Exclusive consumer failure", var21.getCause().getCause());
                            SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer raised exception, attempting restart", false, var21);
                        } else {
                            this.logConsumerException(var21);
                        }
                    } catch (Error var22) {
                        SimpleMessageListenerContainer.this.logger.error("Consumer thread error, thread abort.", var22);
                        SimpleMessageListenerContainer.this.publishConsumerFailedEvent("Consumer threw an Error", true, var22);
                        SimpleMessageListenerContainer.this.getJavaLangErrorHandler().handle(var22);
                        aborted = true;
                    } catch (Throwable var23) {
                        if (SimpleMessageListenerContainer.this.isActive()) {
                            this.logConsumerException(var23);
                        }
                    } finally {
                        if (SimpleMessageListenerContainer.this.getTransactionManager() != null) {
                            ConsumerChannelRegistry.unRegisterConsumerChannel();
                        }

                    }

                    this.start.countDown();
                    this.killOrRestart(aborted);
                    if (routingLookupKey != null) {
                        SimpleResourceHolder.unbind(SimpleMessageListenerContainer.this.getRoutingConnectionFactory());
                    }

                }
            }
        }
```
跟进org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.AsyncMessageProcessingConsumer#mainLoop  
```
            private void mainLoop() throws Exception {
            try {
                //这边执行接受逻辑
                boolean receivedOk = SimpleMessageListenerContainer.this.receiveAndExecute(this.consumer);
                if (SimpleMessageListenerContainer.this.maxConcurrentConsumers != null) {
                    this.checkAdjust(receivedOk);
                }

                long idleEventInterval = SimpleMessageListenerContainer.this.getIdleEventInterval();
                if (idleEventInterval > 0L) {
                    if (receivedOk) {
                        SimpleMessageListenerContainer.this.updateLastReceive();
                    } else {
                        long now = System.currentTimeMillis();
                        long lastAlertAt = SimpleMessageListenerContainer.this.lastNoMessageAlert.get();
                        long lastReceive = SimpleMessageListenerContainer.this.getLastReceive();
                        if (now > lastReceive + idleEventInterval && now > lastAlertAt + idleEventInterval && SimpleMessageListenerContainer.this.lastNoMessageAlert.compareAndSet(lastAlertAt, now)) {
                            SimpleMessageListenerContainer.this.publishIdleContainerEvent(now - lastReceive);
                        }
                    }
                }
            } catch (ListenerExecutionFailedException var10) {
                if (var10.getCause() instanceof NoSuchMethodException) {
                    throw new FatalListenerExecutionException("Invalid listener", var10);
                }
            } catch (AmqpRejectAndDontRequeueException var11) {
            }

        }
```
跟进org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer#receiveAndExecute  
```
    private boolean receiveAndExecute(BlockingQueueConsumer consumer) throws Exception {
        PlatformTransactionManager transactionManager = this.getTransactionManager();
        if (transactionManager != null) {
            try {
                if (this.transactionTemplate == null) {
                    this.transactionTemplate = new TransactionTemplate(transactionManager, this.getTransactionAttribute());
                }

                return (Boolean)this.transactionTemplate.execute((status) -> {
                    RabbitResourceHolder resourceHolder = ConnectionFactoryUtils.bindResourceToTransaction(new RabbitResourceHolder(consumer.getChannel(), false), this.getConnectionFactory(), true);

                    try {
                        //主要方法
                        return this.doReceiveAndExecute(consumer);
                    } catch (RuntimeException var5) {
                        this.prepareHolderForRollback(resourceHolder, var5);
                        throw var5;
                    } catch (Exception var6) {
                        throw new WrappedTransactionException(var6);
                    }
                });
            } catch (WrappedTransactionException var4) {
                throw (Exception)var4.getCause();
            }
        } else {
            return this.doReceiveAndExecute(consumer);
        }
    }
```  
继续跟进org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer#doReceiveAndExecute  
```
    private boolean doReceiveAndExecute(BlockingQueueConsumer consumer) throws Exception {
        Channel channel = consumer.getChannel();
        List<Message> messages = null;
        long deliveryTag = 0L;

        for(int i = 0; i < this.batchSize; ++i) {
            this.logger.trace("Waiting for message from consumer.");
            Message message = consumer.nextMessage(this.receiveTimeout);
            if (message == null) {
                break;
            }
            //默认不开启批量消费消息，一次只消费单条（支持批量发送和批量接受处理，在应用层将10条压缩成一条发送，再将一条解压成10条）
            if (this.consumerBatchEnabled) {
                Collection<MessagePostProcessor> afterReceivePostProcessors = this.getAfterReceivePostProcessors();
                if (afterReceivePostProcessors != null) {
                    Message original = message;
                    deliveryTag = message.getMessageProperties().getDeliveryTag();
                    Iterator var10 = this.getAfterReceivePostProcessors().iterator();

                    while(var10.hasNext()) {
                        MessagePostProcessor processor = (MessagePostProcessor)var10.next();
                        message = processor.postProcessMessage(message);
                        if (message == null) {
                            channel.basicAck(deliveryTag, false);
                            if (this.logger.isDebugEnabled()) {
                                this.logger.debug("Message Post Processor returned 'null', discarding message " + original);
                            }
                            break;
                        }
                    }
                }

                if (message != null) {
                    if (messages == null) {
                        messages = new ArrayList(this.batchSize);
                    }

                    if (this.isDeBatchingEnabled() && this.getBatchingStrategy().canDebatch(message.getMessageProperties())) {
                        this.getBatchingStrategy().deBatch(message, (fragment) -> {
                            messages.add(fragment);
                        });
                    } else {
                        ((List)messages).add(message);
                    }
                }
            } else {
                messages = this.debatch(message);
                if (messages != null) {
                    break;
                }

                try {
                    //这里是将要执行的方法
                    this.executeListener(channel, message);
                } catch (ImmediateAcknowledgeAmqpException var12) {
                    if (this.logger.isDebugEnabled()) {
                        this.logger.debug("User requested ack for failed delivery '" + var12.getMessage() + "': " + message.getMessageProperties().getDeliveryTag());
                    }
                    break;
                } catch (Exception var13) {
                    if (this.causeChainHasImmediateAcknowledgeAmqpException(var13)) {
                        if (this.logger.isDebugEnabled()) {
                            this.logger.debug("User requested ack for failed delivery: " + message.getMessageProperties().getDeliveryTag());
                        }
                    } else {
                        if (this.getTransactionManager() == null) {
                            consumer.rollbackOnExceptionIfNecessary(var13);
                            throw var13;
                        }

                        if (this.getTransactionAttribute().rollbackOn(var13)) {
                            RabbitResourceHolder resourceHolder = (RabbitResourceHolder)TransactionSynchronizationManager.getResource(this.getConnectionFactory());
                            if (resourceHolder != null) {
                                consumer.clearDeliveryTags();
                            } else {
                                consumer.rollbackOnExceptionIfNecessary(var13);
                            }

                            throw var13;
                        }

                        if (this.logger.isDebugEnabled()) {
                            this.logger.debug("No rollback for " + var13);
                        }
                    }
                    break;
                }
            }
        }

        if (messages != null) {
            this.executeWithList(channel, (List)messages, deliveryTag, consumer);
        }

        return consumer.commitIfNecessary(this.isChannelLocallyTransacted());
    }
```
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#executeListener  
```
    protected void executeListener(Channel channel, Object data) {
        if (!this.isRunning()) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Rejecting received message(s) because the listener container has been stopped: " + data);
            }

            throw new MessageRejectedWhileStoppingException();
        } else {
            Object sample = null;
            if (this.micrometerHolder != null) {
                sample = this.micrometerHolder.start();
            }

            try {
                //继续进入该方法
                this.doExecuteListener(channel, data);
                if (sample != null) {
                    this.micrometerHolder.success(sample, data instanceof Message ? ((Message)data).getMessageProperties().getConsumerQueue() : this.queuesAsListString());
                }

            } catch (RuntimeException var6) {
                if (sample != null) {
                    this.micrometerHolder.failure(sample, data instanceof Message ? ((Message)data).getMessageProperties().getConsumerQueue() : this.queuesAsListString(), var6.getClass().getSimpleName());
                }

                Message message;
                if (data instanceof Message) {
                    message = (Message)data;
                } else {
                    message = (Message)((List)data).get(0);
                }

                this.checkStatefulRetry(var6, message);
                this.handleListenerException(var6);
                throw var6;
            }
        }
    }
```  
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#doExecuteListener
```
    private void doExecuteListener(Channel channel, Object data) {
        if (data instanceof Message) {
            Message message = (Message)data;
            if (this.afterReceivePostProcessors != null) {
                Iterator var4 = this.afterReceivePostProcessors.iterator();

                while(var4.hasNext()) {
                    MessagePostProcessor processor = (MessagePostProcessor)var4.next();
                    message = processor.postProcessMessage(message);
                    if (message == null) {
                        throw new ImmediateAcknowledgeAmqpException("Message Post Processor returned 'null', discarding message");
                    }
                }
            }

            if (this.deBatchingEnabled && this.batchingStrategy.canDebatch(message.getMessageProperties())) {
                this.batchingStrategy.deBatch(message, (fragment) -> {
                    this.invokeListener(channel, fragment);
                });
            } else {
                //代码会执行到这个方法
                this.invokeListener(channel, message);
            }
        } else {
            this.invokeListener(channel, data);
        }

    }
```  
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#invokeListener  
```
    protected void invokeListener(Channel channel, Object data) {
        this.proxy.invokeListener(channel, data);
    }
```  
这边调用了代理类，代理类是一个接口  
```
    @FunctionalInterface
    private interface ContainerDelegate {
        void invokeListener(Channel var1, Object var2);
    }
```  
那么接口是什么时候实现的呢，我们可以看到有个org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#initialize方法，这个方法看名字就知道是在bean初始化后调用的  
```
    public void initialize() {
        try {
            synchronized(this.lifecycleMonitor) {
                this.lifecycleMonitor.notifyAll();
            }
            //在这边实例化代理的，我们可以看看this.delegate
            //private final AbstractMessageListenerContainer.ContainerDelegate delegate = this::actualInvokeListener;
            //在这边就已经用actualInvokeListener实现了invokeListener方法
            this.initializeProxy(this.delegate);
            this.checkMissingQueuesFatalFromProperty();
            this.checkPossibleAuthenticationFailureFatalFromProperty();
            this.doInitialize();
            if (!this.isExposeListenerChannel() && this.transactionManager != null) {
                this.logger.warn("exposeListenerChannel=false is ignored when using a TransactionManager");
            }

            if (!this.taskExecutorSet && StringUtils.hasText(this.getListenerId())) {
                this.taskExecutor = new SimpleAsyncTaskExecutor(this.getListenerId() + "-");
                this.taskExecutorSet = true;
            }

            if (this.transactionManager != null && !this.isChannelTransacted()) {
                this.logger.debug("The 'channelTransacted' is coerced to 'true', when 'transactionManager' is provided");
                this.setChannelTransacted(true);
            }

            if (this.messageListener != null) {
                this.messageListener.containerAckMode(this.acknowledgeMode);
            }

            this.initialized = true;
        } catch (Exception var4) {
            throw this.convertRabbitAccessException(var4);
        }
    }
```  
我们先看看初始化代理做了些什么操作  
org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#initializeProxy  
```
    protected void initializeProxy(Object delegate) {
        还记得最开始自动配置时候时的方法拦截器RetryOperationsInterceptor吗，这个时候就会将RetryOperationsInterceptor作为ContainerDelegate的代理，代理她的vokeListener方法。
        if (this.getAdviceChain().length != 0) {
            ProxyFactory factory = new ProxyFactory();
            Advice[] var3 = this.getAdviceChain();
            int var4 = var3.length;

            for(int var5 = 0; var5 < var4; ++var5) {
                Advice advice = var3[var5];
                factory.addAdvisor(new DefaultPointcutAdvisor(advice));
            }

            factory.addInterface(AbstractMessageListenerContainer.ContainerDelegate.class);
            factory.setTarget(delegate);
            this.proxy = (AbstractMessageListenerContainer.ContainerDelegate)factory.getProxy(AbstractMessageListenerContainer.ContainerDelegate.class.getClassLoader());
        }
    }
```
我们先前知道实现proxy.invokeListener的代码在actualInvokeListener方法。最后看看RetryOperationsInterceptor的invoke代理执行了些什么逻辑。  
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#actualInvokeListener  
```
    protected void actualInvokeListener(Channel channel, Object data) {
        Object listener = this.getMessageListener();
        if (listener instanceof ChannelAwareMessageListener) {
            this.doInvokeListener((ChannelAwareMessageListener)listener, channel, data);
        } else {
            if (!(listener instanceof MessageListener)) {
                if (listener != null) {
                    throw new FatalListenerExecutionException("Only MessageListener and SessionAwareMessageListener supported: " + listener);
                }

                throw new FatalListenerExecutionException("No message listener specified - see property 'messageListener'");
            }

            boolean bindChannel = this.isExposeListenerChannel() && this.isChannelLocallyTransacted();
            if (bindChannel) {
                RabbitResourceHolder resourceHolder = new RabbitResourceHolder(channel, false);
                resourceHolder.setSynchronizedWithTransaction(true);
                TransactionSynchronizationManager.bindResource(this.getConnectionFactory(), resourceHolder);
            }

            try {
                //还需要继续跟进
                this.doInvokeListener((MessageListener)listener, data);
            } finally {
                if (bindChannel) {
                    TransactionSynchronizationManager.unbindResource(this.getConnectionFactory());
                }

            }
        }

    }
```   
跟进org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#doInvokeListener(org.springframework.amqp.core.MessageListener, java.lang.Object)  
```
    protected void doInvokeListener(MessageListener listener, Object data) {
        Message message = null;

        try {
            if (data instanceof List) {
                listener.onMessageBatch((List)data);
            } else {
                message = (Message)data;
                //调用到这里就会进入我们刚才写的业务逻辑里，即执行@RabbitListerner注解的方法，感兴趣的可以继续深入，我们现在到这里先结束
                listener.onMessage(message);
            }

        } catch (Exception var5) {
            throw this.wrapToListenerExecutionFailedExceptionIfNeeded(var5, data);
        }
    }
```  
###
刚才我们知道在执行proxy.invokeListener的时候，因为我们配置了重试机制，实际执行的方法是org.springframework.retry.interceptor.RetryOperationsInterceptor#invoke  
```
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        final String name;
        if (StringUtils.hasText(this.label)) {
            name = this.label;
        } else {
            name = invocation.getMethod().toGenericString();
        }
        //这边创建了一个实现RetryCallback的匿名内部类，实现了doWithRetry方法
        //该方法的目的就是执行被代理类的方法，这里的方法是就是我们刚刚提到的方法，就是：
        //org.springframework.amqp.rabbit.listener.AbstractMessageListenerContainer#actualInvokeListener方法
        RetryCallback<Object, Throwable> retryCallback = new RetryCallback<Object, Throwable>() {
            public Object doWithRetry(RetryContext context) throws Exception {
                context.setAttribute("context.name", name);
                if (invocation instanceof ProxyMethodInvocation) {
                    try {
                        return ((ProxyMethodInvocation)invocation).invocableClone().proceed();
                    } catch (Exception var3) {
                        throw var3;
                    } catch (Error var4) {
                        throw var4;
                    } catch (Throwable var5) {
                        throw new IllegalStateException(var5);
                    }
                } else {
                    throw new IllegalStateException("MethodInvocation of the wrong type detected - this should not happen with Spring AOP, so please raise an issue if you see this exception");
                }
            }
        };
        if (this.recoverer != null) {
            RetryOperationsInterceptor.ItemRecovererCallback recoveryCallback = new RetryOperationsInterceptor.ItemRecovererCallback(invocation.getArguments(), this.recoverer);
            //这里recoveryCallback是org.springframework.amqp.rabbit.retry.RejectAndDontRequeueRecoverer
            //我们需要深入execute方法，最终发现实际执行逻辑在org.springframework.retry.support.RetryTemplate#doExecute方法中
            return this.retryOperations.execute(retryCallback, recoveryCallback);
        } else {
            return this.retryOperations.execute(retryCallback);
        }
    }
```
跟进org.springframework.retry.support.RetryTemplate#doExecute方法  
```
    protected <T, E extends Throwable> T doExecute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback, RetryState state) throws E, ExhaustedRetryException {
        RetryPolicy retryPolicy = this.retryPolicy;
        BackOffPolicy backOffPolicy = this.backOffPolicy;
        RetryContext context = this.open(retryPolicy, state);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("RetryContext retrieved: " + context);
        }

        RetrySynchronizationManager.register(context);
        Throwable lastException = null;
        boolean exhausted = false;

        try {
            第一次重试调用实现RetryListener类的open方法
            boolean running = this.doOpenInterceptors(retryCallback, context);
            if (!running) {
                throw new TerminatedRetryException("Retry terminated abnormally by interceptor before first attempt");
            } else {
                BackOffContext backOffContext = null;
                Object resource = context.getAttribute("backOffContext");
                if (resource instanceof BackOffContext) {
                    backOffContext = (BackOffContext)resource;
                }

                if (backOffContext == null) {
                    backOffContext = backOffPolicy.start(context);
                    if (backOffContext != null) {
                        context.setAttribute("backOffContext", backOffContext);
                    }
                }

                while(true) {
                    Object var34;
                    //这边判断是否应该执行，如果上次执行没有失败或者执行次数已经耗尽则停止执行
                    if (this.canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
                        try {
                            if (this.logger.isDebugEnabled()) {
                                this.logger.debug("Retry: count=" + context.getRetryCount());
                            }

                            lastException = null;  
                            //这里执行被代理类的方法，如果没有异常直接返回，不再重试
                            var34 = retryCallback.doWithRetry(context);
                            return var34;
                        } catch (Throwable var31) {
                            Throwable e = var31;
                            lastException = var31;

                            try {
                                //记录这次的异常信息
                                this.registerThrowable(retryPolicy, state, context, e);
                            } catch (Exception var28) {
                                throw new TerminatedRetryException("Could not register throwable", var28);
                            } finally {
                                //失败执行实现RetryListener类的onError方法
                                this.doOnErrorInterceptors(retryCallback, context, var31);
                            }

                            if (this.canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
                                try {
                                    //根据重试规则线程休眠或者其他操作
                                    backOffPolicy.backOff(backOffContext);
                                } catch (BackOffInterruptedException var30) {
                                    lastException = var31;
                                    if (this.logger.isDebugEnabled()) {
                                        this.logger.debug("Abort retry because interrupted: count=" + context.getRetryCount());
                                    }

                                    throw var30;
                                }
                            }

                            if (this.logger.isDebugEnabled()) {
                                this.logger.debug("Checking for rethrow: count=" + context.getRetryCount());
                            }

                            if (this.shouldRethrow(retryPolicy, context, state)) {
                                if (this.logger.isDebugEnabled()) {
                                    this.logger.debug("Rethrow in retry for policy: count=" + context.getRetryCount());
                                }

                                throw wrapIfNecessary(var31);
                            }

                            if (state == null || !context.hasAttribute("state.global")) {
                                continue;
                            }
                        }
                    }

                    if (state == null && this.logger.isDebugEnabled()) {
                        this.logger.debug("Retry failed last attempt: count=" + context.getRetryCount());
                    }

                    exhausted = true;
                    //执行次数耗尽，并且都失败执行recoveryCallback的recover方法
                    var34 = this.handleRetryExhausted(recoveryCallback, context, state);
                    return var34;
                }
            }
        } catch (Throwable var32) {
            throw wrapIfNecessary(var32);
        } finally {
            this.close(retryPolicy, context, state, lastException == null || exhausted);
            this.doCloseInterceptors(retryCallback, context, lastException);
            RetrySynchronizationManager.clear();
        }
    }
```
## 总结  
我们debug代码发现，虽然spring-retry是通过线程休眠的方式不断重新执行消费失败的方法，但是因为消费端默认是一条线程在不断接受消息，如果我们失败重试设置时间过长，会导致消费线程长时间无法执行获取消息的操作，也会导致队列的消息堆积，所以需要手动配置消费线程的数量