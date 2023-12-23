---
title: Spring-源码分析-bean的解析(1)
date: 2017-10-25 16:57:19
tags:
    - Spring
    - Java
categories: 源码分析
---

>  当前版本 Spring 4.3.8

我们一开始需要先定义一个 `Bean` 和一个 `xml`

<!-- more -->

```Java
// bean
package io.github.binglau.bean;

import lombok.Data;

/**
 * 文件描述:
 */

@Data // 简化 setter/getter
public class TestBean {
    private String testStr = "test";
}
```

```xml
<!-- beanFactory.xml -->
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd">

    <bean id="testBean" class="io.github.binglau.bean.TestBean" />

</beans>
```

这时候我们大多数是这么来启动 IoC 的

```Java
package io.github.binglau;

import io.github.binglau.bean.TestBean;
import org.junit.Test;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.core.io.ClassPathResource;

/**
 * 文件描述:
 */

public class BeanFactoryTest {
    @Test
    public void testSimpleLoad() {
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:beanFactory.xml");
        TestBean testBean = (TestBean) context.getBean("testBean");
        System.out.printf("test bean: %s", testBean.getTestStr());
    }
}
```

现在让我们来看看 `ApplicationContext` 代表了什么

```Java
package org.springframework.context;

import org.springframework.beans.factory.HierarchicalBeanFactory;
import org.springframework.beans.factory.ListableBeanFactory;
import org.springframework.beans.factory.config.AutowireCapableBeanFactory;
import org.springframework.core.env.EnvironmentCapable;
import org.springframework.core.io.support.ResourcePatternResolver;

public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {

	String getId();

	String getApplicationName();

	String getDisplayName();

	long getStartupDate();

	ApplicationContext getParent();

	AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;

}
```

其实所谓的 `getBean` 方法是定义在 `ListableBeanFactory` 接口所继承的 `BeanFactory` 接口中的。这样说来，我们应该是可以直接通过 `BeanFactory` 来调用 `Bean` 的，其实有一个根据 `xml` 来实现的 `BeanFactory` ，是这样调用的：

```Java
package io.github.binglau;

import io.github.binglau.bean.TestBean;
import org.junit.Test;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.core.io.ClassPathResource;

/**
 * 文件描述:
 */

public class BeanFactoryTest {
    @Test
    public void testSimpleLoad() {
        BeanFactory context = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));
//        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:beanFactory.xml");
        TestBean testBean = (TestBean) context.getBean("testBean");
        System.out.printf("test bean: %s", testBean.getTestStr());
    }
}
```

好了，现在让我们开始进入 `XmlBeanFactory` 中来分析一下吧

## XmlBeanFactory

```Java
package org.springframework.beans.factory.xml;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.core.io.Resource;


@Deprecated
@SuppressWarnings({"serial", "all"})
public class XmlBeanFactory extends DefaultListableBeanFactory {

	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}

	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
	}

}
```

这里我们可以看出，他是继承了 `DefaultListableBeanFactory`，而  `DefaultListableBeanFactory` 也是整个 bean 加载的核心部分，是 Spring 注册及加载 bean 的默认实现。我们先从全局角度来了解一下它

