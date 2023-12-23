---
title: Spring-源码解析-容器的功能扩展-BeanFactory功能扩展
date: 2017-11-25 13:39:14
tags:
    - Spring
    - Java
categories: 源码分析
---

之前分析是建立在`BeanFactory`  接口以及它的实现类 `XmlBeanFactory` 来进行分析的， `ApplicationContext` 包含了 `BeanFactory` 的所有功能，多数情况我们都会使用 `ApplicationConetxt`。其加载方式如下:

`ApplicationContext ctx = new ClassPathXmlApplicationContext("beanFactory.xml");`

所以我们就以 `ClassPathXmlApplicationContext` 来作为分析的切入点：

<!-- more -->

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
	this(configLocations, refresh, null);
}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
		throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
```

`ClassPathXmlApplicationContext` 可以对数组进行解析并进行加载。而对于解析及功能实现都在 `refresh()` 中实现。

## 设置配置路径

在 `ClassPathXmlApplicationContext` 中支持多个配置文件以数组方式同时传入:

```java
public void setConfigLocations(String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}
```

此函数主要用于解析给定的路径数组，当然，如果数组中包含特殊符号，如`${var}`，那么在resolvePath中会搜寻匹配的系统变量并替换。

## 扩展功能

置了路径之后，便可以根据路径做配置文件的解析以及各种功能的实现了。**可以说 `refresh` 函数中包含了几乎 ApplicationContext 中提供的全部功能**，而且此函数中逻辑非常清晰明了，使我们很容易分析对应的层次及逻辑。

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
         // 准备刷新的上下文环境
		prepareRefresh(); // 1

		// Tell the subclass to refresh the internal bean factory.
         // 初始化 BeanFactory，并进行 XML 文件读取
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory(); // 2

		// Prepare the bean factory for use in this context.
         // 对 BeanFactory 进行各种功能填充
		prepareBeanFactory(beanFactory); // 3

		try {
			// Allows post-processing of the bean factory in context subclasses.
              // 子类覆盖方法做额外的处理
			postProcessBeanFactory(beanFactory); // 4

			// Invoke factory processors registered as beans in the context.
          	  // 激活各种 BeanFactory 处理器
			invokeBeanFactoryPostProcessors(beanFactory); // 5

			// Register bean processors that intercept bean creation.
              // 注册拦截 Bean 创建的 Bean 处理器，这里只是注册，真正的调用是在 getBean 时候
			registerBeanPostProcessors(beanFactory); // 6

			// Initialize message source for this context.
              // 为上下文初始化 Message 源，即不同语言的消息体，国际化处理
			initMessageSource(); // 7

			// Initialize event multicaster for this context.
              // 初始化应用消息广播器，并放入 "applicationEventMulticaster" bean 中
			initApplicationEventMulticaster(); // 8

			// Initialize other special beans in specific context subclasses.
              // 留给子类来初始化其他的 bean
			onRefresh(); // 9

			// Check for listener beans and register them.
              // 在所有注册的 bean 中查找 Listerner bean，注册到消息广播器中
			registerListeners(); // 10

			// Instantiate all remaining (non-lazy-init) singletons.
              // 初始化剩下的单实例(非惰性的)
			finishBeanFactoryInitialization(beanFactory); // 11

			// Last step: publish corresponding event.
              // 完成刷新过程，通知生命周期处理器 lifecycleProcessor 刷新过程，同时发出 ContextRefreshEvent 通知别人
			finishRefresh(); // 12
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

1. 初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。

   在某种情况下项目的使用需要读取某些系统变量，而这个变量的设置很可能会影响着系统的正确性，那么`ClassPathXmlApplicationContext`为我们提供的这个准备函数就显得非常必要，它可以在 Spring 启动的时候提前对必须的变量进行存在性验证。

2. 初始化BeanFactory，并进行XML文件读取。

   之前有提到`ClassPathXmlApplicationContext`包含着`BeanFactory`所提供的一切特征，那么在这一步骤中将会复用 `BeanFactory` 中的配置文件读取解析及其他功能，这一步之后， `ClassPathXmlApplicationContext` 实际上就已经包含了 `BeanFactory` 所提供的功能，也就是可以进行 Bean 的提取等基础操作了。

3. 对BeanFactory进行各种功能填充。

   @Qualifier与@Autowired应该是大家非常熟悉的注解，那么这两个注解正是在这一步骤中增加的支持。

4. 子类覆盖方法做额外的处理。

   Spring 之所以强大，除了它功能上为大家提供了便例外，还有一方面是它的完美架构，开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能。

5. 激活各种BeanFactory处理器。

6. 注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候。

7. 为上下文初始化 Message 源，即对不同语言的消息体进行国际化处理。

8. 初始化应用消息广播器，并放入 `applicationEventMulticaster` bean 中。

9. 留给子类来初始化其他的bean。

10. 在所有注册的bean中查找 listener bean，注册到消息广播器中。

11. 初始化剩下的单实例（非惰性的）。

12. 完成刷新过程，通知生命周期处理器`lifecycleProcessor`刷新过程，同时发出`ContextRefreshEvent`通知别人。

## 环境准备

`prepareRefresh`函数主要是做些准备工作，例如对系统属性及环境变量的初始化及验证。

```java
protected void prepareRefresh() {
	this.startupDate = System.currentTimeMillis();
	this.closed.set(false);
	this.active.set(true);

	if (logger.isInfoEnabled()) {
		logger.info("Refreshing " + this);
	}

	// Initialize any placeholder property sources in the context environment
  	 // 留给子类覆盖
	initPropertySources();

	// Validate that all properties marked as required are resolvable
	// see ConfigurablePropertyResolver#setRequiredProperties
     // 验证需要的属性文件是否都已经放入环境中
	getEnvironment().validateRequiredProperties();

	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

1. `initPropertySources` 正符合 Spring 的开放式结构设计，给用户最大扩展 Spring 的能力。用户可以根据自身的需要重写 `initPropertySources` 方法，并在方法中进行个性化的属性处理及设置。
2. `validateRequiredProperties` 则是通过继承重写的方式对属性进行验证。

## 加载 BeanFactory

经过 `obtainFreshBeanFactory` 之后 `ApplicationContext` 就已经拥有了 `BeanFactory` 的全部功能。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
     // 初始化 BeanFactory，并进行 XML 文件读取，并将得到的 BeanFactory 记录在当前实体的属性中
	refreshBeanFactory();
     // 返回当前实体的 beanFactory 属性
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

```java
protected final void refreshBeanFactory() throws BeansException {
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
         // 创建 DefaultListableBeanFactory
		DefaultListableBeanFactory beanFactory = createBeanFactory(); // 1
         // 为了序列化指定 id，如果需要的话，让这个 BeanFactory 从 id 反序列化到 BeanFactory 对象
		beanFactory.setSerializationId(getId()); // 2
         // 定制 beanFactory，设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
		customizeBeanFactory(beanFactory); // 3
         // 初始化 DocumentReader，并进行 XML 文件读取及解析
		loadBeanDefinitions(beanFactory); // 4
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory; // 5
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

1. 创建 DefaultListableBeanFactory
2. 指定序列化 ID
3. 定制 BeanFactory
4. 加载 BeanDefinition
5. 使用全局变量记录 BeanFactory 类实例

### 定制 `BeanFactory`

增加是否允许覆盖是否允许扩展的设置（通过子类覆盖）

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
     // 如果属性 allowBeanDefinitionOverriding 不为空，设置给 beanFactory 对象相应属性
     // 此属性的含义：是否允许覆盖同名称的不同定义的对象
	if (this.allowBeanDefinitionOverriding != null) {
		beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
	}
     // 如果属性 allowCircularReferences 不为空，设置给 beanFactory 对象相应属性，
     // 此属性的含义：是否允许 bean 之间存在循环依赖
	if (this.allowCircularReferences != null) {
		beanFactory.setAllowCircularReferences(this.allowCircularReferences);
	}
}
```

### 加载 BeanDefinition

与之前解析 `XmlBeanFactory` 一样，初始化 `DefaultListableBeanFactory` 之后需要 `XmlBeanDefinitionReader` 来读取 XML，接下来是初始化 `XmlBeanDefinitionReader` 的步骤

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
     // 为指定的 BeanFactory 创建 XmlBeanDefinitionReader
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
     // 对 beanDefinitionReader 进行环境变量的设置
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
     // 对 BeanDefinitionReader 进行设置，可以覆盖
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```

在初始化了 `DefaultListableBeanFactory` 和 `XmlBeanDefinitionReader` 后就可以进行配置文件的读取了

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		reader.loadBeanDefinitions(configLocations);
	}
}
```

接下来的套路之前基本都已经说到了。

## 功能扩展

在进入函数 `prepareBeanFactory` 前，Spring 已经完成了对配置的解析，而 `ApplicationContext` 在功能上的扩展也由此展开。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
  	 // 设置 beanFactory 的 classLoader 为当前 context 的 classLoader
	beanFactory.setBeanClassLoader(getClassLoader());
  
  	// 设置 beanFactory 的表达式语言处理器，默认可使用 #{bean.xxx} 的形式来调用相关属性值
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 为beanFactory 增加了一个默认的 propertyEditor，这个主要是对 bean 的属性等设置管理的一个工具
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
     // 添加 BeanPostProcessor
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
     // 设置了几个忽略自动装配的接口
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
     // 设置了几个自动装配的特殊规则
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
     // 4.3.4 后添加的
     // 避免 BeanPostProcessor 被 getBeanNamesForType 调用
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
     // 增加对 AspectJ 的支持
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// Register default environment beans.
     // 添加默认的系统环境 bean
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```

上面函数主要进行了几个方面的扩展

-  增加对`SPEL`语言的支持。
-  增加对属性编辑器的支持。
-  增加对一些内置类，比如`EnvironmentAware`、`MessageSourceAware`的信息注入。
-  设置了依赖功能可忽略的接口。
-  注册一些固定依赖的属性。
-  注册 `ApplicationListenerDetector`
-  增加AspectJ的支持（会在后面进行详细的讲解）。
-  将相关环境变量及属性注册以单例模式注册。

### 增加 SPEL 语言的支持

在运行时构建复杂表达式、存取对象图属性、对象方法调用等，并且能与 Spring 功能完美整合，比如能用来配置bean 定义。SpEL 是单独模块，只依赖于 core 模块，不依赖于其他模块，可以单独使用。

我们进入这个方法发现它只是简单指定了一个 `StandardBeanExpressionResolver` 进行语言解析器的注册，之后在我们之前说的 `bean` 进行初始化时候的属性填充时候会调用 `AbstractAutowireCapableBeanFactory` 类的 `applyPropertyValues` 函数来完成功能。同时，也是在这个步骤中一般通过 `AbstractBeanFactory` 中的 `evaluateBeanDefinitionString` 方法区完成 SPEL 的解析。

```java
protected Object evaluateBeanDefinitionString(String value, BeanDefinition beanDefinition) {
	if (this.beanExpressionResolver == null) {
		return value;
	}
	Scope scope = (beanDefinition != null ? getRegisteredScope(beanDefinition.getScope()) : null);
	return this.beanExpressionResolver.evaluate(value, new BeanExpressionContext(this, scope));
}
```

### 增加属性注册编辑器

对于自定义属性的注入如果在原生属性编辑器（`PropertyEditor`）不支持的情况下，可以使用两种方法：

-  自定义属性编辑器

   继承 `PropertyEditorSupport` 重写 `setAsText` 方法

-  注册 Spring 自带的属性编辑器 `CustomDateEditor`

在这里 `beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));` 这段语句中我们可以探究一下 `ResourceEditorRegistrar` 的实现其中的 `doRegisterEditor` 方法中（被核心方法 `registerCustomEditors` 调用）可以看到提到了自定义属性中使用的关键代码。

```java
private void doRegisterEditor(PropertyEditorRegistry registry, Class<?> requiredType, PropertyEditor editor) {
	if (registry instanceof PropertyEditorRegistrySupport) {
		((PropertyEditorRegistrySupport) registry).overrideDefaultEditor(requiredType, editor);
	}
	else {
		registry.registerCustomEditor(requiredType, editor);
	}
}
```

此时我们再回头看 `registerCustomEditors` 可以看到内部无非是注册了一系列的常用类型的属性编辑器。那什么时候能调用到它呢？

借助 idea 的调用链工具我们可以看到

![ResourceEditorRegistrar的registerCustomEditors方法](https://github.com/BingLau7/blog/blob/master/images/blog_37/ResourceEditorRegistrar%E7%9A%84registerCustomEditors%E6%96%B9%E6%B3%95.jpg?raw=true)

在 `AbstractBeanFactory` 中的 `registerCustomEditors` 方法中被调用过，继续下去我们可以看到一个熟悉的方法就是 `AbstractBeanFactory` 类中的 `initBeanWrapper` 方法，这是在 bean 初始化时候使用的一个方法，[详见](https://binglau7.github.io/2017/11/20/Spring-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E2%80%94%E5%88%9B%E5%BB%BA-bean/#instantiateBean-不带参数的构造函数实例化过程)。

此时，逻辑已经明了了，在 bean 的初始化之后会调用 `ResourceEditorRegistrar` 的 `registerCustomEditors` 方法进行批量的通用属性编辑器注册。注册后，在属性填充的环节便可以直接让 Spring 使用这些编辑器进行熟悉的解析了。

既然提到了`BeanWrapper`，这里也有必要强调下，Spring 中用于封装 bean 的是 `BeanWrapper` 类型，而它又间接继承了 `PropertyEditorRegistry` 类型，也就是我们之前反复看到的方法参数 `PropertyEditorRegistry registry`，其实大部分情况下都是 `BeanWrapper`，对于 `BeanWrapper` 在 Spring 中的默认实现是`BeanWrapperImpl`，而 `BeanWrapperImpl` 除了实现 `BeanWrapper` 接口外还继承了`PropertyEditorRegistrySupport`，在 `PropertyEditorRegistrySupport` 中有这样一个方法 `createDefaultEditors` 定义了一系列常用的属性编辑器方便我们进行配置。

### 添加 `ApplicationContextAwareProcessor` 处理器

`beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));` （不加这行自己都忘了自己说到了哪儿）

我们继续通过 `AbstractApplicationContext` 的 `prepareBeanFactory` 方法的主线来进行函数跟踪。对于 `beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this))` 其实主要目的就是注册个 `BeanPostProcessor`，而真正的逻辑还是在 `ApplicationContextAwareProcessor` 中。

`ApplicationContextAwareProcessor` 实现 `BeanPostProcessor` 接口，我们回顾下之前讲过的内容，在 bean 实例化的时候，也就是 Spring 激活 bean 的 `init-method` 的前后，会调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法和 `postProcessAfterInitialization` 方法。同样，对于`ApplicationContextAwareProcessor` 我们也关心这两个方法。

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
	return bean;
}
```

