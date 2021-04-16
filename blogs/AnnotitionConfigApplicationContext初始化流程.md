# AnnotitionConfigApplicationContext

## 一·类层次结构概览

```
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry
```

## 二·构造方法

```
/**
	 * 这个构造方法需要传入一个被javaconfig注解了的配置类
	 * 然后会把这个被注解了javaconfig的类通过注解读取器读取后继而解析
	 * Create a new AnnotationConfigApplicationContext, deriving bean definitions
	 * from the given annotated classes and automatically refreshing the context.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 */
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		//annotatedClasses  appconfig.class
		//这里由于他有父类，故而会先调用父类的构造方法，然后才会调用自己的构造方法
		//在自己构造方法中初始一个读取器和扫描器
		this();
		register(componentClasses);
		refresh();
	}
```

### 1.父类GenericApplicationContext的主要职责
- 实例化一个bean工厂DefaultListableBeanFactory

```
/**
	 * Create a new GenericApplicationContext.
	 * @see #registerBeanDefinition
	 * @see #refresh
	 * 一个工厂
	 */
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```
### 2.AnnotationConfigApplicationContext.this()

```
/**
	 * 初始化一个bean的读取和扫描器
	 * 何谓读取器和扫描器参考上面的属性注释
	 * 默认构造函数，如果直接调用这个默认构造方法，需要在稍后通过调用其register()
	 * 去注册配置类（javaconfig），并调用refresh()方法刷新容器，
	 * 触发容器对注解Bean的载入、解析和注册过程
	 * 这种使用过程我在ioc应用的第二节课讲@profile的时候讲过
	 * Create a new AnnotationConfigApplicationContext that needs to be populated
	 * through {@link #register} calls and then manually {@linkplain #refresh refreshed}.
	 */
	public AnnotationConfigApplicationContext() {
		/**
		 * 父类的构造方法
		 * 创建一个读取注解的Bean定义读取器
		 * 什么是bean定义？BeanDefinition
		 */
		this.reader = new AnnotatedBeanDefinitionReader(this);


		//可以用来扫描包或者类，继而转换成bd
		//但是实际上我们扫描包工作不是scanner这个对象来完成的
		//是spring自己new的一个ClassPathBeanDefinitionScanner
		//这里的scanner仅仅是为了程序员能够在外部调用AnnotationConfigApplicationContext对象的scan方法
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```
#### 2.1 AnnotatedBeanDefinitionReader

```
/**
	 *  这里的BeanDefinitionRegistry registry是通过在AnnotationConfigApplicationContext
	 *  的构造方法中传进来的this
	 *  由此说明AnnotationConfigApplicationContext是一个BeanDefinitionRegistry类型的类
	 *  何以证明我们可以看到AnnotationConfigApplicationContext的类关系：
	 *  GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry
	 *  看到他实现了BeanDefinitionRegistry证明上面的说法，那么BeanDefinitionRegistry的作用是什么呢？
	 *  BeanDefinitionRegistry 顾名思义就是BeanDefinition的注册器
	 *  那么何为BeanDefinition呢？参考BeanDefinition的源码的注释
	 * @param registry
	 */
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}
	
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
//		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
//		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

##### 主要职责
- 添加AnnotationAwareOrderComparator类的对象，主要去排序，能解析@Order注解和@Priority
- 添加ContextAnnotationAutowireCandidateResolver，提供处理延迟加载的功能
- BeanDefinitionMap注册多个后续需要使用到的BeanPostProcessor以及BeanFactoryPostProcessor等

```
//扫描并注册beanDefinition、解析注解、imprt注解的解析等。需要注意的是ConfigurationClassPostProcessor的类型是BeanDefinitionRegistryPostProcessor而 BeanDefinitionRegistryPostProcessor 最终实现BeanFactoryPostProcessor这个接口
ConfigurationClassPostProcessor

//自动装配等
AutowiredAnnotationBeanPostProcessor

RequiredAnnotationBeanPostProcessor

//主要处理@Resource、@PostConstruct和@PreDestroy注解的实现,Resource的处理是由他自己完成,其他两个是由他的父类完成,父类InitDestroyAnnotationBeanPostProcessor的postProcessMergedBeanDefinition,会找出被@PostConstruct和@PreDestroy注解修饰的方法
CommonAnnotationBeanPostProcessor