![容器加载相关类图](https://github.com/BingLau7/blog/blob/master/images/blog_29/%E5%AE%B9%E5%99%A8%E5%8A%A0%E8%BD%BD%E7%9B%B8%E5%85%B3%E7%B1%BB%E5%9B%BE.png?raw=true)

### `DefaultListableBeanFactory` 的各个类功能：

-  AliasRegistry: 定义对 alias 的简单增删改等操作
-  SimpleAliasRegistry: 主要使用 map 作为 alias 的缓存，并对接口 AliasRegistry 进行实现
-  SingletonBeanRegistry: 定义对单例的注册及获取
-  BeanFactory: 定义获取 bean 及 bean 的各种属性
-  DefaultSingletonBeanRegistry: 对接口 SingletonBeanRegistry 各函数的实现
-  HierarchicalBeanFactory: 继承 BeanFactory，也就是在 BeanFactory 定义的功能的基础上增加了对 parentFactory 的支持
-  BeanDefinitionRegistry: 定义对 BeanDefinition 的各种增删改操作
-  FactoryBeanRegistrySupport: 在 DefaultSingletonBeanRegistry 基础上增加了对 FactoryBean 的特殊处理功能
-  ConfigurableBeanFactory: 提供配置 Factory 的各种方法
-  ListableBeanFactory: 根据各种条件获取 bean 的配置清单
-  AbstractBeanFactory: 综合 FactoryBeanRegistrySupport 和 ConfigurableBeanFactory 的功能
-  AutowireCapableBeanFactory: 提供创建 bean、自动注入、初始化以及应用 bean 的后处理器
-  AbstractAutowireCapableBeanFactory: 综合 AbstractBeanFactory 并对接口 AutowireCapableBeanFactory 进行实现
-  ConfigurableListableBeanFactory: BeanFactory 配置清单，指定忽略类型及接口等
-  DefaultListableBeanFactory: 综合上面的所有功能，主要是对 Bean 注册后的处理

`XmlFactoryBean`  主要是针对 XML 文档对 `DefaultListableBeanFactory` 的个性化实现，唯一不同的也就是 `XmlBeanDefinitionReader` 类型的 `reader`。

### ` XmlBeanDefinitionReader`

其中我们看到 `XmlFactoryBean` 构造器中第二句 `this.reader.loadBeanDefinitions(resource);`，但从名字来看它应该是加载 Bean 的主要执行者，而 reader 的定义在顶上 `private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);` 。所谓 `XmlBeanDefinitionReader` 主要是负责读取 Spring 配置文件信息。

![配置文件读取相关类图](https://github.com/BingLau7/blog/blob/master/images/blog_29/%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E8%AF%BB%E5%8F%96%E7%9B%B8%E5%85%B3%E7%B1%BB%E5%9B%BE.png?raw=true)

这张图 Idea 生成有些许差别，于是就自己画了一下。

-  ResourceLoader：定义资源加载器，主要应用于根据给定的资源文件地址返回对应的 Resource
-  BeanDefinitionReader: 主要定义资源文件读取并转换为 BeanDefinition 的各个功能
-  EnvironmentCapable：定义获取 Environment 方法
-  DocumentLoader：定义从资源文件加载到转换为 Document 的功能
-  AbstractBeanDefinitionReader：对 EnvironmentCapable、BeanDefinitionReader 类定义的功能进行实现
-  BeanDefinitionDocumentReader：定义读取 Document 并注册 BeanDefinition 功能
-  BeanDefinitionParserDelegate: 定义解析 Element 的各种方法

经过上面的分析，我们大概能得出 XML 配置文件读取的大致流程：

1. 通过继承自 AbstractBeanDefinitionReader 中的方法，来使用 ResourceLoader 将资源文件路径转换为对应的 Resource 文件
2. 通过 DocumentLoader 对 Resource 文件进行转换，将 Resource 文件转换为 Document 文件
3. 通过实现接口 BeanDefinitionDocumentReader 的 DefaultBeanDefinitionDocumentReader 类对 Document 进行解析，并使用 BeanDefinitionParserDelegate 对 Element 进行解析

### 分析 XmlBeanFactoy

这时候，让我们在回头看 `BeanFactory context = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));`

先从现有信息可以整理出下面这张时序图

![XmlFactoryBean时序图](https://github.com/BingLau7/blog/blob/master/images/blog_29/XmlFactoryBean%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true)

#### 配置文件封装

首先看看 `Resource` 配置文件的加载，也就是 `new ClassPathResource("beanFactory.xml")`

在 Java 中，将不同来源的资源抽象成 URL，通过注册不同的 `handler(URLStreamHandler)`来处理不同来源的资源的读取逻辑，一般 handler 的类型使用不同前缀（协议，Protocol）来识别，如 “file:”、“http:”、“jar:” 等，然而 URL 没有默认定义相对 Classpath 或 ServletContext 等资源的 handler，虽然可以注册自己的 URLStreamHandler 来解析特定的URL前缀（协议），比如 “classpath:” ，然而这需要了解 URL 的实现机制，而且URL也没有提供一些基本的方法，如检查当前资源是否存在、检查当前资源是否可读等方法。 因而 Spring 对其内部使用到的资源实现了自己的抽象结构：Resource接口来封装底层资源。

其内部资源的抽象结构：

```Java
public interface InputStreamSource {

	/**
	 * Return an {@link InputStream} for the content of an underlying resource.
	 * <p>It is expected that each call creates a <i>fresh</i> stream.
	 * <p>This requirement is particularly important when you consider an API such
	 * as JavaMail, which needs to be able to read the stream multiple times when
	 * creating mail attachments. For such a use case, it is <i>required</i>
	 * that each {@code getInputStream()} call returns a fresh stream.
	 * @return the input stream for the underlying resource (must not be {@code null})
	 * @throws java.io.FileNotFoundException if the underlying resource doesn't exist
	 * @throws IOException if the content stream could not be opened
	 */
	InputStream getInputStream() throws IOException;
}

/**
InputSteramSource 抽象了所有 Spring 内部使用到的底层资源：File、URL、Classpath 下的资源和 ByteArray 等、它只有一个方法定义：getInputStream()，该方法返回一个新的 InputStream 对象。
**/
public interface Resource extends InputStreamSource {

	/**
	 * 存在性
	 */
	boolean exists();

	/**
	 * 可读性
	 */
	boolean isReadable();

	/**
	 * 是否处于打开状态
	 */
	boolean isOpen();

	URL getURL() throws IOException;

	URI getURI() throws IOException;

	File getFile() throws IOException;

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	/**
	 * 基于当前资源创建一个相对资源的方法
	 */
	Resource createRelative(String relativePath) throws IOException;

	String getFilename();

	/**
	 * 用于错误处理中的打印信息
	 */
	String getDescription();
}
```

对于不同来源的资源都有其对应的实现：

![资源文件处理相关类图](https://github.com/BingLau7/blog/blob/master/images/blog_29/%E8%B5%84%E6%BA%90%E6%96%87%E4%BB%B6%E5%A4%84%E7%90%86%E7%9B%B8%E5%85%B3%E7%B1%BB%E5%9B%BE.png?raw=true)

其中 `ClassPathResouce` 的实现

```Java
	/**
	 * This implementation opens an InputStream for the given class path resource.
	 * @see java.lang.ClassLoader#getResourceAsStream(String)
	 * @see java.lang.Class#getResourceAsStream(String)
	 */
	@Override
	public InputStream getInputStream() throws IOException {
		InputStream is;
		if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
		if (is == null) {
			throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
		}
		return is;
	}
```

在 `XmlFactoryBean` 中，我们先调用了 `super`，现在看看 `super` 干了什么事情

```Java
	/**
	 * Create a new AbstractAutowireCapableBeanFactory.
	 */
	public AbstractAutowireCapableBeanFactory() {
		super();
		ignoreDependencyInterface(BeanNameAware.class);
		ignoreDependencyInterface(BeanFactoryAware.class);
		ignoreDependencyInterface(BeanClassLoaderAware.class);
	}
```

这里有必要提及一下 `ignoreDependencyInterface` 方法。`ignoreDependencyInterface` 的主要功能是忽略给定接口的自动装配功能，那么，这样做的目的是什么呢？会产生什么样的效果呢？

举例来说，当 A 中有属性 B，那么当 Spring 在获取 A 的 Bean 的时候如果其属性 B 还没有初始化，那么 Spring 会自动初始化 B，这也是 Spring 中提供的一个重要特性。但是，某些情况下，B 不会被初始化，其中的一种情况就是 B 实现了 BeanNameAware 接口。Spring 中是这样介绍的：自动装配时忽略给定的依赖接口，典型应用是通过其他方式解析 Application 上下文注册依赖，类似于 BeanFactory 通过 BeanFactoryAware 进行注入或者 ApplicationContext 通过 ApplicationContextAware 进行注入。

#### 加载 Bean

`this.reader.loadBeanDefinitions(resource)` 的讲解，其时序图

![loadBeanDefinitions函数执行时序图](https://github.com/BingLau7/blog/blob/master/images/blog_29/loadBeanDefinitions%E5%87%BD%E6%95%B0%E6%89%A7%E8%A1%8C%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true)

1. 封装资源文件。当进入 `XmlBeanDefinitionReader` 后首先对参数 `Resource` 使用 `EncodedResource` 类进行封装。
2. 获取输入流。从 `Resource` 中获取对应的 `InputStream` 并构造 `InputSource`
3. 通过构造的 `InputSource` 实例和 `Resource` 实例继续调用函数 `doLoadBeanDefinitions`

##### 时序图中的逻辑实现

```Java
	/**
	 * Load bean definitions from the specified XML file.
	 * @param encodedResource the resource descriptor for the XML file,
	 * allowing to specify an encoding to use for parsing the file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 */
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}
		// 通过属性来记录已经加载的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
             // 从 encodedResource 中获取已经封装的 Resource 对象并再次从 Resource 中获取其中的 inputStream
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
              	 // InputSource 这个类并不来自于 Spring，它的全路径是 org.xml.sax.InputSource
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                 // 真正进入逻辑核心部分
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

```Java
/**
	 * Actually load bean definitions from the specified XML file.
	 * @param inputSource the SAX InputSource to read from
	 * @param resource the resource descriptor for the XML file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 * @see #doLoadDocument
	 * @see #registerBeanDefinitions
	 */
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			Document doc = doLoadDocument(inputSource, resource);
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}

	/**
	 * Actually load the specified document using the configured DocumentLoader.
	 * @param inputSource the SAX InputSource to read from
	 * @param resource the resource descriptor for the XML file
	 * @return the DOM Document
	 * @throws Exception when thrown from the DocumentLoader
	 * @see #setDocumentLoader
	 * @see DocumentLoader#loadDocument
	 */
	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
```

`doLoadBeanDefinitions` 中做了三件事：

1. 获取对 XML 文件的验证模型
2. 加载 XML 文件，并得到对应的 Document
3. 根据返回的 Document 注册 Bean 信息

### 获取 XML 的验证模式

#### DTD 与 XSD 区别

DTD （文档类型定义），保证 XML 文档格式正确的有效方法，可以通过比较 XML 文档和 DTD 文件来看文档是否符合规范，元素和标签使用是否正确。一个 DTD 文档包含：元素的定义规则，元素间关系的定义规则，元素可使用的属性，可使用的实体或符号规则。

XSD （XML Schemas Definition）。XML Scheme 描述了 XML 文档的结构。可以用一个指定的 XML Schema 来验证某个 XML 文档，以检查该 XML 文档是否符合其要求。文档设计者可以通过 XML  Schema 指定一个 XML 文档所允许的结构和内容，并可据此检查一个 XML 文档是否是有效的。XML Schema 本身是一个 XML 文档，它符合 XML 语法结构。可以用通用的 XML 解析器解析它。

在使用 XML Schema 文档中 XML 实例文档进行检验，除了要声明名称空间外（xmlns = http://www.SpringFramework.org/schema/beans），还必须制定该名称空间所对应的 XML Schema 文档的存储位置。通过 schemaLocation 熟悉来指定名称空间所对应的 XML Schema 文档的存储位置，它包含两个部分，一部分是名称空的 URI，另一部分就是该名称空间所标识的 XML Shema 文件位置或 URL 地址(xsi:schemaLocation="http://www.Springframework.org/schema/beans http://www.Springframework.org/schema/beans/Spring-beans.xsd")

#### 验证模式的获取

进入 `Document doc = doLoadDocument(inputSource, resource);`方法

```Java
	protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
	}
```

其中`getValidationModeForResource(resource)`就是获取验证模式。

```Java
	/**
	 * Gets the validation mode for the specified {@link Resource}. If no explicit
	 * validation mode has been configured then the validation mode is
	 * {@link #detectValidationMode detected}.
	 * <p>Override this method if you would like full control over the validation
	 * mode, even when something other than {@link #VALIDATION_AUTO} was set.
	 */
	protected int getValidationModeForResource(Resource resource) {
		int validationModeToUse = getValidationMode();
         // 如果手动指定了验证模式则使用指定的验证模式
		if (validationModeToUse != VALIDATION_AUTO) {
			return validationModeToUse;
		}
         // 如果未指定则使用自动检测
		int detectedMode = detectValidationMode(resource);
		if (detectedMode != VALIDATION_AUTO) {
			return detectedMode;
		}
		// Hmm, we didn't get a clear indication... Let's assume XSD,
		// since apparently no DTD declaration has been found up until
		// detection stopped (before finding the document's root tag).
		return VALIDATION_XSD;
	}
```

`detectValidationMode` 将自动检测验证模式工作委派给专门处理类 `XmlValidationModeDetecotor`，调用了 `XmlValidationModeDetecotor` 的 `vaildationModeDetector` 方法，具体代码如下：

```Java
	/**
	 * Detects which kind of validation to perform on the XML file identified
	 * by the supplied {@link Resource}. If the file has a {@code DOCTYPE}
	 * definition then DTD validation is used otherwise XSD validation is assumed.
	 * <p>Override this method if you would like to customize resolution
	 * of the {@link #VALIDATION_AUTO} mode.
	 */
	protected int detectValidationMode(Resource resource) {
		if (resource.isOpen()) {
			throw new BeanDefinitionStoreException(
					"Passed-in Resource [" + resource + "] contains an open stream: " +
					"cannot determine validation mode automatically. Either pass in a Resource " +
					"that is able to create fresh streams, or explicitly specify the validationMode " +
					"on your XmlBeanDefinitionReader instance.");
		}

		InputStream inputStream;
		try {
			inputStream = resource.getInputStream();
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Unable to determine validation mode for [" + resource + "]: cannot open InputStream. " +
					"Did you attempt to load directly from a SAX InputSource without specifying the " +
					"validationMode on your XmlBeanDefinitionReader instance?", ex);
		}

		try {
			return this.validationModeDetector.detectValidationMode(inputStream);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("Unable to determine validation mode for [" +
					resource + "]: an error occurred whilst reading from the InputStream.", ex);
		}
	}
```

```Java
	/**
	 * Detect the validation mode for the XML document in the supplied {@link InputStream}.
	 * Note that the supplied {@link InputStream} is closed by this method before returning.
	 * @param inputStream the InputStream to parse
	 * @throws IOException in case of I/O failure
	 * @see #VALIDATION_DTD
	 * @see #VALIDATION_XSD
	 */
	public int detectValidationMode(InputStream inputStream) throws IOException {
		// Peek into the file to look for DOCTYPE.
		BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
		try {
			boolean isDtdValidated = false;
			String content;
			while ((content = reader.readLine()) != null) {
				content = consumeCommentTokens(content);
                  // 如果读取的行是空或者是注释则略过
				if (this.inComment || !StringUtils.hasText(content)) {
					continue;
				}
				if (hasDoctype(content)) {
					isDtdValidated = true;
					break;
				}
                  // 读取到 < 开始符号，验证模式一定会在开始符号之前
				if (hasOpeningTag(content)) {
					// End of meaningful data...
					break;
				}
			}
			return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
		}
		catch (CharConversionException ex) {
			// Choked on some character encoding...
			// Leave the decision up to the caller.
			return VALIDATION_AUTO;
		}
		finally {
			reader.close();
		}
	}

	/**
	 * Does the content contain the DTD DOCTYPE declaration?
	 */
	private boolean hasDoctype(String content) {
		return content.contains(DOCTYPE);
	}
```

### 获取 Document

```Java
	/**
	 * Load the {@link Document} at the supplied {@link InputSource} using the standard JAXP-configured
	 * XML parser.
	 */
	@Override
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isDebugEnabled()) {
			logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		return builder.parse(inputSource);
	}
```

`EntityResolver` 解释： 如果 SAX 应用程序需要实现自定义处理外部实体，则必须实现此接口并使用  `setEntityResolver` 方法向 SAX 驱动器注册一个实例。即防止下载 DTD 时候网络错误，提供一个寻找 DTD 声明的方法。

### 解析及注册 BeanDefinitions

```Java
	/**
	 * Register the bean definitions contained in the given DOM document.
	 * Called by {@code loadBeanDefinitions}.
	 * <p>Creates a new instance of the parser class and invokes
	 * {@code registerBeanDefinitions} on it.
	 * @param doc the DOM document
	 * @param resource the resource descriptor (for context information)
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of parsing errors
	 * @see #loadBeanDefinitions
	 * @see #setDocumentReaderClass
	 * @see BeanDefinitionDocumentReader#registerBeanDefinitions
	 */
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
         // 使用 DefaultBeanDefinitionDocumentReader 实例化 BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
         // 在实例化 BeanDefinitionReader 时候会将 BeanDefinitionRegistry 传入，默认使用继承 DefaultListableBeanFactory 的子类
         // 记录统计前 BeanDefinition 的加载个数
		int countBefore = getRegistry().getBeanDefinitionCount();
         // 加载及注册 bean
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
         // 记录本次加载的 BeanDefinition 个数
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}

	/**
	 * This implementation parses bean definitions according to the "spring-beans" XSD
	 * (or DTD, historically).
	 * <p>Opens a DOM Document; then initializes the default settings
	 * specified at the {@code <beans/>} level; then parses the contained bean definitions.
	 */
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
         // 重点：提取 root
		doRegisterBeanDefinitions(root);
	}
```

```Java
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
         // 专门处理解析
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
             // 处理 profile 属性
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
         // 解析前处理，留给子类实现
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
         // 解析后处理，留给子类实现
		postProcessXml(root);

		this.delegate = parent;
	}

	/**
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
         // 对 beans 的处理
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                          // 默认标签解析
						parseDefaultElement(ele, delegate);
					}
					else {
                          // 自定义标签解析
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

## 参考书籍

《Spring源码深度解析》
