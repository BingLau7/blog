---
title: Spring 源码解析—创建 bean
date: 2017-11-20 17:00:18
tags:
    - Spring
    - Java
categories: 源码分析
---

在经历过 `AbstractAutowireCapableBeanFactory#createBean` 中的 `resolveBeforeInstantiation` 方法后，程序有两个选择，如果创建了代理或者说重写了 `InstantiationAwareBeanPostProcessor` 的 `postProcessBeforeInstantiation` 方法并在方法 `postProcessBeforeInstantiation` 中改变了 `bean`，则直接返回就可以了，否则需要进行常规 `bean` 的创建。而这常规 `bean` 的创建就是在 `doCreateBean` 中完成的。

`AbstractAutowireCapableBeanFactory#doCreateBean`

<!-- more -->

```Java
/**
 * Actually create the specified bean. Pre-creation processing has already happened
 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
 * <p>Differentiates between default bean instantiation, use of a
 * factory method, and autowiring a constructor.
 * @param beanName the name of the bean
 * @param mbd the merged bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a new instance of the bean
 * @throws BeanCreationException if the bean could not be created
 * @see #instantiateBean
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 */
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) { // 1
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) { // 2
      	 // 根据指定 bean 使用对应的策略创建新的实例，如：工厂方法、构造函数自动注入、简单初始化
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
	mbd.resolvedTargetType = beanType;

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
              	  // 应用 MergedBeanDefinitionPostProcessor
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName); // 3
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}

	// Eagerly cache singletons to be able to resolve circular references
	// even when triggered by lifecycle interfaces like BeanFactoryAware.
  	// 4
  	// 是否需要提前曝光：单例 & 允许循环依赖 & 当前 bean 正在创建中，检测循环依赖
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
      	// 为了避免后期循环依赖，可以在 bean 初始化完成前将创建实例的 ObjectFactory 加入工厂
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
              	// 对 bean 再一次依赖引用，主要应用 SmartInstantiationAware BeanPostProcessor,
              	// 其中我们熟知的 AOP 就是在这里将 advice 动态织入 bean 中，若没有直接返回 bean，不做任何处理
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
      	// 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性，则会递归初始依赖 bean
       // 5
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
          	// 调用初始化方法，比如 init-method
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
      	// earlySingletonReference 只有在检测到有循环依赖的情况下才会不为空
      	// 6
		if (earlySingletonReference != null) {
          	// 如果 esposedObject 没有在初始化方法中被改变，就是没有被增强
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
                  	  // 检测依赖
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
              	// 因为 bean 创建后其所依赖的 bean 一定是已经创建的，
              	// actualDependentBeans 不为空则表示当前 bean 创建后其依赖的 bean 却没有
              	// 全部创建完，也就是说存在循环依赖
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
      	// 根据 scopes 注册 bean
		registerDisposableBeanIfNecessary(beanName, bean, mbd); // 7
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}
  
	return exposedObject; // 8
}
```

1. 如果是单例则需要首先清除缓存

2. 实例化 bean，将 `BeanDefinition` 转换为 `BeanWrapper`

   转换是一个复杂的过程，但是我们可以尝试概括大致的功能：

   -  如果存在工厂方法则使用工厂方法进行初始化
   -  一个类有多个构造函数，每个构造函数都有不同的参数，所以需要根据参数锁定构造函数并进行初始化
   -  如果既不存在工厂方法也不存在带有参数的构造函数，则使用默认的构造函数进行 bean 的实例化

3. MergedBeanDefinitionPostProcessor的应用

   bean合并后的处理，Autowired注解正是通过此方法实现诸如类型的预解析。

4. 依赖处理

   在Spring中会有循环依赖的情况，例如，当A中含有B的属性，而B中又含有A的属性时就会构成一个循环依赖，此时如果A和B都是单例，那么在Spring中的处理方式就是当创建B的时候，涉及自动注入A的步骤时，并不是直接去再次创建A，而是**通过放入缓存中的ObjectFactory来创建实例**，这样就解决了循环依赖的问题。

5. 属性填充。将所有属性填充至bean的实例中

6. 循环依赖检查

   Sping 中解决循环依赖只对单例有效，而对于 prototype 的 bean，Spring 没有好的解决办法，唯一要做的就是抛出异常。在这个步骤里面会检测已经加载的 bean 是否已经出现了依赖循环，并判断是否需要抛出异常。

7. 注册DisposableBean

   如果配置了destroy-method，这里需要注册以便于在销毁时候调用。

8. 完成创建并返回

## 创建 bean 的实例 (createBeanInstance)

