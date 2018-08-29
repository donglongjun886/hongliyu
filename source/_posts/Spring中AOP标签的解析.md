---
title: Spring中AOP标签的解析
date: 2018-08-27 18:10:57
tags: [AOP,Java,Spring]
categories: Spring
---
### Spring处理AOP的源头
在DefaultBeanDefinitionDocumentReader的parseBeanDefinitions()方法中
```java
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
&lt;bean&gt;标签是默认的Namespace。如果是解析&lt;bean&gt;标签则调用parseDefaultElement()方法，这里是解析&lt;aop:config&gt;，所以调用parseCustomElement()。
### 获取NamespaceHandler
```ava
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
	String namespaceUri = getNamespaceURI(ele);
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
	if (handler == null) {
		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
		return null;
	}
	return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```
从&lt;aop:config&gt;这个Node(参数Element是Node接口的子接口)中拿到namespaceUri=http://www.springframework.org/schema/aop。
具体到AOP这个Node获取的namespaceHandler的是org.springframework.aop.config.AopNamespaceHandler。
具体到AopNamespaceHandler里面有几个parser，是用于具体标签转化的，如下：
```java
public void init() {
	// In 2.0 XSD as well as in 2.1 XSD.
	registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
	registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
	registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
	// Only in 2.0 XSD: moved to context namespace as of 2.1
	registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
}
```
调用findParserForElement()方法获取对应的Parser。
```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
	return findParserForElement(element, parserContext).parse(element, parserContext);
}
```
### bean定义加载
#### 将&lt;aop:before&gt;和&lt;aop:after&gt;转化为RootBeanDefinition
当前节点是&lt;aop:config&gt;，所以对应的Parser是ConfigBeanDefinitionParser，接下来调用parse()方法。
```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
	CompositeComponentDefinition compositeDef =
			new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
	parserContext.pushContainingComponent(compositeDef);
	configureAutoProxyCreator(parserContext, element);
	List<Element> childElts = DomUtils.getChildElements(element);
	for (Element elt: childElts) {
		String localName = parserContext.getDelegate().getLocalName(elt);
		if (POINTCUT.equals(localName)) {
			parsePointcut(elt, parserContext);
		}
		else if (ADVISOR.equals(localName)) {
			parseAdvisor(elt, parserContext);
		}
		else if (ASPECT.equals(localName)) {
			parseAspect(elt, parserContext);
		}
	}
	parserContext.popAndRegisterContainingComponent();
	return null;
}
```
&lt;aop:config&gt;下的节点是&lt;aop:aspect&gt;，执行parseAspect()方法。
```java
private void parseAspect(Element aspectElement, ParserContext parserContext) {
	String aspectId = aspectElement.getAttribute(ID);
	String aspectName = aspectElement.getAttribute(REF);
	try {
		this.parseState.push(new AspectEntry(aspectId, aspectName));
		List<BeanDefinition> beanDefinitions = new ArrayList<BeanDefinition>();
		List<BeanReference> beanReferences = new ArrayList<BeanReference>();
		List<Element> declareParents = DomUtils.getChildElementsByTagName(aspectElement, DECLARE_PARENTS);
		for (int i = METHOD_INDEX; i < declareParents.size(); i++) {
			Element declareParentsElement = declareParents.get(i);
			beanDefinitions.add(parseDeclareParents(declareParentsElement, parserContext));
		}
		// We have to parse "advice" and all the advice kinds in one loop, to get the
		// ordering semantics right.
		NodeList nodeList = aspectElement.getChildNodes();
		boolean adviceFoundAlready = false;
		for (int i = 0; i < nodeList.getLength(); i++) {
			Node node = nodeList.item(i);
			if (isAdviceNode(node, parserContext)) {
				if (!adviceFoundAlready) {
					adviceFoundAlready = true;
					if (!StringUtils.hasText(aspectName)) {
						parserContext.getReaderContext().error(
								"<aspect> tag needs aspect bean reference via 'ref' attribute when declaring advices.",
								aspectElement, this.parseState.snapshot());
						return;
					}
					beanReferences.add(new RuntimeBeanReference(aspectName));
				}
				AbstractBeanDefinition advisorDefinition = parseAdvice(
						aspectName, i, aspectElement, (Element) node, parserContext, beanDefinitions, beanReferences);
				beanDefinitions.add(advisorDefinition);
			}
		}
		AspectComponentDefinition aspectComponentDefinition = createAspectComponentDefinition(
				aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
		parserContext.pushContainingComponent(aspectComponentDefinition);
		List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
		for (Element pointcutElement : pointcuts) {
			parsePointcut(pointcutElement, parserContext);
		}
		parserContext.popAndRegisterContainingComponent();
	}
	finally {
		this.parseState.pop();
	}
}
```
上面代码中有一个isAdviceNode()方法，看下里面做了什么：
```java
/**
 * Return {@code true} if the supplied node describes an advice type. May be one of:
 * '{@code before}', '{@code after}', '{@code after-returning}',
 * '{@code after-throwing}' or '{@code around}'.
 */
private boolean isAdviceNode(Node aNode, ParserContext parserContext) {
	if (!(aNode instanceof Element)) {
		return false;
	}
	else {
		String name = parserContext.getDelegate().getLocalName(aNode);
		return (BEFORE.equals(name) || AFTER.equals(name) || AFTER_RETURNING_ELEMENT.equals(name) ||
				AFTER_THROWING_ELEMENT.equals(name) || AROUND.equals(name));
	}
}
```
调用isAdviceNode()方法的for循环处理的是&lt;aop:aspect&gt;下的&lt;aop:before&gt;、&lt;aop:after&gt;、&lt;aop:after-returning&gt;、&lt;aop:after-throwing&gt;、&lt;aop:around&gt;，如果是这5类标签则调用parseAdvice()方法。
```java
/**
 * Parses one of '{@code before}', '{@code after}', '{@code after-returning}',
 * '{@code after-throwing}' or '{@code around}' and registers the resulting
 * BeanDefinition with the supplied BeanDefinitionRegistry.
 * @return the generated advice RootBeanDefinition
 */
private AbstractBeanDefinition parseAdvice(
		String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext,
		List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {
	try {
		this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));
		// create the method factory bean
		RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
		methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
		methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
		methodDefinition.setSynthetic(true);
		// create instance factory definition
		RootBeanDefinition aspectFactoryDef =
				new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
		aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
		aspectFactoryDef.setSynthetic(true);
		// register the pointcut
		AbstractBeanDefinition adviceDef = createAdviceDefinition(
				adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef,
				beanDefinitions, beanReferences);
		// configure the advisor
		RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
		advisorDefinition.setSource(parserContext.extractSource(adviceElement));
		advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
		if (aspectElement.hasAttribute(ORDER_PROPERTY)) {
			advisorDefinition.getPropertyValues().add(
					ORDER_PROPERTY, aspectElement.getAttribute(ORDER_PROPERTY));
		}
		// register the final advisor
		parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);
		return advisorDefinition;
	}
	finally {
		this.parseState.pop();
	}
}
```
上面方法做了5件事情。

1. 创建方法工厂bean methodDefinition
2. 创建名为aspectFactoryDef的RootBeanDefinition
3. 根据织入方式创建adviceDef，并注册pointcut(将pointcut添加到beanDefinitions或beanReferences)
4. 配置advisorDefinition，将上一步的adviceDef添加到advisorDefinition
5. 将advisorDefinition注册到DefaultListableBeanFactory中

接下来看下createAdviceDefinition()方法，createAdviceDefinition()方法创建了一个RootBeanDefinition实例，解析pointcut并将它与RootBeanDefinition实例关联。
```java
/**
 * Creates the RootBeanDefinition for a POJO advice bean. Also causes pointcut
 * parsing to occur so that the pointcut may be associate with the advice bean.
 * This same pointcut is also configured as the pointcut for the enclosing
 * Advisor definition using the supplied MutablePropertyValues.
 */
