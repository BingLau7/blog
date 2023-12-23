---
title: Spring-源码分析-bean的解析(2)
date: 2017-11-03 16:58:14
tags:
    - Spring
    - Java
categories: 源码分析
---
>  当前版本 Spring 4.3.8

## 默认标签的解析

接上 `parseDefaultElement(ele, delegate);`

<!-- more -->

`DefaultBeanDefinitionDocumentReader#parseDefaultElement`

```java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
      	// import 标签解析
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
      	// 解析 alias 标签 
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
      	// bean 的解析
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
      	// beans 的解析
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

### Bean 标签的解析及注册

```java
	/**
	 * Process the given bean element, parsing the bean definition
	 * and registering it with the registry.
	 */
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele); // 1
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);  // 2
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry()); // 3
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder)); // 4
		}
	}
```

1. 首先委托 `BeanDefinitionDelegate` 类的 `parseBeanDefinitionElement` 方法进行元素解析，返回 `BeanDefinitionHolder` 类型的实例 `bdHolder`，经过这个方法后， `bdHolder` 实例已经包含我们的配置文件中配置的各个属性了，例如 `class、name、id、alias` 之类的属性。
2. 当返回的 `bdHolder` 不为空的情况下若存在默认标签的子节点下再有自定义属性，还需要再次对自定义标签进行解析。
3. 解析完成之后，需要对解析后的 `bdHolder` 进行注册，同样，注册操作委托给了 `BeanDefinitionReaderUtils` 的 `registerBeanDefinition` 方法。
4. 最后发出响应事件，通知想关的监听器，这个 `bean` 已经加载完成了

#### 解析 BeanDefinition

`BeanDefinitionDelegate#parseBeanDefinitionElement`

```java
	/**
	 * Parses the supplied {@code <bean>} element. May return {@code null}
	 * if there were errors during parse. Errors are reported to the
	 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
	 */
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}

	/**
	 * Parses the supplied {@code <bean>} element. May return {@code null}
	 * if there were errors during parse. Errors are reported to the
	 * {@link org.springframework.beans.factory.parsing.ProblemReporter}.
	 */
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
         // 1
         // 解析 id 属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
         // 解析 name 属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
      
         // 分割 name 属性
		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean); // 2
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) { // 3
                  // 如果不存在 beanName 那么根据 Spring 中提供的命名规则为当前 bean 生成对应的 beanName
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray); // 4
		}

		return null;
	}


```

1. 提取元素中的 id 以及 name 属性
2. 进一步解析其他所有属性并统一封装至 `GenericBeanDefinition` 类型的实例中（下面的函数）
3. 如果检测到 bean 没有指定 beanName ，那么使用默认规则为此 Bean 生成 beanName
4. 将获取到的信息封装到 BeanDefinitionHolder 的实例中

```java
/*parseBeanDefinitionElement*/
	/**
	 * Parse the bean definition itself, without regard to name or aliases. May return
	 * {@code null} if problems occurred during the parsing of the bean definition.
	 */
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
          	// 解析parent 属性
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
          	// 创建用于承载属性的 AbstractBeanDefinition 类型的 GenericBeanDefinition
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

          	// 硬编码解析默认 bean 的各种属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
          	// 提取description
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

          	// 解析元数据
			parseMetaElements(ele, bd);
          	// 解析 lookup-override 属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
          	// 解析 replaced-method 属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

          	// 解析构造函数参数
			parseConstructorArgElements(ele, bd);
          	// 解析 property 子元素
			parsePropertyElements(ele, bd);
          	// 解析 qualifier 子元素
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

接下来来看看一些复杂标签的解析:

##### 创建用于属性承载的 BeanDefinition

`BeanDefinition` 是一个接口，在 Spring 中存在三种实现，三种实现均继承了 `AbstactBeanDefinition`，其中 `BeanDefinition` 是配置文件中的 `<bean>` 元素标签在容器中的内部表示形式。`<bean> `元素标签拥有 `class、scope、lazy-init` 等配置属性。`BeanDefinition`则提供了相应的 `beanClass、scope、lazyInit `属性，`BeanDefinition` 和 `<bean>` 中的属性是一一对应的。

-  `RootBeanDefinition` 最常用的实现类，它对应一般性的 `<bean>` 元素标签

   当前版本中， `ConfigurationClassBeanDefinition` 继承与此类

-  `ChildBeanDefinition` 配置文件中的子 `<bean>`

-  `GenericBeanDefinition` 2.5 新加入的 `bean` 文件配置属性定义类，是一站式服务类。

   在当前版本中，有`ScannedGenericBeanDefinition`，`AnnotatedGenericBeanDefinition` 继承与此类

Spring 通过 `BeanDefinition` 将配置文件中的 `<bean>` 配置信息转换为容器的内部表示，并将这些 `BeanDefinition` 注册到 `BeanDefinitionRegistry` 中。Spring 容器的 `BeanDefinitionRegistry` 就像是 Spring 配置信息的内存数据库，主要是以 map 的形式保存的，后续操作直接从 `BeanDefinitionRegistry` 中读取配置信息。

`BeanDefinitionParserDelegate#createBeanDefinition`