//事件监听
EventListenerMethodProcessor...
DefaultEventListenerFactory

...
```

### 3 register(annotatedClasses)


```
/**
	 * 注册单个bean给容器
	 * 比如有新加的类可以用这个方法
	 * 但是注册之后需要手动调用refresh方法去触发容器解析注解
	 *
	 * 有两个意思
	 * 他可以注册一个配置类
	 * 他还可以单独注册一个bean
	 * Register one or more annotated classes to be processed.
	 * <p>Note that {@link #refresh()} must be called in order for the context
	 * to fully process the new classes.
	 * @param annotatedClasses one or more annotated classes,
	 * e.g. {@link Configuration @Configuration} classes
	 * @see #scan(String...)
	 * @see #refresh()
	 */
	public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}
```
### 4 refresh()
- 准备工作包括设置启动时间，是否激活标识位，初始化属性源(property source)配置
- 对工厂进行配置其标准的特征，比如上下文的加载器ClassLoader和post-processors（ApplicationContextAwareProcessor等）回调等
- 在spring的环境中去执行已经被注册的 BeanFactory Processor
- 注册各种BeanPostProcessor
- 初始化应用事件广播器
- 实例化所有剩余的(非lazy-init)单例，包括解决循环依赖，将bean注册到单例池中等工作


```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			////准备工作包括设置启动时间，是否激活标识位，
			// 初始化属性源(property source)配置
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			//返回一个factory 为什么需要返回一个工厂
			//因为要对工厂进行初始化
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			//准备工厂
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.

				//这个方法在当前版本的spring是没用任何代码的
				//可能spring期待在后面的版本中去扩展吧
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				//在spring的环境中去执行已经被注册的 factory processors
				//设置执行自定义的ProcessBeanFactory 和spring内部自己定义的
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				//注册beanPostProcessor
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				//初始化应用事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}
```

以上每个方法都有单独写一篇文章记录的价值，这里先抽几个做一下宏观的解释

#### 1.invokeBeanFactoryPostProcessors
在spring的环境中去执行已经被注册的 工厂后置处理器，包括BeanDefinitionRegistryPostProcessor和BeanFactoryProcessor，主要是执行ConfigurationClassPostProcessor完成bean的扫描，解析注解信息（configuration，import，component）


```
1、getBeanFactoryPostProcessors()得到自己定义的（就是程序员自己写的，并且没有交给spring管理，就是没有加上@Component）
2、得到spring内部自己维护的BeanDefinitionRegistryPostProcessor
org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors
    //调用这个方法
    //循环所有的BeanDefinitionRegistryPostProcessor
    //该方法内部postProcessor.postProcessBeanDefinitionRegistry
    org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
        //调用扩展方法postProcessBeanDefinitionRegistry
        org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
            //拿出的所有bd，然后判断bd时候包含了@Configuration、@Import，@Compent。。。注解
            org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions
                1、的到bd当中描述的类的元数据（类的信息）
                2、判断是不是加了@Configuration   metadata.isAnnotated(Configuration.class.getName())
                3、如果加了@Configuration，添加到一个set当中,把这个set传给下面的方法去解析
                org.springframework.context.annotation.ConfigurationClassUtils#checkConfigurationClassCandidate
                //扫描包
                
                org.springframework.context.annotation.ConfigurationClassParser#parse(java.util.Set<org.springframework.beans.factory.config.BeanDefinitionHolder>)
                    
                    org.springframework.context.annotation.ConfigurationClassParser#parse(org.springframework.core.type.AnnotationMetadata, java.lang.String)
                        //就行了一个类型封装
                        org.springframework.context.annotation.ConfigurationClassParser#processConfigurationClass
                        1、处理内部类 一般不会写内部类
                        org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass
                            //解析扫描的一些基本信息，比如是否过滤，比如是否加入新的包。。。。。
                            org.springframework.context.annotation.ComponentScanAnnotationParser#parse
                                org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan
                                org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#findCandidateComponents
                                    org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents
```