```Java
/**
 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
 * factory method, constructor autowiring, or simple instantiation.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return BeanWrapper for the new instance
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 * @see #instantiateBean
 */
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
	// Make sure bean class is actually resolved at this point.
  	// 解析 class
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

  	// 如果工厂方法不为空则使用工厂方法初始化策略
	if (mbd.getFactoryMethodName() != null)  { // 1
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
  
	// 2
	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
          	// 一个类有多个构造函数，每个构造函数都有不同的参数，所以调用前需要先根据参数锁定构造函数或对应的工厂方法
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
  	// 如果已经解析过则使用解析好的构造函数方法不需要再次锁定
	if (resolved) {
		if (autowireNecessary) {
          	// 构造函数自动注入
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
          	// 使用默认构造函数构造
			return instantiateBean(beanName, mbd);
		}
	}

	// Need to determine the constructor...
  	// 需要根据参数解析构造函数
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
      	 // 构造函数自动注入
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// No special handling: simply use no-arg constructor.
  	// 使用默认构造函数构造
	return instantiateBean(beanName, mbd);
}
```

1. 如果在`RootBeanDefinition`中存在`factoryMethodName`属性，或者说在配置文件中配置了`factory-method`，那么 Spring 会尝试使用`instantiateUsingFactoryMethod(beanName, mbd, args)`方法根据`RootBeanDefinition`中的配置生成bean的实例。
2. 解析构造函数并进行构造函数的实例化。因为一个 bean 对应的类中可能会有多个构造函数，而每个构造函数的参数不同，Spring 在根据参数及类型去判断最终会使用哪个构造函数进行实例化。但是，**判断的过程是个比较消耗性能的步骤，所以采用缓存机制**，如果已经解析过则不需要重复解析而是直接从`RootBeanDefinition`中的属性`resolvedConstructorOrFactoryMethod`缓存的值去取，否则需要再次解析，并将解析的结果添加至 `RootBeanDefinition` 中的属性`resolvedConstructorOrFactoryMethod`中。

### `autowireConstructor`

对于实例的创建Spring中分成了两种情况，一种是通用的实例化，另一种是带有参数的实例化。带有参数的实例化过程相当复杂，因为存在着不确定性，所以在判断对应参数上做了大量工作。