```java
	/**
	 * Create a bean definition for the given class name and parent name.
	 * @param className the name of the bean class
	 * @param parentName the name of the bean's parent bean
	 * @return the newly created bean definition
	 * @throws ClassNotFoundException if bean class resolution was attempted but failed
	 */
	protected AbstractBeanDefinition createBeanDefinition(String className, String parentName)
			throws ClassNotFoundException {

		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}
	/**
	 * Create a new GenericBeanDefinition for the given parent name and class name,
	 * eagerly loading the bean class if a ClassLoader has been specified.
	 * @param parentName the name of the parent bean, if any
	 * @param className the name of the bean class, if any
	 * @param classLoader the ClassLoader to use for loading bean classes
	 * (can be {@code null} to just register bean classes by name)
	 * @return the bean definition
	 * @throws ClassNotFoundException if the bean class could not be loaded
	 */
	public static AbstractBeanDefinition createBeanDefinition(
			String parentName, String className, ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
        // parentName 可能为空
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
              	// 如果 classLoader 不为空，则使用已传入的 classLoader 同一虚拟机加载类对象，否则只是记录 className
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}
```

##### 解析各种属性

```java
	/**
	 * Apply the attributes of the given bean element to the given bean * definition.
	 * @param ele bean declaration element
	 * @param beanName bean name
	 * @param containingBean containing bean definition
	 * @return a bean definition initialized according to the bean element attributes
	 */
	public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			BeanDefinition containingBean, AbstractBeanDefinition bd) {

      	// 解析 singleton 属性。已废弃，使用 scope 设置
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
      	// 解析 scope 属性
      	// https://docs.spring.io/spring/docs/5.0.0.RELEASE/spring-framework-reference/core.html#beans-factory-scopes
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
          	// 在嵌入 beanDefinition 情况下且没有单独指定 scope 属性则使用父类默认的属性
			bd.setScope(containingBean.getScope());
		}

      	// 解析 absract 属性
      	// (抽象bean不能被实例化，只能被子类继承)。
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}
		
      	// 解析 lazy-init 属性
        // 懒初始化，默认false ，即：非懒惰初始化。这是高效的，可以提前暴露错误（如果存在），如果是懒初始化，只有在call 时才初始化。
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
      	// 若没有设置或设置成其他字符都会被设置为 false
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

      	// 解析 autowire 属性
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));
	
      	// 解析 dependency-check 属性
		String dependencyCheck = ele.getAttribute(DEPENDENCY_CHECK_ATTRIBUTE);
		bd.setDependencyCheck(getDependencyCheck(dependencyCheck));
	
      	// 解析 depends-on 属性
      	// 初始化该bean之前必须先初始化谁（name or id 指定）
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

      	// 解析 autowire-candidate 属性
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}
	
      	// 解析 primary 属性
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

      	// 解析 init-method 属性
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			if (!"".equals(initMethodName)) {
				bd.setInitMethodName(initMethodName);
			}
		}
		else {
			if (this.defaults.getInitMethod() != null) {
				bd.setInitMethodName(this.defaults.getInitMethod());
				bd.setEnforceInitMethod(false);
			}
		}

      	// 解析 destory-method 属性
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else {
			if (this.defaults.getDestroyMethod() != null) {
				bd.setDestroyMethodName(this.defaults.getDestroyMethod());
				bd.setEnforceDestroyMethod(false);
			}
		}

      	// 解析 factory-method 属性
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
      	// 解析 factory-bean 属性
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```

