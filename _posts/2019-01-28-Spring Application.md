# Spring Application

## 准备阶段

```java
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    this.webEnvironment = deduceWebEnvironment();
    setInitializers((Collection) getSpringFactoriesInstances(
        ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 配置: Spring Boot Bean 源

​	java配置class 或xml上下文配置文件集合， 用于SpringBoot `BeanDefinitionLoader`读取，并且将配置源解析加载为SpringBean定义	

#### java配置class

添加@SpringBootApplication注解的类

```java
public class AxtApplicationConfiguration {
	public static void main(String[] args) {
		SpringApplication.run(AxtApplication.class, args);
	}
	@SpringBootApplication
	public static class AxtApplication {
	}
}
```

#### xml上下文配置文件集合

**理解:提供兼容之前的版本的方法**

```java
public class AxtApplicationConfiguration {
	public static void main(String[] args) {
		Set<Object> sources = new TreeSet<>();
		sources.add(AxtApplication.class.getName());
		SpringApplication springApplication = new SpringApplication();
 		/**
		 * The sources that will be used to create an ApplicationContext. A valid source 		  * is one of: a class, class name, package, package name, or an XML resource 
		 * location.
		 */       
		springApplication.setSources(sources);
		springApplication.run(args);
	}
	@SpringBootApplication
	public static class AxtApplication {
	}
}
```

**注意:可以通过SpringApplicationBuilder Api进行链式调用**

### 推断web类型 

WebApplicationType:

- NONE : Non-web server
- SERVLET : servlet web server
- REACTIVE : reactive web server

2.x之前 只有servlet和Non-web

```java
private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}
```

2.x 

```java
/**
 * 当servlet和reactive都存在是，取servlet
 * 只存在一个，谁存在取谁
 * 都不存在，非web类型
 */
private WebApplicationType deduceWebApplicationType() {
    if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
        && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

**理解：推断的目的是减少配置**

### 推断引导类(main class)

根据main线程执行堆栈判断实际的引导类

> ```java
> private Class<?> deduceMainApplicationClass() {
>     try {
>         StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
>         for (StackTraceElement stackTraceElement : stackTrace) {
>             if ("main".equals(stackTraceElement.getMethodName())) {
>                 return Class.forName(stackTraceElement.getClassName());
>             }
>         }
>     }
>     catch (ClassNotFoundException ex) {
>         // Swallow and continue
>     }
>     return null;
> }
> ```

### 加载应用上下文初始化器(ApplicationContextInitializer)

利用spring工厂加载机制，实例化`ApplicationContextInitializer`实现类，并排序对象集合。

> - 实现
>
> ```java
> private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
> 			Class<?>[] parameterTypes, Object... args) {
> 		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
> 		// Use names and ensure unique to protect against duplicates
> 		Set<String> names = new LinkedHashSet<String>(
> 				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
> 		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
> 				classLoader, args, names);
> 		AnnotationAwareOrderComparator.sort(instances);
> 		return instances;
> 	}
> ```
>
> - 技术
>   - 实现类: `org.springframework.core.io.support.SpringFactoriesLoader`
>   - 配置资源: `META-INF/spring.factories`
>   - 排序: `AnnotationAwareOrderComparator.sort`