```Java
/**
 * "autowire constructor" (with constructor arguments by type) behavior.
 * Also applied if explicit constructor argument values are specified,
 * matching all remaining arguments with beans from the bean factory.
 * <p>This corresponds to constructor injection: In this mode, a Spring
 * bean factory is able to host components that expect constructor-based
 * dependency resolution.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @param ctors the chosen candidate constructors
 * @param explicitArgs argument values passed in programmatically via the getBean method,
 * or {@code null} if none (-> use constructor argument values from bean definition)
 * @return BeanWrapper for the new instance
 */	
 protected BeanWrapper autowireConstructor(
		String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs) {

	return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

这边的代码量太大了，不太适合贴出来解析，各位最好还是去 idea 里面进行查看，书中作者也觉得这段不符合 Spring 那种『将复杂的逻辑分解，分成N个小函数的嵌套，每一层都是对下一层逻辑的总结及概要，这样使得每一层的逻辑会变得简单容易理解』的规律。我这里就说明一下整体流程，每段流程贴出对应的代码。

#### 构造函数参数的确定

-  根据explicitArgs参数判断

   如果传入的参数`explicitArgs`不为空，那边可以直接确定参数，因为 `explicitArgs` 参数是在调用 Bean 的时候用户指定的，在`BeanFactory`类中存在这样的方法：
   `Object getBean(String name, Object... args) throws BeansException;`
   在获取 bean 的时候，用户不但可以指定 bean 的名称还可以指定 bean 所对应类的构造函数或者工厂方法的方法参数，主要用于静态工厂方法的调用，而这里是需要给定完全匹配的参数的，所以，便可以判断，如果传入参数`explicitArgs`不为空，则可以确定构造函数参数就是它。

   ```Java
   // explicitArgs 通过 getBean 方法传入
   // 如果 getBean 方法调用的时候指定方法参数那么直接使用
   if (explicitArgs != null) {
   	argsToUse = explicitArgs
   }
   ```

-  缓存中获取

   构造函数参数已经记录在缓存中，那么便可以直接拿来使用。而且，这里要提到的是，在缓存中缓存的可能是参数的最终类型也可能是参数的初始类型，例如：构造函数参数要求的是 int 类型，但是原始的参数值可能是String类型的“1”，那么即使在缓存中得到了参数，也需要经过类型转换器的过滤以确保参数类型与对应的构造函数参数类型完全对应。

   ```Java
   else {
        	 // 如果在 getBean 方法时候没有指定则尝试从配置文件中解析
   	Object[] argsToResolve = null;
        	 // 尝试从缓存中获取
   	synchronized (mbd.constructorArgumentLock) {
   		constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
   		if (constructorToUse != null && mbd.constructorArgumentsResolved) {
   			// Found a cached constructor...
                	  // 从缓存中取
   			argsToUse = mbd.resolvedConstructorArguments;
   			if (argsToUse == null) {
                    	  // 配置的构造函数参数 
   				argsToResolve = mbd.preparedConstructorArguments;
   			}
   		}
   	}
        	 // 如果缓存中存在
   	if (argsToResolve != null) {
            	 // 解析参数类型，如给定方法的构造函数 A(int, int) 则通过此方法后就会把配置汇中的 ("1", "1") 转换为 (1, 1)
            	 // 缓存中的值可能是原始值也可能是最终值
   		argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
   	}
   }
   ```

-  配置文件获取

   如果不能根据传入的参数 `explicitArgs` 确定构造函数的参数也无法在缓存中得到相关信息，那么只能开始新一轮的分析了。
   分析从获取配置文件中配置的构造函数信息开始，经过之前的分析，我们知道，Spring中配置文件中的信息经过转换都会通过 `BeanDefinition` 实例承载，也就是参数`mbd`中包含，那么可以通过调用 `mbd.getConstructorArgumentValues()`来获取配置的构造函数信息。有了配置中的信息便可以获取对应的参数值信息了，获取参数值的信息包括直接指定值，如：直接指定构造函数中某个值为原始类型 String 类型，或者是一个对其他 bean 的引用，而这一处理委托给`resolveConstructorArguments`方法，并返回能解析到的参数的个数。

   ```Java
   // Need to resolve the constructor.
   boolean autowiring = (chosenCtors != null ||
   		mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
   ConstructorArgumentValues resolvedValues = null;

   int minNrOfArgs;
   if (explicitArgs != null) {
   	minNrOfArgs = explicitArgs.length;
   }
   else {
        	 // 提取配置文件中的配置的构造函数参数
   	ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
        	 // 用于承载解析后的构造函数参数的值
   	resolvedValues = new ConstructorArgumentValues();
        	 // 能解析到的参数个数
   	minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
   }
   ```


#### 构造函数的确认

经过了第一步后已经确定了构造函数的参数，接下来的任务就是根据构造函数参数在所有构造函数中锁定对应的构造函数，而匹配的方法就是根据参数个数匹配，所以在匹配之前需要先对构造函数按照public构造函数优先参数数量降序、非public构造函数参数数量降序。这样可以在遍历的情况下迅速判断排在后面的构造函数参数个数是否符合条件。

```Java
// Take specified constructors, if any.
Constructor<?>[] candidates = chosenCtors;
if (candidates == null) {
	Class<?> beanClass = mbd.getBeanClass();
	try {
		candidates = (mbd.isNonPublicAccessAllowed() ?
				beanClass.getDeclaredConstructors() : beanClass.getConstructors());
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Resolution of declared constructors on bean Class [" + beanClass.getName() +
				"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
	}
}
// 排序给定的构造函数，public 构造函数优先参数数量排序、非 public 构造函数参数数量降序
AutowireUtils.sortConstructors(candidates);
```

由于在配置文件中并不是唯一限制使用参数位置索引的方式去创建，同样还支持指定参数名称进行设定参数值的情况，如`<constructor-arg name="aa">`，那么这种情况就需要首先确定构造函数中的参数名称。

获取参数名称可以有两种方式

1. 通过注解的方式直接获取，

   ```Java
   String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
   ```

2. 使用Spring中提供的工具类 `ParameterNameDiscoverer` 来获取。构造函数、参数名称、参数类型、参数值都确定后就可以锁定构造函数以及转换对应的参数类型了。

   ```Java
   if (paramNames == null) {
   	ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
   	if (pnd != null) {
   		paramNames = pnd.getParameterNames(candidate);
   	}
   }
   ```

#### 根据确定的构造函数转换对应的参数类型

主要是使用Spring中提供的类型转换器或者用户提供的自定义类型转换器进行转换。

```Java
argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
								getUserDeclaredConstructor(candidate), autowiring);
```

#### 构造参数不确定性的验证

当然，有时候即使构造函数、参数名称、参数类型、参数值都确定后也不一定会直接锁定构造函数，不同构造函数的参数为父子关系，所以Spring在最后又做了一次验证。

```Java
// 探测是否有不确定性的构造函数存在，例如不同构造函数的参数为父子关系
int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
		argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
// Choose this constructor if it represents the closest match.
// 如果它代表着当前最接近的匹配则选择作为构造函数
if (typeDiffWeight < minTypeDiffWeight) {
	constructorToUse = candidate;
	argsHolderToUse = argsHolder;
	argsToUse = argsHolder.arguments;
	minTypeDiffWeight = typeDiffWeight;
	ambiguousConstructors = null;
}
else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
	if (ambiguousConstructors == null) {
		ambiguousConstructors = new LinkedHashSet<Constructor<?>>();
		ambiguousConstructors.add(constructorToUse);
	}
	ambiguousConstructors.add(candidate);
}
```

#### 根据实例化策略以及得到的构造函数及构造函数参数实例化Bean

```Java
if (System.getSecurityManager() != null) {
	final Constructor<?> ctorToUse = constructorToUse;
	final Object[] argumentsToUse = argsToUse;
	beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
		@Override
		public Object run() {
			return beanFactory.getInstantiationStrategy().instantiate(
					mbd, beanName, beanFactory, ctorToUse, argumentsToUse);
		}
	}, beanFactory.getAccessControlContext());
}
else {
	beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
			mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
}
```

后面章节进行描述。

### `instantiateBean` 不带参数的构造函数实例化过程

```Java
if (resolved) {
	if (autowireNecessary) {
		return autowireConstructor(beanName, mbd, null, null);
	}
	else {
		return instantiateBean(beanName, mbd);
	}
}
```

```Java
/**
 * Instantiate the given bean using its default constructor.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @return BeanWrapper for the new instance
 */
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
	try {
		Object beanInstance;
		final BeanFactory parent = this;
		if (System.getSecurityManager() != null) {
			beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					return getInstantiationStrategy().instantiate(mbd, beanName, parent);
				}
			}, getAccessControlContext());
		}
		else {
			beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
		}
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
	}
}
```

### 实例化策略

其实，经过前面的分析，我们已经得到了足以实例化的所有相关信息，完全可以使用最简单的反射方法直接反射来构造实例对象，但是Spring却并没有这么做。

`SimpleInstantiationStrategy#instantiate`