>  简单复习一下 bean 的自动装配 @Autowired （如果设置参数 require = false 则会尝试匹配，无匹配这让 bean 处于未装配状态）
>
>  1. primary（如果配置了的）
>  2. @Qualifier 做限定
>     1. 配合 @Autowired 使用指定需要装配哪个 bean
>     2. 配合 @Component / @Bean 指定限定符，不然在 @Autowired 使用的时候就是默认 beanId
>  3. 符合的类型

##### 解析子元素 meta

meta: 元数据，当需要使用里面的信息时可以通过key获取

>  Finally, the bean definitions should contain matching qualifier values. This example also demonstrates that bean *meta* attributes may be used instead of the `<qualifier/>` sub-elements.If available, the `<qualifier/>` and its attributes would take precedence, but the autowiring mechanism will fallback on the values provided within the `<meta/>` tags if no such qualifier is present

```xml
<bean id="myTestBean" class="bean.MyTestBean">
	<meta key="testStr" value="test">
</bean>
```

```java
	public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
      	// 获取当前节点的所有子元素
		NodeList nl = ele.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
          	// 提取 meta
			if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
				Element metaElement = (Element) node;
				String key = metaElement.getAttribute(KEY_ATTRIBUTE);
				String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
				BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
				attribute.setSource(extractSource(metaElement));
				attributeAccessor.addMetadataAttribute(attribute);
			}
		}
	}
```

##### 解析子元素 lookup-method

获取器注入，特殊方法注入，它是把一个方法声明为返回某种类型的 `bean`， 但实际要返回的 `bean` 是在配置文件里面配置的，此方法可用在设计有些可插拔的功能上，解除程序依赖。

简单来说就是，可以动态配置某个方法的返回值返回的 `bean`

`... <lookup-method name="方法名" bean="需要方法返回的 beanId"> ...`

```java
	/**
	 * Parse lookup-override sub-elements of the given bean element.
	 */
	public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
          	// 仅当在 Spring 默认 bean 的子元素且为 <lookup-method 时有效
			if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
				Element ele = (Element) node;
              	// 获取要修饰的方法
				String methodName = ele.getAttribute(NAME_ATTRIBUTE);
              	// 获取配置返回的 bean
				String beanRef = ele.getAttribute(BEAN_ELEMENT);
				LookupOverride override = new LookupOverride(methodName, beanRef);
				override.setSource(extractSource(ele));
				overrides.addOverride(override);
			}
		}
	}
```

##### 解析子元素 replaced-method

方法替换：可以在运行时用新的方法替换现有的方法。与之前的 `look-up` 不同的是，`replaced-method` 不但可以动态地替换返回实体 `bean`，而且还能动态地更改原有方法的逻辑。

需要实现 `MethodReplacer` 的 `reimplement` 方法，该方法替换其指定方法。

```java
	/**
	 * Parse replaced-method sub-elements of the given bean element.
	 */
	public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
          	// 识别标签
			if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
				Element replacedMethodEle = (Element) node;
              	// 提取需要替换方法
				String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
              	// 提权对应的新的替换方法
				String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
				ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
				// Look for arg-type match elements.
				List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
				for (Element argTypeEle : argTypeEles) {
                  	// 记录参数
					String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
					match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
					if (StringUtils.hasText(match)) {
						replaceOverride.addTypeIdentifier(match);
					}
				}
				replaceOverride.setSource(extractSource(replacedMethodEle));
				overrides.addOverride(replaceOverride);
			}
		}
	}
```

##### 解析子元素 constructor-arg

