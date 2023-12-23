---
title: Spring 源码解析-容器的功能扩展-后续
date: 2017-11-26 01:19:40
tags:
    - Spring
    - Java
categories: 源码分析
---

本文承接自： [Spring-源码解析-容器的功能扩展-BeanFactory功能扩展](https://binglau7.github.io/2017/11/25/Spring-%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E5%AE%B9%E5%99%A8%E7%9A%84%E5%8A%9F%E8%83%BD%E6%89%A9%E5%B1%95-BeanFactory%E5%8A%9F%E8%83%BD%E6%89%A9%E5%B1%95/)

依照前文继续分析: `AbstractApplicationContext#refresh()`

## BeanFactory 的后处理

BeanFacotry 作为 Spring 中容器功能的基础，用于存放所有已经加载的 bean，为了保证程序上的高可扩展性，Spring 针对 BeanFactory 做了大量的扩展，比如我们熟知的 PostProcessor 等都是在这里实现的。

<!-- more -->

### 激活注册的 BeanFactoryPostProcessor

`BeanFactoryPostProcessor` 接口跟 `BeanPostProcessor` 类似，可以对 bean 的定义（配置元数据）进行处理。也就是说，Spring IoC 容器允许 `BeanFactoryPostProcessor` 在容器实际实例化任何其他的 bean 之前读取配置元数据，并有可能修改它。如果你愿意，你可以配置多个 `BeanFactoryPostProcessor`。你还能通过设置 order 属性来控制 `BeanFactoryPostProcessor` 的执行次序（仅当 `BeanFactoryPostProcessor` 实现了`Ordered` 接口时你才可以设置此属性，因此在实现 `BeanFactoryPostProcessor` 时，就应当考虑实现 `Ordered` 接口）。

如果你想改变实际的 bean 实例（例如从配置元数据创建的对象），那么你最好使用 `BeanPostProcessor`。同样地，`BeanFactoryPostProcessor` 的作用域范围是容器级的。它只和你所使用的容器有关。如果你在容器中定义一个 `BeanFactoryPostProcessor`，它仅仅对此容器中的 bean 进行后置处理。`BeanFactoryPostProcessor` 不会对定义在另一个容器中的 bean 进行后置处理，即使这两个容器都是在同一层次上。

#### BeanFactoryPostProcessor 的典型应用：PropertyPlaceholderConfigurer

指定配置文件使用。

`PropertyPlaceholderConfigurer` 这个类间接继承了 `BeanFactoryPostProcessor` 接口。这是一个很特别的接口，当 Spring 加载任何实现了这个接口的 bean 的配置时，都会在 bean 工厂载入所有 bean 的配置之后执行 `postProcessBeanFactory` 方法。在 `PropertyResourceConfigurer` 类中实现了 `postProcessBeanFactory` 方法，在方法中先后调用了`mergeProperties`、`convertProperties`、`processProperties` 这3个方法，分别得到配置，将得到的配置，将得到的配置转换为合适的类型，最后将配置内容告知 `BeanFactory`。

### 激活 BeanFactoryPostProcessor

```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

```java
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<String>();
         // 对 BeanDefinitionRegistry 类型的处理
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessors =
					new LinkedList<BeanDefinitionRegistryPostProcessor>();
			
          	 // 硬编码注册的后处理器
             // 这里有个疑问，书上应该是 Spring 3 版本应该是在这里调用了 getBeanFactoryPostProcessors() 
             // 而在当前 4.3.8 版本时候是在外部调用，而且整体函数被抽离
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryPostProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
                      // 对于 BeanDefinitionRegistryPostPorcessor 类型，在 BeanFactoryPostProcessor
                      // 的基础 上还有自己定义的方法，需要先调用
					registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
					registryPostProcessors.add(registryPostProcessor);
				}
				else {
                      // 记录常规 BeanFactoryPostProcessor
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
             // 1. 这里是之前篇章提过的在之前功能扩展中注册 ApplicationListenerDetector 的意义所在
             // 2. 对于配置中读取的 BeanFactoryPostProcessor 的处理
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
             // 首先处理实现 PriorityOrdered 的 BeanDefinitionRegistryPostProcessors
			List<BeanDefinitionRegistryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
             // 优先级排序
			sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
             // 加入 registryPostProcessors
			registryPostProcessors.addAll(priorityOrderedPostProcessors);
          	 // 激活 BeanDefinitionRegistryPostProcessors 优先级部分
			invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			List<BeanDefinitionRegistryPostProcessor> orderedPostProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					orderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(beanFactory, orderedPostProcessors);
			registryPostProcessors.addAll(orderedPostProcessors);
			invokeBeanDefinitionRegistryPostProcessors(orderedPostProcessors, registry);

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
             // 添加其他类型的 BeanDefinitionRegistryPostProcessors
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);
						registryPostProcessors.add(pp);
						processedBeans.add(ppName);
						pp.postProcessBeanDefinitionRegistry(registry);
						reiterate = true;
					}
				}
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
             // 激活 postProcessBeanFactory 方法，之前激活的是 BeanDefinitionRegistryPostProcessors
			invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
             // 常规 BeanFactoryPostProcessor
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
         // 对于配置读取的 BeanFactoryPostProcessor 的处理
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
         // 对后处理器进行分类
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
             // 已经处理过的，处理 BeanDefinitionRegistery 的时候(应该叫已经添加到处理序列中)
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
         // 按优先级进行排序
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
         // 按 order 进排序
		sortPostProcessors(beanFactory, orderedPostProcessors);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
         // 无序，直接调用
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```

对于 `BeanFactoryPostProcessor` 的处理主要分两种情况进行，一个是对于 `BeanDefinitionRegistry` 类的特殊处理，另一种是对普通的 `BeanFactoryPostProcessor` 进行处理。而对于**每种情况都需要考虑硬编码注入注册的后处理器以及通过配置注入的后处理器**。

对于 `BeanDefinitionRegistry` 类型的处理类的处理主要包括以下内容。

1. 对于硬编码注册的后处理器的处理，主要是通过 `AbstractApplicationContext` 中的添加处理器方法`addBeanFactoryPostProcessor` 进行添加。

   添加后的处理器会存放入 `beanFactoryPostProcessors` 中，而在处理 `BeanFactoryPostProcessor` 时候会首先检测 `beanFactoryPostProcessor` 是否有数据。当然，`BeanDefinitionRegistryPostProcessor` 继承自 `BeanFactoryPostProcessor`，不但有 `BeanFactoryPostProcessor` 的特性，同事还有自己定义的个性化方法，也需要在此调用。所以，这里需要从 `beanFactoryPostProcessors` 中挑出 `BeanDefinitionRegistryPostProcessor` 的后处理器，并进行其 `postProcessBeanDefinitionRegistry` 方法的激活。

2. 记录后处理器主要使用了三个 List 完成

   -  `registryPostProcessors`：记录通过硬编码方式注册的 `BeanDefinitionRegistryPostProcessor` 类型的处理器。
   -  `regularPostProcessors`：记录通过硬编码方式注册的 `BeanFactoryPostProcessor` 类型的处理器。
   -  `registryPostProcessorBeans`：记录通过配置方式注册的 `BeanDefinitionRegistryPostProcessor` 类型的处理器。

3. 对以上所记录的 List 中的后处理器进行统一调用 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法。

4. 对 `beanFactoryPostProcessors` 中非 `BeanDefinitionRegistryPostProcessor` 类型的后处理器进行统一的`BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法调用。