```Java
@Override
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
  	 // 如果有需要覆盖或者动态替换的方法则当然需要使用 cglib 进行动态代理，因为可以在
  	 // 创建代理的同时将方法将动态方法织入类中，但是如果没有需要动态改变的方法，为了
  	 // 方便直接反射就可以了
	if (bd.getMethodOverrides().isEmpty()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
							@Override
							public Constructor<?> run() throws Exception {
								return clazz.getDeclaredConstructor((Class[]) null);
							}
						});
					}
					else {
						constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// Must generate CGLIB subclass.
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```

`CglibSubclassingInstantiationStrategy#instantiate`

```Java
/**
 * An inner class created for historical reasons to avoid external CGLIB dependency
 * in Spring versions earlier than 3.2.
 */
private static class CglibSubclassCreator {

	private static final Class<?>[] CALLBACK_TYPES = new Class<?>[]
			{NoOp.class, LookupOverrideMethodInterceptor.class, ReplaceOverrideMethodInterceptor.class};

	private final RootBeanDefinition beanDefinition;

	private final BeanFactory owner;

	CglibSubclassCreator(RootBeanDefinition beanDefinition, BeanFactory owner) {
		this.beanDefinition = beanDefinition;
		this.owner = owner;
	}

	/**
	 * Create a new instance of a dynamically generated subclass implementing the
	 * required lookups.
	 * @param ctor constructor to use. If this is {@code null}, use the
	 * no-arg constructor (no parameterization, or Setter Injection)
	 * @param args arguments to use for the constructor.
	 * Ignored if the {@code ctor} parameter is {@code null}.
	 * @return new instance of the dynamically generated subclass
	 */
	public Object instantiate(Constructor<?> ctor, Object... args) {
		Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
		Object instance;
		if (ctor == null) {
			instance = BeanUtils.instantiateClass(subclass);
		}
		else {
			try {
				Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
				instance = enhancedSubclassConstructor.newInstance(args);
			}
			catch (Exception ex) {
				throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
						"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
			}
		}
		// SPR-10785: set callbacks directly on the instance instead of in the
		// enhanced class (via the Enhancer) in order to avoid memory leaks.
		Factory factory = (Factory) instance;
		factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
				new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
				new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
		return instance;
	}

	/**
	 * Create an enhanced subclass of the bean class for the provided bean
	 * definition, using CGLIB.
	 */
	private Class<?> createEnhancedSubclass(RootBeanDefinition beanDefinition) {
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(beanDefinition.getBeanClass());
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		if (this.owner instanceof ConfigurableBeanFactory) {
			ClassLoader cl = ((ConfigurableBeanFactory) this.owner).getBeanClassLoader();
			enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(cl));
		}
		enhancer.setCallbackFilter(new MethodOverrideCallbackFilter(beanDefinition));
		enhancer.setCallbackTypes(CALLBACK_TYPES);
		return enhancer.createClass();
	}
}
```