```java
	/**
	 * Parse a constructor-arg element.
	 */
	public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
      	// 提取对应元素
		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		if (StringUtils.hasLength(indexAttr)) {
			try {
				int index = Integer.parseInt(indexAttr);
				if (index < 0) {
					error("'index' cannot be lower than 0", ele);
				}
				else {
					try {
						this.parseState.push(new ConstructorArgumentEntry(index));
                      	// 解析 ele 对应的属性元素
						Object value = parsePropertyValue(ele, bd, null);
                      	// 使用 ConstructorArgumentValues.ValueHolder 类型封装解析出来的元素
						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
						if (StringUtils.hasLength(typeAttr)) {
							valueHolder.setType(typeAttr);
						}
						if (StringUtils.hasLength(nameAttr)) {
							valueHolder.setName(nameAttr);
						}
						valueHolder.setSource(extractSource(ele));
                      	// 不允许重复指定相同的参数
						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
							error("Ambiguous constructor-arg entries for index " + index, ele);
						}
						else {
                          	// 将封装的信息添加到当前 BeanDefinition 的 constructorArgumentValues 中的 indexedArgumentValue 属性中
							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
						}
					}
					finally {
						this.parseState.pop();
					}
				}
			}
			catch (NumberFormatException ex) {
				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
			}
		}
		else {
          	// 没有 index 属性则忽略去属性，自动寻找
			try {
				this.parseState.push(new ConstructorArgumentEntry());
              	// 解析 constructor-arg 元素
				Object value = parsePropertyValue(ele, bd, null);
				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
				if (StringUtils.hasLength(typeAttr)) {
					valueHolder.setType(typeAttr);
				}
				if (StringUtils.hasLength(nameAttr)) {
					valueHolder.setName(nameAttr);
				}
				valueHolder.setSource(extractSource(ele));
				bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
			}
			finally {
				this.parseState.pop();
			}
		}
	}
```

解析构造函数配置中子元素的过程

```java
	/**
	 * Get the value of a property element. May be a list etc.
	 * Also used for constructor arguments, "propertyName" being null in this case.
	 */
	public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
		String elementName = (propertyName != null) ?
						"<property> element for property '" + propertyName + "'" :
						"<constructor-arg> element";

		// Should only have one child element: ref, value, list, etc.
      	// 一个属性只能对应一种类型：ref、value、list等
		NodeList nl = ele.getChildNodes();
		Element subElement = null;
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
          	// 对应 description 或者 meta 不处理
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
					!nodeNameEquals(node, META_ELEMENT)) {
				// Child element is what we're looking for.
				if (subElement != null) {
					error(elementName + " must not contain more than one sub-element", ele);
				}
				else {
					subElement = (Element) node;
				}
			}
		}
		
      	// 解析 constructor-arg 上的 ref 属性
		boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
      	// 解析 constructor-arg 上的 value 属性
		boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
		if ((hasRefAttribute && hasValueAttribute) ||
				((hasRefAttribute || hasValueAttribute) && subElement != null)) {
          	/**
          		在 constructor-arg 上不存在：
          		1. 同时既有 ref 属性又有 value 属性
          		2. 存在 ref 属性或者 value 属性且又有子元素
          	**/
			error(elementName +
					" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
		}

		if (hasRefAttribute) {
          	// ref 属性的处理，使用 RuntimeBeanRefence 封装对应的 ref 名称
			String refName = ele.getAttribute(REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error(elementName + " contains empty 'ref' attribute", ele);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (hasValueAttribute) {
          	// value 属性的处理，使用 TypedStringValue 封装
			TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
			valueHolder.setSource(extractSource(ele));
			return valueHolder;
		}
		else if (subElement != null) {
          	// 解析子元素
			return parsePropertySubElement(subElement, bd);
		}
		else {
			// Neither child element nor "ref" or "value" attribute found.
          	// 即没有 ref 也没有 value 也没有子元素，报错
			error(elementName + " must specify a ref or value", ele);
			return null;
		}
	}
```

所谓子元素

```xml
<constructor-arg>
	<map>
		<entry key="key" value="value"/>
	</map>
</constructor-arg>
```



