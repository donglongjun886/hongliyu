---
title: Spring中bean的创建
date: 2018-08-22 16:31:13
tags: [Java,Spring]
categories: Spring
---
### 模板方法createBean()
AbstractAutowireCapableBeanFactory的createBean方法实现了AbstractBeanFactory的createBean()方法。
```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}
```
执行完resolveBeforeInstantiation方法后，程序有两种选择，如果重写了InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation()方法并在方法中改变了bean，则可以直接返回，否则要调用doCreateBean()进行常规bean的创建。
```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
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
```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName)
			throws BeansException {

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

### 创建常规bean
#### 创建实例
调用createBeanInstance()方法利用反射通过构造函数实例化bean
```java
if (instanceWrapper == null) {
	instanceWrapper = createBeanInstance(beanName, mbd, args);
}
```

#### postProcessMergedBeanDefinition
调用applyMergedBeanDefinitionPostProcessors()方法来执行MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition()方法
```java
// Allow post-processors to modify the merged bean definition.
synchronized (mbd.postProcessingLock) {
	if (!mbd.postProcessed) {
		applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
		mbd.postProcessed = true;
	}
}
```
```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName)
			throws BeansException {

	try {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
	catch (Exception ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Post-processing failed of bean type [" + beanType + "] failed", ex);
	}
}
```

#### 调用populateBean()方法
```java
populateBean(beanName, mbd, instanceWrapper);
```
执行InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation()方法来决定是否进行属性赋值（依赖注入）
```java
// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
// state of the bean before properties are set. This can be used, for example,
// to support styles of field injection.
boolean continueWithPropertyPopulation = true;

if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
				continueWithPropertyPopulation = false;
				break;
			}
		}
	}
}

if (!continueWithPropertyPopulation) {
	return;
}
```
收集属性名称及相应的对象到pvs
```java
if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
		mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
	MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

	// Add property values based on autowire by name if applicable.
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
		autowireByName(beanName, mbd, bw, newPvs);
	}

	// Add property values based on autowire by type if applicable.
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
		autowireByType(beanName, mbd, bw, newPvs);
	}

	pvs = newPvs;
}
```
执行applyPropertyValues()方法将依赖注入bean属性
```java
applyPropertyValues(beanName, mbd, bw, pvs);
```

#### 调用initializeBean()方法
```java
// Initialize the bean instance.
Object exposedObject = bean;
try {
	populateBean(beanName, mbd, instanceWrapper);
	if (exposedObject != null) {
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
}
```
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
		invokeAwareMethods(beanName, bean);
	}
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}
	try {
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```
调用invokeAwareMethods()方法，如果bean实现了BeanNameAware、BeanClassLoaderAware、BeanFactoryAware接口，则会执行相应的setBeanName()、setBeanClassLoader()、setBeanFactory()方法。
```java
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
调用applyBeanPostProcessorsBeforeInitialization()方法执行BeanPostProcessor的postProcessBeforeInitialization()方法
```java
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
调用invokeInitMethods()方法，如果bean实现了InitializingBean接口则调用afterPropertiesSet()方法。
```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
		throws Throwable {
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
			((InitializingBean) bean).afterPropertiesSet();
		}
	}
	if (mbd != null) {
		String initMethodName = mbd.getInitMethodName();
		if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
```
通过调用invokeCustomInitMethod()方法执行指定的初始化方法（属性ini-method指定的方法）。
```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
	String initMethodName = mbd.getInitMethodName();
	final Method initMethod = (mbd.isNonPublicAccessAllowed() ?
			BeanUtils.findMethod(bean.getClass(), initMethodName) :
			ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
	if (initMethod == null) {
		if (mbd.isEnforceInitMethod()) {
			throw new BeanDefinitionValidationException("Couldn't find an init method named '" +
					initMethodName + "' on bean with name '" + beanName + "'");
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("No default init method named '" + initMethodName +
						"' found on bean with name '" + beanName + "'");
			}
			// Ignore non-existent default lifecycle methods.
			return;
		}
	}
	if (logger.isDebugEnabled()) {
		logger.debug("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
	}
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
			@Override
			public Object run() throws Exception {
				ReflectionUtils.makeAccessible(initMethod);
				return null;
			}
		});
		try {
			AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
				@Override
				public Object run() throws Exception {
					initMethod.invoke(bean);
					return null;
				}
			}, getAccessControlContext());
		}
		catch (PrivilegedActionException pae) {
			InvocationTargetException ex = (InvocationTargetException) pae.getException();
			throw ex.getTargetException();
		}
	}
	else {
		try {
			ReflectionUtils.makeAccessible(initMethod);
			initMethod.invoke(bean);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}
```
调用applyBeanPostProcessorsAfterInitialization()方法执行BeanPostProcessor的postProcessAfterInitialization()方法。
```java
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
#### 处理循环依赖
```java
if (earlySingletonExposure) {
	Object earlySingletonReference = getSingleton(beanName, false);
	if (earlySingletonReference != null) {
		if (exposedObject == bean) {
			exposedObject = earlySingletonReference;
		}
		else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
			String[] dependentBeans = getDependentBeans(beanName);
			Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
			for (String dependentBean : dependentBeans) {
				if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
					actualDependentBeans.add(dependentBean);
				}
			}
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
```
#### 注册disposableBean
```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
	AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
	if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
		if (mbd.isSingleton()) {
			// Register a DisposableBean implementation that performs all destruction
			// work for the given bean: DestructionAwareBeanPostProcessors,
			// DisposableBean interface, custom destroy method.
			registerDisposableBean(beanName,
					new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
		}
		else {
			// A bean with a custom scope...
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