## 记录创建 bean 的 ObjectFactory

在 `doCreate` 函数中有这样一段代码：

```Java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
// 4
// 是否需要提前曝光：单例 & 允许循环依赖 & 当前 bean 正在创建中，检测循环依赖
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
		isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
	if (logger.isDebugEnabled()) {
		logger.debug("Eagerly caching bean '" + beanName +
				"' to allow for resolving potential circular references");
	}
  	// 为了避免后期循环依赖，可以在 bean 初始化完成前将创建实例的 ObjectFactory 加入工厂
	addSingletonFactory(beanName, new ObjectFactory<Object>() {
		@Override
		public Object getObject() throws BeansException {
          	// 对 bean 再一次依赖引用，主要应用 SmartInstantiationAware BeanPostProcessor,
          	// 其中我们熟知的 AOP 就是在这里将 advice 动态织入 bean 中，若没有直接返回 bean，不做任何处理
			return getEarlyBeanReference(beanName, mbd, bean);
		}
	});
}
```

-  `mbd.isSingleton`: 是否是单例


-  `this.allowCircularReferences`: 是否允许循环依赖，很抱歉，并没有找到在配置文件中如何配置，但是在 AbstractRefreshableApplicationContext 中提供了设置函数，可以通过硬编码的方式进行设置或者可以通过自定义命名空间进行配置，其中硬编码的方式代码如下: 

   ```Java
   ClassPathXmlApplicationContext bf = new ClassPathXmlApplicationContext ("aspectTest.xml");
   bf.setAllowBeanDefinitionOverriding(false);
   ```

-  `isSingletonCurrentlyInCreation(beanName)`: 该bean是否在创建中。在Spring中，会有个专门的属性默认为`DefaultSingletonBeanRegistry`的`singletonsCurrentlyInCreation`来记录bean的加载状态，在bean开始创建前会将beanName记录在属性中，在bean创建结束后会将 beanName 从属性中移除。

   记录状态的点，不同 scope 的位置不一样，以  `singleton` 为例，在singleton下记录属性的函数是在`DefaultSingletonBeanRegistry`类的 `public Object getSingleton(String beanName, ObjectFactory singletonFacotry)` 函数的 `beforeSingletonCreation(beanName)` 和 `afterSingletonCreation(beanName)` 中，在这两段函数中分别是 `this.singletonsCurrentlyInCreation.add(beanName)` 与 `this.singletonsCurrentlyInCreation.remove(beanName)` 来进行状态的记录与移除。

## 属性注入

```Java
// Initialize the bean instance.
Object exposedObject = bean;
try {
	populateBean(beanName, mbd, instanceWrapper);
	if (exposedObject != null) {
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
}
```

```Java
/**
 * Populate the bean instance in the given BeanWrapper with the property values
 * from the bean definition.
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @param bw BeanWrapper with bean instance
 */
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
	PropertyValues pvs = mbd.getPropertyValues();

	if (bw == null) {
		if (!pvs.isEmpty()) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
		}
		else {
			// Skip property population phase for null instance.
          	 // 没有可填充的属性
			return;
		}
	}

	// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
	// state of the bean before properties are set. This can be used, for example,
	// to support styles of field injection.
     // 给 InstantiationAwareBeanPostprocessors 最优一次机会在属性设置前来改变 bean
     // 如：可以用来支持属性注入的类型
	boolean continueWithPropertyPopulation = true;

	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) { // 1
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
              	  // 返回值为是否继续填充 bean
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					continueWithPropertyPopulation = false;
					break;
				}
			}
		}
	}
	// 如果后处理器发出停止填充命令则终止后续的运行
	if (!continueWithPropertyPopulation) {
		return;
	}

  	 // 2
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

		// Add property values based on autowire by name if applicable.
      	 // 根据名称自动注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}

		// Add property values based on autowire by type if applicable.
      	 // 根据类型自动注入
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}

		pvs = newPvs;
	}

  	 // 后处理器已经初始化
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
  	 // 需要依赖检查
	boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

	if (hasInstAwareBpps || needsDepCheck) {
		PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		if (hasInstAwareBpps) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) { // 3
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                  	  // 对所有需要依赖检查的属性进行后处理
					pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvs == null) {
						return;
					}
				}
			}
		}
		if (needsDepCheck) {
          	 // 依赖检查，对应 depends-on 属性， 3.0 已经弃用此属性
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}
	}
 
  	 // 将属性应用到 bean 中
	applyPropertyValues(beanName, mbd, bw, pvs); // 4
}
```

