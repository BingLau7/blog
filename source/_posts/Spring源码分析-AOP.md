---
title: Spring源码分析-AOP
date: 2017-12-18 22:28:01
tags:
    - Spring
    - Java
categories: 源码分析
---

## 动态 AOP 使用示例

1. 创建用于拦截的 bean

   ```java
   @Data
   public class TestBean {
       private String testStr = "test";

       public void test() {
           System.out.println("test");
       }
   }
   ```

2. 创建 Advisor

   ```java
   @Aspect
   public class AspectJTest {
       @Pointcut("execution(* *.test(..))")
       public void test() {

       }

       @Before("test()")
       public void beforeTest() {
           System.out.println("beforeTest");
       }

       @After("test()")
       public void afterTest() {
           System.out.println("afterTest");
       }

       @Around("test()")
       public Object aroundTest(ProceedingJoinPoint p) {
           System.out.println("brfore around");
           Object o = null;
           try {
               o = p.proceed();
           } catch (Throwable e) {
               e.printStackTrace();
           }
           System.out.println("after around");
           return o;
       }
   }
   ```

3. 创建配置文件

   ```xml
       <aop:aspectj-autoproxy />
       <bean id="testBean" class="io.github.binglau.bean.TestBean" />
       <bean class="io.github.binglau.AspectJTest" />
   ```

4. 测试

   ```java
       @Test
       public void testAop() {
           ApplicationContext ctx = new ClassPathXmlApplicationContext("beanFactory.xml");
           TestBean bean = (TestBean) ctx.getBean("testBean");
           bean.test();
       }
   ```

5. 不出意外结果

   ```
   brfore around
   beforeTest
   test
   after around
   afterTest
   ```

可知 `<aop:aspectj-autoproxy />` 是开启 aop 的关键，我们不妨由此入手。

<!--more-->

## 动态 AOP 自定义标签

