## BeanFactory

![beanFactory-001](G:\Career\images\spring\beanFactory-001.PNG)

### Bean容器初始化

1. 通过初始化ClasspathXmlApplicationContext来加载资源和初始化BeanFactory； 

```java
	//测试代码启动Bean容器
	ApplicationContext ac = new ClassPathXmlApplicationContext("spring/aop.xml");
	
	//构造方法
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
	
	//构造方法
	//refresh=true: 刷新Context，加载Bean定义和创建单例Bean
	public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
	    //调用父类构造方法：主要指定resourcePatternResolver，用来资源解析。
		super(parent);
		//解析配置文件：解析通配符，将文件路径设置到configLocations成员变量。
		setConfigLocations(configLocations);
		//刷新Context，加载Bean，创建单例Bean
		if (refresh) {
			//调用父类AbstractApplicationContext的refresh方法。
			refresh();
		}
	}
```

2. AbstractApplicationContext的refresh。

   ```java
   public void refresh() throws BeansException, IllegalStateException {
      synchronized (this.startupShutdownMonitor) {
         // 刷新前的准备工作：这只状态标量、环境变量中通配符解析、校验required属性是否存在、创建earlyApplicationEvents
         prepareRefresh();
   
         // 创建了DefaultListableBeanFactory，加载BeanDefinitions
         ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
   
         // Prepare the bean factory for use in this context.
         prepareBeanFactory(beanFactory);
   
         try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);
   
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);
   
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
   
            // Initialize message source for this context.
            initMessageSource();
   
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();
   
            // Initialize other special beans in specific context subclasses.
            onRefresh();
   
            // Check for listener beans and register them.
            registerListeners();
   
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);
   
            // Last step: publish corresponding event.
            finishRefresh();
         }
   
         catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
               logger.warn("Exception encountered during context initialization - " +
                     "cancelling refresh attempt: " + ex);
            }
   
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();
   
            // Reset 'active' flag.
            cancelRefresh(ex);
   
            // Propagate exception to caller.
            throw ex;
         }
   
         finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
         }
      }
   }
   
   protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
       refreshBeanFactory();
       ConfigurableListableBeanFactory beanFactory = getBeanFactory();
       if (logger.isDebugEnabled()) {
           logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
       }
       return beanFactory;
   }
   
   //刷新bean工厂
   protected final void refreshBeanFactory() throws BeansException {
       if (hasBeanFactory()) {
           destroyBeans();
           closeBeanFactory();
       }
       try {
           DefaultListableBeanFactory beanFactory = createBeanFactory();
           beanFactory.setSerializationId(getId());
           customizeBeanFactory(beanFactory);
           //加载Bean
           loadBeanDefinitions(beanFactory);
           synchronized (this.beanFactoryMonitor) {
               this.beanFactory = beanFactory;
           }
       }
       catch (IOException ex) {
           throw new ApplicationContextxception("I/O error parsing bean definition source for " + getDisplayName(), ex);
       }
   }
   
   protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
      // Create a new XmlBeanDefinitionReader for the given BeanFactory.
      XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
   
      // Configure the bean definition reader with this context's
      // resource loading environment.
      beanDefinitionReader.setEnvironment(this.getEnvironment());
      beanDefinitionReader.setResourceLoader(this);
      beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
   
      // Allow a subclass to provide custom initialization of the reader,
      // then proceed with actually loading the bean definitions.
      initBeanDefinitionReader(beanDefinitionReader);
      loadBeanDefinitions(beanDefinitionReader);
   }
   
   protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
       Resource[] configResources = getConfigResources();
       if (configResources != null) {
           reader.loadBeanDefinitions(configResources);
       }
       String[] configLocations = getConfigLocations();
       if (configLocations != null) {
           //加载BeanDefinition
           reader.loadBeanDefinitions(configLocations);
       }
   }
   
   //AbstractBeanDefinitionReader
   public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
       Assert.notNull(locations, "Location array must not be null");
       int counter = 0;
       for (String location : locations) {
           counter += loadBeanDefinitions(location);
       }
       return counter;
   }
   
   //AbstractBeanDefinitionReader
   public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
       ResourceLoader resourceLoader = getResourceLoader();
       if (resourceLoader == null) {
           throw new BeanDefinitionStoreException(
               "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
       }
   
       if (resourceLoader instanceof ResourcePatternResolver) {
           // Resource pattern matching available.
           try {
               //获取资源
               Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
               int loadCount = loadBeanDefinitions(resources);
               if (actualResources != null) {
                   for (Resource resource : resources) {
                       actualResources.add(resource);
                   }
               }
               if (logger.isDebugEnabled()) {
                   logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
               }
               return loadCount;
           }
           catch (IOException ex) {
               throw new BeanDefinitionStoreException(
                   "Could not resolve bean definition resource pattern [" + location + "]", ex);
           }
       }
       else {
           // Can only load single resources by absolute URL.
           Resource resource = resourceLoader.getResource(location);
           int loadCount = loadBeanDefinitions(resource);
           if (actualResources != null) {
               actualResources.add(resource);
           }
           if (logger.isDebugEnabled()) {
               logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
           }
           return loadCount;
       }
   }
   
   //PathMatchingResourcePatternResolver
   public Resource[] getResources(String locationPattern) throws IOException {
       Assert.notNull(locationPattern, "Location pattern must not be null");
       if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
           // a class path resource (multiple resources for same name possible)
           if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
               // a class path resource pattern
               return findPathMatchingResources(locationPattern);
           }
           else {
               // all class path resources with the given name
               return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
           }
       }
       else {
           // Generally only look for a pattern after a prefix here,
           // and on Tomcat only after the "*/" separator for its "war:" protocol.
           int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 :
                            locationPattern.indexOf(":") + 1);
           if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
               // a file pattern
               return findPathMatchingResources(locationPattern);
           }
           else {
               // a single resource with the given name
               return new Resource[] {getResourceLoader().getResource(locationPattern)};
           }
       }
   }
   
   //XmlBeanDefinitionReader
   public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
       Assert.notNull(encodedResource, "EncodedResource must not be null");
       if (logger.isInfoEnabled()) {
           logger.info("Loading XML bean definitions from " + encodedResource.getResource());
       }
   
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
           InputStream inputStream = encodedResource.getResource().getInputStream();
           try {
               InputSource inputSource = new InputSource(inputStream);
               if (encodedResource.getEncoding() != null) {
                   inputSource.setEncoding(encodedResource.getEncoding());
               }
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
   
   //XmlBeanDefinitionReader
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
   
   //XmlBeanDefinitionReader
   public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
       BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
       int countBefore = getRegistry().getBeanDefinitionCount();
       documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
       return getRegistry().getBeanDefinitionCount() - countBefore;
   }
   
   //DefaultBeanDefinitonDocumentReader
   public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
       this.readerContext = readerContext;
       logger.debug("Loading bean definitions");
       Element root = doc.getDocumentElement();
       doRegisterBeanDefinitions(root);
   }
   
   //DefaultBeanDefinitonDocumentReader
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
   
   //DefaultBeanDefinitonDocumentReader
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
   
   //DefaultBeanDefinitonDocumentReader
   private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
       if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
           importBeanDefinitionResource(ele);
       }
       else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
           processAliasRegistration(ele);
       }
       else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
           processBeanDefinition(ele, delegate);
       }
       else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
           // recurse
           doRegisterBeanDefinitions(ele);
       }
   }
   
   //DefaultBeanDefinitonDocumentReader
   protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
       BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
       if (bdHolder != null) {
           bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
           try {
               // Register the final decorated instance.
               BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
           }
           catch (BeanDefinitionStoreException ex) {
               getReaderContext().error("Failed to register bean definition with name '" +
                                        bdHolder.getBeanName() + "'", ele, ex);
           }
           // Send registration event.
           getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
       }
   }
   
   //BeanDefinitionParserDelegate
   public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
       String id = ele.getAttribute(ID_ATTRIBUTE);
       String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
   
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
   
       AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
       if (beanDefinition != null) {
           if (!StringUtils.hasText(beanName)) {
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
           return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
       }
   
       return null;
   }
   
   //
   public AbstractBeanDefinition parseBeanDefinitionElement(
   			Element ele, String beanName, BeanDefinition containingBean) {
   
       this.parseState.push(new BeanEntry(beanName));
   
       String className = null;
       if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
           className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
       }
   
       try {
           String parent = null;
           if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
               parent = ele.getAttribute(PARENT_ATTRIBUTE);
           }
           AbstractBeanDefinition bd = createBeanDefinition(className, parent);
   
           parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
           bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
   
           parseMetaElements(ele, bd);
           parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
           parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
   
           parseConstructorArgElements(ele, bd);
           parsePropertyElements(ele, bd);
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

```java

```