1. `InstantiationAwareBeanPostProcessor`处理器的`postProcessAfterInstantiation`函数的应用，此函数可以控制程序是否继续进行属性填充。
2. 根据注入类型（byName/byType），提取依赖的 bean，并统一存入`PropertyValues`中。
3. 应用`InstantiationAwareBeanPostProcessor`处理器的`postProcessPropertyValues`方法，对属性获取完毕填充前对属性的再次处理，典型应用是 `RequiredAnnotationBeanPostProcessor` 类中对属性的验证。
4. 将所有`PropertyValues`中的属性填充至`BeanWrapper`中。

在上面的步骤中有几个地方是我们比较感兴趣的，它们分别是依赖注入（ `autowireByName`/`autowireByType`）以及属性填充，那么，接下来进一步分析这几个功能的实现细节。

### `autowireByName`

```Java
protected void autowireByName(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

     // 寻找 bw 中需要依赖注入的属性
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		if (containsBean(propertyName)) {
          	 // 递归初始化相关的 bean 
			Object bean = getBean(propertyName);
			pvs.add(propertyName, bean);
             // 注册依赖
			registerDependentBean(propertyName, beanName);
			if (logger.isDebugEnabled()) {
				logger.debug("Added autowiring by name from bean name '" + beanName +
						"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
						"' by name: no matching bean found");
			}
		}
	}
}
```

### `autowireByType`

```Java
protected void autowireByType(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}

	Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
     // 寻找 bw 中需要依赖注入的属性
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		try {
			PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			// Don't try autowiring by type for type Object: never makes sense,
			// even if it technically is a unsatisfied, non-simple property.
			if (Object.class != pd.getPropertyType()) {
              	  // 探测指定属性的 set 方法
				MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
				// Do not allow eager init for type matching in case of a prioritized post-processor.
				boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
				DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
				Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
              	  // 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在 
              	  // autowireBeanNames 中，当属性存在多个封装 bean 时，如:
                  // @Autowired private List<A> aList; 将会找到所有匹配 A 类型
                  // 的 bean 并将其注入
				if (autowiredArgument != null) {
					pvs.add(propertyName, autowiredArgument);
				}
				for (String autowiredBeanName : autowiredBeanNames) {
                  	  // 注册依赖
					registerDependentBean(autowiredBeanName, beanName);
					if (logger.isDebugEnabled()) {
						logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
								propertyName + "' to bean named '" + autowiredBeanName + "'");
					}
				}
				autowiredBeanNames.clear();
			}
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
		}
	}
}
```

现根据名称自动匹配的第一步就是寻找 `bw` 中需要依赖注入的属性，同样对于根据类型自动匹配的实现来讲第一步也是寻找`bw`中需要依赖注入的属性，然后遍历这些属性并寻找类型匹配的 bean，其中最复杂的就是寻找类型匹配的 bean。

#### `DefaultListableBeanFactory#resolveDependency`

```Java
public Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName,
		Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

	descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
	if (javaUtilOptionalClass == descriptor.getDependencyType()) {
		return new OptionalDependencyFactory().createOptionalDependency(descriptor, requestingBeanName);
	}
	else if (ObjectFactory.class == descriptor.getDependencyType() ||
			ObjectProvider.class == descriptor.getDependencyType()) {
         // ObjectFactory 类注入的特殊处理
		return new DependencyObjectProvider(descriptor, requestingBeanName);
	}
	else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
         // javaxInjectProviderClass 类注入的特殊处理
		return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
	}
	else {
		Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
				descriptor, requestingBeanName);
		if (result == null) {
              // 通用逻辑处理
			result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
		}
		return result;
	}
}
```