5. 普通 beanFactory 处理。

### 注册 BeanPostProcessor

真正的调用是在 bean 的实例化阶段进行的。

```java
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```

```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
         /**
         * BeanPostProcessorChecker 是一个普通的信息打印，可能会有些情况，
         * 当 Spring 的配置中的后处理器还没有被注册就已经开始了 bean 的初始化时
         * 便会打印出 BeanPostProcessorChecker 中设定的信息
         **/
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
         // 使用 PriorityOrdered 接口保证顺序
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
         // MergedBeanDefinitionPostProcessor
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
         // 使用 Ordered 接口保证顺序
		List<String> orderedPostProcessorNames = new ArrayList<String>();
         // 无序 BeanPostProcessor
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
         // 注册所有实现 PriorityOrdered 的 BeanPostProcessors
		sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
         // 注册所有实现 Ordered 的 BeanPostProcessors
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(beanFactory, orderedPostProcessors);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
         // 注册所有无序的 BeanPostProcessors
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
         // 注册所有 MergedBeanDefinitionPostProcessor 类型的 BeanPostProcessor, 并非重复注册
         // 在 beanFactory.addBeanPostProcessor 中会先移除以及存在的 BeanPostProcessor
		sortPostProcessors(beanFactory, internalPostProcessors);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
         // 添加 ApplicationListener 探测器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

对比 `BeanFactoryPostProcessor` 的处理要区分两种情况，一种方式是通过硬编码方式的处理，另一种是通过配置文件方式的处理。

对于 `BeanFactoryPostProcessor` 的处理，不但要实现注册功能，而且**还要实现对后处理器的激活操作**，所以需要载入配置中的定义，并进行激活；而对于 `BeanPostProcessor` 并**不需要马上调用**，再说，硬编码的方式实现的功能是将后处理器提取并调用，这里并不需要调用，当然不需要考虑硬编码的方式了，这里的功能只需要将配置文件的 `BeanPostProcessor` 提取出来并注册进入 beanFactory 就可以了。

### 初始化 ApplicationEventMulticaster

#### 具体使用

1. 定义事件

   ```java
   public class TestEvent extends ApplicationEvent {
       public String msg;

       public TestEvent(Object source) {
           super(source);
       }

       public TestEvent(Object source, String msg) {
           super(source);
           this.msg = msg;
       }

       public void print() {
           System.out.println(msg);
       }
   }
   ```

2. 定义监听器

   ```java
   public class TestListener implements ApplicationListener {
       @Override
       public void onApplicationEvent(ApplicationEvent event) {
           if (event instanceof TestEvent) {
               TestEvent testEvent = (TestEvent) event;
               testEvent.print();
           }
       }
   }
   ```

   ```xml
   <bean id="testListener" class="io.github.binglau.TestListener" />
   ```

3. 测试

   ```java
       @Test
       public void testEvent() {
           ApplicationContext ctx = new ClassPathXmlApplicationContext("beanFactory.xml");
           TestEvent event = new TestEvent("hello", "msg");
           ctx.publishEvent(event);
       }
   ```

#### 具体实现

`initApplicationEventMulticaster` 的方式比较简单，无非考虑两种情况。

-  如果用户自定义了事件广播器，那么使用用户自定义的事件广播器。
-  如果用户没有自定义事件广播器，那么使用默认的 `ApplicationEventMulticaster`。

```java
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
						APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
						"': using default [" + this.applicationEventMulticaster + "]");
			}
		}
	}