```java
	public Object parsePropertySubElement(Element ele, BeanDefinition bd) {
		return parsePropertySubElement(ele, bd, null);
	}

	/**
	 * Parse a value, ref or collection sub-element of a property or
	 * constructor-arg element.
	 * @param ele subelement of property element; we don't know which yet
	 * @param defaultValueType the default type (class name) for any
	 * {@code <value>} tag that might be created
	 */
	public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
		if (!isDefaultNamespace(ele)) {
			return parseNestedCustomElement(ele, bd);
		}
		else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
			BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
			if (nestedBd != null) {
				nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
			}
			return nestedBd;
		}
		else if (nodeNameEquals(ele, REF_ELEMENT)) {
			// A generic reference to any name of any bean.
			String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
			boolean toParent = false;
			if (!StringUtils.hasLength(refName)) {
              	 // 解析 local
				// A reference to the id of another bean in the same XML file.
				refName = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
				if (!StringUtils.hasLength(refName)) {
                  	// 解析 parent
					// A reference to the id of another bean in a parent context.
					refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
					toParent = true;
					if (!StringUtils.hasLength(refName)) {
						error("'bean', 'local' or 'parent' is required for <ref> element", ele);
						return null;
					}
				}
			}
			if (!StringUtils.hasText(refName)) {
				error("<ref> element contains empty target attribute", ele);
				return null;
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
			ref.setSource(extractSource(ele));
			return ref;
		}
      	// 对 idref 元素的解析
		else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
			return parseIdRefElement(ele);
		}
      	// 对 value 子元素的解析
		else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
			return parseValueElement(ele, defaultValueType);
		}
      	// 对 null 子元素的解析
		else if (nodeNameEquals(ele, NULL_ELEMENT)) {
			// It's a distinguished null value. Let's wrap it in a TypedStringValue
			// object in order to preserve the source location.
			TypedStringValue nullHolder = new TypedStringValue(null);
			nullHolder.setSource(extractSource(ele));
			return nullHolder;
		}
       	// 解析 array 子元素
		else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
			return parseArrayElement(ele, bd);
		}
      	// 解析 list 子元素 
		else if (nodeNameEquals(ele, LIST_ELEMENT)) {
			return parseListElement(ele, bd);
		}
      	// 解析 set 子元素
		else if (nodeNameEquals(ele, SET_ELEMENT)) {
			return parseSetElement(ele, bd);
		}
      	// 解析 map 子元素
		else if (nodeNameEquals(ele, MAP_ELEMENT)) {
			return parseMapElement(ele, bd);
		}
      	// 解析 parse 子元素
		else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
			return parsePropsElement(ele);
		}
		else {
			error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
			return null;
		}
	}
```

##### 解析子元素 property

```java
	/**
	 * Parse property sub-elements of the given bean element.
	 */
	public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}
	/**
	 * Parse a property element.
	 */
	public void parsePropertyElement(Element ele, BeanDefinition bd) {
      	// 获取配置元素中 name 的值
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		this.parseState.push(new PropertyEntry(propertyName));
		try {
          	// 不允许多次对同一属性配置
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
			Object val = parsePropertyValue(ele, bd, propertyName);
			PropertyValue pv = new PropertyValue(propertyName, val);
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			this.parseState.pop();
		}
	}
```

##### 解析子元素 qualifier