… 过

```java
@Override
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
	AccessControlContext acc = null;

	if (System.getSecurityManager() != null &&
			(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware || bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware || bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
		acc = this.applicationContext.getBeanFactory().getAccessControlContext();
	}

	if (acc != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareInterfaces(bean);
				return null;
			}
		}, acc);
	}
	else {
		invokeAwareInterfaces(bean);
	}

	return bean;
}
```

```java
private void invokeAwareInterfaces(Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof EnvironmentAware) {
			((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
		}
		if (bean instanceof EmbeddedValueResolverAware) {
			((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
		}
		if (bean instanceof ResourceLoaderAware) {
			((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
		}
		if (bean instanceof ApplicationEventPublisherAware) {
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}
}
```

实现这些 `Aware` 接口的 bean 在被初始化之后，可以取得一些对应的资源。

### 设置忽略依赖

`invokeAwareInterfaces` 方法中间接调用的 `Aware` 类已经不是普通的 bean 了，如 `ResourceLoaderAware`、`ApplicationEventPublisherAware` 等，那么当然需要在 Spring 做 bean 的依赖注入的时候忽略它们。而 `ignoreDependencyInterface` 的作用正是在此。

```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

### 注册依赖

Spring中有了忽略依赖的功能，当然也必不可少地会有注册依赖的功能。

```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

当注册了依赖解析后，例如当注册了对 `BeanFactory.class` 的解析依赖后，当 bean 的属性注入的时候，一旦检测到属性为 `BeanFactory` 类型便会将 `beanFactory` 的实例注入进去。