```

由此推断其中默认实现的广播器 `SimpleApplicationEventMulticaster` 中有逻辑来存储监听器并在合适的时候调用。

```java
	public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

可以推断，当产生 Spring 事件的时候会默认使用 `SimpleApplicationEventMulticaster` 的 `multicastEvent` 来广播事件，遍历所有监听器，并使用监听器中的 `onApplicationEvent` 方法来进行监听器的处理。而对于每个监听器来说其实都可以获取到产生的事件，但是是否进行处理则由事件监听器来决定。

### 注册监听器

```java
	protected void registerListeners() {
		// Register statically specified listeners first.
         // 硬编码方式注册的监听器处理
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
         // 配置文件注册的监听器处理
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
         // 提前发布一些消息告诉我们已经有了一个广播器
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```

## 初始化非延迟加载单例

完成 BeanFactory 的初始化工作，其中包括 ConversionService 的设置、配置冻结以及非延迟加载的 bean 的初始化工作。

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					return getEnvironment().resolvePlaceholders(strVal);
				}
			});
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
         // 冻结所有的 bean 定义，说明注册的 bean 定义将不被修改或任何进一步的处理
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
         // 初始化剩下的单实例(非惰性)
		beanFactory.preInstantiateSingletons();
	}
```

### ConversionService 的设置

之前我们提到过使用自定义类型转换器，那么，在Spring中还提供了另一种转换方式：使用 `Converter`。

#### 具体使用

1. 定义转换器

   ```java
   public class String2DateConverter implements Converter<String, LocalDate> {
       static DateTimeFormatter formatter;

       static {
           formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
       }

       @Override
       public LocalDate convert(String source) {
           try {
               return LocalDate.parse(source, formatter);
           } catch (Exception e) {
               return null;
           }
       }
   }
   ```

2. 注册

   ```java
       <bean id="conversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
           <property name="converters">
               <list>
                   <bean class="io.github.binglau.String2DateConverter" />
               </list>
           </property>
       </bean>
   ```

3. 测试（方便起见直接调用测试）

   ```java
       @Test
       public void testConverter() {
           DefaultConversionService conversionService = new DefaultConversionService();
           conversionService.addConverter(new String2DateConverter());

           LocalDate date = conversionService.convert("2017-09-22", LocalDate.class);
           System.out.println(date);
       }
   ```

### 冻结配置

```java
	@Override
	public void freezeConfiguration() {
		this.configurationFrozen = true;
		this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
	}
