# Spring Framework 源码学习指南

## 项目概述

Spring Framework 7.0.0-SNAPSHOT — Spring 生态的基石，也是 Spring Boot 的底层依赖。当前版本处于 **main** 分支开发中，Java 17+ 字节码目标，JDK 23 工具链编译。

---

## 构建系统

- **Gradle** 多模块项目，使用 `gradlew` 包装器（无需本地安装 Gradle）
- **Java**: 工具链 JDK 23 (BellSoft Liberica)，字节码目标 17（`-release 17`）
- **Kotlin**: 2.1.10
- `./gradlew check antora` — 完整 CI 构建（编译 + 测试 + 检查 + 文档）
- `./gradlew :spring-xxx:test` — 运行单个模块测试
- `./gradlew :spring-xxx:test --tests "*SomeTest*"` — 运行单个测试类
- `./gradlew runtimeHintsTest` — 运行 RuntimeHints 代理测试（需 `spring-core-test` 模块支持）
- 缓存：Gradle 构建缓存开启，依赖缓存秒级过期（`cacheChangingModulesFor 0`）
- 并行编译：`org.gradle.parallel=true`

### 自定义 Gradle 插件（buildSrc）

| 插件 ID | 作用 |
|---------|------|
| `org.springframework.build.conventions` | 总约定：Java/Kotlin/Checkstyle/Test/ArchUnit |
| `org.springframework.build.optional-dependencies` | 提供 `optional` 依赖配置（不传递） |
| `org.springframework.build.multiReleaseJar` | 多版本 JAR 支持（如 Java 21 源码集） |
| `org.springframework.build.runtimehints-agent` | 为 GraalVM 原生镜像的 RuntimeHints 测试 |
| `org.springframework.architecture` | ArchUnit 架构规则检查 |

---

## 模块结构（21 个 spring-* 模块）

### 核心容器（Core Container）
```
spring-expression          ← SpEL，无外部依赖
spring-core                ← 核心工具、ASM/CGLIB/JavaPoet/Objenesis（重打包）、类型解析、IO、转换、AOT
spring-beans               ← BeanFactory、BeanDefinition、属性访问，依赖 spring-core
spring-aop                 ← 代理 AOP（JDK 动态代理 + CGLIB），依赖 spring-beans + spring-core
spring-context             ← ApplicationContext、注解配置、调度、JMX、缓存，依赖 aop+beans+core+expression
```

### 数据访问（Data Access）
```
spring-jdbc                ← JDBC 框架
spring-tx                  ← 事务抽象（@Transactional、PlatformTransactionManager）
spring-orm                 ← JPA/Hibernate 集成
spring-oxm                 ← XML 编组（JAXB）
spring-r2dbc               ← 响应式数据库访问
```

### Web
```
spring-web                 ← HTTP 抽象、客户端、服务器端公共设施，依赖 beans+core
spring-webmvc              ← Servlet 栈 MVC（DispatcherServlet），依赖 web+context
spring-webflux             ← 响应式栈 Web（DispatcherHandler），依赖 web+context+core
spring-websocket           ← WebSocket 支持
spring-messaging           ← Spring 集成消息抽象
```

### 其他
```
spring-jms                 ← JMS 消息
spring-instrument          ← 类加载期 Instrumentation
spring-aspects             ← AspectJ 方面（注解事务、缓存等）
spring-context-support     ← 第三方集成（Quartz、Velocity、FreeMarker 等）
spring-context-indexer     ← 注解索引（加速类路径扫描）
spring-test                ← 测试支持（TestContext、MockHttpServletRequest 等）
spring-core-test           ← 核心测试基础设施（RuntimeHintsAgent 等）
```

### 基础设施模块
```
framework-platform         ← BOM 依赖版本管理
framework-bom              ← 发布的 Bill of Materials
framework-docs             ← Antora 参考文档源
framework-api              ← API 文档 + Schema ZIP
integration-tests          ← 集成测试
```

### 模块依赖图（自底向上）
```
spring-core
  └─ spring-beans
       ├─ spring-aop
       │    └─ spring-context
       │         └─ spring-webmvc (Servlet)
       │         └─ spring-webflux (Reactive)
       └─ spring-web
```

---

## 关键源码入口

### IoC 容器
- **BeanFactory 层级**: `spring-beans/src/main/java/org/springframework/beans/factory/`
  - `BeanFactory.java` — 最顶层接口
  - `ListableBeanFactory.java` — 可列举的 BeanFactory
  - `HierarchicalBeanFactory.java` — 层级 BeanFactory
  - `ConfigurableBeanFactory.java` — 可配置接口
  - `ConfigurableListableBeanFactory.java` → `DefaultListableBeanFactory.java`（核心实现）
  - `AutowireCapableBeanFactory.java` — 自动装配接口
  - `BeanDefinition.java` — Bean 定义元信息

