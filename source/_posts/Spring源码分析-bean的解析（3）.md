---
title: Spring-源码分析-bean的解析(3)
date: 2017-11-12 16:58:42
tags:
    - Spring
    - Java
categories: 源码分析
---

>  当前版本 Spring 4.3.8

## 自定义标签的解析

### 自定义标签使用

在很多情况下，当使用默认配置过于繁琐时候，解析工作或许是一个不得不考虑的负担。Spring 提供了可扩展 Scheme 的支持。大概需要以下几个步骤：

<!-- more -->

1. 创建一个需要扩展的组件

   ```java
   package io.github.binglau.bean;

   import lombok.Data;

   /**
    * 文件描述:
    */

   @Data
   public class User {
       private String userName;
       private String email;
   }

   ```

2. 定义一个 XSD 文件描述组件内容

   ```java
   <?xml version="1.0" encoding="UTF-8" ?>
   <xsd:schema xmlns="http://www.binglau.com/schema/user"
               xmlns:xsd="http://www.w3.org/2001/XMLSchema"
               targetNamespace="http://www.binglau.com/schema/user"
               elementFormDefault="qualified">

       <xsd:element name="user">
           <xsd:complexType>
               <xsd:attribute name="id" type="xsd:string"/>
               <xsd:attribute name="userName" type="xsd:string"/>
               <xsd:attribute name="email" type="xsd:string"/>
           </xsd:complexType>
       </xsd:element>
   </xsd:schema>
   ```

3. 创建一个文件，实现 `BeanDefinitionParser` 接口，用来解析 XSD 文件中的定义和组件定义

   ```java
   package io.github.binglau;

   import io.github.binglau.bean.User;
   import org.springframework.beans.factory.support.BeanDefinitionBuilder;
   import org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser;
   import org.springframework.util.StringUtils;
   import org.w3c.dom.Element;

   /**
    * 文件描述:
    */

   public class UserBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
       // Element 对应的类
       @Override
       protected Class<?> getBeanClass(Element element) {
           return User.class;
       }

       // 从 element 中解析并提取对应的元素
       @Override
       protected void doParse(Element element, BeanDefinitionBuilder builder) {
           String userName = element.getAttribute("userName");
           String email = element.getAttribute("email");
           // 将提取的数据放入 BeanDefinitionBuilder 中，待到完成所有 bean 的解析后统一注册到 beanFactory 中
           if (StringUtils.hasText(userName)) {
               builder.addPropertyValue("userName", userName);
           }
           if (StringUtils.hasText(email)) {
               builder.addPropertyValue("email", email);
           }
       }
   }

   ```

4. 创建一个 Handler 文件，扩展自 NamespaceHandlerSupport，目的是将组件注册到 Spring 容器

   ```java
   package io.github.binglau;

   import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

   /**
    * 文件描述:
    */

   public class MyNamespaceHandler extends NamespaceHandlerSupport {
       @Override
       public void init() {
           registerBeanDefinitionParser("user", new UserBeanDefinitionParser());
       }
   }

   ```

5. 编写 Spring.handlers 和 Spring.schemas 文件（resources/MATE-INF）

   ```shell
   # Spring.handlers
   http\://www.binglau.com/schema/user=io.github.binglau.MyNamespaceHandler

   # Spring.schemas
   http\://www.binglau.com/schema/user.xsd=user-xsd.xsd
   ```

测试：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:myname="http://www.binglau.com/schema/user"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
       http://www.binglau.com/schema/user http://www.binglau.com/schema/user.xsd">

    <myname:user id="testbean" userName="aaa" email="bbb"/>

    <bean id="testBean" class="io.github.binglau.bean.TestBean" />

</beans>
```

```java
package io.github.binglau;

import io.github.binglau.bean.User;
import org.junit.Test;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.xml.XmlBeanFactory;
import org.springframework.core.io.ClassPathResource;

/**
 * 文件描述:
 */

public class BeanFactoryTest {
    @Test
    public void testSimpleLoad() {
        BeanFactory context = new XmlBeanFactory(new ClassPathResource("beanFactory.xml"));
        User user = (User)context.getBean("testbean");
        System.out.println(user);
    }
}