之前讲过 Spring 中的[自定义注解](https://binglau7.github.io/2017/11/12/Spring%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-bean%E7%9A%84%E8%A7%A3%E6%9E%90%EF%BC%883%EF%BC%89/)，如果声明了自定义的注解，那么就一定会在程序中的某个地方注册了对应的解析器。我们搜索整个代码，尝试找到注册的地方，全局搜索后我们发现了在 `AopNamespaceHandler` 中对应着这样一段函数。中间我们看到了 `aspectj-autoproxy`

```java
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```

### 注册 `AnnotationAwareAspectJAutoProxyCreator`

所有解析器，因为是对 `BeanDefinitionParser` 接口的统一实现，入口都是从 `parse` 函数开始的，`AspectJAutoProxyBeanDefinitionParser` 的 `parse` 函数如下：

```java
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
         // 注册 AnnotationAwareAspectJAutoProxyCreator
		AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
         // 对于注解中子类的处理
		extendBeanDefinition(element, parserContext);
		return null;
	}
```

其中 `registerAspectJAnnotationAutoProxyCreatorIfNecessary` 很明显值得注意，这也是其关键逻辑的实现。

```java
	public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {
         // 注册或升级 AutoProxyCreator 定义 beanName 为 org.Springframework.aop.config.internalAutoProxyCreator 的 BeanDefinition
		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
         // 对于 proxy-target-class 以及 expose-proxy 属性的处理
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
         // 注册组件并通知，便于监听器做进一步处理
         // 其中 beanDefinition 的 className 为 AnnotationAwareAspectJAutoProxyCreator
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
```

#### 注册或者升级 `AnnotationAwareAspectJAutoProxyCreator`

对于 AOP 的实现，基本上都是靠 `AnnotationAwareAspectJAutoProxyCreator` 去完成，它可以根据 `@Point` 注解定义的切点来自动代理相匹配的 bean。但是为了配置简便，Spring 使用了自定义配置来帮助我们自动注册 `AnnotationAwareAspectJAutoProxyCreator` ，其注册过程就是在这里实现的。

```java
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
		return registerOrEscalateApcAsRequired(**AnnotationAwareAspectJAutoProxyCreator**.class, registry, source);
	}
```

```java
	private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
         // 如果已经存在了自动代理创建器且存在的自动代理创建器与现状的不一致那么需要根据优先级来判断
         // 到底需要使用哪个
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
             // 	public static final String AUTO_PROXY_CREATOR_BEAN_NAME =
             //  "org.springframework.aop.config.internalAutoProxyCreator";
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
				if (currentPriority < requiredPriority) {
                      // 改变 bean 最重要的就是改变 bean 所对应的 className 属性
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
             // 如果已经存在自动代理创建器并且与将要创建的一直，那么无需再次创建
			return null;
		}
		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
		return beanDefinition;
	}
```

以上代码中实现了自动注册 `AnnotationAwareAspectJAutoProxyCreator` 类的功能，同时这里还涉及了一个优先级的问题，如果已经存在了自动代理创建器，而且存在的自动代理创建器与现在的不一致，那么需要根据优先级来判断到底需要使用哪个。

#### 处理 proxy-target-class 以及 expose-proxy 属性

```java
	private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, Element sourceElement) {
		if (sourceElement != null) {
             // 对于 proxy-target-class 属性的处理
			boolean proxyTargetClass = Boolean.valueOf(sourceElement.getAttribute(PROXY_TARGET_CLASS_ATTRIBUTE));
			if (proxyTargetClass) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
             // 对于 expose-proxy 属性的处理
			boolean exposeProxy = Boolean.valueOf(sourceElement.getAttribute(EXPOSE_PROXY_ATTRIBUTE));
			if (exposeProxy) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
```

```java
	// 强制使用的过程其实也是一个属性设置的过程
	public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
		}
	}

	public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
		}
	}
```

##### proxy-target-class

Spring AOP 部分使用 **JDK 动态代理**或者 **CGLIB** 来为目标对象创建代理。（建议尽量使用 JDK 的动态代理），**如果被代理的目标对象实现了至少一个接口，则会使用 JDK 动态代理。**所有该目标类型实现的接口都将被代理。**若该目标对象没有实现任何接口，则创建一个CGLIB代理。**如果你希望强制使用CGLIB代理，（例如希望代理目标对象的所有方法，而不只是实现自接口的方法）那也可以。但是需要考虑以下两个问题。

-  无法通知（advise）Final 方法，因为它们不能被覆写。
-  你需要将CGLIB二进制发行包放在classpath下面。

如果需要强制使用 CGLIB 代理，则需要将 `<aop:config>` 的 `proxy-target-class` 属性设为 true:

`<aop:config proxy-target-class="true"> ... </aop:config>`

当需要使用 CGLIB 代理和 `@AspectJ` 自动代理支持，可以按照以下方式设置 `<aop:aspectj-autoproxy>` 的 `proxy-target-class` 属性：

`<aop:aspectj-autoproxy proxy-target-class="true"/>`

##### JDK 动态代理 vs CGLIB 代理

-  JDK动态代理：其代理对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理。
-  CGLIB 代理：实现原理类似于 JDK 动态代理，只是它在运行期间生成的代理对象是针对目标类扩展的子类。CGLIB是高效的代码生成包，底层是依靠ASM（开源的Java字节码编辑类库）操作字节码实现的，性能比JDK强。

具体可以看之前的博客 [AOP 的道理](https://binglau7.github.io/2017/06/11/AOP%E7%9A%84%E9%81%93%E7%90%86/)

##### expose-proxy

有时候目标对象内部的自我调用将无法实施切面中的增强，如下示例：

```java
public interface AService {
	public void a();
	public void b();
}

@Service()
public class AServiceImpl1 implements AService{
	@Transactional(propagation = Propagation.REQUIRED)
	public void a() {
		this.b();
	}
  
	@Transactional(propagation = Propagation.REQUIRES_NEW)
	public void b() {
	}
}
```

此处的 this 指向目标对象，因此调用 `this.b()` 将不会执行 b 事务切面，即不会执行事务增强，因此 b 方法的事务定义`@Transactional(propagation = Propagation.REQUIRES_NEW)`将不会实施，为了解决这个问题，我们可以这样做：

`<aop:aspectj-autoproxy expose-proxy="true"/>`

然后将以上代码中的 `this.b();` 修改为 `((AService) AopContext.currentProxy()).b();` 即可。通过以上的修改便可以完成对 a 和 b 方法的同时增强。

最后注册组件并通知，便于监听器做进一步处理，这里就不再一一赘述了。

## 创建 AOP 代理

上文讲解了通过自定义配置完成了对 `AnnotationAwareAspectJAutoProxyCreator` 类型的自动注册，那么这个类到底做了什么工作来完成 AOP 的操作呢？

首先让我们看看 `AnnotationAwareAspectJAutoProxyCreator` 类的层次结构

![AnnotationAwareAspectJAutoProxyCreator类的层次结构图](https://github.com/BingLau7/blog/blob/master/images/blog_41/AnnotationAwareAspectJAutoProxyCreator%E7%B1%BB%E5%B1%82%E6%AC%A1%E7%BB%93%E6%9E%84.jpg?raw=true)

我们看到 `AnnotationAwareAspectJAutoProxyCreator` 实现了 `BeanPostProcessor` 接口，而实现了该接口之后，当 Spring 加载这个 Bean 时会在实例化之前调用其 `postProcessAfterInitialization` 方法，其逻辑也由此开始。

其父类 `AbstractAutoProxyCreator#postProcessAfterInitialization`

```java
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
         // 根据给定的 bean 的 class 和 name 构建出个 key，格式: beanClassName_beanName
		Object cacheKey = getCacheKey(beanClass, beanName);

         // 对 bean 不存在的一些处理或者这个 bean 无需增强
		if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
             // 跳过一些特殊的类
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		if (beanName != null) {
			TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
			if (targetSource != null) {
				this.targetSourcedBeans.add(beanName);
                  // 如果存在增强方法则创建代理
				Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
                  // 创建代理
				Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
		}

		return null;
	}
```

大致主流程氛围三步（判断条件走完之后）：

1. 获取需要增强的 targetSource
2. 获取增强器
3. 根据需要增强的 targetSource 和增强器创建代理​

### 获取 targetSource

>  这部分代码与书上不符，属于多出来的步骤

这里顺便贴出 TargetSource 的说明

```java
/**
 * A {@code TargetSource} is used to obtain the current "target" of
 * an AOP invocation, which will be invoked via reflection if no around
 * advice chooses to end the interceptor chain itself.
 *
 * <p>If a {@code TargetSource} is "static", it will always return
 * the same target, allowing optimizations in the AOP framework. Dynamic
 * target sources can support pooling, hot swapping, etc.
 *
 * <p>Application developers don't usually need to work with
 * {@code TargetSources} directly: this is an AOP framework interface.
 */
```

这里可以看，TargetSource 是用来获取调用 AOP 的那个当前『目标』。每当AOP代理处理一个方法调用时都会向TargetSource 的实现请求一个目标实例。

使用 Spring AOP 的开发者通常不需要直接和 TargetSource 打交道，但这提供了一种强大的方式来支持池化（pooling），热交换（hot swappable）和其它高级目标。 例如，一个使用池来管理实例的 TargetSource 可以为每个调用返回一个不同的目标实例。

如果你不指定一个TargetSource，一个缺省实现将被使用，它包装一个本地对象。对于每次调用它将返回相同的目标（像你期望的那样）。

[官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-targetsource)

```java
	/**
	 * Create a target source for bean instances. Uses any TargetSourceCreators if set.
	 * Returns {@code null} if no custom TargetSource should be used.
	 * <p>This implementation uses the "customTargetSourceCreators" property.
	 * Subclasses can override this method to use a different mechanism.
	 * @param beanClass the class of the bean to create a TargetSource for
	 * @param beanName the name of the bean
	 * @return a TargetSource for this bean
	 * @see #setCustomTargetSourceCreators
	 */
	protected TargetSource getCustomTargetSource(Class<?> beanClass, String beanName) {
		// We can't create fancy target sources for directly registered singletons.
		if (this.customTargetSourceCreators != null &&
				this.beanFactory != null && this.beanFactory.containsBean(beanName)) {
			for (TargetSourceCreator tsc : this.customTargetSourceCreators) {
				TargetSource ts = tsc.getTargetSource(beanClass, beanName);
				if (ts != null) {
					// Found a matching TargetSource.
					if (logger.isDebugEnabled()) {
						logger.debug("TargetSourceCreator [" + tsc +
								" found custom TargetSource for bean with name '" + beanName + "'");
					}
					return ts;
				}
			}
		}

		// No custom TargetSource found.
		return null;
	}
```

### 获取增强器

```java
	@Override
	protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}

	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```

对于指定 bean 的增强方法的获取一定是包含两个步骤的，获取所有的增强以及寻找所有增强中适用于 bean 的增强并应用，那么 `findCandidateAdvisors` 与 `findAdvisorsThatCanApply` 便是做了这两件事情。当然，如果无法找到对应的增强器便返回  null。

由于我们分析的是使用注解进行的 AOP，所以对于 `findCandidateAdvisors` 的实现其实是由 `AnnotationAwareAspectJAutoProxyCreator` 类完成的

```java
	@Override
	protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
         // 当使用注解方式配置 AOP 的时候并不是丢弃了对 XML 配置的支持
         // 在这里调用父类方法加载配置文件中的 AOP 声明
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		return advisors;
	}
```

`AnnotationAwareAspectJAutoProxyCreator` 间接继承了 `AbstractAdvisorAutoProxyCreator`，在实现获取增强的方法中除了保留父类的获取配置文件中定义的增强外，同时添加了获取 Bean 的注解增强的功能，那么其实现正是由`this.aspectJAdvisorsBuilder.buildAspectJAdvisors()`来实现的。

首先在没有接触代码的情况下，让我们理一下思路，看看增强器解析步骤

1. 获取所有 beanName，这一步骤中所有再 beanFactory 中注册的 Bean 都会被提取出来
2. 遍历所有 beanName，并找出声明 AspectJ 注解的类，进行进一步处理
3. 对标记为 AspectJ 注解的类进行增强器的提取
4. 将提取结果加入缓存

接下来看看实现，首先是对 Spring 中所有类进行分析，提取 Advisor

```java
	/**
	 * Look for AspectJ-annotated aspect beans in the current bean factory,
	 * and return to a list of Spring AOP Advisors representing them.
	 * <p>Creates a Spring Advisor for each AspectJ advice method.
	 * @return the list of {@link org.springframework.aop.Advisor} beans
	 * @see #isEligibleBean
	 */
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new LinkedList<Advisor>();
					aspectNames = new LinkedList<String>();
                      // 获取所有 beanName
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
                      // 循环所有的 beanName 找出对应的增强方法
					for (String beanName : beanNames) {
                           // 不合法的 bean 则略过，由子类定义规则，默认返回 true
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
                          // 获取对应的 bean 的类型
						Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
                          // 如果存在 Aspect 注解
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                   // 解析标记 AspectJ 注解中的增强方法
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
      
         // 记录在缓存中
		List<Advisor> advisors = new LinkedList<Advisor>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```

至此我们已经完成了 Advisor 的提取，其中增强器获取的工作委托给了 `getAdvisors` 方法实现

```java
	@Override
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
         // 获取标记为 AspectJ 的类
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
         // 获取标记为 AspectJ 的 name
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
         // 验证
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new LinkedList<Advisor>();
		for (Method method : getAdvisorMethods(aspectClass)) {
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
          	 // 如果寻找的增强器不为空而且又配置了增强延迟初始化那么需要在首位加入同步实例化增强器
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
      	 // 获取 DeclareParents 注解
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}

	private List<Method> getAdvisorMethods(Class<?> aspectClass) {
		final List<Method> methods = new LinkedList<Method>();
		ReflectionUtils.doWithMethods(aspectClass, new ReflectionUtils.MethodCallback() {
			@Override
			public void doWith(Method method) throws IllegalArgumentException {
				// Exclude pointcuts
                  // 对声明为 pointcut 的方法不处理
				if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
					methods.add(method);
				}
			}
		});
		Collections.sort(methods, METHOD_COMPARATOR);
		return methods;
	}
```

#### 普通增强器的获取

普通增强器的获取逻辑通过 `getAdvisor` 方法实现，实现步骤包括对切点的注解的获取以及根据注解信息生成增强

```java
	@Override
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

      	 // 切点信息获取
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
		 // 根据切点信息生成增强器
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
```

##### 切点信息获取

所谓获取切点信息就是制定注解的表达式信息的获取，如 `@Before("test()")`

```java
	private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
	     // 获取方法上的注解
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

         // 使用 AspectJExpressionPointcut 实例封装获取的信息
		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
         // 提取得到的注解中的表达式，如
         // @Pointcut("execution(* *.*test*(..))") 中的 execution(* *.*test*(..))
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		ajexp.setBeanFactory(this.beanFactory);
		return ajexp;
	}
```

```java
	protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
         // 设置敏感的注解类
		Class<?>[] classesToLookFor = new Class<?>[] {
				Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
		for (Class<?> c : classesToLookFor) {
			AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
			if (foundAnnotation != null) {
				return foundAnnotation;
			}
		}
		return null;
	}
```

```java
	// 获取指定方法上的注解并使用 AspectJAnnotation 封装
	private static <A extends Annotation> AspectJAnnotation<A> findAnnotation(Method method, Class<A> toLookFor) {
		A result = AnnotationUtils.findAnnotation(method, toLookFor);
		if (result != null) {
			return new AspectJAnnotation<A>(result);
		}
		else {
			return null;
		}
	}
```

##### 根据切点信息生成增强

所有的增强都由 Advisor 的实现类 `InstantiationModelAwarePointcutAdvisorImpl` 统一封装

```java
	public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

      	 // test()
		this.declaredPointcut = declaredPointcut;
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		this.methodName = aspectJAdviceMethod.getName();
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
      	 // public void test.AspectJTest.beforeTest()
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		this.aspectInstanceFactory = aspectInstanceFactory;
         // 0
		this.declarationOrder = declarationOrder;
         // test.AspectJTest
		this.aspectName = aspectName;

		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			// Static part of the pointcut is a lazy type.
			Pointcut preInstantiationPointcut = Pointcuts.union(
					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
			// If it's not a dynamic pointcut, it may be optimized out
			// by the Spring AOP infrastructure after the first evaluation.
			this.pointcut = new PerTargetInstantiationModelPointcut(
					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
			this.lazy = true;
		}
		else {
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
	}
```

在封装过程中只是简单地将信息封装在类的实例中，所有的信息单纯地赋值，在**实例初始化的过程中还完成了对于增强器的初始化**。因为不同的增强所体现的逻辑是不同的，比如`@Before("test()")`与@`After("test()"`标签的不同就是增强器增强的位置不同，所以就需要不同的增强器来完成不同的逻辑，而根据注解中的信息初始化对应的增强器就是在`instantiateAdvice`函数中实现的。

```java
	private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
		return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
	}

	@Override
	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		validate(candidateAspectClass);

		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// If we get here, we know we have an AspectJ method.
		// Check that it's an AspectJ-annotated class
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
		}

		AbstractAspectJAdvice springAdvice;

      	 // 根据不同的注解类型封装不同的增强器
		switch (aspectJAnnotation.getAnnotationType()) {
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		springAdvice.calculateArgumentBindings();
		return springAdvice;
	}
```

>  这里书上描述得有些奇怪，直接进入了 `MethodBeforeAdviceInterceptor` 解析了，但是我们并不知道这与 `AspectJMethodBeforeAdvice` 有何关系。后面查阅了一下网上资料。
>
>  http://lgbolgger.iteye.com/blog/2117214
>
>  https://www.dengxiangxing.com/post/16229
>
>  http://www.voidcn.com/article/p-erexljic-bqk.html

![Advice 结构图](https://github.com/BingLau7/blog/blob/master/images/blog_41/Advice%20%E7%BB%93%E6%9E%84%E5%9B%BE.jpg?raw=true)

这里我们可以看出，`AspectJMethodBeforeAdvice` 和 `AspectJAfterReturningAdvice` 均没有实现  `MethodInterceptor` ，但是我们会发现 `AspectJMethodBeforeAdvice#before` 方法是被 `MethodBeforeAdviceInterceptor#invoke` 调用的（通过 idea 的 `call Hierarchy`），所以根据网上资料推测 `MethodInterceptor` 的 `invoke` 方法是直接调用的整个 AOP 流程的，而 `AspectJMethodBeforeAdvice` 虽然没有实现 `MethodInterceptor` 但是通过了 `MethodBeforeAdviceInterceptor` 适配器来适配解决了这个问题。而这二者的适配器分别是 `MethodBeforeAdviceAdapter`, `AfterReturningAdviceAdapter`。

>  不如我们后面开个 Spring 中的设计模式专题吧？

这里我们后面再说其适配的时机。

#### 增加同步实例化增强器

如果寻找的增强器不为空而且又配置了增强延迟初始化，那么就需要在首位加入同步实例增强器。同步实例化增强器 `SyntheticInstantiationAdvisor` 如下：

```java
	@SuppressWarnings("serial")
	protected static class SyntheticInstantiationAdvisor extends DefaultPointcutAdvisor {

		public SyntheticInstantiationAdvisor(final MetadataAwareAspectInstanceFactory aif) {
			super(aif.getAspectMetadata().getPerClausePointcut(), new MethodBeforeAdvice() {
              	 // 目标方法前调用，类似 @Before
				@Override
				public void before(Method method, Object[] args, Object target) {
					// Simply instantiate the aspect
					aif.getAspectInstance();
				}
			});
		}
	}
```

#### 获取 `DeclareParents` 注解

`DeclareParents` 主要用于引介增强的注解形式的实现，而其实现方式与普遍增强很类似，只不过使用 `DeclareParentsAdvisor` 对功能进行封装

```java
	/**
	 * Build a {@link org.springframework.aop.aspectj.DeclareParentsAdvisor}
	 * for the given introduction field.
	 * <p>Resulting Advisors will need to be evaluated for targets.
	 * @param introductionField the field to introspect
	 * @return {@code null} if not an Advisor
	 */
	private Advisor getDeclareParentsAdvisor(Field introductionField) {
		DeclareParents declareParents = introductionField.getAnnotation(DeclareParents.class);
		if (declareParents == null) {
			// Not an introduction field
			return null;
		}

		if (DeclareParents.class == declareParents.defaultImpl()) {
			throw new IllegalStateException("'defaultImpl' attribute must be set on DeclareParents");
		}

		return new DeclareParentsAdvisor(
				introductionField.getType(), declareParents.value(), declareParents.defaultImpl());
	}
```

### 寻找匹配的增强器

前面的函数中已经完成了所有增强器的解析，但是对于所有增强器来讲，并不一定都适用于当前的 Bean，还要挑取适合的增强器，也就是满足我们配置的通配符的增强器。具体实现在 `findAdvisorsThatCanApply` 中。

还记得之前说过 `AnnotationAwareAspectJAutoProxyCreator` 间接继承了 `AbstractAdvisorAutoProxyCreator`，而 `findAdvisorsThatCanApply` 就是在 `AbstractAdvisorAutoProxyCreator` 中被调用的(其调用链可追溯到`postProcessAfterInitialization`)。

```java
	/**
	 * Search the given candidate Advisors to find all Advisors that
	 * can apply to the specified bean.
	 * @param candidateAdvisors the candidate Advisors
	 * @param beanClass the target's bean class
	 * @param beanName the target's bean name
	 * @return the List of applicable Advisors
	 * @see ProxyCreationContext#getCurrentProxiedBeanName()
	 */
	protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
             // 过滤已经得到的 advisors
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}
```

继续看 `findAdvisorsThatCanApply`

```java
	/**
	 * Determine the sublist of the {@code candidateAdvisors} list
	 * that is applicable to the given class.
	 * @param candidateAdvisors the Advisors to evaluate
	 * @param clazz the target class
	 * @return sublist of Advisors that can apply to an object of the given class
	 * (may be the incoming List as-is)
	 */
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
         // 首先处理引介增强
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
             // 引介增强已被处理
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
          	 // 对于普通 bean 的处理
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
```

`findAdvisorsThatCanApply` 函数的主要功能是寻找所有增强器中适用于当前 class 的增强器。引介增强与普通的增强是处理不一样的，所以分开处理。而对于真正的匹配在 `canApply` 中实现。

### 创建代理

`AbstractAutoProxyCreator#createProxy`

```java
	/**
	 * Create an AOP proxy for the given bean.
	 * @param beanClass the class of the bean
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @param targetSource the TargetSource for the proxy,
	 * already pre-configured to access the bean
	 * @return the AOP proxy for the bean
	 * @see #buildAdvisors
	 */
	protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
         // 获取当前类中相关属性
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
             // 决定对于给定的 bean 是否应该使用 targetClass 而不是他的接口代理
          	 // 检查 proxyTargetClass 设置以及 preserveTargetClass 属性
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
                 // 添加代理接口
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
             // 加入增强器
			proxyFactory.addAdvisor(advisor);
		}

         // 设置要代理的类
		proxyFactory.setTargetSource(targetSource);
         // 定制代理
		customizeProxyFactory(proxyFactory);

         // 用来控制代理工程被配置之后，是否还允许修改通知
         // 缺省值为 false(即在代理被配置之后，不允许修改代理的配置)
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

对于代理类的创建及处理， Spring 委托给了 `ProxyFactory` 去处理，而在此函数中主要是对 `ProxyFactory` 的初始化操作，进而对真正的创建代理做准备，这些初始化操作包括如下内容：

1. 获取当前类的属性
2. 添加代理接口
3. 封装 Advisor 并加入到 ProxyFactory 中
4. 设置要代理的类
5. 当然在 Spring 中还为子类提供了定制的函数 `customizeProxyFactory`，子类可以在此函数中进行对 `ProxyFactory` 的进一步封装
6. 进行获取代理操作

其中，封装 Advisor 并加入到 `ProxyFactory` 中以及创建代理是两个相对繁琐的过程，可以通过 `ProxyFactory` 提供的 `addAdvisor` 方法直接将增强器置入代理创建工程中，但是拦截器封装为增强器还是急需要一定的逻辑的。

#### 封装 Advisor 并加入到 ProxyFactory 中

```java
	/**
	 * Determine the advisors for the given bean, including the specific interceptors
	 * as well as the common interceptor, all adapted to the Advisor interface.
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @return the list of Advisors for the given bean
	 */
	protected Advisor[] buildAdvisors(String beanName, Object[] specificInterceptors) {
		// Handle prototypes correctly...
		Advisor[] commonInterceptors = resolveInterceptorNames();

		List<Object> allInterceptors = new ArrayList<Object>();
		if (specificInterceptors != null) {
             // 加入拦截器
			allInterceptors.addAll(Arrays.asList(specificInterceptors));
			if (commonInterceptors.length > 0) {
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isDebugEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}

		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
             // 拦截器进行封装转化为 Advisor
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}
```

```java
	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
         // 如果要封装的对象本身就是 Advisor 类型的那么无需再做过多处理
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
         // 因为此封装方法只对 Advisor 与 Advice 两种类型的数据有效，如果不是将不能封装
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}
		Advice advice = (Advice) adviceObject;
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
             // 如果是 MethodInterceptor 类型则使用 DefaultPointcutAdvisor 封装
			return new DefaultPointcutAdvisor(advice);
		}
         // 如果存在 Advisor 的适配器那么也同样需要进行封装
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}
```

之前上文所说的适配器流程便在这里了。

由于 Spring 中涉及过多的拦截器、增强器、增强方法等方式来对逻辑进行增强，所以非常有必要统一封装成 Advisor 来进行代理的创建，完成了增强的封装过程，那么解析最重要的一步就是代理的创建与获取了。

`ProxyFactory#getProxy`

```java
	public Object getProxy() {
		return createAopProxy().getProxy();
	}
```

#### 创建代理

```java
	/**
	 * Subclasses should call this to get a new AOP proxy. They should <b>not</b>
	 * create an AOP proxy with {@code this} as an argument.
	 */
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}
```

```java
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
         // 见下文描述
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

从 if 语句可以看出有三个方面影响 Spring 选择 JDKProxy or CglibProxy：

-  optimize: 用来控制通过 CGLIB 创建的代理是否使用激进的优化策略。除非完全了解 AOP 代理如何处理优化，否则不推荐用户使用这个设置。目前这个属性仅用于 CGLIB 代理，对于 JDK 动态代理（缺省代理）无效。
-  proxyTargetClass：这个属性为 true 时，目标类本身被代理而不是目标类的接口。如果这个属性值被设为 true，CGLIB 代理将被创建，设置方式： `<aop:aspectj-autoproxy proxy-target-class="true">`
-  hasNoUserSuppliedProxyInterfaces: 是否存在代理接口

##### 对 JDK 与 Cglib 方式的总结

-  如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理实现 AOP
-  如果目标对象实现了接口，可以强制使用 CGLIB 实现 AOP
-  如果目标对象没有实现接口，必须采用 CGLIB 库，Spring 会自动在 JDK 动态代理和 CGLIB 之间转换

###### 如何强制使用 CGLIB 实现 AOP

1. 添加 CGLIB 库
2. 在 Spring 配置文件中加入 `<aop:aspectj-autoproxy proxy-target-class="true">`

###### JDK 动态代理和 CGLIB 字节码生成的区别

-  JDK 动态代理只能对实现了接口的类生成代理，而不能针对类
-  CGLIB 是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，因为是继承，所以该类或方法最好不要声明成 final

#### 获取代理

确定了使用哪种代理方式后便可以进行代理的创建了，但是创建之前我们有必要回顾一下两种方式的使用方法。这里可以参考我之前的[AOP的道理](https://binglau7.github.io/2017/06/11/AOP%E7%9A%84%E9%81%93%E7%90%86/)。

##### JDK 代理分析

继续之前的跟踪，可达 `JdkDynamicAopProxy#getProxy`

```java
	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

之前文章中分析过 JDK 动态代理的关键是创建自定义的 `InvocationHandler`，而 `InvocationHandler` 中包含了需要覆盖的函数 `getProxy`，而当前方法正是完成了这个操作。

既然`JdkDynamicAopProxy` 实现了 `InvocationHandler` 那它肯定有 `invoke` 函数

```java
	/**
	 * Implementation of {@code InvocationHandler.invoke}.
	 * <p>Callers will see exactly the exception thrown by the target,
	 * unless a hook method throws an exception.
	 */
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
             // equals 方法的处理
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
             // hash 方法的处理
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
             // isAssignableFrom(Class cls) 方法如果调用这个方法的 class 或者接口与其参数
             // 表示的类或接口相同或者是参数 cls 表示的类或接口的父类，则返回 true
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

             // 有时候目标对象内部的自我调用将无法实施切面中的增强则需要通过此属性暴露代理
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// Get the interception chain for this method.
             // 获取当前方法的拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                  // 如果拦截器链为空直接调用切点方法
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// We need to create a method invocation...
                  // 将拦截器封装在 ReflectiveMethodInvocation，以便于使用其 proceed 
                  // 进行链接表用拦截器
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
                  // 执行拦截器链
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
             // 返回结果
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
```

上面的函数中最主要的工作就是创建了一个拦截器链，并使用 `ReflectiveMethodInvocation` 类进行了链的封装，而在 `ReflectiveMethodInvocation` 类的 `proceed` 方法中实现了拦截器的逐一调用，那么我们继续来探究，在 `proceed` 方法中是怎么实现前置增强的目标方法前调用后置增强在目标方法后调用的逻辑呢？

```java
	@Override
	public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
         // 执行完所有增强后执行切点方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

         // 获取下一个要执行的拦截器
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
             // 动态匹配
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
                  // 不匹配则不执行拦截器
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
             /**
             * 普通拦截器，直接调用拦截器，比如：
             * ExposeInvocationInterceptor
             * DelegatePerTargetObjectIntroductionInterceptor,
             * MethodBeforeAdviceInterceptor
             * AspectJAroundAdvice
             * AspectJAfterAdvice
             **/
             // 将 this 作为参数传递以保证当前实例中调用链的执行
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

`ReflectiveMethodInvocation` 中的主要职责是维护了链接调用的计数器，记录着当前调用链接的位置，以便链可以有序地进行下去，那么在这个方法中并没有我们之前设想的维护各种增强的顺序，而是将此工作委托给了各个增强器，使各个增强器在内部进行逻辑实现。

##### CGLIB 代理分析

我们参考上述知道其入口也是在 getProxy 方法中

```java
	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
		}

		try {
			Class<?> rootClass = this.advised.getTargetClass();
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
			if (ClassUtils.isCglibProxyClass(rootClass)) {
				proxySuperClass = rootClass.getSuperclass();
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);
				}
			}

			// Validate the class, writing log messages as necessary.
             // 验证 class
			validateClassIfNecessary(proxySuperClass, classLoader);

			// Configure CGLIB Enhancer...
             // 配置及设置 Enhancer
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

             // 设置拦截器
			Callback[] callbacks = getCallbacks(rootClass);
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// Generate the proxy class and create a proxy instance.
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Throwable ex) {
			// TargetSource.getTarget() failed
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}

     // 生成代理类以及创建代理
	protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
		enhancer.setInterceptDuringConstruction(false);
		enhancer.setCallbacks(callbacks);
		return (this.constructorArgs != null ?
				enhancer.create(this.constructorArgTypes, this.constructorArgs) :
				enhancer.create());
	}
```

以上函数完整地阐述了一个创建 Spring 中的 Enhancer 的过程，这里最重要的是通过 getCallbacks 方法设置拦截器链

```java
	private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		// Parameters used for optimization choices...
		boolean exposeProxy = this.advised.isExposeProxy();
		boolean isFrozen = this.advised.isFrozen();
		boolean isStatic = this.advised.getTargetSource().isStatic();

		// Choose an "aop" interceptor (used for AOP calls).
         // 将拦截器封装在 DynamicAdvisedInterceptor
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

		// Choose a "straight to target" interceptor. (used for calls that are
		// unadvised but can return this). May be required to expose the proxy.
		Callback targetInterceptor;
		if (exposeProxy) {
			targetInterceptor = isStatic ?
					new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource());
		}
		else {
			targetInterceptor = isStatic ?
					new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedInterceptor(this.advised.getTargetSource());
		}

		// Choose a "direct to target" dispatcher (used for
		// unadvised calls to static targets that cannot return this).
		Callback targetDispatcher = isStatic ?
				new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp();

		Callback[] mainCallbacks = new Callback[] {
                  // 将拦截器链加入 Callback 中
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		Callback[] callbacks;

		// If the target is a static one and the advice chain is frozen,
		// then we can make some optimizations by sending the AOP calls
		// direct to the target using the fixed chain for that method.
		if (isStatic && isFrozen) {
			Method[] methods = rootClass.getMethods();
			Callback[] fixedCallbacks = new Callback[methods.length];
			this.fixedInterceptorMap = new HashMap<String, Integer>(methods.length);

			// TODO: small memory optimization here (can skip creation for methods with no advice)
			for (int x = 0; x < methods.length; x++) {
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(methods[x], rootClass);
				fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
						chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
				this.fixedInterceptorMap.put(methods[x].toString(), x);
			}

			// Now copy both the callbacks from mainCallbacks
			// and fixedCallbacks into the callbacks array.
			callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
			System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
			System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
			this.fixedInterceptorOffset = mainCallbacks.length;
		}
		else {
			callbacks = mainCallbacks;
		}
		return callbacks;
	}
```

在 getCallback 中 Spring 考虑了很多情况，但是对于我们来说，只需要理解最常用的就可以了，比如将 advised 属性封装在 `DynamicAdvisedInterceptor` 并加入在 callbacks 中，这么做的目的是什么呢？然后调用呢?

就我们所知，CGLIB 对于方法的拦截是通过将自定义的拦截器（实现 `MethodInterceptor` 接口）加入 Callback 中并在调用代理时直接激活拦截器中的 intercept 方法来实现的，那么在 getCallback 中正式实现了这样一个目的，`DynamicAdvisedInterceptor` 继承自 `MethodInterceptor` ，加入 Callback 中后，在再次调用代理时会直接调用 `DynamicAdvisedInterceptor` 中的 intercept 方法，由此推断，对于 CGLIB 方式实现的代理，其核心逻辑必然在 `DynamicAdvisedInterceptor` 中的 `intercept` 中

```java
		@Override
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Class<?> targetClass = null;
			Object target = null;
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// May be null. Get as late as possible to minimize the time we
				// "own" the target, in case it comes from a pool...
				target = getTarget();
				if (target != null) {
					targetClass = target.getClass();
				}
                  // 获取拦截器链
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
                      // 如果拦截器链为空则直接激活原方法
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
                      // 进入链
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null) {
					releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```

上述实现与 JDK 方式实现代理中的 invoke 方法大同小异，首先是构造链，然后封装此链进行串联调用，稍有区别的是 JDK 中直接构建 `ReflectiveMethodInvocation` ，而再 cglib 中使用 `CglibMethodInvocation` 。`CglibMethodInvocation` 继承自 `ReflectiveMethodInvocation`，但是 `proceed` 方法并没有重写。