- **ApplicationContext 层级**: `spring-context/src/main/java/org/springframework/context/support/`
  - `AbstractApplicationContext.java` — 核心模板实现（refresh → 整个容器启动流程）
  - `GenericApplicationContext.java` — 通用实现
  - `AnnotationConfigApplicationContext.java` — 注解驱动入口
  - `ClassPathXmlApplicationContext.java` — XML 驱动入口
  - `PostProcessorRegistrationDelegate.java` — BeanPostProcessor 注册逻辑

- **注解配置解析**: `spring-context/src/main/java/org/springframework/context/annotation/`
  - `ConfigurationClassPostProcessor.java` — `@Configuration` 核心处理
  - `ConfigurationClassParser.java` — 解析 `@Import`、`@ComponentScan`、`@Bean` 等
  - `ConfigurationClassBeanDefinitionReader.java` — 将解析结果注册为 BeanDefinition
  - `AnnotatedBeanDefinitionReader.java` — 注册单个注解 Bean
  - `ClassPathBeanDefinitionScanner.java` — 包扫描
  - `ClassPathScanningCandidateComponentProvider.java` — 候选组件检测

### AOP
- `spring-aop/src/main/java/org/springframework/aop/framework/`
  - `JdkDynamicAopProxy.java` — JDK 动态代理
  - `CglibAopProxy.java` — CGLIB 代理
  - `DefaultAopProxyFactory.java` — 代理工厂选择策略
  - `AdvisedSupport.java` — 代理配置持有者
  - `AopProxyUtils.java` — AOP 工具方法

### Spring MVC
- `spring-webmvc/src/main/java/org/springframework/web/servlet/`
  - `DispatcherServlet.java` — 前端控制器入口（doDispatch 为核心方法）
  - `HandlerMapping.java` — 请求→处理器映射
  - `HandlerAdapter.java` — 处理器适配
  - `ViewResolver.java` — 视图解析
  - `HandlerInterceptor.java` — 拦截器
  - `config/` → `WebMvcConfigurationSupport.java`, `EnableWebMvc.java`
  - `mvc/` → 注解驱动支持

### Spring WebFlux
- `spring-webflux/src/main/java/org/springframework/web/reactive/`
  - `DispatcherHandler.java` — 响应式前端控制器

### 事务
- `spring-tx/src/main/java/org/springframework/transaction/`
  - `PlatformTransactionManager.java` — 事务管理器接口
  - `annotation/EnableTransactionManagement.java`
  - `interceptor/TransactionInterceptor.java`
  - `interceptor/TransactionAspectSupport.java`

### 测试
- `spring-test/src/main/java/org/springframework/test/`
  - `context/TestContextManager.java` — 测试上下文管理器
  - `context/junit5/SpringExtension.java` — JUnit 5 扩展
  - `web/servlet/MockHttpServletRequest.java` — Mock 请求

---

## 代码规范

### 编码风格
- **缩进**: Tab，每个 Tab = 4 空格（`.editorconfig`，Eclipse 设置）
- **编码**: UTF-8
- **换行**: LF
- **格式化**: Spring Java Format（checkstyle 强制执行）
  - Checkstyle 配置: `src/checkstyle/checkstyle.xml`
  - Checkstyle 工具版本: 10.21.2
- **Java 编译参数**:
  - main: `-Xlint:all -Werror`（所有警告视为错误）
  - test: 部分 lint 禁用（`-varargs -fallthrough -rawtypes -deprecation -unchecked`）

### 空安全
- 所有 `package-info.java` 必须标注 `@NullMarked`
- 使用 JSpecify 注解（`org.jspecify.annotations`）进行空值标记
- NullAway 静态分析开启（`NullAway:JSpecifyMode=true`）
- 禁止导入 `org.springframework.lang.NonNull` / `Nullable`（使用 JSpecify）

### 架构规则（ArchUnit 强制）
- 包之间无循环依赖（剔除 asm、cglib、javapoet、objenesis 重打包包）
- 禁止 import `reactor.core.support.Assert`
- 禁止 import `org.slf4j.LoggerFactory`
- 禁止 import `org.springframework.lang.NonNull` / `Nullable`
- Java 文件禁止依赖 Kotlin 注解（`org.jetbrains.annotations`）
- `String.toLowerCase()` / `toUpperCase()` 必须传 `Locale.ROOT`

### NoHTTP 检查
- 源代码禁止出现 HTTP URL 中的明文 `http://`
- 例外配置: `src/nohttp/allowlist.lines`

---

## 测试约定