```Java
public Object doResolveDependency(DependencyDescriptor descriptor, String beanName,
		Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {

	InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
	try {
		Object shortcut = descriptor.resolveShortcut(this);
		if (shortcut != null) {
			return shortcut;
		}

		Class<?> type = descriptor.getDependencyType();
         // 用于支持 Spring 中的注解 @Value
		Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
		if (value != null) {
			if (value instanceof String) {
				String strVal = resolveEmbeddedValue((String) value);
				BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
				value = evaluateBeanDefinitionString(strVal, bd);
			}
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			return (descriptor.getField() != null ?
					converter.convertIfNecessary(value, type, descriptor.getField()) :
					converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
		}

		Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
		if (multipleBeans != null) {
			return multipleBeans;
		}

         // 根据属性类型找到 beanFactory 中所有类型的匹配 bean，
         // 返回值的构成为： key: 匹配的 beanName, value: beanName 对应的实例化后的 bean(通过 getBean(beanName) 返回)
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
		if (matchingBeans.isEmpty()) {
          	  // 如果 autowire 的 require 属性为 true 而找到的匹配项却为空则只能抛出异常
			if (descriptor.isRequired()) {
				raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
			}
			return null;
		}

		String autowiredBeanName;
		Object instanceCandidate;

		if (matchingBeans.size() > 1) {
			autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
			if (autowiredBeanName == null) {
				if (descriptor.isRequired() || !indicatesMultipleBeans(type)) {
					return descriptor.resolveNotUnique(type, matchingBeans);
				}
				else {
					// In case of an optional Collection/Map, silently ignore a non-unique case:
					// possibly it was meant to be an empty collection of multiple regular beans
					// (before 4.3 in particular when we didn't even look for collection beans).
					return null;
				}
			}
			instanceCandidate = matchingBeans.get(autowiredBeanName);
		}
		else {
			// We have exactly one match.
			Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
			autowiredBeanName = entry.getKey();
			instanceCandidate = entry.getValue();
		}

		if (autowiredBeanNames != null) {
			autowiredBeanNames.add(autowiredBeanName);
		}
		return (instanceCandidate instanceof Class ?
				descriptor.resolveCandidate(autowiredBeanName, type, this) : instanceCandidate);
	}
	finally {
		ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
	}
}
```

### `applyPropertyValues`

程序运行到这里，已经完成了对所有注入属性的获取，但是获取的属性是以`PropertyValues`形式存在的，还并没有应用到已经实例化的bean中，这一工作是在`applyPropertyValues`中。

```Java
/**
 * Apply the given property values, resolving any runtime references
 * to other beans in this bean factory. Must use deep copy, so we
 * don't permanently modify this property.
 * @param beanName the bean name passed for better exception information
 * @param mbd the merged bean definition
 * @param bw the BeanWrapper wrapping the target object
 * @param pvs the new property values
 */
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	if (pvs == null || pvs.isEmpty()) {
		return;
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (System.getSecurityManager() != null) {
		if (bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}
	}

	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
         // 如果 mpvs 中的值已经被转换为对应的类型那么可以直接设置到 beanWapper 中
		if (mpvs.isConverted()) {
			// Shortcut: use the pre-converted values as-is.
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		original = mpvs.getPropertyValueList();
	}
	else {
         // 如果 pvs 并不是使用 MutablePropertyValues 封装的类型，那么直接使用原始的属性获取方法
		original = Arrays.asList(pvs.getPropertyValues());
	}

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
     // 获取对应的解析器
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

	// Create a deep copy, resolving any references for values.
	List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
	boolean resolveNecessary = false;
     // 遍历属性，将属性转换为对应类的对应属性的类型
	for (PropertyValue pv : original) {
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			Object originalValue = pv.getValue();
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
			// Possibly store converted value in merged bean definition,
			// in order to avoid re-conversion for every created bean instance.
			if (resolvedValue == originalValue) {
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		mpvs.setConverted();
	}

	// Set our (possibly massaged) deep copy.
	try {
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}
```

## 初始化 bean

