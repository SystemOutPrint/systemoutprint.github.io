---
layout:       post
title:        "Spring IOC小解"
subtitle:     "Spring IOC小解"
date:         2017-09-11 12:00:00
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - IOC
---

## 0x01 Ioc堆栈

		DefaultListableBeanFactory(AbstractAutowireCapableBeanFactory).createBean(String, RootBeanDefinition, Object[]) line: 450	
		AbstractBeanFactory$1.getObject() line: 306	
		DefaultListableBeanFactory(DefaultSingletonBeanRegistry).getSingleton(String, ObjectFactory<?>) line: 230	
		DefaultListableBeanFactory(AbstractBeanFactory).doGetBean(String, Class<T>, Object[], boolean) line: 302	
		DefaultListableBeanFactory(AbstractBeanFactory).getBean(String, Class<T>) line: 202	
		DependencyDescriptor.resolveCandidate(String, Class<?>, BeanFactory) line: 207	
		DefaultListableBeanFactory.findAutowireCandidates(String, Class<?>, DependencyDescriptor) line: 1214	
		DefaultListableBeanFactory.doResolveDependency(DependencyDescriptor, String, Set<String>, TypeConverter) line: 1054	
		DefaultListableBeanFactory.resolveDependency(DependencyDescriptor, String, Set<String>, TypeConverter) line: 1019	
		AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject(Object, String, PropertyValues) line: 566	
		InjectionMetadata.inject(Object, String, PropertyValues) line: 88	
		AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues(PropertyValues, PropertyDescriptor[], Object, String) line: 349	
		DefaultListableBeanFactory(AbstractAutowireCapableBeanFactory).populateBean(String, RootBeanDefinition, BeanWrapper) line: 1214	
		DefaultListableBeanFactory(AbstractAutowireCapableBeanFactory).doCreateBean(String, RootBeanDefinition, Object[]) line: 543	
		DefaultListableBeanFactory(AbstractAutowireCapableBeanFactory).createBean(String, RootBeanDefinition, Object[]) line: 482	
		AbstractBeanFactory$1.getObject() line: 306	
		DefaultListableBeanFactory(DefaultSingletonBeanRegistry).getSingleton(String, ObjectFactory<?>) line: 230	
		DefaultListableBeanFactory(AbstractBeanFactory).doGetBean(String, Class<T>, Object[], boolean) line: 302	
		DefaultListableBeanFactory(AbstractBeanFactory).getBean(String) line: 197	
		DefaultListableBeanFactory.preInstantiateSingletons() line: 776	
		AnnotationConfigEmbeddedWebApplicationContext(AbstractApplicationContext).finishBeanFactoryInitialization(ConfigurableListableBeanFactory) line: 861	
		AnnotationConfigEmbeddedWebApplicationContext(AbstractApplicationContext).refresh() line: 541	
		AnnotationConfigEmbeddedWebApplicationContext(EmbeddedWebApplicationContext).refresh() line: 122	
		SpringApplication.refresh(ApplicationContext) line: 759	
		
## 0x02 强行对堆栈分析

Ioc的复杂在于在Spring中，无论是什么，都会被定义成Bean注册到BeanFactory中，所以继承层级会很深，各个方法的实现复杂度会很高，在阅读的时候要保持着清晰的思路，抽丝剥茧的去读。<br>
上述堆栈想讲的实际上就是从ApplicationContext调到BeanFactory中的getBean方法获取Bean对象。<br>
getBean是对Bean初始化的地方，观察堆栈会发现，getBean其实是一个递归的方法。<br>
Bean的初始化过程类似于class的加载机制，当要初始化Bean的时候，要先对Bean中的属性进行初始化，这就出发了一系列的递归初始化。<br>

__doGetBean__
```java
protected <T> T doGetBean(
		final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
		throws BeansException {
	final String beanName = transformedBeanName(name);
	Object bean;

	// 先从缓存中取
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// 否则就去尝试创建
		// 检查是否是循环引用
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// 在父BeanFactory中是否存在该bean
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// 保证Bean的依赖都被初始化过
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
					if (isDependent(beanName, dependsOnBean)) {
						throw new BeanCreationException();
					}
					registerDependentBean(dependsOnBean, beanName);
					getBean(dependsOnBean);
				}
			}

			/**
			 * 根据scope分别去创建Bean。
			 */
			if (mbd.isSingleton()) {
				// double-check的另一种写法
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					@Override
					public Object getObject() throws BeansException {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// 每调一次都要去实例化一次
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
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, 
					new ObjectFactory<Object>() {
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
					throw new BeanCreationException(ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```
__doCreateBean__
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
	// 先从缓存里找BeanWrapper
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	// 没有就创建一个空壳壳
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			mbd.postProcessed = true;
		}
	}

	// 从缓存中查找单例的引用，来实现循环依赖。
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// 真正对Bean初始化的地方
	Object exposedObject = bean;
	try {
		// 先初始化依赖
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			// 然后对Bean初始化
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

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
					throw new BeanCurrentlyInCreationException();
				}
			}
		}
	}

	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```