- **测试框架**: JUnit 5（Platform），使用 `useJUnitPlatform()`
- **测试类命名**: `*Tests.class` 或 `*Test.class`
- **模拟库**: Mockito + AssertJ Assertions
- **CI 重试**: 失败测试最多重试 3 次（仅 CI 环境，`CI=true`）
- **系统属性**: `java.awt.headless=true`, `io.netty.leakDetection.level=paranoid`
- **JVM 参数**: `--add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/java.util=ALL-UNNAMED -Xshare:off`
- **可选**: `testGroups` 项目属性可传递到系统属性

### 测试特殊标记
- `@EnabledIfRuntimeHintsAgent` — 需要 RuntimeHints 代理的测试（通过 `runtimeHintsTest` 任务运行）
- `testFixtures` — 使用 Gradle `java-test-fixtures` 插件共享测试工具

---

## IDE 导入须知

### IntelliJ IDEA
1. 先预编译: `./gradlew :spring-core:compileTestJava :spring-oxm:compileTestJava`
2. 导入 `build.gradle` 作为项目
3. **排除 `spring-aspects` 模块**（因为 AspectJ 类型在 IDEA 中无法编译）
4. 如点"Rebuild Project"，需重新执行 `./gradlew :spring-oxm:compileTestJava`
5. 不要提交 `.iml`、`.ipr`、`.iws` 文件（已在 `.gitignore`）

### Eclipse
- 先执行 `./gradlew eclipse` 生成项目文件
- 使用 `src/eclipse/` 下的预置设置
- 多版本 JAR 的 Java 21 源码在 Eclipse 中不可用

---

## 特殊机制

### 重打包（Repack）
- spring-core 通过 ShadowJar **重打包** javapoet（`com.squareup.javapoet` → `org.springframework.javapoet`）和 objenesis
- 这是为了规避外部依赖的直接暴露
- IDE 中需要先 `compileTestJava` 生成这些重打包 JAR

### 多版本 JAR（Multi-Release JAR）
- spring-core 配置了 `releaseVersions 21`，在 `src/main/java21/` 下可放置 Java 21+ 专属实现
- 构建时自动选择对应版本的源码编译

### RuntimeHints GraalVM 支持
- 使用 `spring-core-test` 模块的 RuntimeHintsAgent
- 通过 `runtimeHintsTest` 任务执行带代理的测试
- 用于验证 GraalVM 原生镜像所需的配置提示

### Annotation Indexer
- `spring-context-indexer` 是编译时注解处理器，生成 `META-INF/spring.components`
- 替代运行时类路径扫描，提升大型项目启动速度

---

## Git 工作流

- 主分支: `main`（6.0.x、6.1.x、6.2.x 为旧版本维护分支）
- PR 提交到 `main`，backport 按需处理
- 提交信息格式: 首行 55 字符，正文每行 72 字符，结尾 `Closes gh-XXXXX`
- 每个提交须包含 `Signed-off-by` 尾注（DCO）
- 提交前整理 commits（squash 同逻辑的多次修改）
- 当前版本: `7.0.0-SNAPSHOT`（在 `gradle.properties` 中定义）

---

## 学习路径建议

要理解 Spring Boot 源码，建议按以下顺序阅读 Spring Framework 核心：

1. **spring-core** → `DefaultListableBeanFactory`（核心 IoC 容器）、`ResolvableType`（泛型解析）、`Resource`/`ResourceLoader`
2. **spring-beans** → `BeanDefinition` 体系、`BeanFactory` 接口层级、`BeanPostProcessor` 扩展点
3. **spring-context** → `AbstractApplicationContext.refresh()`（容器启动 13 步流程）、`ConfigurationClassPostProcessor`（注解配置驱动）
4. **spring-aop** → 代理创建机制、`@Transactional` 原理前置知识
5. **spring-web** → HTTP 抽象、`HttpMessageConverter`、Filter 链
6. **spring-webmvc** → `DispatcherServlet.doDispatch()` 请求处理流程
7. **spring-tx** → `@Transactional` 声明式事务实现
8. **spring-test** → `TestContextManager`、`SpringExtension`（JUnit 5 集成）

---

## 常见陷阱

- 不要直接 import `org.springframework.lang.NonNull` — 使用 `org.jspecify.annotations.NullMarked`
- 不要调用 `String.toLowerCase()` / `toUpperCase()` 不传 `Locale.ROOT` — ArchUnit 会拦截
- `spring-context` 的测试需要 `@Inject TCK`（JUnit Vintage Engine）
- 修改 `spring-core` 的重打包类（asm、cglib）需重新生成 ShadowJar
- Gradle 配置变更后执行 `--refresh-dependencies`（缓存秒级过期）
