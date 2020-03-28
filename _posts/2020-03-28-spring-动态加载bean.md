---
layout: post
title: "spring 容器动态加载bean的实现"
description: "动态加载bean"
keywords: ImportSelector,source,analysis
tags: ImportSelector 代码解析
---

## 前言
	在很多应用场景，我们需要根据配置或者条件判断，来决定是否加载某些bean。为了解决动态加载bean的需求，spring提供了 ```ImportSelector``` 这个工具.
	一个具体的应用，就是spring boot 的自动配置功能。比如，服务自动发现功能```@EnableDiscoveryClient```。

```java
// import选择器，用于自定义import的策略。
// selectImports 返回的是配置类。
public interface ImportSelector {

	/**
	 * Select and return the names of which class(es) should be imported based on
	 * the {@link AnnotationMetadata} of the importing @{@link Configuration} class.
	 */
	String[] selectImports(AnnotationMetadata importingClassMetadata);

}


/**
 * Annotation to enable a DiscoveryClient implementation.
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
/**
 * 这里，使用了 EnableDiscoveryClientImportSelector 这个 ImportSelector
 */
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 */
	boolean autoRegister() default true;
}
```

当Spring 扫描到EnableDiscoveryClient注解时，根据 @import注解，加载EnableDiscoveryClientImportSelector，并执行ImportSelector接口约定的selectImports函数.
EnableDiscoveryClientImportSelector的具体实现内容是:提取类路径下所有jar包的/META-INF/spring.factories文件中Key为``` @EnableDiscoveryClient```注解类全路径的值,并返回。
在nacos服务自动发现的/META-INF/spring.factories中，有这样一行配置:
```properties
# 这一行配置，就是用于标记服务发现的配置类为: NacosDiscoveryClientAutoConfiguration
org.springframework.cloud.client.discovery.EnableDiscoveryClient=\
org.springframework.cloud.alibaba.nacos.NacosDiscoveryClientAutoConfiguration
```
所以，EnableDiscoveryClientImportSelector 最终返回的是: ["org.springframework.cloud.alibaba.nacos.NacosDiscoveryClientAutoConfiguration" ].

Spring 容器会解析 ```NacosDiscoveryClientAutoConfiguration```这个配置类，提取bean并注入容器，这样便完成了服务自动发现的必要功能。
```java
/**
 * @author xiaojing
 */
@Configuration
@ConditionalOnMissingBean(DiscoveryClient.class)
@ConditionalOnNacosDiscoveryEnabled
@EnableConfigurationProperties
public class NacosDiscoveryClientAutoConfiguration {
	/**
         * spring cloud 约定的服务发现客户端 DiscoveryClient
         */
	@Bean
	public DiscoveryClient nacosDiscoveryClient() {
		return new NacosDiscoveryClient();
	}

	@Bean
	@ConditionalOnMissingBean
	public NacosDiscoveryProperties nacosProperties() {
		return new NacosDiscoveryProperties();
	}
}
```