```

### 初始化非延迟加载

`ApplicationContext` 实现的默认行为就是在启动时将所有单例 bean 提前进行实例化。提前实例化意味着作为初始化过程的一部分，`ApplicationContext` 实例会创建并配置所有的单例 bean。通常情况下这是一件好事，因为这样在配置中的任何错误就会即刻被发现（否则的话可能要花几个小时甚至几天）。而这个实例化的过程就是在 `finishBeanFactoryInitialization` 中完成的。

```java
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

## finishRefresh

在 Spring 中还提供了 `Lifecycle` 接口，`Lifecycle` 中包含 `start/stop` 方法，实现此接口后 Spring 会保证在启动的时候调用其 start 方法开始生命周期，并在 Spring 关闭的时候调用 stop 方法来结束生命周期，通常用来配置后台程序，在启动后一直运行（如对 MQ 进行轮询等）。而 `ApplicationContext` 的初始化最后正是保证了这一功能的实现。

```java
	protected void finishRefresh() {
		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

### initLifecycleProcessor

当 ApplicationContext 启动或停止时，它会通过 `LifecycleProcessor` 来与所有声明的 bean 的周期做状态检查，而 `LifecycleProcessor` 的使用前首先需要初始化。

```java
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate LifecycleProcessor with name '" +
						LIFECYCLE_PROCESSOR_BEAN_NAME +
						"': using default [" + this.lifecycleProcessor + "]");
			}
		}
	}
```

### onRefresh

启动所有实现了 `Lifecycle` 接口的 bean

```java
	@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}
```

```java
	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<Integer, LifecycleGroup>();
		for (Map.Entry<String, ? extends Lifecycle> entry : lifecycleBeans.entrySet()) {
			Lifecycle bean = entry.getValue();
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(entry.getKey(), bean);
			}
		}
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<Integer>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```

### publishEvent

当晚餐 ApplicationContext 初始化的时候，要通过 Spring 中的事件发布机制来发出 ContextRefreshedEvent 事件，以保证对应的监听器可以做进一步的逻辑处理。

```java
	protected void publishEvent(Object event, ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<Object>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

### registerApplicationContext

注册 LiveBeansView MBean（如果处于活动状态）

关于 LiveBeansView：

>  Adapter for live beans view exposure, building a snapshot of current beans and their dependencies from either a local `ApplicationContext` (with a local `LiveBeansView` bean definition) or all registered ApplicationContexts (driven by the ["spring.liveBeansView.mbeanDomain"](https://docs.spring.io/autorepo/docs/spring/4.2.1.RELEASE/javadoc-api/org/springframework/context/support/LiveBeansView.html#MBEAN_DOMAIN_PROPERTY_NAME) environment property).

```java
	static void registerApplicationContext(ConfigurableApplicationContext applicationContext) {
		String mbeanDomain = applicationContext.getEnvironment().getProperty(MBEAN_DOMAIN_PROPERTY_NAME);
		if (mbeanDomain != null) {
			synchronized (applicationContexts) {
				if (applicationContexts.isEmpty()) {
					try {
						MBeanServer server = ManagementFactory.getPlatformMBeanServer();
						applicationName = applicationContext.getApplicationName();
						server.registerMBean(new LiveBeansView(),
								new ObjectName(mbeanDomain, MBEAN_APPLICATION_KEY, applicationName));
					}
					catch (Throwable ex) {
						throw new ApplicationContextException("Failed to register LiveBeansView MBean", ex);
					}
				}
				applicationContexts.add(applicationContext);
			}
		}
	}
```