```java
	/**
	 * Parse qualifier sub-elements of the given bean element.
	 */
	public void parseQualifierElements(Element beanEle, AbstractBeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ELEMENT)) {
				parseQualifierElement((Element) node, bd);
			}
		}
	}
	/**
	 * Parse a qualifier element.
	 */
	public void parseQualifierElement(Element ele, AbstractBeanDefinition bd) {
		String typeName = ele.getAttribute(TYPE_ATTRIBUTE);
		if (!StringUtils.hasLength(typeName)) {
			error("Tag 'qualifier' must have a 'type' attribute", ele);
			return;
		}
		this.parseState.push(new QualifierEntry(typeName));
		try {
			AutowireCandidateQualifier qualifier = new AutowireCandidateQualifier(typeName);
			qualifier.setSource(extractSource(ele));
			String value = ele.getAttribute(VALUE_ATTRIBUTE);
			if (StringUtils.hasLength(value)) {
				qualifier.setAttribute(AutowireCandidateQualifier.VALUE_KEY, value);
			}
			NodeList nl = ele.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (isCandidateElement(node) && nodeNameEquals(node, QUALIFIER_ATTRIBUTE_ELEMENT)) {
					Element attributeEle = (Element) node;
					String attributeName = attributeEle.getAttribute(KEY_ATTRIBUTE);
					String attributeValue = attributeEle.getAttribute(VALUE_ATTRIBUTE);
					if (StringUtils.hasLength(attributeName) && StringUtils.hasLength(attributeValue)) {
						BeanMetadataAttribute attribute = new BeanMetadataAttribute(attributeName, attributeValue);
						attribute.setSource(extractSource(attributeEle));
						qualifier.addMetadataAttribute(attribute);
					}
					else {
						error("Qualifier 'attribute' tag must have a 'name' and 'value'", attributeEle);
						return;
					}
				}
			}
			bd.addQualifier(qualifier);
		}
		finally {
			this.parseState.pop();
		}
	}
```

#### AbstractBeanDefinition 属性

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {

	// 忽略常量
	
	private volatile Object beanClass;
	
	// bean 的作用范围，对应 bean 属性 scope
	private String scope = SCOPE_DEFAULT;

	// 是否是抽象，对应 abstract
	private boolean abstractFlag = false;

	// 是否延迟加载，对应 lazy-init
	private boolean lazyInit = false;

	// 自动注入模式，对应 autowire
	private int autowireMode = AUTOWIRE_NO;

	// 依赖检查， 3.0 弃用
	private int dependencyCheck = DEPENDENCY_CHECK_NONE;

	// 用来表示一个 bean 的实例化依靠另一个 bean 先实例化，对应 depend-on
	private String[] dependsOn;

	// autowire-candidate 属性设置为 false，这样容器在查找自动装配对象时，
	// 将不考虑该 bean，即它不会被考虑作为其他 bean 自动装配的候选者，但是该 bean 本身还是可以
	// 使用自动装配来注入其他 bean 的。
	// 对应 bean 属性 autowire-candidate
	private boolean autowireCandidate = true;

	// 自动装配时当出现多个 bean 候选者时，将作为首选者，对应 primary
	private boolean primary = false;

	// 用于记录 Qualifier，对应子元素 qualifier
	private final Map<String, AutowireCandidateQualifier> qualifiers =
			new LinkedHashMap<String, AutowireCandidateQualifier>(0);

	// 允许访问非公开的构造器和方法，程序设置
	private boolean nonPublicAccessAllowed = true;

	/**
	是否以一种宽松模式解析构造函数，默认为 true
	如果为 false，则在如下情况
	interface ITest{}
	class ITestImpl implements ITest{};
	class Main{
      Main(ITest i) {}
      Main(ITestImpl i) {}
	}
	抛出异常，因为 Spring 无法准确定位哪个构造函数
	程序设置
	**/
	private boolean lenientConstructorResolution = true;

	/**
	对应 bean 属性 factory-bean
	<bean id="instanceFactoryBean" class="xxx"/>
	<bean id="currentTime" factory-bean="instanceFactoryBean" factory-method="createTime"/>
	**/
	private String factoryBeanName;

	// 对应 factory-method
	private String factoryMethodName;

	// 记录构造函数注入属性，对于 constructor-arg
	private ConstructorArgumentValues constructorArgumentValues;

	// 普通属性集合
	private MutablePropertyValues propertyValues;

	// 方法重写的持有者，记录 lookup-method、replaced-method 元素
	private MethodOverrides methodOverrides = new MethodOverrides();

	// 初始化方法，对应 init-method
	private String initMethodName;

	// 销毁方法，对应 destory-method
	private String destroyMethodName;

	// 是否执行 init-method，程序设定
	private boolean enforceInitMethod = true;

	// 是否执行 destory-method，程序设置
	private boolean enforceDestroyMethod = true;

	// 是否是用户定义的而不是应用程序本身定义的，创建 AOP 时候为 true，程序设置
	private boolean synthetic = false;

	/**
	定义这个 bean 的应用
	APPLICATION: 用户
	INFRASTRUCTURE：完全内部使用，与用户无光
	SUPPORT: 某些复杂配置的一部分
	程序设置
	**/
	private int role = BeanDefinition.ROLE_APPLICATION;
	
	// bean 的描述信息
	private String description;

	// 这个 bean 定义的资源
	private Resource resource;
	// ...各种 set/get
}
```

#### 解析默认表情中的自定义标签元素

之前的解析标签起始函数:

```java
	/**
	 * Process the given bean element, parsing the bean definition
	 * and registering it with the registry.
	 */
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele); // 1
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);  // 2
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry()); // 3
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder)); // 4
		}
	}
