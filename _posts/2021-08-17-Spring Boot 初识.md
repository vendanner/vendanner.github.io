---
layout:     post
title:      Spring Boot 初识
subtitle:   
date:       2021-08-17
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Spring Boot
---

### 自动配置

**Auto Configuration**

spring-boot-autoconfiguration jar 包含 Spring Boot 自动配置代码。

- 开启：@EnableAutoConfiguration
  - @SpringBootApplication 已包含 @EnableAutoConfiguration，实际使用直接 @SpringBootApplication
  - 若不想使用一些自动配置，@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class }) 使用数据源的自动配置

Spring Boot 都包含哪些自动配置？

```java
org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#selectImports
  ->org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getAutoConfigurationEntry
  ->org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getCandidateConfigurations

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 打开本jar 下的 META-INF/spring.factories 文件
    // org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的值就是 Spring Boot 包含的所有自动配置类
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

#### 原理

自动配置是如何生效的呢？Condition：通过条件判断生成对应 Bean，完成自动配置

```java
public class DataSourceAutoConfiguration {
	  @Conditional({DataSourceAutoConfiguration.EmbeddedDatabaseCondition.class})
    @ConditionalOnMissingBean({DataSource.class, XADataSource.class})
    @Import({EmbeddedDataSourceConfiguration.class})
    protected static class EmbeddedDatabaseConfiguration {
        protected EmbeddedDatabaseConfiguration() {
        }
    }
}
```

- 满足 DataSourceAutoConfiguration.EmbeddedDatabaseCondition getMatchOutcome 函数里的条件
- 不包含 DataSource.class, XADataSource.class Bean
- EmbeddedDataSourceConfiguration 生成 EmbeddedDataSource

#### 观察

Spring Boot 自动配置有很多，怎么知道哪些配置生效/没生效。在程序启动时加 “--debug”，日志级别 DEBUG

- 自动配置已匹配

```shell
Positive matches:
-----------------

   AopAutoConfiguration matched:
      - @ConditionalOnProperty (spring.aop.auto=true) matched (OnPropertyCondition)

   AopAutoConfiguration.AspectJAutoProxyingConfiguration matched:
      - @ConditionalOnClass found required class 'org.aspectj.weaver.Advice' (OnClassCondition)
   ...
```

- 自动配置未匹配

```shell
Negative matches:
-----------------

   ActiveMQAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'javax.jms.ConnectionFactory' (OnClassCondition)
   ...
```

- 自动配置未包含

```shell
Exclusions:
-----------

    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

- 不需要满足条件的自动配置

```shell
Unconditional classes:
----------------------

    org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration

    com.tuya.middleware.dubai.utils.Config
    ...
```

#### 自定义自动配置

- 编写 Java Config：@Configuration
- 添加条件：@Conditional
- 让Spring Boot 能够找到：META-INF/spring.factories 里写入 Class



### 起步依赖

**Starter Dependency**

- dependencyManagement：统一管理 jar 版本



### Actuator

监控和管理应用程序