大家应该记得在 bean 配置时 bean 中有一个`init-method`的属性，这个属性的作用是在 bean 实例化前调用`init-method`指定的方法来根据用户业务进行相应的实例化。我们现在就已经进入这个方法了，首先看一下这个方法的执行位置，Spring中程序已经执行过bean的实例化，并且进行了属性的填充，而就在这时将会调用用户设定的初始化方法。

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
         // 对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
      	 // 应用后处理器
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
         // 激活用户自定义的 init 方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}

	if (mbd == null || !mbd.isSynthetic()) {
         // 后处理器应用
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

### 激活 Aware 方法

Spring 中提供一些 Aware 相关接口，比如 `BeanFactoryAware`、`ApplicationContextAware`、`ResourceLoaderAware`、`ServletContextAware` 等，实现这些 Aware 接口的 bean 在被初始之后，可以取得一些相对应的资源，例如实现`BeanFactoryAware` 的 bean 在初始后， Spring 容器将会注入 `BeanFactory` 的实例，而实现`ApplicationContextAware`的bean，在bean被初始后，将会被注入`ApplicationContext`的实例等。

#### Aware 的使用

1. 定义 bean

   ```java
   public class Hello {
       public void say() {
           System.out.println("hello");
       }
   }
   ```

2. 定义 `BeanFactoryAware` 类型的 bean

   ```Java
   public class TestAware implements BeanFactoryAware {
       private BeanFactory beanFactory;

       // 声明 bean 的时候 Spring 会自动注入 BeanFactory
       @Override
       public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
           this.beanFactory = beanFactory;
       }

       public void testAware() {
           // 通过 hello 这个 bean id 从 beanFactory 获取实例
           Hello hello = (Hello) beanFactory.getBean("hello");
           hello.say();
       }
   }
   ```

3. 测试

   ```Java
   public static void main(String[] args) {
       ApplicationContext ctx = new ClassPathXmlApplicationContext("testAware.xml");
       TestAware testAware = (TestAware) ctx.getBean("testAware");
       testAware.testAware();
   }

   /**
   hello
   **/
   ```

#### `invokeAwareMethods`

```Java
private void invokeAwareMethods(final String beanName, final Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware) {
			((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
		}
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

### 处理器的应用

`BeanPostProcessor`相信大家都不陌生，这是 Spring 中开放式架构中一个必不可少的亮点，给用户充足的权限去更改或者扩展 Spring ，而除了 `BeanPostProcessor` 外还有很多其他的 `PostProcessor`，当然大部分都是以此为基础，继承自 `BeanPostProcessor`。`BeanPostProcessor` 的使用位置就是这里，在调用客户自定义初始化方法前以及调用自定义初始化方法后分别会调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 和`postProcessAfterInitialization`方法，使用户可以根据自己的业务需求进行响应的处理。

```Java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
		result = beanProcessor.postProcessBeforeInitialization(result, beanName);
		if (result == null) {
			return result;
		}
	}
	return result;
}
```

```Java
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
		result = beanProcessor.postProcessAfterInitialization(result, beanName);
		if (result == null) {
			return result;
		}
	}
	return result;
}
```

### 激活自定义的 init 方法

客户定制的初始化方法除了我们熟知的使用配置`init-method`外，还有使自定义的 bean 实现`InitializingBean`接口，并在`afterPropertiesSet`中实现自己的初始化业务逻辑。

`init-method`与`afterPropertiesSet`都是在初始化 bean 时执行，执行顺序是`afterPropertiesSet`先执行，而`init-method`后执行。

在`invokeInitMethods`方法中就实现了这两个步骤的初始化方法调用。

```Java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
		throws Throwable {
	// 首先会检查是否是 InitializingBean，如果是的话需要调用 afterPropertiesSet 方法
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (logger.isDebugEnabled()) {
			logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
		}
		if (System.getSecurityManager() != null) {
			try {
				AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
					@Override
					public Object run() throws Exception {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}
				}, getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
              // 属性初始化后的处理
			((InitializingBean) bean).afterPropertiesSet();
		}
	}

	if (mbd != null) {
		String initMethodName = mbd.getInitMethodName();
		if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
              // 调用自定义初始化方法
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
```

## 注册 `DisposableBean`

Spring 中不但提供了对于初始化方法的扩展入口，同样也提供了销毁方法的扩展入口，对于销毁方法的扩展，除了我们熟知的配置属性`destroy-method`方法外，用户还可以注册后处理器`DestructionAwareBeanPostProcessor`来统一处理bean的销毁方法，代码如下(`doCreateBean`中)：

```Java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
	AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
	if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
		if (mbd.isSingleton()) {
			// Register a DisposableBean implementation that performs all destruction
			// work for the given bean: DestructionAwareBeanPostProcessors,
			// DisposableBean interface, custom destroy method.
              // 单例模式下注册需要销毁的 bean，此方法中会处理实现 DisposableBean 的 bean，
              // 并且堆所有的 bean 使用 DestructionAwareBeanPostProcessors 处理
              // DisposableBean DestructionAwareBeanPostProcessors
			registerDisposableBean(beanName,
					new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
		}
		else {
			// A bean with a custom scope...
              // 自定义 scope 的处理
			Scope scope = this.scopes.get(mbd.getScope());
			if (scope == null) {
				throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
			}
			scope.registerDestructionCallback(beanName,
					new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
		}
	}
}
```