```

接下来介绍 2 `bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);`

适用场景：

```xml
<bean id="test" class="test.MyClass">
	<mybean:user username="aaa" />
</bean>
```

```java
	public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder) {
		return decorateBeanDefinitionIfRequired(ele, definitionHolder, null);
	}

	// containingBd 为父类的 bean，为了使用父类的 scope 属性
	public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder definitionHolder, BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = definitionHolder;

		// Decorate based on custom attributes first.
		NamedNodeMap attributes = ele.getAttributes();
      	// 遍历所有的属性，看看是否有适用于修饰的属性
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
		NodeList children = ele.getChildNodes();
      	// 遍历所有的子节点，看看是否有使用于修饰的子元素
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}

	public BeanDefinitionHolder decorateIfRequired(
			Node node, BeanDefinitionHolder originalDef, BeanDefinition containingBd) {

      	// 获取自定义标签的命名空间
		String namespaceUri = getNamespaceURI(node);
      	// 对于非默认标签进行修饰
		if (!isDefaultNamespace(namespaceUri)) {
          	// 根据命名空间找到对应的处理器
			NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
			if (handler != null) {
              	// 进行修饰
				return handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
			}
			else if (namespaceUri != null && namespaceUri.startsWith("http://www.springframework.org/")) {
				error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
			}
			else {
				// A custom namespace, not to be handled by Spring - maybe "xml:...".
				if (logger.isDebugEnabled()) {
					logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
				}
			}
		}
		return originalDef;
	}
```

#### 注册解析的 BeanDefinition

```java
	/**
	 * Register the given bean definition with the given bean factory.
	 * @param definitionHolder the bean definition including name and aliases
	 * @param registry the bean factory to register with
	 * @throws BeanDefinitionStoreException if registration failed
	 */
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
      	// 使用 beanName 做唯一标识注册
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
      	// 注册所有的别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

##### 通过 beanName 注册 BeanDefinition

```java
	// DefaultListableBeanFactory
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
              	/**
              	注册前的最后一次校验，这里的校验不同于之前的 XML 文件校验，
              	主要是对于 AbstractBeanDefinition 属性中的 methodOverrides 校验，
              	校验 methodOverrides 是否与工厂方法并存或者 methodOverrides 对应的方法根本不存在
              	**/
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}
		
		BeanDefinition oldBeanDefinition;
		// 这个 beanDefinitionMap 是 ConcurrentHashMap，考虑并发
		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
          	// 如果对应的 BeanName 已经注册且在配置中配置了 bean 不允许被覆盖，则抛出异常
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
          	// 注册 beanDefinition
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
      	// 不属于 AbstractBeanDefinition 了
      	// 疑问，什么不属于 AbstactBeanDefinition 的 BeanDefinition 的实现？
		else {
          	// 
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
              	// 这里为什么还需要锁？
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
                  	// 这里需要锁！
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
              	// 记录 beanName
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
          	// 清楚解析之前留下的对应 beanName 的缓存
			resetBeanDefinition(beanName);
		}
	}
```