private AbstractBeanDefinition createAdviceDefinition(
		Element adviceElement, ParserContext parserContext, String aspectName, int order,
		RootBeanDefinition methodDef, RootBeanDefinition aspectFactoryDef,
		List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {
	RootBeanDefinition adviceDefinition = new RootBeanDefinition(getAdviceClass(adviceElement, parserContext));
	adviceDefinition.setSource(parserContext.extractSource(adviceElement));
	adviceDefinition.getPropertyValues().add(ASPECT_NAME_PROPERTY, aspectName);
	adviceDefinition.getPropertyValues().add(DECLARATION_ORDER_PROPERTY, order);
	if (adviceElement.hasAttribute(RETURNING)) {
		adviceDefinition.getPropertyValues().add(
				RETURNING_PROPERTY, adviceElement.getAttribute(RETURNING));
	}
	if (adviceElement.hasAttribute(THROWING)) {
		adviceDefinition.getPropertyValues().add(
				THROWING_PROPERTY, adviceElement.getAttribute(THROWING));
	}
	if (adviceElement.hasAttribute(ARG_NAMES)) {
		adviceDefinition.getPropertyValues().add(
				ARG_NAMES_PROPERTY, adviceElement.getAttribute(ARG_NAMES));
	}
	ConstructorArgumentValues cav = adviceDefinition.getConstructorArgumentValues();
	cav.addIndexedArgumentValue(METHOD_INDEX, methodDef);
	Object pointcut = parsePointcutProperty(adviceElement, parserContext);
	if (pointcut instanceof BeanDefinition) {
		cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcut);
		beanDefinitions.add((BeanDefinition) pointcut);
	}
	else if (pointcut instanceof String) {
		RuntimeBeanReference pointcutRef = new RuntimeBeanReference((String) pointcut);
		cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcutRef);
		beanReferences.add(pointcutRef);
	}
	cav.addIndexedArgumentValue(ASPECT_INSTANCE_FACTORY_INDEX, aspectFactoryDef);
	return adviceDefinition;
}
```
看下上面的getAdviceClass()方法：
```java
/**
 * Gets the advice implementation class corresponding to the supplied {@link Element}.
 */