/**
结果：
User(userName=aaa, email=bbb)
**/
```



### 自定义标签解析

```java
	/**
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
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

其中的 `delegate.parseCustomElement(ele)` 既是对自定义标签的解析

`BeanDefinitionParserDelegate.parseCustomElement`

```java
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}
	// containingBd 为父类 bean，对顶层元素的解析应设置为 null
	public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
      	// 获取对应的命名空间
		String namespaceUri = getNamespaceURI(ele);
      	// 根据命名空间找到对应的 NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
      	// 调用自定义的 NamespaceHandler 进行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```

#### 获取标签的命名空间

直接调用 `org.w3c.dom.Node` 中的方法

```java
	public String getNamespaceURI(Node node) {
		return node.getNamespaceURI();
	}
```

#### 提取自定义标签处理器

`private final XmlReaderContext readerContext` 即 `new ClassPathResource("beanFactory.xml")`

在 `readerContext` 初始化的时候其属性 `namespaceHandlerResolver` 已经被初始化为 `DefaultNamespaceHandlerResolver` 实例，所以，这里实际调用了 `DefaultNamespaceHandlerResolver `的方法。

```java
	/**
	 * Locate the {@link NamespaceHandler} for the supplied namespace URI
	 * from the configured mappings.
	 * @param namespaceUri the relevant namespace URI
	 * @return the located {@link NamespaceHandler}, or {@code null} if none found
	 */
	@Override
	public NamespaceHandler resolve(String namespaceUri) {
      	// 获取所有已有配置的 handler 映射
		Map<String, Object> handlerMappings = getHandlerMappings();
      	// 根据命名空间找到对应的信息
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
          	// 已经做过解析的情况，直接从缓存读取
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
          	// 没有做过解析，则返回的是类路径
			String className = (String) handlerOrClassName;
			try {
              	// 使用反射 将类路径转换为类
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
              	// 初始化类
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
              	// 调用自定义的 NamespaceHandler 的初始化方法
				namespaceHandler.init();
              	// 记录在缓存
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("NamespaceHandler class [" + className + "] for namespace [" +
						namespaceUri + "] not found", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Invalid NamespaceHandler class [" + className + "] for namespace [" +
						namespaceUri + "]: problem with handler class file or dependent class", err);
			}
		}
	}
```

在上面的调用`namespaceHandler.init();`中参考之前的自定义标签使用

```java
    public void init() {
        registerBeanDefinitionParser("user", new UserBeanDefinitionParser());
    }
```

当得到自定义命名空间处理后回马上进行 `BeanDefinitionParser` 的注册以支持自定义标签

注册后，命名空间处理器就可以根据标签的不同来调用不同的解析器进行解析。`getHandlerMappings` 主要功能就是读取 `Spring.handlers` 配置文件并将配置文件缓存在 map 中。

```java
	/**
	 * Load the specified NamespaceHandler mappings lazily.
	 */
	private Map<String, Object> getHandlerMappings() {
      	// 如果没有被缓存则开始进行缓存
		if (this.handlerMappings == null) {
			synchronized (this) {
				if (this.handlerMappings == null) {
					try {
                      	// this.handlerMappingsLocation 在构造函数中被初始化为 META-INF/Spring.handlers
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
                      	// 将 Properties 格式文件合并到 Map 格式的 handlerMappings 中
						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return this.handlerMappings;
	}
```

#### 标签解析

`NamespaceHandlerSupport#parse`

```java
	/**
	 * Parses the supplied {@link Element} by delegating to the {@link BeanDefinitionParser} that is
	 * registered for that {@link Element}.
	 */
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
      	// 寻找解析器并进行解析操作
		return findParserForElement(element, parserContext).parse(element, parserContext);
	}
	/**
	 * Locates the {@link BeanDefinitionParser} from the register implementations using
	 * the local name of the supplied {@link Element}.
	 */
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
      	// 获取元素名称，也就是 <myname:user> 中的 user，若在上面的示例中，则此时 localName 为 user
		String localName = parserContext.getDelegate().getLocalName(element);
      	// 根据 user 找到对应的解析器，也就是在 
      	// registerBeanDefinitionParser("user", new UserBeanDefinitionParser()) 注册的解析器
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```

而对于 parse 方法的处理

`AbstractBeanDefinitionParser#parse`

```java
	@Override
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
      	// 真正的解析工作
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
              	// 将 AbstractBeanDefinition 转换为 BeanDefinitionHolder 并注册
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
                  	// 需要通知监听器则进行处理
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				parserContext.getReaderContext().error(ex.getMessage(), element);
				return null;
			}
		}
		return definition;
	}
```

`AbstractSingleBeanDefinitionParser#parseInternal`

```java
	/**
	 * Creates a {@link BeanDefinitionBuilder} instance for the
	 * {@link #getBeanClass bean Class} and passes it to the
	 * {@link #doParse} strategy method.
	 * @param element the element that is to be parsed into a single BeanDefinition
	 * @param parserContext the object encapsulating the current state of the parsing process
	 * @return the BeanDefinition resulting from the parsing of the supplied {@link Element}
	 * @throws IllegalStateException if the bean {@link Class} returned from
	 * {@link #getBeanClass(org.w3c.dom.Element)} is {@code null}
	 * @see #doParse
	 */
	@Override
	protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);
		if (parentName != null) {
			builder.getRawBeanDefinition().setParentName(parentName);
		}
      	// 获取自定义标签中的 class，此时会调用自定义解析器如 UserBeanDefinitionParser 中的 getBeanClass 方法
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
          	// 若子类没有重写 getBeanClass 方法则尝试检查子类是否重写 getBeanClassName 方法
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
		if (parserContext.isNested()) {
          	// 若存在父类则使用父类的 scope 属性
			// Inner bean definition must receive same scope as containing bean.
			builder.setScope(parserContext.getContainingBeanDefinition().getScope());
		}
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
          	// 配置延迟加载
			builder.setLazyInit(true);
		}
      	// 调用子类重写的 doParse 方法进行解析
		doParse(element, parserContext, builder);
		return builder.getBeanDefinition();
	}

	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		doParse(element, builder);
	}
```

## 参考书籍

《Spring源码深度解析》