1. 对 `AbstractBeanDefinition` 的校验。在解析 XML 文件的时候我们提过校验，但是此校验非彼校验，之前的校验时针对于 XML 格式的校验，而此时的校验时针是对于 `AbstractBeanDefinition` 的 `methodOverrides` 属性的。
2. 对 `beanName` 已经注册的情况的处理。如果设置了不允许 `bean` 的覆盖，则需要抛出异常，否则直接覆盖
3. 加入 map 缓存
4. 清除解析之前留下的对应 `beanName` 的缓存

##### 通过别名注册 BeanDefinition

```java
	@Override
	public void registerAlias(String name, String alias) {
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
      	// 如果 beanName 与 alias 相同的话不记录 alias，并删除对应的 alias
		if (alias.equals(name)) {
			this.aliasMap.remove(alias);
		}
		else {
			String registeredName = this.aliasMap.get(alias);
			if (registeredName != null) {
				if (registeredName.equals(name)) {
					// An existing alias - no need to re-register
					return;
				}
              	// 如果 alias 不允许被覆盖则抛出异常
				if (!allowAliasOverriding()) {
					throw new IllegalStateException("Cannot register alias '" + alias + "' for name '" +
							name + "': It is already registered for name '" + registeredName + "'.");
				}
			}
          	// 当 A->B 存在时，若再次出现 A->C->B 时候则会抛出异常
			checkForAliasCircle(name, alias);
			this.aliasMap.put(alias, name);
		}
	}
```

1. `alias` 与 `beanName` 相同情况处理。若 `alias` 与 `beanName` 并名称相同则不需要处理并删除掉原有 `alias`。
2. `alias` 覆盖处理。若 `aliasName` 已经使用并已经指向了另一 `beanName` 则需要用户的设置进行处理。
3. `alias` 循环检查。当 A->B 存在时，若再次出现 A->C->B 时候则会抛出异常。
4. 注册 `alias`。

#### 通知监听器解析及注册完成

`// 扩展`

### alias 标签的解析

```java
	/**
	 * Process the given alias element, registering the alias with the registry.
	 */
	protected void processAliasRegistration(Element ele) {
      	// 获取 beanName
		String name = ele.getAttribute(NAME_ATTRIBUTE);
      	// 获取 alias
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
              	// 注册 alias
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
          	// 别名注册后通知监听器做相应处理
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}
```

### import 标签的解析

```java
	/**
	 * Parse an "import" element and load the bean definitions
	 * from the given resource into the bean factory.
	 */
	protected void importBeanDefinitionResource(Element ele) {
      	// 获取 resource 属性
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
      	// 如果不存在 resource 属性则不做任何处理
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}"
      	// 解析系统属性，格式如："${user.dir}"
		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

		// Discover whether the location is an absolute or relative URI
      	// 判定 location 是决定 URI 还是相对 RUI
		boolean absoluteLocation = false;
		try {
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
		}

		// Absolute or relative?
		if (absoluteLocation) {
			try {
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
          	// 如果是相对地址则根据相对地址计算出绝对地址
			try {
				int importCount;
              	// Resource 存在多个子实现类，如 VfsResource，FileSystemResource 等,
              	// 而每个 resource 的 createRelative 方式实现都不一样，所以这里先使用子类的方法尝试解析
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				if (relativeResource.exists()) {
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					actualResources.add(relativeResource);
				}
				else {
                  	// 如果解析不成功，则使用默认的解析器 ResourcePatternResolver 进行解析
					String baseLocation = getReaderContext().getResource().getURL().toString();
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
      	// 解析后进行监听器激活处理
		Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}
```

1. 获取 resource 属性所表示的路径
2. 解析路径中的系统属性，格式如 "${user.dir}"
3. 判定 location 是绝对路径还是相对路径
4. 如果是绝对路径则递归调用 bean 的解析过程，进行另一次的解析
5. 如果是相对路径则计算出绝对路径并进行解析
6. 通知监听器，解析完成

### 嵌入式 beans 标签的解析

递归调用解析 `bean`

```java
	/**
	 * Register each bean definition within the given root {@code <beans/>} element.
	 */
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
```

## 参考书籍

《Spring源码深度解析》