private Class<?> getAdviceClass(Element adviceElement, ParserContext parserContext) {
	String elementName = parserContext.getDelegate().getLocalName(adviceElement);
	if (BEFORE.equals(elementName)) {
		return AspectJMethodBeforeAdvice.class;
	}
	else if (AFTER.equals(elementName)) {
		return AspectJAfterAdvice.class;
	}
	else if (AFTER_RETURNING_ELEMENT.equals(elementName)) {
		return AspectJAfterReturningAdvice.class;
	}
	else if (AFTER_THROWING_ELEMENT.equals(elementName)) {
		return AspectJAfterThrowingAdvice.class;
	}
	else if (AROUND.equals(elementName)) {
		return AspectJAroundAdvice.class;
	}
	else {
		throw new IllegalArgumentException("Unknown advice kind [" + elementName + "].");
	}
}
```
根据element获取对应advice的实现类。

1. before对应AspectJMethodBeforeAdvice.class
2. after对应AspectJAfterAdvice.class
3. after-returning对应AspectJAfterReturningAdvice.class
4. after-throwing对应AspectJAfterThrowingAdvice.class
5. around对应AspectJAroundAdvice.class

#### 将RootBeanDefinition注册到DefaultListableBeanFactory中去
代码就是上面org.springframework.aop.config.ConfigBeanDefinitionParser.parseAdvice()方法的最后一步：
```java
// register the final advisor
parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);
```
```java
public String registerWithGeneratedName(BeanDefinition beanDefinition) {
	String generatedName = generateBeanName(beanDefinition);
	getRegistry().registerBeanDefinition(generatedName, beanDefinition);
	return generatedName;
}
```
上面的generateBeanName()方法获取注册的beanName，具体创建beanName的在BeanDefinitionReaderUtils的generateBeanName()方法中：
```java
public static String generateBeanName(
		BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean)
		throws BeanDefinitionStoreException {
	String generatedBeanName = definition.getBeanClassName();
	if (generatedBeanName == null) {
		if (definition.getParentName() != null) {
			generatedBeanName = definition.getParentName() + "$child";
		}
		else if (definition.getFactoryBeanName() != null) {
			generatedBeanName = definition.getFactoryBeanName() + "$created";
		}
	}
	if (!StringUtils.hasText(generatedBeanName)) {
		throw new BeanDefinitionStoreException("Unnamed bean definition specifies neither " +
				"'class' nor 'parent' nor 'factory-bean' - can't generate bean name");
	}
	String id = generatedBeanName;
	if (isInnerBean) {
		// Inner bean: generate identity hashcode suffix.
		id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + ObjectUtils.getIdentityHexString(definition);
	}
	else {
		// Top-level bean: use plain class name.
		// Increase counter until the id is unique.
		int counter = -1;
		while (counter == -1 || registry.containsBeanDefinition(id)) {
			counter++;
			id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + counter;
		}
	}
	return id;
}
```
根据代码注解，这里的beanName应该是bean的className+'#'+全局计数器的形式,其中className应该是org.springframework.aop.aspectj.AspectJPointcutAdvisor，依次类推，beanName应该是org.springframework.aop.aspectj.AspectJPointcutAdvisor#1、org.springframework.aop.aspectj.AspectJPointcutAdvisor#2、org.springframework.aop.aspectj.AspectJPointcutAdvisor#3等等。

#### 处理&lt;aop:pointcut&gt;
回到之前ConfigBeanDefinitionParser的parseAspect()方法
```java
List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
for (Element pointcutElement : pointcuts) {
	parsePointcut(pointcutElement, parserContext);
}
```
parsePointcut()方法解析pointcut,并将解析结果注册在BeanDefinitionRegistry
```java
/**
 * Parses the supplied {@code <pointcut>} and registers the resulting
 * Pointcut with the BeanDefinitionRegistry.
 */
private AbstractBeanDefinition parsePointcut(Element pointcutElement, ParserContext parserContext) {
	String id = pointcutElement.getAttribute(ID);
	String expression = pointcutElement.getAttribute(EXPRESSION);
	AbstractBeanDefinition pointcutDefinition = null;
	try {
		this.parseState.push(new PointcutEntry(id));
		pointcutDefinition = createPointcutDefinition(expression);
		pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));
		String pointcutBeanName = id;
		if (StringUtils.hasText(pointcutBeanName)) {
			parserContext.getRegistry().registerBeanDefinition(pointcutBeanName, pointcutDefinition);
		}
		else {
			pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
		}
		parserContext.registerComponent(
				new PointcutComponentDefinition(pointcutBeanName, pointcutDefinition, expression));
	}
	finally {
		this.parseState.pop();
	}
	return pointcutDefinition;
}
```
createPointcutDefinition创建了pointcut的bean定义。
```java
protected AbstractBeanDefinition createPointcutDefinition(String expression) {
	RootBeanDefinition beanDefinition = new RootBeanDefinition(AspectJExpressionPointcut.class);
	beanDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
	beanDefinition.setSynthetic(true);
	beanDefinition.getPropertyValues().add(EXPRESSION, expression);
	return beanDefinition;
}
```