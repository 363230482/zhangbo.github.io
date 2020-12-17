```
new 实例化 -> populateBean注入依赖 -> Aware接口 ->
BeanPostProcessor.beforeInit.. -> @PostConstruct ->
InitializingBean.afterPropertiesSet -> @Bean#init-method ->
BeanPostProcessor.afterInit... -> 生成代理对象wrapIfNecessary -> 
初始化完成，可以使用 ->
@PreDestroy --> DisposableBean.destroy -> @Bean#destroy-method
```

首先在AbstractApplicationContext。refresh()方法中

## scan bean definition
@Configuration注解配置的类，在ConfigurationClassParser.retrieveBeanMethodMetadata(SourceClass sourceClass)解析配置的元数据，
然后在ConfigurationClassBeanDefinitionReader.loadBeanDefinitions(Set<ConfigurationClass> configurationModel)方法中加载bean。
解析成BeanDefinition后，注册到DefaultListableBeanFactory.registerBeanDefinition(String beanName, BeanDefinition beanDefinition)


## 反射生成bean
```
AbstractApplicationContext.refresh() -->
AbstractApplicationContext.finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) --> 
DefaultListableBeanFactory.preInstantiateSingletons() --> 
AbstractBeanFactory.getBean(String name) --> 
// 解决循环依赖,见下方
AbstractBeanFactory.doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)-->
DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory) -->
AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args) --> 
AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) --> 
AbstractAutowireCapableBeanFactory.createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) --> 
// 构造器
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireConstructor
// 普通方法
AbstractAutowireCapableBeanFactory.instantiateBean(final String beanName, final RootBeanDefinition mbd)  --> 
SimpleInstantiationStrategy.instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) --> 
BeanUtils.instantiateClass(constructorToUse) --> 
Constructor.newInstance(args) 
```

## 循环依赖-三级缓存
```
org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean() {
    // 简单步骤
    // 从缓存中获取
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String)
        // 缓存中获取方法源码
        protected Object getSingleton(String beanName, boolean allowEarlyReference) {
            // singletonObjects 一级缓存，存放已经初始化好的bean，可以直接被使用
            Object singletonObject = this.singletonObjects.get(beanName);
            // isSingletonCurrentlyInCreation方法中singletonsCurrentlyInCreation用于存放正在创建
            // (要整个bean的初始化过程结束才从singletonsCurrentlyInCreation移除)的bean
            if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
                synchronized (this.singletonObjects) {
                    // earlySingletonObjects 二级缓存，存放已经实例化好，但还没有进行初始化的bean
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null && allowEarlyReference) {
                        // singletonFactories 三级缓存，bean factory缓存
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            // 实例化完成，放入earlySingletonObjects
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
            return singletonObject;
        }
    // 缓存中没有则进入初始化
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)
    // 将bean name 加入 singletonsCurrentlyInCreation
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#beforeSingletonCreation
    // bean 初始化过程，包括实例化，组装属性值，初始化全过程
    org.springframework.beans.factory.ObjectFactory#getObject
    org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
    // 将bean name 移除 singletonsCurrentlyInCreation
    org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#afterSingletonCreation
    // 将初始化好的bean加入一级缓存
    protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}
```

## 依赖注入
在生成bean后AbstractAutowireCapableBeanFactory.doCreateBean()，后续即会调用populateBean()注入依赖
```
AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) --> 
AbstractAutowireCapableBeanFactory.populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) --> 
AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues( PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) -- > 
InjectionMetadata.inject(Object target, String beanName, PropertyValues pvs) -->
AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement.inject(Object bean, String beanName, PropertyValues pvs)-->
DefaultListableBeanFactory.resolveDependency(DependencyDescriptor descriptor, String requestingBeanName,
      Set<String> autowiredBeanNames, TypeConverter typeConverter) 
DefaultListableBeanFactory. doResolveDependency(DependencyDescriptor descriptor, String beanName,
      Set<String> autowiredBeanNames, TypeConverter typeConverter) 
DefaultListableBeanFactory.findAutowireCandidates(String beanName,Class<?> requiredType, DependencyDescriptor descriptor) 
DefaultListableBeanFactory.addCandidateEntry(Map<String, Object> candidates, String candidateName,
      DependencyDescriptor descriptor, Class<?> requiredType)
DependencyDescriptor.resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory)
AbstractBeanFactory.getBean(String name, Class<T> requiredType)
AbstractBeanFactory.getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd)
FactoryBeanRegistrySupport.getObjectFromFactoryBean(FactoryBean<?> factory,String beanName,boolean shouldPostProcess)
FeignClientFactoryBean.getObject()

返回 value，然后 -->
if (value != null) {
   ReflectionUtils.makeAccessible(field);
   field.set(bean, value);
}
```

## 生成代理对象
在依赖注入后AbstractAutowireCapableBeanFactory.doCreateBean()，后续即会调用initializeBean()生成代理对象
```
AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) --> 
AbstractAutowireCapableBeanFactory.initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) --> 
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) --> 
AbstractAutoProxyCreator.postProcessAfterInitialization(Object bean, String beanName) --> 
AbstractAutoProxyCreator.wrapIfNecessary(Object bean, String beanName, Object cacheKey) --> 
AbstractAutoProxyCreator.createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) --> 
ProxyFactorygetProxy(ClassLoader classLoader) -- > 
DefaultAopProxyFactory.createAopProxy(AdvisedSupport config) --> 通过该方法选择是用JDK或者cglib代理
cglib: --> CglibAopProxy.getProxy(ClassLoader classLoader) -->
cglib: --> ObjenesisCglibAopProxy.createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) 
jdkProxy: --> JdkDynamicAopProxy.getProxy(ClassLoader classLoader) --> 
jdkProxy: --> Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)
DefaultAopProxyFactory.createAopProxy(AdvisedSupport config) --> 方法如下：
// 选择是用JDK或者cglib代理，如果是接口，默认JDK动态代理，否则cglib动态代理
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      Class<?> targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException("TargetSource cannot determine target class: " +
               "Either an interface or a target is required for proxy creation.");
      }
      // 如果是接口或者已经被jdk代理过的类，使用jdk代理
      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         return new JdkDynamicAopProxy(config);
      }
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      return new JdkDynamicAopProxy(config);
   }
}

DefaultSingletonBeanRegistry.addSingleton(String beanName, Object singletonObject)
```