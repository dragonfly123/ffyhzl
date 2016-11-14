### Spring Cloud ConfigServer简介 ###
#### 1. 实现的主要功能
Spring Cloud ConfigServer 的主要功能是对传统的配置文件放在提供一个服务的形式，提
供配置中心，任何客户段均可通过web服务访问配置。从而实现配置的统一管理，有效的将测试开发隔离开，保证一个统一的开发环境，不用受到配置参数的干扰，更加有利于开发、测试。   

#### 2. SpringBoot中几个常用的配置的简单介绍
 * 一个简单的<strong>Spring.factories</strong>   

        # Bootstrap components
        org.springframework.cloud.bootstrap.BootstrapConfiguration=\
        org.springframework.cloud.config.server.bootstrap.ConfigServerBootstrapConfiguration

        # Application listeners
        org.springframework.context.ApplicationListener=\
        org.springframework.cloud.config.server.bootstrap.ConfigServerBootstrapApplicationListener      
        # Autoconfiguration
        org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
        org.springframework.cloud.config.server.config.EncryptionAutoConfiguration,\
        org.springframework.cloud.config.server.config.SingleEncryptorAutoConfiguration

* <strong>BootstrapConfiguration</strong>简介
 Spring.factories中的配置项， org.springframework.cloud.bootstrap.BootstrapConfiguration，会在Spring启动之前的准备上下文阶段将其加入到容器中。我们在介绍 [@Configuration配置解析]一文中曾详细解释Configuration中的解析。BootstrapConfiguration是在刷新（refresh）阶段将class作为预选的配置类@Configuration注入到容器中的，在[@Configuration配置解析]阶段会解析此class。
 + <i><strong>加载BootstrapConfiguration配置的类</strong></i>
````
List<String> names = SpringFactoriesLoader
				.loadFactoryNames(BootstrapConfiguration.class, classLoader);
		for (String name : StringUtils.commaDelimitedListToStringArray(
				environment.getProperty("spring.cloud.bootstrap.sources", ""))) {
			names.add(name);
		}
		// TODO: is it possible or sensible to share a ResourceLoader?
		SpringApplicationBuilder builder = new SpringApplicationBuilder()
				.profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
				.environment(bootstrapEnvironment)
				.properties("spring.application.name:" + configName)
				.registerShutdownHook(false).logStartupInfo(false).web(false);
		List<Class<?>> sources = new ArrayList<>();
		for (String name : names) {
			Class<?> cls = ClassUtils.resolveClassName(name, null);
			try {
				cls.getDeclaredAnnotations();
			}
			catch (Exception e) {
				continue;
			}
			sources.add(cls);
		}
		builder.sources(sources.toArray(new Class[sources.size()]));
		AnnotationAwareOrderComparator.sort(sources);
		final ConfigurableApplicationContext context = builder.run();
````
 + <i><strong>refresh阶段的prepareContext</strong></i>
````  
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		postProcessApplicationContext(context);
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

           // Add boot specific singleton beans
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		Set<Object> sources = getSources();//获取需要加载的class源
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[sources.size()]));//加载class
		listeners.contextLoaded(context);
      }
````
* <strong>ApplicationListener</strong>简介
 org.springframework.context.ApplicationListener会在SpringBoot中几个典型事件产生后调用onApplicationEvent方法。详见[Spring事件发布系统]。

* <strong>Autoconfiguration</strong>简介
org.springframework.boot.autoconfigure.EnableAutoConfiguration是SpringBoot中最常见配置，用于自动配置。只要有<strong>@EnableAutoConfiguration</strong>注解，该配置的所有类都会自动加载到Spring容器中。


[@Configuration配置解析]:http://www.cnblogs.com/dragonfei/p/5925114.html "Configuration"
[Spring事件发布系统]:http://www.cnblogs.com/dragonfei/p/5911259.html "Spring 事件发布系统"
