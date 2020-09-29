new 实例化 -> set property注入依赖 -> Aware接口 ->
BeanPostProcessor.beforeInit.. -> @PostConstruct ->
InitializingBean.afterPropertiesSet -> 自定义init-method ->
BeanPostProcessor.afterInit... -> 初始化完成，可以使用 ->
@PreDestroy --> DisposableBean.destroy -> 自定义destroy-method

首先在AbstractApplicationContext。refresh()方法中

@Configuration注解配置的类，在ConfigurationClassParser.retrieveBeanMethodMetadata(SourceClass sourceClass)解析配置的元数据，
然后在ConfigurationClassBeanDefinitionReader.loadBeanDefinitions(Set<ConfigurationClass> configurationModel)方法中加载bean。
解析成BeanDefinition后，注册到DefaultListableBeanFactory.registerBeanDefinition(String beanName, BeanDefinition beanDefinition)


反射生成bean：
AbstractApplicationContext.refresh() -->
AbstractApplicationContext.finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) --> 
DefaultListableBeanFactory.preInstantiateSingletons() --> 
AbstractBeanFactory.getBean(String name) --> 
AbstractBeanFactory. doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)-->
DefaultSingletonBeanRegistry.getSingleton(String beanName, ObjectFactory<?> singletonFactory) -->
AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args) --> 
AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) --> 
AbstractAutowireCapableBeanFactory.createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) --> 
AbstractAutowireCapableBeanFactory.instantiateBean(final String beanName, final RootBeanDefinition mbd)  --> 
SimpleInstantiationStrategy.instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) --> 
BeanUtils.instantiateClass(constructorToUse) --> 
Constructor.newInstance(args) 


依赖注入：在生成bean后AbstractAutowireCapableBeanFactory.doCreateBean()，后续即会调用populateBean()注入依赖
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


生成代理对象：在依赖注入后AbstractAutowireCapableBeanFactory.doCreateBean()，后续即会调用initializeBean()生成代理对象
AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) --> 
AbstractAutowireCapableBeanFactory.initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) --> 
AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) --> 
AbstractAutoProxyCreator.postProcessAfterInitialization(Object bean, String beanName) --> 
AbstractAutoProxyCreator.wrapIfNecessary(Object bean, String beanName, Object cacheKey) --> 
AbstractAutoProxyCreator.createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) --> 
ProxyFactorygetProxy(ClassLoader classLoader)-- > 
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
