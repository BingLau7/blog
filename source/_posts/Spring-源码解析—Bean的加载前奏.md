---
title: Spring 源码解析—Bean的加载前奏
date: 2017-11-14 16:59:44
tags:
    - Spring
    - Java
categories: 源码分析
---

`User user = (User)context.getBean("testbean");`

由这句入手

<!-- more -->

`AbstractBeanFactory#getBean`

```Java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}

@SuppressWarnings("unchecked")
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {
	// 提取对应的 beanName
	final String beanName = transformedBeanName(name);
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
  	/**
  	检查缓存中或者实例工厂中是否有对应的实例
  	为什么首先会使用这段代码呢，因为在创建单例 bean 的时候会存在依赖注入的情况，
  	而在创建依赖的时候为了避免循环依赖，Spring 创建 bean 的原则是不等 bean 创建
  	完成就会将创建的 bean 的 ObjectFactory 提前曝光，也就是将 ObjectFactory 加入
  	缓存中，一旦下个 bean 创建时候需要依赖上个 bean 则直接使用 ObjectFactory
  	**/
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
      	// 返回对应的实例，有时候存在诸如 BeanFactory 的情况并不是直接返回实例本身而是返回指定方法返回的实例
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
      	/**
      	只有在单例情况才会尝试解决依赖循环，原型模型情况，如果存在
      	A 中有 B 的熟悉，B 中有 A 的属性，那么当依赖注入的时候，就会产生当 A 还未创建完的时候
      	因为对于 B 的创建再次返回创建 ，造成依赖循环，也就是下面的情况
      	**/
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		BeanFactory parentBeanFactory = getParentBeanFactory();
      	// 如果 beanDefinitionMap 中也就是在所有已经加载的类中不包括 beanName 则
      	// 尝试从 parentBeanFactory 中检测
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
          	// 递归到 BeanFactory 中寻找
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

      	// 如果不是仅仅做类型检查则是创建 bean，这里要进行记录
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
          	// 将存储 XML 配置文件的 GernericBeanDefinition 转换为 RootBeanDefinition，
          	// 如果指定 BeanName 是子 Bean 的话同时会合并父类的相关属性
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			String[] dependsOn = mbd.getDependsOn();
          	// 若存在依赖则需要递归实例化依赖的 bean
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
                  	  // 缓存依赖调用
					registerDependentBean(dep, beanName);
					getBean(dep);
				}
			}

			// Create bean instance.
          	// 实例化依赖的 bean 后便可以实例化 mbd 本身了
          	// singleton 模式的创建
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					@Override
					public Object getObject() throws BeansException {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
              	// prototype 模式的创建 (new)
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
              	// 指定的 scope 上实例化 bean
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
  	// 检查需要的类型是否符合 bean 的实际类型
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

大致过程：

1. 转换对应 beanName。这里的 beanName 可能是别名，可能是 FactoryBean，所以需要一系列的解析：
   1. 去除 FactoryBean 的修饰符，也就是如果 `name="&aa"`，那么会首先去除 `&` 使得 `name=aa`
   2. 取指定 alias 所表示的最终 beanName，例如别名 A 指向名称为 B 的 bean 则返回 B；若别名 A 指向别名 B，别名 B 又指向名称为 C 的 bean 则返回 C

2. 尝试从缓存中加载单例

   单例在 Spring 的同一个容器内只会被创建一次，后续在获取 bean，就直接从单例缓存中获取了。当然这里也只是尝试加载，首先尝试从缓存中加载，如果加载不成功则再次尝试从 singletonFactories 中加载。

   由于存在依赖注入的问题，所以在 Spring 中创建 bean 的原则是不等 bean 创建完成就会将创建的 ObjectFactory 提早曝光加入到缓存中，一旦下一个 bean 创建需要依赖上一个 bean 则直接使用 ObjectFactory。

3. bean 的实例化

4. 原型模式的依赖检查

   只有单例会尝试解决循环依赖。

5. 在非 singleton 下检测 parentBeanFactory，看是否需要进入 parentBeanFactory 中加载（当前 BeanFactory 中无该 bean 且 parentBeanFactory 存在且存在该 bean）

6. 将存储 XML 配置文件的 GernericBeanDefinition 转换为 RootBeanDefinition。方便 Bean 的后续处理。

7. 寻找依赖

8. 针对不同的  scope 进行 bean 的创建

9. 类型转换（requiredType = true）


## FactoryBean 的使用

一般来说，Spring 是通过反射机制利用 bean 的 class 属性指定实现类来实例化 bean 的。FactoryBean 是为了对付配置 bean 的复杂性的。

```Java
public interface FactoryBean<T> {
	T getObject() throws Exception;
	Class<?> getObjectType();
  	// 如果返回 true 则 getObject() 时候会将实例放入 Spring 容器中单实例缓存池中
	boolean isSingleton(); 
}
```

实现了 `XxxFactoryBean`之后，解析`<bean id="xxx" class="xx.xx.XxxFactoryBean" />`时候会调用该其实现的 `getObject()` 方法

## 缓存中获取单例 bean

```Java
@Override
public Object getSingleton(String beanName) {
  	// 设置 true 表示允许早期依赖
	return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  	// 检查缓存中是否存在实例
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      	// 如果为空，则锁定全局变量并进行处理
		synchronized (this.singletonObjects) {
          	// 如果此 bean 正在加载则不处理
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
              	// 当某些方法需要提前初始化时候会调用 addSingletonFactory 方法
              	// 将对应的 ObjectFactory 初始化策略存储在 singletonFactories
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
                  	// 调用预先设定的 getObject 方法
					singletonObject = singletonFactory.getObject();
                  	// 记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

首先尝试从 `singletonObjects` 中获取实例，如果获取不到，则从 `earlySingletonObjects` 里面获取，如果还取不到，则尝试从 `singletonFactories` 里面获取 `beanName` 对应的 `ObjectFactory`，然后调用这个 `ObjectFactory` 的 getObject 来创建 bean，并放到 `earlySingletonObjects` 中，然后从 singletonFactories 中 remove 掉这个 ObjectFactory。



-  singletonObjects: 用于保存 BeanName 和创建 bean 实例之间的关系，beanName -> beanInstance
-  singletonFactories：用于保存 BeanName 和创建  bean 的工厂之间的关系，beanName -> ObjectFactory
-  earlySingletonObjects：也是保存 BeanName 和创建 bean 实例之间的关系，与singletonObjects的不同之处在于，当一个单例bean被放到这里面后，那么当bean还在创建过程中，就可以通过getBean方法获取到了，其目的是用来检测循环引用。
-  registeredSingletons：用来保存当前所有已注册的bean。

## 从 bean 的实例中获取对象

无论是从缓存中获取到的 bean 还是通过不同的 scope 策略加载的 bean 都只是最原始的 bean 状态，并不一定是我们最终想要的 bean。举个例子，假如我们需要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，但是我们真正需要的是工厂 bean 中定义的 factory-method 方法中返回的 bean ，而`getObjectForBeanInstance` 方法就是完成这个工作的。

```Java
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

	// Don't let calling code try to dereference the factory if the bean isn't a factory.
     // 如果指定的 name 是工厂相关(以 & 为前缀)且 beanInstance 又不是 FactoryBean 类型则验证不通过
	if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
		throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
	}

	// Now we have the bean instance, which may be a normal bean or a FactoryBean.
	// If it's a FactoryBean, we use it to create a bean instance, unless the
	// caller actually wants a reference to the factory.
  	// 现在我们有了个 bean 的实例，这个实例可能会是正常的 bean 或者 FactoryBean
  	// 如果是 FactoryBean 我们使用它创建实例，但如果用户想要直接获取工厂实例而不是工程对应的
  	// getObject 方法对应的实例那么传入的 name 应该包含前缀 &
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}

  	// 加载 FactoryBean
	Object object = null;
	if (mbd == null) {
      	// 尝试从缓存中加载 bean
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
		// Return bean instance from factory.
      	// 到这里已经明确指定 beanInstance 一定是 FactoryBean 类型
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// Caches object obtained from FactoryBean if it is a singleton.
      	// containsBeanDefinition 检测 beanDefinitionMap 中也就是在所有已经加载的类中
      	// 检测是否定义 beanName
		if (mbd == null && containsBeanDefinition(beanName)) {
          	// 将存储 XML 配置文件的 GernericBeanDefinition 转换为 RootBeanDefinition，
          	// 如果指定 BeanName 是子 Bean 的话同时会合并父类的相关属性
			mbd = getMergedLocalBeanDefinition(beanName);
		}
      	// 是否是用户定义而不是应用程序本身定义的
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```

1. 对 FactoryBean 正确性的验证
2. 对非 FactoryBean 不做任何处理
3. 对 bean 进行转换
4. 将从 Factory 中解析 bean 的工作委托给 `getObjectFromFactoryBean`

```Java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
	if (factory.isSingleton() && containsSingleton(beanName)) {
		synchronized (getSingletonMutex()) {
			Object object = this.factoryBeanObjectCache.get(beanName);
			if (object == null) {
				object = doGetObjectFromFactoryBean(factory, beanName);
				// Only post-process and store if not put there already during getObject() call above
				// (e.g. because of circular reference processing triggered by custom getBean calls)
				Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
				if (alreadyThere != null) {
					object = alreadyThere;
				}
				else {
					if (object != null && shouldPostProcess) {
						try {
							object = postProcessObjectFromFactoryBean(object, beanName);
						}
						catch (Throwable ex) {
							throw new BeanCreationException(beanName,
									"Post-processing of FactoryBean's singleton object failed", ex);
						}
					}
					this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
				}
			}
			return (object != NULL_OBJECT ? object : null);
		}
	}
	else {
		Object object = doGetObjectFromFactoryBean(factory, beanName);
		if (object != null && shouldPostProcess) {
			try {
              	 // 调用 ObjectFactory 的后处理器
				object = postProcessObjectFromFactoryBean(object, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
			}
		}
		return object;
	}
}
```

这部分代码只是做了一件事：返回的 bean 如果是单例，那就必须要**保证全局唯一**，同时，也因为是单例的，所以不被重复创建，可以使用缓存来提高性能，也就是说已经加载过就要记录下来以便于下次复用，否则的话就直接获取了。

所以我们最后是在 `doGetObjectFromFactoryBean `中看到了自己想要的方法

```Java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
		throws BeanCreationException {

	Object object;
	try {
		if (System.getSecurityManager() != null) {
			AccessControlContext acc = getAccessControlContext();
			try {
              	// 需要权限验证
				object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
					@Override
					public Object run() throws Exception {
							return factory.getObject();
						}
					}, acc);
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
          	// 直接调用 getObject 方法
			object = factory.getObject();
		}
	}
	catch (FactoryBeanNotInitializedException ex) {
		throw new BeanCurrentlyInCreationException(beanName, ex.toString());
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
	}

	// Do not accept a null value for a FactoryBean that's not fully
	// initialized yet: Many FactoryBeans just return null then.
	if (object == null && isSingletonCurrentlyInCreation(beanName)) {
		throw new BeanCurrentlyInCreationException(
				beanName, "FactoryBean which is currently in creation returned null from getObject");
	}
	return object;
}
```

我们接下来看 `doGetObjectFromFactoryBean` 获取对象之后，最后返回对象的过程中操作 `postProcessObjectFromFactoryBean` 做了哪些工作？

`AbstractAutowireCapableBeanFactory#postProcessObjectFromFactoryBean`

```Java
protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
	return applyBeanPostProcessorsAfterInitialization(object, beanName);
}

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

对于后处理器，后面来进行介绍。这里我们只需要了解在 Spring 获取 bean 的规则中有这样一条：**尽可能保证所有 bean 初始化后都会调用注册的 BeanPostProcesser 的 postProcessAfterInitialization 方法进行处理**，在世纪开发过程中可以根据此特性设计自己的逻辑业务。

## 获取单例

之前我们讲解了从缓存中获取单例的过程，那么，如果缓存中不存在已经加载的单例bean就需要从头开始bean的加载过程了，而Spring中使用getSingleton的重载方法实现bean的加载过程。

`AbstractBeanFactory#getBean` 片段

```Java
// 实例化依赖的 bean 后便可以实例化 mbd 本身了
 // singleton 模式的创建	
if (mbd.isSingleton()) {
	sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
		@Override
		public Object getObject() throws BeansException {
			try {
				return createBean(beanName, mbd, args);
			}
			catch (BeansException ex) {
				// Explicitly remove instance from singleton cache: It might have been put there
				// eagerly by the creation process, to allow for circular reference resolution.
				// Also remove any beans that received a temporary reference to the bean.
				destroySingleton(beanName);
				throw ex;
			}
		}
	});
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

`DefaultSingletonBeanRegistry#getSingleton`

```Java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "'beanName' must not be null");
  	// 全局变量需要同步
	synchronized (this.singletonObjects) {
      	// 首先检查对应的 bean 是否已经加载过了，因为 singleton 模式其实就是复用以创建的 bean，
      	// 所以这步是必须的
		Object singletonObject = this.singletonObjects.get(beanName); // 1
      	// 如果为空才可以进行 singleton 的 bean 的初始化
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'"); // 2
			}
			beforeSingletonCreation(beanName); // 3
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<Exception>();
			}
			try {
              	// 初始化 bean
				singletonObject = singletonFactory.getObject(); // 4
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName); // 5
			}
			if (newSingleton) {
              	// 加入缓存
				addSingleton(beanName, singletonObject); // 6
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null); // 7
	}
}
```

1. 检查缓存是否已经加载过了

2. 若没有加载，则记录 beanName 的正在加载状态

3. 加载单例前记录加载状态，通过 `this.singletonsCurrentlyInCreation.add(beanName)`，以便于对循环依赖进行检测

   ```Java
   	protected void beforeSingletonCreation(String beanName) {
   		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
   			throw new BeanCurrentlyInCreationException(beanName);
   		}
   	}
   ```

4. 通过调用参数传入的 `ObjectFactory` 的个体 `Object` 方法实例化 `bean`

5. 加载单例后的处理方法调用。当 `bean` 加载结束后需要移除缓存中对该 `bean` 的正在加载状态的记录

   ```Java
   	protected void afterSingletonCreation(String beanName) {
   		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
   			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
   		}
   	}
   ```

6. 将结果记录至缓存并删除加载 `bean` 过程中所记录的各种辅助状态

   ```Java
   	protected void addSingleton(String beanName, Object singletonObject) {
   		synchronized (this.singletonObjects) {
   			this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));
   			this.singletonFactories.remove(beanName);
   			this.earlySingletonObjects.remove(beanName);
   			this.registeredSingletons.add(beanName);
   		}
   	}
   ```

7. 返回处理结果

虽然我们已经从外部了解了加载bean的逻辑架构，但现在我们还并没有开始对bean加载功能的探索，之前提到过， bean 的加载逻辑其实是在传入的 ObjectFactory 类型的参数singletonFactory中定义的，我们反推参数的获取，得到如下代码：

```Java
sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
	@Override
	public Object getObject() throws BeansException {
		try {
			return createBean(beanName, mbd, args);
		}
		catch (BeansException ex) {
			// Explicitly remove instance from singleton cache: It might have been put there
			// eagerly by the creation process, to allow for circular reference resolution.
			// Also remove any beans that received a temporary reference to the bean.
			destroySingleton(beanName);
			throw ex;
		}
	}
});
```

`ObjectFactory` 的核心部分其实只是调用了 `createBean` 的方法，所以，继续~

## 准备创建 bean

`AbstractAutowireCapableBeanFactory#createBean`

```Java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
  	// 锁定 class，根据设置的 class 属性或者根据 className 来解析 Class
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName); // 1
  	// 如果解析成功则 clone RootBeanDefinition 并且设置其 bean 类为解析之后的 class
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
  	// 验证及准备覆盖的方法
	try {
		mbdToUse.prepareMethodOverrides(); // 2
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
      	// 给 BeanPostProcessors 一个机会来返回代理来替代真正的实例
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse); // 3
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	Object beanInstance = doCreateBean(beanName, mbdToUse, args); // 4
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}
```

1. 根据设置的 class 属性或者根据 className 来解析 Class
2. 对 override 属性进行标记及验证（`lookup-method` and `replace-method`）
3. 应用初始化前的后处理器，解析指定 bean 是否存在初始化前的短路操作
4. 创建 bean

### 处理 `override` 属性

```Java
public void prepareMethodOverrides() throws BeanDefinitionValidationException {
	// Check that lookup methods exists.
	MethodOverrides methodOverrides = getMethodOverrides();
	if (!methodOverrides.isEmpty()) {
		Set<MethodOverride> overrides = methodOverrides.getOverrides();
		synchronized (overrides) {
			for (MethodOverride mo : overrides) {
				prepareMethodOverride(mo);
			}
		}
	}
}

protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
  	// 获取对应类中对应方法名的个数
	int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
	if (count == 0) {
		throw new BeanDefinitionValidationException(
				"Invalid method override: no method with name '" + mo.getMethodName() +
				"' on class [" + getBeanClassName() + "]");
	}
	else if (count == 1) {
		// Mark override as not overloaded, to avoid the overhead of arg type checking.
      	// 标记 MethoOverride 暂未被覆盖，避免参数类型检查的开销
		mo.setOverloaded(false);
	}
}
```

对于方法的匹配来讲，如果一个类中存在若干个重载方法，那么，在函数调用及增强的时候还需要根据参数类型进行匹配，来最终确认当前调用的到底是哪个函数。但是，Spring将一部分匹配工作在这里完成了，如果当前类中的方法只有一个，那么就设置重载该方法没有被重载，这样在后续调用的时候便可以直接使用找到的方法，而不需要进行方法的参数匹配验证了，而且还可以提前对方法存在性进行验证，正可谓一箭双雕。

### 实例化的前置处理

**AOP基于前置处理后的短路判断**

```Java
try {
	// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
  	// 给 BeanPostProcessors 一个机会来返回代理来替代真正的实例
	Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  	// 提供一个短路判断，当经过处理之后的 bean 若不为空，则直接返回结果。
  	// 我们所熟知的 AOP 功能就是基于这里的判断
	if (bean != null) {
		return bean;
	}
}
```

```Java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
  	// 如果尚未被解析
	if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
		// Make sure bean class is actually resolved at this point.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			Class<?> targetType = determineTargetType(beanName, mbd);
			if (targetType != null) {
				bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
				if (bean != null) {
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
		}
		mbd.beforeInstantiationResolved = (bean != null);
	}
	return bean;
}
```

其中 `applyBeanPostProcessorsBeforeInstantiation` 与 `applyBeanPostProcessorsAfterInitialization` 分别对应实例化前后的处理器，实现也挺简单的，无非是对后处理器中的所有 `InstantiationAwareBeanPostProcessor` 类型的后处理器进行 `postProcessBeforeInstantiation` 方法和 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法的调用

####  实例化前的后处理器应用

```Java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
	}
	return null;
}
```

#### 实例化后的后处理器应用

```Java
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

在讲解从缓存中获取单例bean的时候就提到过，Spring中的规则是在bean的初始化后尽可能保证将注册的后处理器的postProcessAfterInitialization方法应用到该bean中，因为如果返回的bean不为空，那么便不会再次经历普通bean的创建过程，所以只能在这里应用后处理器的postProcessAfterInitialization方法。

## 循环依赖

### 什么是循环依赖

循环依赖就是循环引用，就是两个或多个 bean 相互之间持有对方。循环依赖不是循环调用，循环调用是指方法之间的环调用的，循环调用除非有终止条件，否则无法解决。

### Spring 如何解决循环依赖

我们先来定义一个循环引用类：

```Java
package io.github.binglau.circle;

/**
 * 文件描述:
 */

public class TestA {
    private TestB testB;

    public void a() {
        testB.b();
    }

    public TestB getTestB() {
        return testB;
    }

    public void setTestB(TestB testB) {
        this.testB = testB;
    }
}


package io.github.binglau.circle;

/**
 * 文件描述:
 */

public class TestB {
    private TestC testC;

    public void b() {
        testC.c();
    }

    public TestC getTestC() {
        return testC;
    }

    public void setTestC(TestC testC) {
        this.testC = testC;
    }
}

package io.github.binglau.circle;

/**
 * 文件描述:
 */

public class TestC {
    private TestA testA;

    public void c() {
        testA.a();
    }

    public TestA getTestA() {
        return testA;
    }

    public void setTestA(TestA testA) {
        this.testA = testA;
    }
}
```

在 Spring 中将循环依赖的处理分成了 3 中情况

#### 构造器循环依赖

无法解决，抛出 `BeanCurrentlyInCreationException` 异常

Spring容器将每一个正在创建的bean标识符放在一个“当前创建bean池”中，bean标识符在创建过程中将一直保持在这个池中，因此如果在创建 bean 过程中发现自己已经在“当前创建bean池”里时，将抛出BeanCurrentlyInCreationException异常表示循环依赖；而对于创建完毕的bean将从“当前创建bean池”中清除掉。

直观的测试

1. 创建配置文件

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans-2.0.xsd">

       <bean id="testA" class="io.github.binglau.circle.TestA">
           <constructor-arg index="0" ref="testB"/>
       </bean>

       <bean id="testB" class="io.github.binglau.circle.TestB">
           <constructor-arg index="0" ref="testC"/>
       </bean>

       <bean id="testC" class="io.github.binglau.circle.TestC">
           <constructor-arg index="0" ref="testA"/>
       </bean>

   </beans>
   ```

2. 创建测试用例

   ```Java
       @Test(expected = BeanCurrentlyInCreationException.class)
       public void testCircleByConstructor() throws Throwable {
           try {
               new ClassPathXmlApplicationContext("test.xml");
           } catch (Exception e) {
               Throwable el = e.getCause().getCause().getCause();
               throw el;
           }
       }
   ```

#### setter 循环依赖

表示通过 setter 注入方式构成的循环依赖。对于 setter 注入造成的依赖是通过 Spring 容器提前暴露刚完成构造器注入但未完成其他步骤（如 setter 注入）的 bean 来完成的，而且只能解决单例作用域的 bean 循环依赖。通过提前暴露一个单例工厂方法，从而使其他 bean 能引用到该 bean ，如下代码所示：

```Java
addSingletonFactory(beanName, new ObjectFactory() {
　 public Object getObject() throws BeansException {
　　 return getEarlyBeanReference(beanName, mbd, bean);
  }
});
```

具体步骤如下：

1. Spring 容器创建单例 `testA` bean，首先根据无参构造器创建 bean，并暴露一个 `ObjectFactory` 用于返回一个提前暴露一个创建中的 bean，并将`testA`标识符放到 『当前创建 bean 池』，然后进行 setter 注入 `testB`。
2. Spring 容器创建单例`testB `bean，首先根据无参构造器创建 bean，并暴露一个`ObjectFactory`用于返回一个提前暴露一个创建中的 bean，并将“testB”标识符放到『当前创建bean池』，然后进行 setter 注入 `circle`。
3. Spring 容器创建单例`testC` bean，首先根据无参构造器创建 bean，并暴露一个`ObjectFactory`用于返回一个提前暴露一个创建中的 bean，并将`testC`标识符放到『当前创建bean池』，然后进行setter注入`testA`。进行注入`testA`时由于提前暴露了`ObjectFactory`工厂，从而使用它返回提前暴露一个创建中的 bean。
4. 最后在依赖注入`testB`和`testA`，完成 setter 注入。

#### prototype 范围的依赖处理

对于 `prototype` 作用域 bean，Spring 容器无法完成依赖注入，因为 Spring 容器不进行缓存 `prototype` 作用域的 bean，因此无法提前暴露一个创建中的 bean。

>  关于创建 bean 详见下篇文章
