# Spring Boot + Spring Cloud 微服务面试题（上篇）

> 本篇覆盖 Spring Boot 核心面试题 30 道，每题包含：题目、核心答案、追问方向、避坑提示。
> 下篇覆盖 Spring Cloud + 微服务架构面试题。

---

## 1. @SpringBootApplication 注解的自动配置原理，三大注解各自作用

### 题目
`@SpringBootApplication` 是 Spring Boot 应用的入口注解，请描述它的组成三大注解各自的作用，以及自动配置是如何触发的。

### 核心答案

`@SpringBootApplication` 是组合注解，本质等价于以下三个注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration       // 标注该类是 Bean 配置类（内部会自动扫描 @Bean）
@EnableAutoConfiguration // 开启自动配置（核心！）
@ComponentScan       // 组件扫描，默认扫描同包及子包
public @interface SpringBootApplication { ... }
```

**三大注解各自作用：**

| 注解 | 作用 |
|------|------|
| `@Configuration` | 声明这是一个配置类，Spring 容器会将其中标注的 `@Bean` 方法注册为 Bean。同时具有 `proxyBeanMethods` 属性支持 Lite 模式 |
| `@EnableAutoConfiguration` | 启用 Spring Boot 自动配置机制，是自动配置的"开关"，内部通过 `@Import(AutoConfigurationImportSelector.class)` 导入自动配置类 |
| `@ComponentScan` | 组件扫描，默认从声明 `@SpringBootApplication` 类的包路径开始，扫描 `@Component`、`@Service`、`@Repository`、`@Controller` 等并注册为 Bean |

**自动配置触发链：**

1. `@EnableAutoConfiguration` → `@Import(AutoConfigurationImportSelector)`
2. `AutoConfigurationImportSelector` 实现了 `DeferredImportSelector` 接口
3. 在 `selectImports()` 方法中，调用 `SpringFactoriesLoader.loadFactoryNames()` 读取 `META-INF/spring.factories`（Spring Boot 2.7+ 改为 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）
4. 加载所有声明的自动配置类，通过 `@Conditional` 系列注解进行条件匹配
5. 匹配成功的配置类被注册为 Bean，完成自动装配

### 追问方向
- 自动配置类的加载顺序如何控制？（`@AutoConfigureBefore`、`@AutoConfigureOrder`）
- 为什么有时候自动配置不生效？如何排查？
- `spring.factories` 和 `AutoConfiguration.imports` 的区别是什么？

### 避坑提示
- 不要把 `@SpringBootApplication` 等同于 `@ComponentScan` + `@EnableAutoConfiguration`，忽略了 `@Configuration` 的 `proxyBeanMethods` 默认为 true 这个细节
- 自动配置不生效的常见原因是自定义配置覆盖了自动配置类（可以通过 `@EnableConfigurationProperties` 或 `exclude` 排查）

---

## 2. Spring Boot 的 starter 机制是怎么实现的，为什么引入 starter 就能生效

### 题目
Spring Boot 的 starter 机制让开发者只需引入一个依赖就能获得整套功能，请描述 starter 的实现原理以及为什么引依赖就能生效。

### 核心答案

**Starter 的本质：**

Starter 是一个约定大于配置的 Maven 依赖包，包含以下组件：

```
spring-boot-starter-xxx/
├── pom.xml                    # 声明功能依赖 + 传递依赖版本
└── src/main/resources/
    └── META-INF/
        └── spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
            # 列出该 starter 的自动配置类（Spring Boot 3.x）
        # 或 META-INF/spring.factories（Spring Boot 2.x）
```

**为什么引入 starter 就生效：**

1. **依赖传递**：starter 的 `pom.xml` 中声明了所有必要的依赖（如 `spring-boot-starter` 自身），开发者引入一个坐标， Maven/Gradle 解析传递依赖后，相关类就进入了 classpath

2. **自动配置类注册**：starter 的 `AutoConfiguration.imports`（或 `spring.factories`）中注册了配置类，Spring Boot 在启动时扫描并加载这些配置类

3. **条件匹配**：配置类上标注了 `@Conditional*` 注解（如 `@ConditionalOnClass`），只有当 classpath 中存在特定类时才会生效（如 Redis starter 检测到 `Jedis.class` 存在才注册 Jedis 相关的 Bean）

4. **配置属性绑定**：starter 通常提供默认配置，同时支持用户通过 `application.yml` 覆盖（如 `spring.redis.host`）

**一个典型 starter 的结构示例：**

```xml
<!-- my-starter/pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>my-library</artifactId>
    </dependency>
</dependencies>
```

```properties
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfigure.MyAutoConfiguration
```

```java
@Configuration
@ConditionalOnClass(MyLibrary.class) // classpath 中有 MyLibrary 才生效
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    // 注册需要的 Bean
}
```

### 追问方向
- 自定义 starter 如何给用户提供默认配置，同时允许用户覆盖？
- starter 的 `spring-boot-starter` 和 `spring-boot-starter-parent` 区别是什么？
- `@ConditionalOnMissingBean` 在 starter 中的作用是什么？

### 避坑提示
- 区分"功能 starter"（如 `spring-boot-starter-web`）和"parent"（`spring-boot-starter-parent`），前者是依赖包，后者提供版本管理
- 不要在 starter 的自动配置类中直接写死配置值，应该用 `@ConfigurationProperties` 绑定到属性类，让用户可配置

---

## 3. Spring Boot 自动配置类的加载时机和条件判断（@Conditional 系列注解）

### 题目
Spring Boot 自动配置类的加载时机是什么？`@Conditional` 系列注解是如何控制配置类是否生效的？请列举常用条件注解及其作用。

### 核心答案

**加载时机：**

自动配置类的加载发生在 Spring 容器刷新（`refresh()`）阶段，具体在 `ConfigurationClassPostProcessor` 处理 `@Configuration` 配置类时，通过 `AutoConfigurationImportSelector` 导入。

时序：`SpringApplication.run()` → `prepareContext()` → `load(context, sources)` → 处理 `@Import(AutoConfigurationImportSelector)` → `selectImports()` 返回配置类名 → 注册为 Bean

**@Conditional 系列注解：**

| 注解 | 作用 |
|------|------|
| `@ConditionalOnClass` | classpath 存在指定类才生效 |
| `@ConditionalOnMissingClass` | classpath 不存在指定类才生效 |
| `@ConditionalOnBean` | 容器中存在指定 Bean 才生效 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean 才生效 |
| `@ConditionalOnProperty` | 配置属性满足条件才生效（如 `spring.redis.host=xxx`） |
| `@ConditionalOnWebApplication` | 是 Web 应用才生效 |
| `@ConditionalOnNotWebApplication` | 非 Web 应用才生效 |
| `@ConditionalOnExpression` | SpEL 表达式为 true 才生效 |
| `@ConditionalOnJava` | JDK 版本满足条件才生效 |

**条件判断的组合使用：**

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)         // classpath 有 Redis 客户端
@ConditionalOnMissingBean(RedisTemplate.class)    // 容器中没有自定义 RedisTemplate
@AutoConfigureAfter(RedisAutoConfiguration.class) // 在 Redis 自动配置之后
public class MyRedisAutoConfiguration {
    @Bean
    @ConditionalOnProperty(name = "my.redis.enabled", havingValue = "true")
    public MyRedisTemplate myRedisTemplate() {
        return new MyRedisTemplate();
    }
}
```

**执行顺序的决定因素：**
1. `@AutoConfigureBefore` / `@AutoConfigureAfter` 注解控制同包内配置类的顺序
2. `AutoConfiguration.imports` 文件中的顺序（Spring Boot 3.x）
3. `@AutoConfigureOrder` 可调整顺序（数值越小越先执行）

### 追问方向
- `@ConditionalOnMissingBean` 的坑：为什么说它可能导致用户的自定义 Bean 被跳过？
- Spring Boot 2.7 后自动配置加载机制有什么变化？
- 如何用 `@Configuration` + `@Conditional` 自己实现一个条件注册 Bean？

### 避坑提示
- `@ConditionalOnMissingBean` 是在自动配置阶段检查的，如果用户 Bean 在配置类加载**之后**才定义，仍然会生效。所以定义顺序很重要
- 不要在自动配置类中使用 `@Primary` 覆盖用户手动定义的 Bean（应该让用户的配置优先级更高）

---

## 4. Spring Boot 如何实现多环境配置（application-{profile}.yml）

### 题目
Spring Boot 如何实现多环境配置？`application-{profile}.yml` 的加载规则是什么？不同环境的配置如何切换？

### 核心答案

**多环境配置实现方式：**

Spring Boot 支持通过**激活不同的 Profile** 来切换环境配置：

```bash
# 激活方式1：命令行参数
java -jar app.jar --spring.profiles.active=dev

# 激活方式2：环境变量
export SPRING_PROFILES_ACTIVE=prod

# 激活方式3：JVM 参数
java -Dspring.profiles.active=test -jar app.jar

# 激活方式4：application.yml 中指定
# spring.profiles.active=dev
```

**配置文件加载规则：**

Spring Boot 加载配置文件的优先级顺序（从高到低）：

1. `application-{profile}.yml`（高优先级）
2. `application.yml`（基础配置，作为默认）
3. `classpath:application-{profile}.yml`
4. `classpath:application.yml`

**示例结构：**

```
src/main/resources/
├── application.yml          # 公共配置（数据库连接池大小、日志级别等）
├── application-dev.yml      # 开发环境（localhost、debug 日志）
├── application-test.yml     # 测试环境（测试数据库）
└── application-prod.yml     # 生产环境（高并发、安全配置）
```

```yaml
# application.yml
spring:
  profiles:
    active: dev   # 默认激活 dev
  datasource:
    url: jdbc:mysql://localhost:3306/common_db  # 公共配置
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: root
    password: root
server:
  port: 8080
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/prod_db
    username: prod_user
    password: ${DB_PASSWORD}  # 生产环境从环境变量读取
server:
  port: 80
logging:
  level:
    root: WARN
    com.example: INFO
```

**@Profile 注解：**

除了配置文件级别的 profile，还可以使用 `@Profile("dev")` 注解在类或方法级别指定激活的 profile 才注册该 Bean：

```java
@Service
@Profile("dev")
public class DevStorageService implements StorageService { ... }

@Service
@Profile("prod")
public class ProdStorageService implements StorageService { ... }
```

### 追问方向
- 如果 application.yml 和 application-dev.yml 中有相同的 key，哪个生效？
- 如何在代码中获取当前激活的 profile？
- `@PropertySource` 能读取 YAML 吗？

### 避坑提示
- 公共配置写在主 `application.yml`，环境特有配置写在 `application-{profile}.yml`，避免重复配置
- 不要在 application-prod.yml 中明文写密码，使用环境变量 `${DB_PASSWORD}` 引用

---

## 5. 外部化配置优先级，ConfigFileApplicationListener 加载顺序

### 题目
Spring Boot 支持从多种来源加载配置（命令行参数、环境变量、配置文件等），请描述外部化配置的整体优先级，以及 `ConfigFileApplicationListener` 的加载顺序。

### 核心答案

**外部化配置优先级（从高到低）：**

```
1. 命令行参数（--server.port=9000）
2. SPRING_APPLICATION_JSON（内嵌 JSON 的环境变量或请求体）
3. ServletConfig/ServletContext 参数
4. JNDI 属性
5. Java System 属性（-D参数）
6. OS 环境变量（操作系统级）
7. random.* 配置（RandomValuePropertySource）
8. jar 包外部的 application-{profile}.yml / .properties
9. jar 包内部的 application-{profile}.yml / .properties
10. jar 包外部的 application.yml / .properties
11. jar 包内部的 application.yml / .properties
12. @PropertySource 加载的配置文件（默认不加载）
13. SpringApplication.setDefaultProperties() 设置的默认属性
```

**ConfigFileApplicationListener（Spring Boot 2.4 之前）加载顺序：**

`ConfigFileApplicationListener` 实现了 `EnvironmentPostProcessor` 接口，在 Spring 容器启动早期（`prepareEnvironment` 阶段）加载 `application.yml` 相关配置：

```
Bootstrap ApplicationListener 链
    ↓
ConfigFileApplicationListener.postProcessEnvironment()
    ↓
加载 application-{profile}.yml（从 classpath 和当前目录查找）
    ↓
应用 @PropertySource 注解（如果有）
    ↓
PropertySources-placeholder 解析
```

Spring Boot 2.4 之后，引入了 `ConfigDataEnvironmentPostProcessor`，替代了原来的 `ConfigFileApplicationListener`，支持 `spring.config.import` 导入外部配置：

```yaml
# application.yml
spring:
  config:
    import: optional:file:./config/external.yml
```

**关键加载顺序原则：**
- 高优先级的配置会**覆盖**低优先级中同名的配置
- 后加载的配置会覆盖先加载的同名配置
- Profile 特定的配置（`application-dev.yml`）会与主配置（`application.yml`）**合并**而非完全替换

### 追问方向
- `spring.config.location` 和 `spring.config.additional-location` 的区别？
- 如何让 Spring Boot 在启动时不加载任何配置文件？
- `@PropertySource` 为什么默认不加载 YAML 文件？

### 避坑提示
- OS 环境变量的优先级高于 jar 包内的配置文件，这在某些部署场景下可能与预期不符
- 不要同时在 application.yml 和 application-prod.yml 中写相同的 `@ConfigurationProperties` 类属性，容易造成排查困难

---

## 6. Spring Boot Actuator 端点有哪些，如何自定义健康检查指标

### 题目
Spring Boot Actuator 提供了哪些内置端点？如何自定义一个健康检查指标（Health Indicator）？

### 核心答案

**常用 Actuator 端点：**

| 端点 | 路径 | 作用 |
|------|------|------|
| health | `/actuator/health` | 应用健康状态 |
| info | `/actuator/info` | 应用自定义信息 |
| env | `/actuator/env` | 环境变量 |
| beans | `/actuator/beans` | 容器中所有 Bean |
| configprops | `/actuator/configprops` | 配置属性 |
| mappings | `/actuator/mappings` | 所有 URL 映射 |
| metrics | `/actuator/metrics` | 指标数据 |
| loggers | `/actuator/loggers` | 日志级别 |
| heapdump | `/actuator/heapdump` | 堆转储（需授权） |
| threaddump | `/actuator/threaddump` | 线程转储 |
| scheduledtasks | `/actuator/scheduledtasks` | 定时任务 |
| healthchecks | `/actuator/health` | 聚合各组件健康状态 |

**在 application.yml 中暴露端点：**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
      base-path: /actuator   # 端点基础路径
  endpoint:
    health:
      show-details: always   # 显示详细信息（生产环境建议 authorized）
  health:
    db:
      enabled: true          # 启用数据库健康检查
    redis:
      enabled: true          # 启用 Redis 健康检查
```

**自定义健康检查指标：**

方式一：实现 `HealthIndicator` 接口

```java
@Component
public class MyServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            // 执行健康检查逻辑
            boolean healthy = checkMyService();
            if (healthy) {
                return Health.up()
                    .withDetail("service", "MyService")
                    .withDetail("status", "running")
                    .build();
            } else {
                return Health.down()
                    .withDetail("error", "Service unavailable")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("service", "MyService")
                .build();
        }
    }

    private boolean checkMyService() {
        // 实际检查逻辑，如 ping 数据库、调用接口等
        return true;
    }
}
```

方式二：使用 `@HealthIndicator` 注解（Spring Boot 3.x）

```java
@Component
@HealthIndicator(name = "myService")
public class MyServiceHealth {
    public Health health() {
        // ...
    }
}
```

**聚合多个健康检查：**

`/actuator/health` 会自动聚合所有 `HealthIndicator` 的结果：

```json
{
  "status": "DOWN",
  "components": {
    "db": { "status": "UP" },
    "redis": { "status": "DOWN", "details": { "error": "Connection refused" } },
    "myService": { "status": "UP" }
  }
}
```

### 追问方向
- 如何自定义 metrics 指标并暴露到 `/actuator/metrics`？
- `HealthIndicator` 返回 DOWN 后，应用状态码是多少？
- Actuator 的安全配置如何做，防止端点被未授权访问？

### 避坑提示
- 生产环境 `show-details` 应该设置为 `authorized`，避免暴露敏感信息
- 自定义的 `HealthIndicator` 不要在检查逻辑中做耗时操作，会导致健康检查端点响应慢

---

## 7. Spring Boot 嵌入式 Web 容器（Tomcat/Undertow/Jetty）是如何启动的

### 题目
Spring Boot 内置了 Tomcat/Undertow/Jetty 容器，请描述嵌入式 Web 容器的启动流程，以及如何切换不同的容器。

### 核心答案

**嵌入式容器的启动流程：**

```
SpringApplication.run()
    ↓
prepareContext() → 加载配置、扫描 Bean
    ↓
refreshContext() → AbstractApplicationContext.refresh()
    ↓
onRefresh()  ← ServletWebServerApplicationContext 重写了此方法
    ↓
createWebServer() → 获取 WebServerFactory（Tomcat/Undertow/Jetty）
    ↓
webServer.start() → 启动 HTTP 监听
    ↓
reset() → 发布 ServletContextInitialized 事件
```

**具体步骤：**

1. **WebServerFactory 的创建**：`ServletWebServerApplicationContext.onRefresh()` 调用 `createWebServer()`
2. **容器选择**：根据 classpath 中的类决定使用哪个容器
   - classpath 有 `org.apache.catalina.startup.Tomcat` → Tomcat
   - classpath 有 `io.undertow.Undertow` → Undertow
   - classpath 有 `org.eclipse.jetty.server.Server` → Jetty
3. **端口分配**：若未指定端口，从 `Environment` 获取 `server.port`（默认 8080），`server.address` 绑定地址
4. **Servlet 容器初始化**：创建 `Tomcat/Undertow/Jetty` 实例，配置线程池、连接器、上下文
5. **Tomcat 特殊流程**：`TomcatWebServer` 初始化 `Tomcat` 对象 → 创建 `Connector`（监听端口）→ 配置 `Context`（应用上下文）→ `_host` 和 `_engine`

**切换嵌入式容器：**

排除默认 Tomcat，引入其他容器：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 切换到 Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

**Undertow vs Tomcat vs Jetty 对比：**

| 特性 | Tomcat | Undertow | Jetty |
|------|--------|----------|-------|
| 线程模型 | BIO/NIO（可选） | NIO | NIO |
| 并发性能 | 一般 | 高 | 高 |
| 内存占用 | 较高 | 低 | 低 |
| WebSocket | 支持 | 支持 | 支持 |
| Servlet 版本 | 4.0 | 4.0 | 4.0 |
| 轻量级 | 一般 | 轻量 | 轻量 |

### 追问方向
- 如何配置嵌入式 Tomcat 的线程池、连接数、最大连接数？
- Undertow 的 XNIO 线程模型是什么？
- 嵌入式容器与传统部署（war 包到外部容器）的区别？

### 避坑提示
- 不要在嵌入式容器中使用 `@PreDestroy` 做容器关闭时的长时间清理，会因为 `Thread.interrupt()` 导致非预期行为
- 切换容器时确认 Spring Boot 版本对应的 Servlet API 版本兼容性

---

## 8. Spring Boot 的异常处理机制（ErrorController、BASIC 错误页面、@ExceptionHandler）

### 题目
Spring Boot 提供了哪些异常处理机制？请描述 `ErrorController`、`@ExceptionHandler`、错误页面（4xx.html / error.html）的处理流程和优先级。

### 核心答案

**异常处理机制总览：**

```
请求进来 → Filter → DispatcherServlet
                ↓
        HandlerExceptionResolver（异常解析链）
                ↓
        1. @ExceptionHandler（Controller 内优先）
        2. @ControllerAdvice（全局）
        3. ErrorController（页面/API 兜底）
        ↓
        响应返回（JSON 或错误页面）
```

**@ExceptionHandler（Controller 局部异常处理）：**

```java
@GetMapping("/divide")
public Result<Integer> divide(int a, int b) {
    if (b == 0) throw new BizException("除数不能为零");
    return Result.ok(a / b);
}

@ExceptionHandler(BizException.class)
public Result<Void> handleBizException(BizException e) {
    return Result.error(e.getMessage());
}
```

**@ControllerAdvice（全局异常处理）：**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public Result<Void> handleBiz(BizException e) {
        return Result.error(400, e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleGeneral(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "系统繁忙，请稍后重试");
    }
}
```

**ErrorController（兜底异常处理）：**

Spring Boot 默认注册了 `BasicErrorController`，当没有 `@ExceptionHandler` 能处理时，由它处理：

```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController {

    @RequestMapping(produces = MediaType.TEXT_HTML_VALUE) // HTML 请求 → 错误页面
    public ModelAndView errorHtml() {
        return new ModelAndView("error", ...); // 渲染 error.html
    }

    @RequestMapping // 其他请求 → JSON
    public ResponseEntity<Map<String, Object>> error() {
        Map<String, Object> body = getErrorAttributes(...);
        return new ResponseEntity<>(body, status);
    }
}
```

**自定义 ErrorController：**

```java
@RestController
public class MyErrorController implements ErrorController {
    @RequestMapping("/error")
    public Result<Void> handleError(HttpServletRequest request) {
        Integer status = (Integer) request.getAttribute("jakarta.servlet.error.status_code");
        return Result.error(status, "自定义错误：" + status);
    }
}
```

**错误页面机制：**

在 `src/main/resources/templates/error/` 下放置错误页面：

- `404.html` → 处理 404 错误
- `5xx.html` → 处理所有 5xx 错误
- `error.html` → 兜底错误页面

**处理优先级：**

```
1. @ExceptionHandler（在当前 Controller 中定义） — 最高优先级
2. @ControllerAdvice 中的 @ExceptionHandler
3. @ResponseStatus 注解的异常（如 @ResponseStatus(HttpStatus.NOT_FOUND)）
4. ErrorController.error() — 兜底
5. 容器级异常处理（如 Jetty/Tomcat 的 Valve）
```

### 追问方向
- `@ExceptionHandler` 如何获取 HttpServletRequest、HttpServletResponse？
- `@ControllerAdvice` 的 `basePackages`、`annotations`、`assignableTypes` 属性分别控制什么？
- Spring Boot 3.x 中异常处理有什么变化？

### 避坑提示
- `@ExceptionHandler` 方法的返回值如果是 `ResponseEntity<?>`，会绕过 Spring Boot 的默认错误处理格式
- 全局 `@ControllerAdvice` 默认会拦截所有 Controller 的异常，但如果 Controller 内部已有 `@ExceptionHandler`，会优先使用内部的

---

## 9. Spring Boot 配置绑定（@ConfigurationProperties）原理，对比 @Value

### 题目
`@ConfigurationProperties` 是 Spring Boot 中推荐的配置绑定方式，请描述它的工作原理，以及与 `@Value` 的区别和使用场景。

### 核心答案

**@ConfigurationProperties 工作原理：**

1. **Bean 注册**：标注了 `@ConfigurationProperties` 的类需要在类级别或方法级别标注 `@EnableConfigurationProperties(XxxProperties.class)` 或 `@ConfigurationPropertiesScan`（Spring Boot 2.2+）来触发注册

2. **属性绑定**：Spring Boot 的 `Binder` 类会绑定 `Environment` 中的属性值到 Java Bean 的字段
   - 支持松散绑定（relaxed binding）：`my-config.secure-boolean` = `MyConfig.secureBoolean`
   - 支持复杂类型转换：`String → Integer、Duration、DataSize、Pattern` 等

3. **元数据生成**：编译时通过 `spring-boot-configuration-processor` 自动生成 `META-INF/spring-configuration-metadata.json`，提供 IDE 提示

**使用示例：**

```java
@Component
@ConfigurationProperties(prefix = "myapp")
public class MyAppProperties {
    private String name;
    private int timeout = 5000;
    private List<String> servers = new ArrayList<>();
    private Security security = new Security();

    public static class Security {
        private boolean enabled = true;
        private String token;
        // getters/setters
    }
}
```

```yaml
myapp:
  name: my-application
  timeout: 3000
  servers:
    - server1.example.com
    - server2.example.com
  security:
    enabled: true
    token: ${TOKEN_SECRET}
```

**@ConfigurationProperties vs @Value 对比：**

| 特性 | @ConfigurationProperties | @Value |
|------|--------------------------|--------|
| 绑定目标 | 整个对象（批量绑定） | 单个字段 |
| 宽松绑定 | 支持（recommended-size） | 不支持（严格 SpEL） |
| 占位符解析 | 支持 `${app.name}` | 支持 `#{}`（SpEL） |
| SpEL 表达式 | 不支持 | 支持 `#{systemProperties['os.name']}` |
| 默认值 | 可在 Bean 中定义 | `@Value("xxx")` 指定 |
| 元数据 | 支持 IDE 提示 | 不支持 |
| 校验 | 可配合 `@Validated` | 不支持 |
| 注册为 Bean | 需要配合 `@Component` | 不需要 |
| 类型转换 | 自动（String→多种类型） | 自动（String→类型） |

**使用场景：**
- **@ConfigurationProperties**：配置类、需要批量绑定、配置项多、团队协作、需要在多处注入使用的配置
- **@Value**：简单的单值注入、SpEL 表达式场景、临时调试

### 追问方向
- 什么是松散绑定（Relaxed Binding）？有哪些命名规范？
- `@ConfigurationProperties` 如何结合 `@Validated` 做配置校验？
- 自定义属性转换器如何实现？

### 避坑提示
- `@ConfigurationProperties` 类如果不加 `@Component`，需要通过 `@EnableConfigurationProperties` 显式注册，否则不会生效（Spring Boot 2.2+ 的 `@ConfigurationPropertiesScan` 自动处理）
- `@Value` 中的 SpEL 表达式 `#{...}` 与占位符 `${...}` 不要混淆，前者是 SpEL 表达式，后者是属性占位符

---

## 10. Spring Boot 启动流程：从 main 方法到容器刷新完整链路

### 题目
请描述 Spring Boot 从 `main()` 方法到容器刷新（refresh）的完整启动链路，每个关键步骤做了什么。

### 核心答案

**完整启动链路：**

```
main(String[] args)
    ↓
SpringApplication.run(Application.class, args)
    ↓
1. new SpringApplication(primarySources)
        - 判断 WebApplicationType（NONE/SERVLET/REACTIVE）
        - 加载 spring.factories 中的 BootstrapRegistryInitializer
        - 加载 ApplicationContextInitializer
        - 加载 ApplicationListener
        - 推断 main 方法所在类作为主配置类
    ↓
2. run(String... args)
    ↓
3. prepareEnvironment()
        - 创建 ConfigurableEnvironment
        - 加载命令行参数 --xxx=yyy
        - 加载 JVM 系统属性 -Dxxx=yyy
        - 加载 OS 环境变量
        - 触发 EnvironmentPostProcessor（加载 application.yml）
        - 触发 ApplicationContextInitializer.initialize()
    ↓
4. printBanner() 打印 Banner
    ↓
5. createApplicationContext()
        - 根据 WebApplicationType 创建：
          - REACTIVE → AnnotationConfigReactiveWebServerApplicationContext
          - SERVLET → AnnotationConfigServletWebServerApplicationContext
          - NONE → AnnotationConfigApplicationContext
    ↓
6. prepareContext()
        - 设置 environment、classloader
        - 加载 primarySources（主配置类）
        - 应用 ApplicationContextInitializer
        - 加载 BeanDefinition（扫描 @Component 等）
    ↓
7. refreshContext() ← 关键步骤
        ↓
        AbstractApplicationContext.refresh()
        ├── prepareRefresh()          — 初始化 PropertySources
        ├── obtainFreshBeanFactory()  — 获取 BeanFactory
        ├── prepareBeanFactory()      — 设置 BeanFactory 上下文
        ├── postProcessBeanFactory()   — 子类扩展
        ├── invokeBeanFactoryPostProcessors() — BeanFactory 后置处理（扫描 @Configuration 等）
        ├── registerBeanPostProcessors()      — 注册 BeanPostProcessor
        ├── initMessageSource()               — 国际化
        ├── initApplicationEventMulticaster() — 事件广播器
        ├── onRefresh()              ← 创建 Web 服务器
        │       ↓
        │   createWebServer()
        │       → Tomcat/Undertow/JettyFactory 创建容器
        │       → webServer.start()
        ├── registerListeners()      — 注册监听器
        ├── finishBeanFactoryInitialization() — 单例 Bean 预实例化
        ├── finishRefresh()          — 容器刷新完成
        │       → publishEvent(new ContextRefreshedEvent())
        │       → WebServerStartStopLifecycle.start()
    ↓
8. afterRefresh()（空方法，模板方法供扩展）
    ↓
9. listeners.running() — 发布 ApplicationStartedEvent
    ↓
10. callRunners() — 执行 ApplicationRunner 和 CommandLineRunner
```

**关键细节：**

- `prepareEnvironment` 阶段加载配置，**早于容器创建**，所以 `-D` 参数和命令行参数可以影响环境
- `refresh()` 中的 `onRefresh()` 是 Web 容器初始化的入口，Servlet 环境和非 Servlet 环境在这里分叉
- `finishBeanFactoryInitialization()` 中完成所有非懒加载单例 Bean 的实例化，**非常耗时**
- `callRunners()` 在容器完全就绪后执行，此时所有 Bean 已注入完毕

### 追问方向
- `prepareEnvironment` 和 `prepareContext` 的区别是什么？
- BeanFactoryPostProcessor 和 BeanPostProcessor 的区别？
- Spring Boot 2.4 之后 `ConfigFileApplicationListener` 加载配置文件的时机变化？

### 避坑提示
- 不要在 BeanFactoryPostProcessor 中依赖其他 Bean（此时其他 Bean 可能还未创建）
- `finishBeanFactoryInitialization()` 阶段实例化所有非懒加载单例，如果某个 Bean 初始化很慢，会拖慢整个启动时间

---

## 11. SpringApplication.run() 做了什么，SpringApplication 和 ApplicationContext 的关系

### 题目
`SpringApplication.run()` 方法到底做了什么？`SpringApplication` 实例和 `ApplicationContext` 实例之间的关系是什么？

### 核心答案

**SpringApplication.run() 做了什么：**

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return new SpringApplication(primarySource).run(args);
}

// 实际执行流程
public ConfigurableApplicationContext run(String... args) {
    // 1. 记录启动时间
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();

    // 2. 引导 BootstrapContext（引导上下文）
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();

    // 3. 配置 Headless 属性（无头模式，服务器部署）
    configureHeadlessProperty();

    // 4. 获取并启动 SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, mainApplicationClass);

    // 5. 准备 Environment
    ConfigurableEnvironment environment = prepareEnvironment(bootstrapContext, listeners, args);

    // 6. 打印 Banner
    Banner printedBanner = printBanner(environment);

    // 7. 创建 ApplicationContext
    ConfigurableApplicationContext context = createApplicationContext();

    // 8. 准备 Context
    prepareContext(bootstrapContext, context, environment, listeners, printedBanner);

    // 9. 刷新 Context（核心）
    refreshContext(context);

    // 10. 刷新后处理
    afterRefresh(context, args);

    // 11. 停止计时并发布启动完成事件
    stopWatch.stop();
    listeners.started(context);
    callRunners(context, args);

    // 12. 发布 ApplicationReadyEvent
    listeners.running(context);

    return context;
}
```

**SpringApplication 和 ApplicationContext 的关系：**

```
SpringApplication（引导器/启动器）
    ├── 负责启动流程的编排
    ├── 持有 BootstrapContext
    ├── 持有 ApplicationContextFactory（用于创建 ApplicationContext）
    └── 不属于 IoC 容器，只是启动工具

ApplicationContext（Spring 容器/IoC 容器）
    ├── 是真正的 Bean 容器
    ├── 由 SpringApplication 在 run() 过程中创建
    ├── 实现了 ConfigurableApplicationContext 接口
    └── 包含所有 BeanDefinition 和单例 Bean
```

**创建时机：**

```
SpringApplication.run()
    ↓
createApplicationContext()  ← 创建 ApplicationContext
    ↓
refreshContext(context)     ← 填充 Bean
```

**可以获取的方式：**

```java
// 在 Bean 中实现 ApplicationContextAware
@Component
public class MyBean implements ApplicationContextAware {
    private ApplicationContext ctx;
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
}

// 或实现 EnvironmentAware
// 或实现 BeanFactoryAware
```

### 追问方向
- `SpringApplication` 为什么要单独持有一个 `BootstrapContext`？
- `ApplicationContext` 和 `BeanFactory` 的区别？
- `SpringApplicationRunListener` 的加载时机和作用？

### 避坑提示
- 不要在 `SpringApplication.run()` 返回之前尝试从容器中获取 Bean（容器还未刷新，Bean 不存在）
- `SpringApplication` 本身不是 Bean，除非在创建时手动注册为 Bean

---

## 12. Spring Boot 与 Spring Cloud 版本对应关系

### 题目
Spring Boot 和 Spring Cloud 的版本之间有严格的对应关系，请说明版本对应规则，以及如何在项目中选择兼容的版本。

### 核心答案

**版本对应规则：**

Spring Cloud 是基于 Spring Boot 的版本进行设计的，遵循 **London Underground** 版本命名规则（从 Angel 到 Wellington），每个 Spring Cloud 版本声明了它所支持的 Spring Boot 版本范围。

**主要版本对应（截至 Spring Cloud 2023.x）：**

| Spring Cloud 版本 | Spring Boot 版本 | 日期 |
|-------------------|-----------------|------|
| 2023.0.x (Leyton) | 3.2.x | 2024-01 |
| 2022.0.x (Kilburn) | 3.0.x, 3.1.x | 2023-01 |
| 2021.0.x (Jubilee) | 2.6.x, 2.7.x | 2022-01 |
| 2020.0.x (Ilford) | 2.4.x, 2.5.x | 2021-01 |
| Hoxton | 2.2.x, 2.3.x | 2020-01 |
| Greenwich | 2.1.x | 2019-01 |
| Finchley | 2.0.x | 2018-01 |

**Spring Boot 3.x 对应关系（Spring Cloud 2022.0.0+）：**

```
Spring Boot 3.2.x → Spring Cloud 2023.0.x
Spring Boot 3.1.x → Spring Cloud 2022.0.x
Spring Boot 3.0.x → Spring Cloud 2022.0.x
```

**Spring Boot 2.x 经典对应（Spring Cloud 2021.x）：**

```
Spring Boot 2.7.x → Spring Cloud 2021.0.5 (2023-03 已停止维护)
Spring Boot 2.6.x → Spring Cloud 2021.0.x
Spring Boot 2.5.x → Spring Cloud 2020.0.x
Spring Boot 2.4.x → Spring Cloud 2020.0.x（2021.0.x 也支持）
```

**pom.xml 配置示例（Spring Cloud 2021.0.5 + Spring Boot 2.7.x）：**

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
</parent>

<properties>
    <spring-cloud.version>2021.0.8</spring-cloud.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 追问方向
- Spring Cloud 的子项目（如 Feign、Hystrix、Gateway）各自对 Spring Boot 版本的要求？
- Spring Boot 3.x 需要 JDK 17+，Spring Cloud 2022.0+ 支持 JDK 17 的变化是什么？
- 如何查看当前 Spring Cloud 版本支持哪些 Spring Boot 版本？

### 避坑提示
- **不要混用版本**：Spring Boot 2.7 + Spring Cloud 2022.0 会导致依赖冲突，因为 2022.0 要求 Spring Boot 3.x
- 使用 `spring-cloud-starter-parent` 作为 parent 可以自动管理版本，避免手动对齐
- 维护老项目时，升级 Spring Boot 版本前必须先确认 Spring Cloud 版本是否支持

---

## 13. Spring Boot 读取配置的几种方式（Environment、@ConfigurationProperties、@PropertySource）

### 题目
Spring Boot 支持多种配置读取方式，请列举并比较 `Environment`、`@ConfigurationProperties`、`@PropertySource` 三种方式的作用、特点和使用场景。

### 核心答案

**1. Environment API**

`Environment` 是 Spring Boot 配置属性源（PropertySource）的抽象容器，提供对所有配置属性的统一访问：

```java
@Autowired
private Environment env;

// 获取属性值
String port = env.getProperty("server.port");
String dbUrl = env.getProperty("spring.datasource.url");
Integer timeout = env.getProperty("myapp.timeout", Integer.class, 5000);
```

`Environment` 内部维护了一个 `MutablePropertySources` 列表，按优先级排序：
- `SystemEnvironmentPropertySource`（OS 环境变量）
- `ServletConfigPropertySource`（Servlet 配置）
- `ServletContextPropertySource`（Servlet 上下文）
- `CommandLinePropertySource`（命令行参数）
- `ApplicationConfigurationPropertySource`（application.yml）
- `RandomValuePropertySource`（random.*）

**2. @ConfigurationProperties（推荐方式）**

用于将外部配置批量绑定到 POJO：

```java
@Component
@ConfigurationProperties(prefix = "myapp")
public class MyAppConfig {
    private String host;
    private int port;
}
```

特点：
- 批量绑定，类型安全
- 支持松散绑定
- 支持 IDE 元数据提示
- 可结合 `@Validated` 做校验
- 编译时生成配置元数据

**3. @PropertySource**

将外部 `.properties` 文件加载为 PropertySource：

```java
@PropertySource(value = "classpath:custom.properties", ignoreResourceNotFound = true)
@PropertySource(value = "file:./config/external.properties")
public class AppConfig {
    // 需要配合 @Value 使用
    @Value("${my.custom.prop}")
    private String customProp;
}
```

**对比：**

| 特性 | Environment | @ConfigurationProperties | @PropertySource |
|------|-------------|---------------------------|-----------------|
| 配置来源 | 所有 PropertySource | application.yml + 自定义 | 指定文件 |
| 返回类型 | String/Object | POJO 对象 | 配合 @Value |
| 批量绑定 | 否（需逐个获取） | 是 | 否 |
| 类型转换 | 自动 | 自动 | 自动 |
| 校验 | 不支持 | 支持 @Validated | 不支持 |
| 默认值 | getProperty 重载支持 | Bean 中定义 | @Value 指定 |
| 加载时机 | 任何时候 | 容器刷新 | 容器刷新 |

**混合使用示例：**

```java
@Configuration
@PropertySource("classpath:custom.properties")
@EnableConfigurationProperties(MyAppConfig.class)
public class AppConfig {

    @Autowired
    private Environment env;

    @Autowired
    private MyAppConfig myAppConfig;  // @ConfigurationProperties 绑定
}
```

### 追问方向
- `@PropertySource` 默认不支持 YAML，如何让它支持？
- 如何自定义 PropertySource 并插入到 Environment 的指定优先级？
- `@PropertySource` 和 `spring.config.import` 的区别？

### 避坑提示
- `@PropertySource` 不会加载 YAML 文件，只支持 `.properties` 文件
- `@PropertySource` 默认在找不到文件时抛出异常，设置 `ignoreResourceNotFound = true` 可以忽略
- `Environment.getProperty()` 返回的是 String，即使属性值是数字也需要手动转换或使用重载方法指定类型

---

## 14. Spring Boot 的 Banner 如何定制

### 题目
Spring Boot 启动时打印的 Banner 是如何定制的？可以定制哪些内容？

### 核心答案

**Banner 定制方式：**

**方式一：文本 Banner（文本文件）**

在 `src/main/resources/` 下放置 `banner.txt`：

```
________________            __________________
\                \          /                /
 \                \        /                /
  \                \      /                /
   |  ____  __  __  ____|____  __  __  ____|
   | |    ||  |/  |/    |    \|  |/  |/    |
   | |____||__|\__|____ |____|\__|\__|____|
   |                                     |
   |  MyApp v1.0.0                       |
   |  Spring Boot ${spring-boot.version}    |
   |                                     |
   /_____________________________________\
```

**Banner 占位符：**

| 占位符 | 说明 |
|--------|------|
| `${application.version}` | 应用版本号 |
| `${application.title}` | 应用标题 |
| `${spring-boot.version}` | Spring Boot 版本 |
| `${spring-boot.formatted-version}` | 带前缀的 Spring Boot 版本（v3.2.0） |
| `${Ansi.NAME}` | ANSI 彩色转义码 |

**方式二：图片 Banner**

将图片命名为 `banner.gif`、`banner.jpg` 或 `banner.png`，放在 `src/main/resources/` 下：

```
src/main/resources/
└── banner.gif   # Spring Boot 会将图片的像素逐行打印到控制台
```

**方式三：ANSI 彩色 Banner**

```txt
__        __             _____                          _
\ \      / /__  _ __ __| ____|__ _  __ __ _  __| | ___| |_ ___
 \ \ /\ / / _ \| '__/ _` |  _| / _` |/ _` |/ _` |/ _ \ __/ __|
  \ V  V / (_) | | | (_| | |__| (_| | (_| | (_| |  __/ |_\__ \
   \_/\_/ \___/|_|  \__,_|_____\__,_|\__,_|\__,_|\___|\__|___/
:: Spring Boot :: (v${spring-boot.version})
```

**方式四：代码设置（代码配置）**

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(MyApplication.class);
        app.setBannerMode(Banner.Mode.OFF); // 关闭 Banner
        // 或使用自定义 Banner
        app.setBanner(new ResourceBanner(new ClassPathResource("banner-custom.txt")));
        app.run(args);
    }
}
```

**Banner 模式：**

| 模式 | 说明 |
|------|------|
| `Banner.Mode.CONSOLE` | 打印到控制台（默认） |
| `Banner.Mode.LOG` | 打印到日志 |
| `Banner.Mode.OFF` | 关闭 Banner |
| `Banner.Mode.EXIT` | 打印后直接退出（不启动应用） |

**动态生成 Banner：**

```java
public class MyBanner implements Banner {
    @Override
    public void printBanner(Environment environment, Class<?> sourceClass, PrintStream out) {
        out.println("========================================");
        out.println("  Application: " + environment.getProperty("spring.application.name"));
        out.println("  Profile: " + Arrays.toString(environment.getActiveProfiles()));
        out.println("========================================");
    }
}
```

### 追问方向
- 如何在 Spring Boot 2.x 和 3.x 中分别定制 Banner？
- Banner 图片的 ASCII 艺术是如何生成的？
- 如何在 Banner 中显示当前激活的 Profile？

### 避坑提示
- `banner.txt` 文件编码建议使用 UTF-8，否则中文可能显示乱码
- 图片 Banner 在日志文件中不生效，只在控制台有效

---

## 15. Spring Boot 日志配置，默认日志框架是什么，日志级别配置

### 题目
Spring Boot 默认使用什么日志框架？如何配置日志级别？有哪些配置方式？

### 核心答案

**默认日志框架：**

Spring Boot 2.x 开始默认使用 **Logback** 作为日志实现（Spring Boot 1.x 使用 Log4j）。

日志门面（抽象层）：**SLF4J**（Simple Logging Facade for Java）

依赖链：
```
应用代码 → SLF4J API → Logback（默认实现）
                          ↑
            如果切换 → Log4j2（spring-boot-starter-log4j2）
```

**日志配置方式：**

**方式一：application.yml 中配置（最简单）**

```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    com.example.service: TRACE
    org.springframework.web: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: /var/log/myapp.log
    max-size: 100MB
    max-history: 30
```

**方式二：logback-spring.xml（推荐，完整控制）**

在 `src/main/resources/` 下放置 `logback-spring.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}/}spring.log}"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <springProfile name="dev">
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="WARN">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="FILE"/>
        </root>
    </springProfile>
</configuration>
```

**日志级别：**

```
TRACE < DEBUG < INFO < WARN < ERROR < FATAL（OFF）
```

**常用日志框架切换：**

排除 Logback，引入 Log4j2：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

### 追问方向
- Logback 的 `RollingFileAppender` 有哪几种滚动策略？
- 如何在运行时动态修改日志级别？
- SLF4J 的占位符 `{}` 和字符串拼接 `+` 的性能区别？

### 避坑提示
- 不要在日志级别为 DEBUG 时使用 `logger.info("User: " + user)` 做字符串拼接，应用 `logger.debug("User: {}", user)` 的占位符方式，前者即使不打印也会执行字符串拼接
- 生产环境不要设置 `root: DEBUG`，日志量会非常大影响性能

---

## 16. Spring Boot Bean 的懒加载（@Lazy），单例 Bean 的线程安全问题

### 题目
Spring Boot 中 Bean 的懒加载是如何实现的？单例 Bean 在多线程环境下存在哪些线程安全问题？如何解决？

### 核心答案

**@Lazy 懒加载：**

`@Lazy` 注解可以让 Bean 在首次使用时才创建，而非应用启动时：

```java
@Component
@Lazy
public class HeavyService {
    // 第一次调用时才实例化
}
```

**@Lazy 原理：**

1. 标注 `@Lazy` 后，Spring 不会在 `finishBeanFactoryInitialization()` 阶段预实例化该 Bean
2. 首次 `getBean()` 时，通过 `BeanWrapper` + JDK/CGLIB 代理触发真实 Bean 的创建
3. `DefaultListableBeanFactory.createBean()` 中检查 `AbstractBeanFactory.isLazyInit()` 标志

```java
// 源码简析
if (mbd.isLazyInit() && !containsSingleton(beanName)) {
    // 不在启动时创建，等到 getBean 时再创建
}
```

**@Lazy 在 @Configuration 类中的特殊行为：**

```java
@Configuration
public class AppConfig {
    @Bean
    @Lazy
    public MyService myService() {
        return new MyService();
    }
}
```

`@Configuration` 的 `proxyBeanMethods = true` 时，`@Lazy` 实际代理的是配置类而非返回的 Bean，导致第一次调用时创建的是代理对象。

**单例 Bean 的线程安全问题：**

Spring 默认注册的 Bean 是**单例（Singleton）**，在多线程环境下存在以下问题：

1. **可变共享状态**：Bean 中的实例字段（不是 static，也不是 final）会被多个线程并发访问和修改

```java
@Component
public class SharedCounter {
    private int count = 0;  // 危险！多个线程共享此字段

    public void increment() {
        count++;  // 非原子操作，存在竞态条件
    }
}
```

2. **非线程安全的集合**：`HashMap`、`ArrayList`、`SimpleDateFormat` 等非线程安全类在多线程环境下可能数据错乱

**解决方案：**

| 方案 | 适用场景 |
|------|----------|
| 使用无状态 Bean（推荐） | Bean 不持有可变状态，只做逻辑处理 |
| 使用 `ThreadLocal` | 需要线程本地变量时 |
| 使用同步：`synchronized` 或 `ReentrantLock` | 必须修改共享状态时（性能差） |
| 使用线程安全集合 | 如 `ConcurrentHashMap`、`CopyOnWriteArrayList` |
| 改变 Bean 作用域为 `prototype`（不推荐） | 每个请求一个实例，但无法利用容器缓存优势 |
| 使用 `java.util.concurrent` 包 | `AtomicInteger`、`AtomicReference` 等原子类 |

```java
// 推荐：无状态 Bean
@Component
public class OrderService {
    // 不持有任何实例字段，只有方法参数和局部变量
    public Order createOrder(String userId, List<Item> items) {
        // 所有变量都是局部变量，天然线程安全
        return new Order(userId, items);
    }
}

// 使用 ThreadLocal
@Component
public class RequestContext {
    private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

    public static void set(User user) { currentUser.set(user); }
    public static User get() { return currentUser.get(); }
}
```

### 追问方向
- `@Lazy` 和 `BeanFactoryPostProcessor` 的懒加载有什么区别？
- 为什么 `@Configuration(proxyBeanMethods = true)` 的类中 `@Lazy` 修饰的 @Bean 实际上代理的是配置类？
- `@Scope("prototype")` 的 Bean 何时销毁？

### 避坑提示
- 绝大多数 Spring Bean 应该是无状态的，将业务逻辑放在无状态服务类中，状态放到数据库或缓存
- `SimpleDateFormat` 是线程不安全的，不要定义为 Bean 的实例字段

---

## 17. Spring Boot Runner 和 ApplicationRunner 的区别

### 题目
Spring Boot 提供了 `ApplicationRunner` 和 `CommandLineRunner` 两个接口用于在应用启动后执行代码，请描述它们的区别和使用场景。

### 核心答案

**共同点：**

两者都在 `SpringApplication.run()` 完成容器刷新后，`callRunners()` 阶段执行，都在**所有单例 Bean 初始化完毕后**执行，用于应用启动后的初始化任务。

```java
// 执行时机
refreshContext(context);
afterRefresh(context, args);
callRunners(context, args);  // ← 两者都在这里执行
```

**区别：**

| 特性 | ApplicationRunner | CommandLineRunner |
|------|-------------------|-------------------|
| 参数类型 | `ApplicationArguments` | `String[] args` |
| 接口包 | `org.springframework.boot` | `org.springframework.boot` |
| 方法名 | `run(ApplicationArguments args)` | `run(String... args)` |
| 优点 | 提供命名参数（`--name=value`）解析 | 直接获取原始命令行参数 |
| 访问参数 | `args.getOptionValues("name")` | 直接使用 `args[0]` |
| Option 参数解析 | 内置支持 `--foo=bar` | 需要手动解析 |

**使用示例：**

```java
@Component
@Order(1)  // 指定执行顺序
public class MyApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner 运行");
        System.out.println("非选项参数: " + args.getNonOptionArgs());
        System.out.println("选项参数 foo: " + args.getOptionValues("foo"));
        // java -jar app.jar --foo=bar hello
        // args.getOptionValues("foo") → ["bar"]
        // args.getNonOptionArgs() → ["hello"]
    }
}
```

```java
@Component
@Order(2)
public class MyCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("CommandLineRunner 运行");
        System.out.println("原始参数: " + Arrays.toString(args));
    }
}
```

**执行顺序控制：**

1. `@Order(value)` 注解（值越小优先级越高）
2. 实现 `Ordered` 接口

```java
@Component
public class FirstRunner implements ApplicationRunner, Ordered {
    @Override
    public int getOrder() { return 0; }  // 最先执行

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // ...
    }
}
```

**常见使用场景：**

- `ApplicationRunner`：解析命令行参数（如 `--spring.config.location`），初始化缓存预热数据，数据库初始化脚本
- `CommandLineRunner`：打印启动信息，执行一次性批处理任务，记录启动参数

### 追问方向
- `@Order` 的值相同会怎样？
- Runner 抛出异常会导致应用启动失败吗？
- `ApplicationArguments` 和 `String[] args` 的具体区别是什么？

### 避坑提示
- 如果有多个 Runner 且它们有依赖关系，不要依赖 `@Order` 做严格排序，可能存在不确定性，应该通过 `@Autowired` + 方法调用确保执行顺序
- Runner 中的代码应该快速完成，避免在启动时执行耗时很长的任务，影响应用启动时间

---

## 18. Spring Boot 打成的 jar 包为什么可以直接运行，内嵌容器原理

### 题目
传统的 Spring MVC 应用需要打成 war 包部署到外部 Tomcat，而 Spring Boot 应用可以打成 jar 包直接运行，这是如何实现的？内嵌容器的原理是什么？

### 核心答案

**Spring Boot JAR 直接运行的原理：**

Spring Boot 的 jar 包称为 **Fat JAR**（或 Uber JAR），结构如下：

```
myapp.jar
├── META-INF/
│   └── MANIFEST.MF
│       Main-Class: org.springframework.boot.loader.JarLauncher
│       Start-Class: com.example.MyApplication
├── org/springframework/boot/loader/          ← Spring Boot Loader（嵌套 JAR 加载器）
│   ├── JarLauncher.class
│   ├── LaunchedURLClassLoader.class
│   └── ...
├── BOOT-INF/
│   ├── classes/                              ← 应用类文件
    │   └── com/example/MyApplication.class
│   └── lib/                                  ← 依赖 JAR（嵌套 JAR）
│       ├── spring-boot-3.2.x.jar
│       ├── spring-core-6.x.jar
│       └── ...
```

**MANIFEST.MF 关键内容：**

```mf
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.example.MyApplication
```

**启动流程：**

```
java -jar myapp.jar
    ↓
系统类加载器加载 JarLauncher
    ↓
JarLauncher.main()
    ↓
创建 LaunchedURLClassLoader（自定义 ClassLoader）
    ↓
从 BOOT-INF/lib/ 中加载所有依赖 JAR（通过 nested JAR 机制）
    ↓
加载 BOOT-INF/classes/ 中的应用类
    ↓
反射调用 Start-Class 的 main() 方法
    ↓
SpringApplication.run()
```

**嵌套 JAR 加载原理（Spring Boot 2.3+ 使用 Z JAR）：**

Spring Boot 2.3 之前使用 `Spring Boot Loader` 的 `ExplodedJarArchive` 手动解压和加载嵌套 JAR。

Spring Boot 2.3+ 使用 **Z JAR** 格式（基于 JDK 9+ 的模块化特性），但仍然兼容旧格式：

1. `JarLauncher` 通过特殊的 `ClassLoader` 实现从 `BOOT-INF/lib/` 下的多个 JAR 中加载类
2. 每个嵌套 JAR 都有自己的 `MANIFEST.MF`，由 `NestedJarArchiveEntry` 表示
3. `LaunchedURLClassLoader` 重写了 `findClass()` 和 `loadClass()`，按嵌套 JAR 的顺序查找类

**内嵌容器的启动：**

```
SpringApplication.run()
    ↓
refreshContext(context)
    ↓
ServletWebServerApplicationContext.onRefresh()
    ↓
createWebServer()
    ↓
根据 classpath 选择 Tomcat/Undertow/Jetty
    ↓
Tomcat tomcat = new Tomcat()
    tomcat.setPort(port)
    tomcat.getConnector()  // 创建连接器
    Context context = tomcat.addContext("", baseDir)
    // 注册 DispatcherServlet
    → StandardWrapper沙箱
    tomcat.start()
    ↓
内嵌 Tomcat 监听 HTTP 请求
```

内嵌容器本质上是**在应用进程中**实例化 `Tomcat/Undertow/Jetty` 对象，调用其 API 启动 HTTP 服务，而不是依赖外部容器。

**与 war 包部署的对比：**

| 维度 | Spring Boot Fat JAR | 传统 war 包部署 |
|------|---------------------|-----------------|
| 部署方式 | `java -jar` 直接运行 | 部署到外部容器 |
| 容器位置 | 内嵌（应用进程内） | 独立进程 |
| 端口配置 | `server.port` | 容器配置文件 |
| 类加载器 | Spring Boot Loader | 容器类加载器 |
| 依赖隔离 | 独立（Fat JAR 包含所有依赖） | 容器共享 lib |
| 热部署 | 支持（需额外配置） | 容器支持 |
| 启动速度 | 快（内嵌） | 慢（依赖外部容器） |

### 追问方向
- `JarLauncher` 和 `WarLauncher` 的区别？
- Spring Boot 2.3+ 的 Z JAR 格式是什么？和旧格式有什么区别？
- 如何在 IDE 中以普通方式运行 Spring Boot 应用（不用 java -jar）？

### 避坑提示
- 不要在 Windows 下使用 FAT32 文件系统存放 Spring Boot Fat JAR，因为 FAT32 不支持大于 4GB 的文件
- 如果 jar 内嵌的 Tomcat 版本与外部容器版本冲突（比如部署到外部 Tomcat），不要用 jar 方式，用 war 方式

---

## 19. Spring Boot 配置文件的加密方案（jasypt）

### 题目
Spring Boot 应用的配置文件中有敏感信息（如数据库密码、API Key），如何加密这些配置？常用的 jasypt 加密方案如何使用？

### 核心答案

**加密必要性：**

`application.yml` 中的数据库密码、第三方 API 密钥等敏感信息明文存储有安全风险，需要加密存储，运行时解密。

**jasypt-spring-boot 集成：**

jasypt（Java Simplified Encryption）提供对称加密功能，Spring Boot 2.x 推荐使用 `jasypt-spring-boot-starter`。

**使用步骤：**

**1. 添加依赖：**

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

**2. 配置加密密钥（生产环境禁止硬编码）：**

```yaml
jasypt:
  encryptor:
    password: ${JASYPT_PASSWORD}   # 从环境变量获取，不要写死在配置中
    algorithm: PBEWithMD5AndDES   # 加密算法
    iv-generator-classname: org.jasypt.iv.RandomIvGenerator
```

**3. 加密明文：**

方式一：使用命令行工具

```bash
# 下载 jasypt CLI 工具或使用 Maven 插件
java -cp jasypt-*.jar org.jasypt.intf.cli.JasyptPBEStringCLI \
    password=MySecretPassword \
    algorithm=PBEWithMD5AndDES \
    input="mypassword123"

# 输出：ENC(encrypted_output_here)
```

方式二：使用测试类或工具类加密

```java
@Autowired
private StringEncryptor encryptor;

public void encrypt() {
    String encrypted = encryptor.encrypt("mypassword123");
    System.out.println("加密结果: ENC(" + encrypted + ")");
}
```

**4. 在配置文件中使用密文：**

```yaml
spring:
  datasource:
    password: ENC(加密后的密文)

myapp:
  api-key: ENC(加密后的密文)
```

**5. 启动时注入密钥：**

```bash
# 方式1：环境变量
export JASYPT_PASSWORD=MySecretPassword
java -jar myapp.jar

# 方式2：命令行
java -Djasypt.encryptor.password=MySecretPassword -jar myapp.jar

# 方式3：运维工具（如 Vault、AWS Secrets Manager）
```

**jasypt-spring-boot 3.x 支持 Spring Boot 3.x：**

```xml
<dependency>
    <groupId>com.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>4.0.0</version>
</dependency>
```

**加密算法选择：**

| 算法 | 安全性 | 兼容性 |
|------|--------|--------|
| PBEWithMD5AndDES | 低（MD5+DES 已不安全） | 高 |
| PBEWithMD5AndTripleDES | 中 | 高 |
| PBEWithSHA1AndDESede | 中 | 高 |
| PBEWithSHA256AndAES-256 | 高（推荐） | Java 8+ |

**其他加密方案：**

1. **Spring Cloud Config + Vault**：结合 Spring Cloud Config 的配置中心 + HashiCorp Vault 密钥管理
2. **Jasypt + Spring Cloud Config**：将加密后的配置放在配置中心
3. **Spring Boot 2.4+ 原生支持**：`spring.config.import` + external encrypted source

### 追问方向
- jasypt 的 `PBEWithMD5AndDES` 为什么被认为不安全？
- 如何自定义 `StringEncryptor` 实现使用自己的加密算法？
- jasypt 在集群部署中如何统一管理密钥？

### 避坑提示
- **绝对不要把加密密钥写在 application.yml 中**，否则攻击者拿到 jar 包后可以用同样的密钥解密
- 使用 `-Djasypt.encryptor.password` 参数时，命令行历史会记录密码，建议使用环境变量
- 如果使用 CI/CD 部署，密钥应该通过 CI/CD 平台的 secrets 管理机制注入

---

## 20. Spring Boot + MyBatis 的集成，SqlSessionFactory 创建过程

### 题目
Spring Boot 如何集成 MyBatis？`SqlSessionFactory` 是如何被创建并注册到容器的？

### 核心答案

**集成方式：**

Spring Boot 通过 `mybatis-spring-boot-starter` 自动配置 MyBatis：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
```

**自动配置过程：**

**1. MyBatisAutoConfiguration 加载：**

```java
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@EnableConfigurationProperties(MybatisProperties.class)
public class MybatisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        // ... 设置其他属性
        return factory.getObject();
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

**2. SqlSessionFactory 创建流程：**

```
SqlSessionFactoryBean.getObject()
    ↓
afterPropertiesSet()
    ↓
buildSqlSessionFactory()
    ↓
1. 读取 MybatisProperties（从 application.yml 的 mybatis.* 配置）
    - mapperLocations（classpath*:mapper/**/*.xml）
    - typeAliasesPackage（别名包扫描）
    - configLocation（mybatis-config.xml）
    ↓
2. 设置 DataSource（自动从容器注入）
    ↓
3. 创建 Configuration 对象
    ↓
4. 配置事务管理器（如果启用了 @Transactional）
    ↓
5. 注册 TypeHandler（处理 Java Type → JDBC Type 转换）
    ↓
6. 解析 Mapper XML（如果有）
    ↓
7. 注册已扫描的 Mapper 接口
    ↓
return new DefaultSqlSessionFactory(config)
```

**关键配置属性（application.yml）：**

```yaml
mybatis:
  mapper-locations: classpath*:mapper/**/*.xml   # Mapper XML 路径
  type-aliases-package: com.example.entity       # 实体类别名包
  config-location: classpath:mybatis-config.xml # 配置文件
  configuration:
    map-underscore-to-camel-case: true           # 驼峰映射
    log-impl: SLF4J                              # 日志实现
```

**Mapper 接口扫描注册：**

方式一：`@MapperScan` 注解

```java
@SpringBootApplication
@MapperScan("com.example.mapper")
public class MyApplication { }
```

方式二：`@Mapper` 注解（逐个标注）

```java
@Mapper
public interface UserMapper {
    @Select("SELECT * FROM users WHERE id = #{id}")
    User findById(Long id);
}
```

**@MapperScan 原理：**

```java
@Retention(RetentionPolicy.RUNTIME)
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {
    String[] value() default {};
    String[] basePackages() default {};
    // ...
}
```

`MapperScannerRegistrar` 实现了 `ImportBeanDefinitionRegistrar`，在 `registerBeanDefinitions()` 中注册 `MapperScannerConfigurer`（一个 `BeanDefinitionRegistryPostProcessor`），在容器刷新时扫描指定包下所有 `@Mapper` 标注的接口并注册为 `MapperFactoryBean`。

### 追问方向
- `SqlSession` 和 `SqlSessionTemplate` 的关系？
- MyBatis 的 `TypeHandler` 是如何处理 Java 类型和 JDBC 类型转换的？
- 为什么 Mapper 接口不需要实现类？

### 避坑提示
- 如果 Mapper XML 和 Mapper 接口不在同一包下，需要正确配置 `mapper-locations`
- `SqlSessionFactory` 应该全局只有一个，Spring Boot 自动配置会保证这一点，不要手动创建多个

---

## 21. Spring Boot 2.x 新特性（响应式编程、WebFlux、Coroutine）

### 题目
Spring Boot 2.x 引入了哪些重要新特性？请重点描述响应式编程、WebFlux 以及 Kotlin Coroutine 支持。

### 核心答案

**Spring Boot 2.x 主要新特性：**

| 特性 | 描述 |
|------|------|
| Spring Boot 2.0 | Java 8+ baseline, Spring 5, WebFlux, JUnit 5 |
| Spring Boot 2.1 | Java 11+ 支持, 性能优化, 默认 HiKariCP |
| Spring Boot 2.2 | Java 13 支持, `@ConfigurationPropertiesScan`, JUnit 5 原生支持 |
| Spring Boot 2.3 | Spring 5.3, Cloud LoadBalancer, 容器镜像构建 |
| Spring Boot 2.4 | `spring.config.import`, Config Data API |
| Spring Boot 2.5 | GraalVM 支持, Docker/Buildpacks 支持 |
| Spring Boot 2.6 | PathPatternParser, 循环依赖检测 |
| Spring Boot 2.7 | Spring Security 自动配置, 自动配置导入文件 |
| Spring Boot 2.8+ | Java 17 baseline |

**响应式编程（Reactive Programming）：**

响应式编程是一种异步非阻塞的编程范式，基于**事件流**和**背压（backpressure）**机制。

核心概念：
- `Publisher<T>`：发布者，发出数据流
- `Subscriber<T>`：订阅者，消费数据流
- `Subscription`：订阅关系
- `Backpressure`：消费者告诉生产者自己的处理能力，防止生产者过快

**Spring WebFlux（响应式 Web 框架）：**

Spring 5 引入了 `Spring WebFlux`，提供完全异步非阻塞的 Web 框架，替代传统的基于 Servlet 的 Spring MVC：

```java
// WebFlux 路由方式
@Configuration
public class WebFluxConfig {
    @Bean
    public RouterFunction<ServerResponse> route(ArticleHandler handler) {
        return route(GET("/articles/{id}"), handler::getArticle)
                .and(route(GET("/articles"), handler::listArticles))
                .and(route(POST("/articles"), handler::createArticle));
    }
}

@Component
public class ArticleHandler {
    public Mono<ServerResponse> getArticle(ServerRequest request) {
        String id = request.pathVariable("id");
        return articleRepository.findById(id)
                .flatMap(article -> ServerResponse.ok().bodyValue(article))
                .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> listArticles(ServerRequest request) {
        return articleRepository.findAll()
                .collectList()
                .flatMap(articles -> ServerResponse.ok().bodyValue(articles));
    }
}
```

**WebFlux vs Spring MVC：**

| 维度 | Spring MVC（阻塞） | WebFlux（非阻塞） |
|------|-------------------|-------------------|
| 线程模型 | 一请求一线程 | 少量线程处理大量请求（事件循环） |
| 并发模型 | 同步阻塞 | 异步非阻塞 |
| 适用场景 | CRUD、CPU 密集型 | 高并发 IO、微服务间通信 |
| 事务 | 传统声明式事务 | 响应式事务（R2DBC） |
| 调试 | 同步，可直接打断点 | 异步，调试复杂 |

**Kotlin Coroutine 支持：**

Spring Boot 2.2+ 支持 Kotlin Coroutine，提供更简洁的异步代码写法：

```kotlin
@GetMapping("/articles/{id}")
suspend fun getArticle(@PathVariable id: Long): Article {
    return articleRepository.findById(id)
}

@GetMapping("/articles")
fun getArticles(): Flow<Article> {
    return articleRepository.findAll()
}
```

Spring WebFlux 支持 `CoroutineSubscriber` 将 Coroutine 的 `Flow` 转换为响应式流。

**响应式数据访问（R2DBC）：**

Spring Boot 2.3+ 支持 R2DBC（Reactive Relational Database Connectivity），实现完全异步的数据库访问：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
```

### 追问方向
- 什么场景下应该选择 WebFlux 而不是 Spring MVC？
- WebFlux 的默认容器是什么？Undertow 还是 Netty？
- Kotlin Coroutine 和 Project Reactor 的 Mono/Flux 如何选择？

### 避坑提示
- WebFlux 不能直接使用 Spring MVC 的同步组件（如 `JdbcTemplate`），需要使用 R2DBC 或 WebClient
- 如果团队不熟悉响应式编程，不要为了赶时髦引入 WebFlux，增加了复杂度但收益不明显

---

## 22. Spring Boot + Redis 集成，Lettuce vs Jedis 区别

### 题目
Spring Boot 如何集成 Redis？Lettuce 和 Jedis 两个客户端库有什么区别？

### 核心答案

**集成方式：**

Spring Boot 通过 `spring-boot-starter-data-redis` 集成 Redis，默认使用 **Lettuce** 作为客户端：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**自动配置：**

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public RedisConnectionFactory redisConnectionFactory() {
        // 根据配置创建 LettuceConnectionFactory 或 JedisConnectionFactory
        return new LettuceConnectionFactory(...);
    }

    @Bean
    @ConditionalOnMissingBean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // 配置序列化器
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}
```

**application.yml 配置：**

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: ${REDIS_PASSWORD}
    database: 0
    timeout: 3000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 2
        max-wait: 1000ms
```

**Lettuce vs Jedis 对比：**

| 特性 | Lettuce | Jedis |
|------|---------|-------|
| 线程安全性 | 线程安全（连接池共享） | 早期版本线程不安全，需要连接池 |
| 连接协议 | 支持 RESP v2（Redis 6+） | RESP v2（部分） |
| 异步支持 | 原生支持异步、响应式（WebClient 集成） | 异步支持较弱 |
| 集群支持 | 原生支持 Redis Cluster、Sentinel | 支持 |
| 连接池 | Commons Pool 2（基于 Netty） | Commons Pool 2 |
| 性能 | 高（Netty 异步IO） | 较好 |
| 维护状态 | 活跃（Spring Data Redis 默认） | 活跃 |
| Redis 6+ ACL | 支持 | 支持 |

**Lettuce 的优势：**

```java
// Lettuce 支持异步
RedisAsyncCommands<String, String> async = connection.async();
async.set("key", "value").thenAccept(result -> {
    System.out.println("异步设置成功");
});

// Lettuce 支持响应式
RedisReactiveCommands<String, String> reactive = connection.reactive();
reactive.get("key").subscribe(value -> {
    System.out.println("响应式获取: " + value);
});
```

**切换到 Jedis：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

**RedisTemplate 的序列化配置：**

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // 使用 JSON 序列化
        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(Object.class);

        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(serializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

### 追问方向
- Lettuce 连接池是如何工作的？为什么它是线程安全的？
- Redis 集群模式下，Lettuce 如何路由请求？
- 如何在 Spring Boot 中使用 Redis 的 Pub/Sub 功能？

### 避坑提示
- Redis 6+ 支持多线程 IO（IO threads），Lettuce 可以利用此特性，但需要确认 Jedis 版本
- 不要在生产环境使用 `spring.redis.host=localhost`，应该使用集群或哨兵模式确保高可用

---

## 23. Spring Boot 中的 ImportBeanDefinitionRegistrar 和 @ImportSelector

### 题目
Spring Boot 自动配置大量使用了 `ImportBeanDefinitionRegistrar` 和 `ImportSelector`，请描述它们的作用、区别和使用场景。

### 核心答案

**共同点：**

两者都通过 `@Import` 导入到 Spring 容器，都是 `DeferredImportSelector`（Spring Boot 自动配置的核心机制）的基石，用于在应用启动时**动态注册 Bean 定义**。

**@ImportSelector：**

`ImportSelector` 负责**根据条件返回需要导入的类名数组**，Spring 容器会根据这些类名注册 Bean：

```java
public interface ImportSelector {
    String[] selectImports(AnnotationMetadata importingClassMetadata);
    // 返回需要注册的 Bean 的全限定类名
}
```

**典型应用：自动配置类选择**

```java
public class MyAutoConfigurationImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 可以根据注解元数据判断是否需要导入某些配置
        Map<String, Object> attrs = importingClassMetadata.getAnnotationAttributes("com.example.EnableMyFeature");
        boolean enabled = (boolean) attrs.get("enabled");
        if (enabled) {
            return new String[] {
                "com.example.config.FeatureAConfig",
                "com.example.config.FeatureBConfig"
            };
        }
        return new String[0];
    }
}
```

Spring Boot 的 `AutoConfigurationImportSelector` 继承自 `DeferredImportSelector`，在 `selectImports()` 中读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 中的配置类。

**ImportBeanDefinitionRegistrar：**

`ImportBeanDefinitionRegistrar` 直接**操作 BeanDefinitionRegistry**，可以手动注册 Bean 定义，比 `ImportSelector` 更底层和灵活：

```java
public interface ImportBeanDefinitionRegistrar {
    void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                 BeanDefinitionRegistry registry);
    // 可以用 registry.registerBeanDefinition() 手动注册
}
```

**典型应用：Mapper 接口扫描注册**

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata meta,
                                        BeanDefinitionRegistry registry) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(MapperScannerConfigurer.class);
        builder.addPropertyValue("basePackage", "com.example.mapper");
        AbstractBeanDefinition definition = builder.getBeanDefinition();
        registry.registerBeanDefinition("mapperScannerConfigurer", definition);
    }
}
```

`MapperScannerConfigurer` 是一个 `BeanDefinitionRegistryPostProcessor`，会在容器刷新时扫描指定包下的 `@Mapper` 接口并注册为 `MapperFactoryBean`。

**DeferredImportSelector（Spring Boot 自动配置的核心）：**

`DeferredImportSelector` 是 `ImportSelector` 的子接口，特点是在所有 `@Configuration` 类处理完毕后**最后执行**，且可以指定分组（`Group`）：

```java
// AutoConfigurationImportSelector 继承链
AutoConfigurationImportSelector
    → DeferredImportSelector
    → ImportSelector
```

执行顺序：`ImportSelector` → 普通 `@Configuration` → `DeferredImportSelector`

**@EnableAutoConfiguration 的实现：**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

`AutoConfigurationImportSelector.selectImports()` 返回所有自动配置类的类名，Spring Boot 根据 `@Conditional` 注解进行过滤后注册。

**对比：**

| 特性 | ImportSelector | ImportBeanDefinitionRegistrar |
|------|---------------|-------------------------------|
| 返回值 | String[]（类名） | void（直接操作 registry） |
| 控制方式 | 间接（返回要注册的类） | 直接（手动注册 BeanDefinition） |
| 灵活性 | 只能返回类名，由容器实例化 | 可完全控制 BeanDefinition 属性 |
| 使用场景 | 自动配置、动态导入 | MyBatis @MapperScan、扩展性注册 |
| 执行时机 | 与 @Configuration 同时处理 | 与 @Configuration 同时处理 |

### 追问方向
- `DeferredImportSelector.Group` 接口的作用是什么？
- `ImportBeanDefinitionRegistrar` 如何获取 `@Import` 注解上传递的参数？
- 为什么 Spring Boot 选择 `DeferredImportSelector` 而非直接 `ImportSelector`？

### 避坑提示
- `ImportSelector` 只能返回类名字符串，不能注册 `BeanDefinitionRegistryPostProcessor`，如果需要更细粒度控制，用 `ImportBeanDefinitionRegistrar`
- 在 `registerBeanDefinitions` 中不要调用 `BeanFactory.getBean()`，此时 Bean 还未实例化

---

## 24. SpringApplicationBuilder 如何链式调用，parent/child 容器

### 题目
`SpringApplicationBuilder` 是如何实现链式调用的？Spring Boot 中 parent 和 child 容器的概念是什么？何时需要多容器架构？

### 核心答案

**SpringApplicationBuilder 链式调用原理：**

`SpringApplicationBuilder` 实现了**流式 API（Builder Pattern）**，每个方法返回 `this`（实际上是 `SpringApplicationBuilder` 实例）：

```java
// 链式调用示例
new SpringApplicationBuilder(Application.class)
    .bannerMode(Banner.Mode.OFF)
    .profiles("dev")
    .sources(Application.class)
    .run(args);
```

源码结构：

```java
public class SpringApplicationBuilder {
    private final SpringApplicationBuilder parent;
    private final List<SpringApplication> children = new ArrayList<>();

    // 每个方法返回 this，实现链式
    public SpringApplicationBuilder sources(Class<?>... sources) {
        this.sources.addAll(Arrays.asList(sources));
        return this;
    }

    public SpringApplicationBuilder bannerMode(Banner.Mode bannerMode) {
        this.bannerMode = bannerMode;
        return this;
    }

    public SpringApplicationBuilder profiles(String... profiles) {
        this.profiles.addAll(Arrays.asList(profiles));
        return this;
    }
}
```

**parent/child 容器架构：**

Spring Boot 支持创建**分层 ApplicationContext**：

```
Root ApplicationContext（父容器）
    ├── shared beans（共享 Bean，如数据库连接池）
    │
    └── Servlet Web ApplicationContext（子容器）
            ├── Controllers
            ├── ViewResolver
            └── Web 相关的 Bean
```

**创建父子容器：**

```java
// SpringApplicationBuilder 方式
new SpringApplicationBuilder()
    .sources(ParentConfig.class)           // 父容器配置
    .child(ChildWebConfig.class)           // 子容器配置
    .bannerMode(Banner.Mode.OFF)
    .run(args);
```

等价于 Spring Framework 原生方式：

```java
// 创建父容器
AnnotationConfigApplicationContext parentContext =
    new AnnotationConfigApplicationContext();
parentContext.scan("com.example.parent");
parentContext.refresh();

// 创建子容器，指定父容器
AnnotationConfigServletWebServerApplicationContext childContext =
    new AnnotationConfigServletWebServerApplicationContext(parentContext);
childContext.scan("com.example.child");
childContext.refresh();
```

**Bean 查找顺序：**

子容器可以**看见**父容器的 Bean（向上查找），但父容器**看不见**子容器的 Bean（向下隔离）：

```java
@Component
public class ChildBean {
    @Autowired
    private ParentService parentService;  // ✓ 可以注入父容器 Bean

    @Autowired
    private ChildService childService;     // ✓ 可以注入同容器 Bean
}

@Component
public class ParentBean {
    @Autowired
    private ParentService parentService;   // ✓ 可以注入同容器 Bean

    // @Autowired
    // private ChildService childService;    // ✗ 父容器看不到子容器 Bean
}
```

**典型使用场景：**

1. **模块化拆分**：不同子模块有独立的上下文，但共享某些基础 Bean
2. **传统 Web 应用**：Spring MVC 分层（Root WebApplicationContext + Servlet WebApplicationContext）
3. **微服务测试**：在同一 JVM 中启动多个 Spring Boot 应用进行集成测试

**创建多个独立容器（不是父子）：**

```java
// 两个完全独立的容器（不是父子关系）
SpringApplication app1 = new SpringApplication(App1.class);
ConfigurableApplicationContext ctx1 = app1.run(args);

SpringApplication app2 = new SpringApplication(App2.class);
ConfigurableApplicationContext ctx2 = app2.run(args);
```

### 追问方向
- Spring MVC 的父子容器是如何实现的？和 Spring Boot 中的父子容器有什么区别？
- 父子容器中 Bean 的懒加载行为有什么特殊性？
- 什么情况下需要用 `ApplicationContextFactory` 自定义创建方式？

### 避坑提示
- 父子容器中如果有同名的 Bean，会发生什么？（子容器优先，会覆盖父容器）
- 在大多数单体 Spring Boot 应用中，不需要使用父子容器架构，直接单容器即可

---

## 25. Spring Boot 自定义 Starter 的步骤和要点

### 题目
如何编写一个 Spring Boot 自定义 Starter？请描述完整的步骤和关键要点。

### 核心答案

**自定义 Starter 命名规范：**

```
spring-boot-starter-xxx       → 官方维护
xxx-spring-boot-starter      → 社区/第三方维护（约定）
```

**完整步骤：**

**第一步：创建项目结构**

```
my-starter/
├── pom.xml
└── src/main/java/com/example/autoconfigure/
│   ├── MyAutoConfiguration.java     ← 自动配置类
│   └── MyProperties.java            ← 配置属性类
└── src/main/resources/META-INF/
    └── spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

**第二步：pom.xml 配置**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-starter</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.2.0</spring-boot.version>
    </properties>

    <dependencies>
        <!-- 依赖 Spring Boot 基础设施 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>${spring-boot.version}</version>
            <scope>provided</scope>
        </dependency>
        <!-- 可选：starter 本身不需要这个 -->
        <!-- <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <version>${spring-boot.version}</version>
        </dependency> -->
    </dependencies>
</project>
```

**第三步：配置属性类**

```java
@ConfigurationProperties(prefix = "my.config")
public class MyProperties {
    private String host = "localhost";
    private int port = 8080;
    private boolean enabled = true;

    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
}
```

**第四步：自动配置类**

```java
@Configuration
@ConditionalOnClass(MyService.class)              // classpath 中有 MyService 才加载
@ConditionalOnProperty(                          // 配置中 enabled=true 才生效
    prefix = "my.config",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = false                       // 如果没有配置，默认不生效
)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Autowired
    private MyProperties properties;

    @Bean
    @ConditionalOnMissingBean                    // 如果用户已自定义，则不注册
    public MyService myService() {
        return new MyService(properties.getHost(), properties.getPort());
    }
}
```

**第五步：注册自动配置类**

Spring Boot 3.x 方式（`AutoConfiguration.imports`）：

```txt
# src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfigure.MyAutoConfiguration
```

Spring Boot 2.x 方式（`spring.factories`）：

```properties
# src/main/resources/META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfigure.MyAutoConfiguration
```

**第六步：可选的 meta-data 生成**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <version>${spring-boot.version}</version>
    <optional>true</optional>
</dependency>
```

**关键要点总结：**

| 要点 | 说明 |
|------|------|
| `@ConditionalOnClass` | 避免 classpath 不存在时加载失败 |
| `@ConditionalOnMissingBean` | 允许用户覆盖默认配置 |
| `@ConditionalOnProperty` | 支持用户通过配置开关控制是否启用 |
| `@EnableConfigurationProperties` | 将 `@ConfigurationProperties` 类注册为 Bean |
| 配置属性类 | 使用 `@ConfigurationProperties` 提供用户可配置属性 |
| 正确命名 | 遵循 `xxx-spring-boot-starter` 命名约定 |
| 自动配置注册 | Spring Boot 3.x 用 `AutoConfiguration.imports`，2.x 用 `spring.factories` |
| 避免版本依赖 | starter 不要引入具体实现库的版本，由使用者管理 |

**使用者引入：**

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

```yaml
my:
  config:
    enabled: true
    host: 127.0.0.1
    port: 9090
```

### 追问方向
- `@AutoConfigureOrder` 和 `@AutoConfigureAfter` 在自定义 Starter 中的作用？
- 如何在自定义 Starter 中使用 `@ConditionalOnBean` 检测其他 starter 提供的 Bean？
- Spring Boot 3.x 的 `AutoConfiguration.imports` 和 `spring.factories` 混用会怎样？

### 避坑提示
- **不要在 starter 中引入具体依赖的版本号**，由 parent 或使用者管理，避免版本冲突
- **一定要加 `@ConditionalOnMissingBean`**，否则用户无法覆盖自动配置的 Bean
- **一定不要用 `@Primary` 标记自动配置的 Bean**，这会覆盖用户的自定义 Bean

---

## 26. Spring Boot 3.x 与 2.x 的主要区别

### 题目
Spring Boot 3.x 相对于 2.x 有哪些重大变化？升级时需要注意什么？

### 核心答案

**Spring Boot 3.x 主要变化：**

| 变化项 | Spring Boot 2.x | Spring Boot 3.x |
|--------|-----------------|-----------------|
| Java 版本 | Java 8~17 | Java 17~21（最低 17） |
| Spring Framework | 5.0~5.3 | 6.0~6.2 |
| Jakarta EE | JavaEE 8 (`javax.*`) | Jakarta EE 9+ (`jakarta.*`) |
| 嵌入式容器 | Tomcat 9.x | Tomcat 10.x（需要 Servlet 5.0） |
| 自动配置注册 | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 配置文件导入 | `spring.config.location` | `spring.config.import` |
| `spring.factories` | 支持 | 废弃 |
| 注解 | `javax.annotation.*` | `jakarta.annotation.*` | |
| 加密 | Jasypt 3.x | Jasypt 4.x |

**Java 版本要求：**

Spring Boot 3.x 要求 **JDK 17 最低**，推荐 JDK 21。不再支持 Java 8、11、15、16。

**Jakarta EE 迁移（最大破坏性变更）：**

所有 `javax.*` 包迁移到 `jakarta.*`：

```java
// Spring Boot 2.x
import javax.servlet.Filter;
import javax.persistence.Entity;
import javax.annotation.PostConstruct;

// Spring Boot 3.x
import jakarta.servlet.Filter;
import jakarta.persistence.Entity;
import jakarta.annotation.PostConstruct;
```

这意味着 Spring Boot 3.x 的应用**无法直接使用** Spring Boot 2.x 的依赖库（除非库已提供 Jakarta EE 9+ 版本）。

**自动配置注册文件变化：**

```
# Spring Boot 2.x: META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration

# Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyAutoConfiguration
```

**spring.config.import 改进：**

```yaml
# Spring Boot 2.x
spring.config.location=file:./config/

# Spring Boot 3.x
spring:
  config:
    import: optional:file:./config/application.yml
```

**新特性：**

1. **AOT 编译支持**：提前编译（Ahead-of-Time），结合 GraalVM 生成原生镜像，启动时间从秒级降到毫秒级
2. **Observability**：原生支持 Micrometer Tracing（OpenTelemetry 集成）
3. **Spring Security 自动配置改进**：`SecurityFilterChain` Bean 的自动检测
4. **Record 类支持**：`@ConfigurationProperties` 支持 Java Record 作为属性类
5. **`@ConfigurationPropertiesScan`**：替代 `@EnableConfigurationProperties`

```java
// Spring Boot 3.x 可以用 Record 作为配置类
@ConfigurationProperties(prefix = "myapp")
public record MyProperties(String name, int port) {}
```

**升级注意事项：**

1. **依赖库兼容性**：检查所有第三方依赖是否已支持 Jakarta EE
2. **代码迁移**：`javax.*` → `jakarta.*`
3. **配置文件**：`spring.factories` → `AutoConfiguration.imports`
4. **Java 版本**：JDK 17+
5. **测试框架**：JUnit 5 原生支持，不需要 vintage engine
6. **Spring Cloud 兼容性**：Spring Cloud 2022.0+ 才支持 Spring Boot 3.x

### 追问方向
- Spring Boot 3.x 如何使用 GraalVM 构建原生镜像？
- `spring.factories` 废弃后，Spring Boot 2.x 升级到 3.x 时如何迁移？
- Spring Boot 3.x 对 WebSocket、Validation API 等的影响？

### 避坑提示
- **不要在 Spring Boot 3.x 项目中引入任何 `javax.*` 依赖**，会导致 `ClassNotFoundException`
- 升级前先确认所有依赖的 3.x 兼容版本，否则编译都可能失败
- Spring Boot 3.x 的 `spring-boot-starter-parent` 3.x 无法 parent 2.x 的项目

---

## 27. @EnableAutoConfiguration 注解的工作机制

### 题目
`@EnableAutoConfiguration` 是开启自动配置的入口注解，请详细描述它的工作机制。

### 核心答案

**@EnableAutoConfiguration 注解定义：**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

核心机制是 **`@Import(AutoConfigurationImportSelector.class)`**。

**完整工作流程：**

```
1. 标注 @EnableAutoConfiguration（在 @SpringBootApplication 中间接标注）
        ↓
2. Spring 容器启动，处理 @Configuration 类时发现 @Import(AutoConfigurationImportSelector)
        ↓
3. AutoConfigurationImportSelector 实现 ImportSelector 接口
   + DeferredImportSelector 接口（在所有 @Configuration 处理完毕后执行）
        ↓
4. AutoConfigurationImportSelector.selectImports() 被调用
        ↓
5. SpringFactoriesLoader.loadFactoryNames(
       EnableAutoConfiguration.class,
       classLoader
   )
   → 读取 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   → 返回所有自动配置类全限定名（如 120+ 个）
        ↓
6. 返回的自动配置类数组被注册为 Bean 定义
        ↓
7. BeanFactoryPostProcessor（ConfigurationClassPostProcessor）处理这些配置类
        ↓
8. 对每个自动配置类，按 @Conditional* 注解进行条件匹配
        ↓
9. 匹配成功的配置类被实例化并注册为 Bean
```

**AutoConfigurationImportSelector 源码关键逻辑：**

```java
// Spring Boot 3.x 实现
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // 从 spring.factories 或 AutoConfiguration.imports 加载
    List<String> configurations = SpringFactoriesLoader
        .loadFactoryNames(getAnnotationClass(), getClass().getClassLoader());
    // ...过滤、去重
    return configurations.toArray(new String[0]);
}

private Class<?> getAnnotationClass() {
    return EnableAutoConfiguration.class;
}
```

**Spring Boot 3.x 读取的文件：**

路径：`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

内容示例（Spring Boot 3.2 自动配置列表的片段）：

```
# 内容为纯文本，每行一个类名
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
...
```

**条件匹配过滤：**

加载到 120+ 自动配置类后，通过 `@ConditionalOnClass`、`@ConditionalOnMissingBean` 等注解逐个过滤：

```java
// 例如 RedisAutoConfiguration 的条件
@Configuration
@ConditionalOnClass(RedisOperations.class)      // classpath 有 RedisOperations
@@EnableConfigurationProperties(RedisProperties.class)
@AutoConfigureAfter(RedisAutoConfiguration.class) // 在某配置之后
public class RedisAutoConfiguration {
    // ...
}
```

**exclude/excludeName 属性：**

```java
@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class})
@EnableAutoConfiguration(excludeName = "com.example.MyAutoConfiguration")
```

等价于在 `@SpringBootApplication` 中：

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

**排除自动配置的多种方式：**

```yaml
# application.yml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

```properties
# 或 application.properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 追问方向
- `@EnableAutoConfiguration` 的 `exclude` 和 `spring.autoconfigure.exclude` 哪个优先级更高？
- `AutoConfigurationImportSelector` 的 `Deferred` 特性在什么时候起作用？
- 如何自定义 `ImportSelector` 实现完全自己控制的自动配置？

### 避坑提示
- 不要在启动类上用 `@ComponentScan` 和 `@EnableAutoConfiguration` 叠加，容易导致组件扫描重复
- 排除自动配置时，`exclude` 属性中放的是**类名**，不是全限定类名的字符串形式

---

## 28. Spring Boot 如何判断是否 Web 环境（WebApplicationType）

### 题目
Spring Boot 如何自动判断当前应用是 Web 环境（SERVLET）、响应式环境（REACTIVE）还是非 Web 环境（NONE）？`WebApplicationType` 的判断逻辑是什么？

### 核心答案

**WebApplicationType 枚举：**

```java
public enum WebApplicationType {
    NONE,      // 非 Web 应用
    SERVLET,   // 基于 Servlet 的 Web 应用（Tomcat/Jetty/Undertow）
    REACTIVE   // 响应式 Web 应用（WebFlux/Netty）
}
```

**判断逻辑：**

`SpringApplication` 构造函数中通过 `WebApplicationType.deduceFromClasspath()` 自动判断：

```java
// SpringApplication 构造方法中
this.webApplicationType = WebApplicationType.deduceFromClasspath();

static WebApplicationType deduceFromClasspath() {
    // 1. 如果存在 Servlet 类且不存在 WebFlux 相关的类 → SERVLET
    if (ClassUtils.isPresent("jakarta.servlet.Servlet", null)
            && !ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", null)) {
        return WebApplicationType.SERVLET;
    }
    // 2. 如果存在 WebFlux 的 DispatcherHandler → REACTIVE
    if (!ClassUtils.isPresent("jakarta.servlet.Servlet", null)
            && ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", null)) {
        return WebApplicationType.REACTIVE;
    }
    // 3. 其他情况 → NONE
    return WebApplicationType.NONE;
}
```

**判断逻辑详解：**

| 条件 | 结果 |
|------|------|
| 有 `jakarta.servlet.Servlet` + 无 `org.springframework.web.reactive.DispatcherHandler` | `SERVLET` |
| 无 `jakarta.servlet.Servlet` + 有 `org.springframework.web.reactive.DispatcherHandler` | `REACTIVE` |
| 两者都没有 | `NONE` |
| 两者都有 | `SERVLET`（优先 Servlet） |

**优先级原因**：Spring Boot 认为 Servlet 环境比响应式更主流。

**手动指定 WebApplicationType：**

```java
// 方式1：代码指定
new SpringApplicationBuilder()
    .web(WebApplicationType.SERVLET)  // 强制指定
    .sources(Application.class)
    .run(args);

// 方式2：配置文件（Spring Boot 2.x）
# application.properties
spring.main.web-application-type=reactive
```

**不同 WebApplicationType 创建的 ApplicationContext：**

| WebApplicationType | ApplicationContext 类型 |
|--------------------|------------------------|
| `SERVLET` | `AnnotationConfigServletWebServerApplicationContext` |
| `REACTIVE` | `AnnotationConfigReactiveWebServerApplicationContext` |
| `NONE` | `AnnotationConfigApplicationContext` |

**手动排除 Servlet 后变成 NONE：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

排除后 classpath 中没有 `jakarta.servlet.Servlet`，且没有 `DispatcherHandler`，应用就变成 `NONE`（非 Web 应用）。

### 追问方向
- 如果 classpath 中同时有 Servlet 和 WebFlux，Spring Boot 会选择哪个？
- WebFlux 的 `AnnotationConfigReactiveWebServerApplicationContext` 和 Servlet 的有什么本质区别？
- 如何强制让 Spring Boot 认为一个没有 Web 依赖的应用是 Web 应用？

### 避坑提示
- 手动指定 `web(WebApplicationType.SERVLET)` 后，即使排除了 Tomcat，也可能启动失败（因为没有 Servlet 容器）
- 如果启动的是 REACTIVE 应用，却引入了 Spring MVC 的 `@Controller`，启动会失败，因为 REACTIVE 容器不支持 Servlet

---

## 29. Spring Boot 的反射机制（introspection）

### 题目
Spring Boot 大量使用反射机制来发现和管理 Bean，请描述 Spring Boot 中的反射机制（introspection）是如何工作的，以及 `IntrospectionFailureReason` 和 `BeanInfo` 的作用。

### 核心答案

**Spring 中的 Introspection（内省）：**

Java Bean Introspection 是 Java SE 提供的机制，用于在运行时**发现 Bean 的属性（PropertyDescriptor）和方法（Method）**。

Spring Framework 在此基础上封装了 `org.springframework.beans` 包，提供：
- `PropertyDescriptor`：描述 Bean 的属性（getter/setter）
- `BeanWrapper`：包装 Bean，提供统一的对象属性访问 API
- `BeanInfo`：描述一个类的元信息（属性、方法、事件）

**Spring Boot 中 Introspection 的使用场景：**

**1. @ConfigurationProperties 绑定：**

```java
@Component
@ConfigurationProperties(prefix = "myapp")
public class MyProperties {
    private String name;
    private int port;
    // getters/setters
}
```

Spring Boot 的 `Binder` 类通过 `PropertyDescriptor` 读取 `name`、`port` 的 getter/setter，建立起 YAML 中的 `myapp.name` 与 `MyProperties.name` 的绑定关系。

**2. BeanWrapper 使用：**

```java
public static void main(String[] args) {
    MyProperties props = new MyProperties();
    BeanWrapper bw = new BeanWrapperImpl(props);
    bw.setPropertyValue("name", "MyApp");
    bw.setPropertyValue("port", 8080);

    // 也可以读取
    String name = (String) bw.getPropertyValue("name");
}
```

**3. Spring Boot 2.0 引入的 JSR-305 @ConstructorBinding：**

```java
@Component
@ConfigurationProperties(prefix = "myapp")
@ConstructorBinding  // 从构造函数绑定属性
public class MyProperties {
    private final String name;
    private final int port;

    public MyProperties(String name, int port) {
        this.name = name;
        this.port = port;
    }
}
```

**Spring Boot 的内省优化（IntrospectionCache）：**

Spring Framework 维护了一个 `ConcurrentHashMap` 缓存 `BeanInfo`：

```java
// CachedIntrospectionResults 内部
private static final Map<Class<?>, CachedIntrospectionResults> cache =
    new ConcurrentHashMap<>(64);

public static CachedIntrospectionResults forClass(Class<?> beanClass) {
    return cache.computeIfAbsent(beanClass,
        CachedIntrospectionResults::new);
}
```

**CachedIntrospectionResults** 缓存了 `PropertyDescriptor[]`、`MethodDescriptor[]` 等内省结果，避免重复内省同一个类。

**Spring Boot 对 Kotlin 的 Introspection：**

Kotlin 的数据类没有 Java 风格的 getter/setter，需要特殊处理：

```kotlin
// Kotlin 数据类
data class MyProperties(val name: String, val port: Int)
```

Spring Boot 2.x+ 通过 `KotlinIntrospector` 提供对 Kotlin 类的内省支持，将 Kotlin 属性映射为 Bean 属性。

**IntrospectionFailureReason（Spring 5 新增）：**

当内省失败时，Spring 会给出详细的原因枚举：

```java
public enum IntrospectionFailureReason {
    NOT_AN_INTROSPECTION_TARGET_CLASS,     // 不是内省目标类（如 Object.class）
    NO_BEAN_CLASS,                          // Bean 类为空
    NO_BEAN_PROPERTY_DESCRIPTORS,           // 没有 Bean 属性描述符
    ACCESS_TYPE_DENIED,                     // 访问被拒绝（如 private 字段）
    INVALID_GETTER_METHOD,                  // Getter 方法签名无效
    INVALID_SETTER_METHOD_PARAMETER_TYPE, // Setter 方法参数类型不匹配
    // ...
}
```

这些信息对配置绑定失败的诊断非常有帮助，Spring Boot 会在配置绑定失败时输出 `IntrospectionFailureReason`。

**典型使用：BeanInfo 自定义（JavaBeans 规范）：**

如果一个类提供了自定义 `BeanInfo`，Spring 会优先使用它：

```java
public class MyPropertiesBeanInfo extends SimpleBeanInfo {
    @Override
    public PropertyDescriptor[] getPropertyDescriptors() {
        try {
            PropertyDescriptor name = new PropertyDescriptor("name", MyProperties.class);
            name.setDescription("应用名称");
            return new PropertyDescriptor[] { name };
        } catch (IntrospectionException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 追问方向
- `@ConstructorBinding` 和传统的 setter 绑定有什么区别？什么场景下必须用构造器绑定？
- Spring 的 `BeanWrapperImpl` 和 `DirectFieldAccessor` 有什么区别？
- Spring Boot 如何处理 Kotlin data class 的属性内省？

### 避坑提示
- 避免在启动时频繁通过反射创建大量 Bean，会触发大量内省操作，可以考虑用 `cached` 模式的 BeanInfo
- Kotlin 的 data class 使用 `@ConfigurationProperties` 时，不需要加 `@Component`，用 `@ConfigurationPropertiesScan` 统一注册

---

## 30. Spring Boot 配置类为什么要加 @Configuration 和 @Configuration 区别

### 题目
Spring Boot 配置类中标注 `@Configuration` 和标注 `@Component` 有什么区别？配置类一定要加 `@Configuration` 吗？

### 核心答案

**@Configuration 和 @Component 的本质区别：**

```java
@Component
public class MyConfig1 {
    @Bean
    public MyService myService() {
        return new MyService();
    }
}

@Configuration
public class MyConfig2 {
    @Bean
    public MyService myService() {
        return new MyService();
    }
}
```

两者都会被 Spring 扫描并注册为 Bean，都能定义 `@Bean` 方法。但有一个**关键区别**：`@Configuration` 默认开启了 **`proxyBeanMethods = true`**。

**proxyBeanMethods 的作用：**

```java
@Configuration  // proxyBeanMethods = true（默认）
public class MyConfig {
    @Bean
    public MyService myService() {
        System.out.println("创建 MyService");
        return new MyService();
    }
}

@Component
public class MyComponent {
    @Autowired
    private MyConfig config;

    @Autowired
    private MyService service1;

    @Autowired
    private MyService service2;
}
```

当 `@Configuration` 的 `proxyBeanMethods = true` 时，Spring Boot 会为配置类生成一个 **CGLIB 代理**，拦截所有 `@Bean` 方法调用，确保**每次调用返回的是容器中注册的同一个单例 Bean**：

```
service1 == service2  → true（同一个对象）
myService() 只打印一次 "创建 MyService"
```

而 `@Component`（或 `@Configuration(proxyBeanMethods = false)`）不会代理，每次调用 `@Bean` 方法都执行方法体，创建**新对象**：

```
service1 != service2  → true（不同对象）
myService() 打印两次 "创建 MyService"
```

**源码层面解释：**

`@Configuration` 标注的类，Spring 在 `ConfigurationClassPostProcessor` 处理时会生成 CGLIB 子类：

```java
// 伪代码
@Configuration
↓ 生成代理类
public class ConfigurationClass$$CGLIB$$ extends MyConfig {
    private MyService myService;  // 缓存

    @Override
    public MyService myService() {
        if (this.myService == null) {
            this.myService = super.myService();  // 调用原始方法
        }
        return this.myService;  // 返回缓存的单例
    }
}
```

**使用场景：**

| 场景 | 推荐 | 原因 |
|------|------|------|
| Bean 之间有依赖 | `@Configuration` | 保证同一实例 |
| @Bean 方法调用另一个 @Bean | `@Configuration` | 必须用代理，否则每次调都 new |
| 配置类不需要代理 | `@Configuration(proxyBeanMethods = false)` | 禁用代理提升性能 |
| 纯工具类，不需要 Spring 容器管理 Bean | `@Component` | 只是组件扫描 |

**@Component 的典型使用场景：**

```java
@Component
public class StringToLocalDateConverter implements Converter<String, LocalDate> {
    @Override
    public LocalDate convert(String source) {
        return LocalDate.parse(source);
    }
}
```

这个类不是"配置类"，只是需要一个 Spring 管理的单例实例，使用 `@Component`。

**Spring Boot 3.x 的变化：**

Spring Boot 3.x 中，`@Configuration` 的默认行为不变，但引入了 `@ConfigurationProperties` 支持 Record 类型，且 `@Configuration` 和 `@Component` 在 Bean 扫描层面的行为完全一致。

**最佳实践：**

- **定义配置 Bean 统一用 `@Configuration`**，因为它明确表达了"这是一个配置类"的语义
- **如果 @Bean 方法之间没有依赖**，且追求性能（避免代理开销），可以使用 `@Configuration(proxyBeanMethods = false)`
- **不要混用 `@Component` + 大量 `@Bean`**，因为 `@Bean` 方法之间的调用会失去代理保护

### 追问方向
- `proxyBeanMethods = false` 时，如果 @Bean A 依赖 @Bean B，会发生什么？
- `@Import(MyConfig.class)` 导入的配置类需要加 `@Configuration` 吗？
- `@Configuration` 的代理是如何绕过 `final` 方法限制的？

### 避坑提示
- `@Bean` 方法参数注入（如 `@Bean public MyService service(DataSource ds)`）不需要依赖代理机制，因为参数本身由容器注入
- 如果在 `@Configuration` 类中调用另一个 `@Bean` 方法且期望返回同一个对象，**必须使用代理**（默认行为），否则每次调用都会创建新对象

---

## 21. Nacos AP和CP模式切换，Raft协议在Nacos中的应用

### 题目
Nacos 同时支持 AP（最终一致性）和 CP（强一致性）两种模式，请描述 Nacos 的 Raft 协议实现、AP/CP 模式如何切换，以及它们各自的适用场景。

### 核心答案

**CAP 定理与 Nacos：**

CAP 定理指出分布式系统无法同时满足 Consistency（一致性）、Availability（可用性）、Partition tolerance（分区容错）。Nacos 通过选择不同的协议同时支持 AP 和 CP：

| 模式 | 协议 | 适用场景 | 数据一致性 |
|------|------|----------|------------|
| AP |  Distro | 服务注册/发现 | 最终一致性 |
| CP |  Raft | 配置管理 | 强一致性 |

**Raft 协议在 Nacos CP 模式中的应用：**

Nacos 的 CP 模式基于 Raft 协议实现选主和日志复制：

```
Raft 角色：Leader → Follower → Candidate
         ↓
     Leader 选举（Term 递增 + 投票）
         ↓
     日志复制（Write Request → Leader 写本地 → 复制到 Follower → 半数确认 → Commit）
         ↓
     Leader 挂了 → 重新选举（已有日志完整性检查）
```

**Leader 选举流程：**
1. 节点启动后是 Follower
2. 收到心跳超时 → 转为 Candidate，Term++，发起投票
3. 获得集群半数以上投票 → 成为 Leader
4. 其他 Candidate 发现已有 Leader → 转为 Follower

**AP/CP 模式切换：**

```yaml
# application.yml
spring:
  cloud:
    nacos:
      discovery:
        # 默认 AP（服务注册发现）
        # 切换 CP：curl -X PUT 'http://localhost:8848/nacos/v1/ns/operator/switches?entry=serverMode&value=CP'
      config:
        # CP 模式用于配置管理
```

```java
// 通过 API 切换
@Bean
public NacosServiceManager nacosServiceManager() {
    return new NacosServiceManager();
}

// 切换到 CP 模式
nacosServiceManager.updateClusterMetadata("CP模式");
```

**Distro 协议（AP 模式）：**

Nacos AP 模式使用自研的 Distro 协议：
- 每个节点负责部分数据，通过异步复制实现最终一致
- 节点间通过心跳检测故障，节点挂了将请求转发给其他节点
- 适合服务发现的高可用场景

### 追问方向
- Raft 的"过半确认"机制在 Nacos 中如何实现？
- Nacos 配置变更时，Raft Leader 挂了怎么办？
- Distro 协议和 Raft 协议在数据分片上的区别是什么？

### 避坑提示
- 服务注册默认是 AP 模式，切换到 CP 后注册会变慢（需要 Leader 确认），但配置变更是强一致的
- CP 模式下节点数建议使用奇数（3/5/7），避免平票问题
- 不要在同一个命名空间混用 AP 和 CP，CP 模式下服务注销需要主动注销而非依赖超时删除

---

## 22. Sentinel的@SentinelResource和Feign熔断的整合

### 题目
Sentinel 的 `@SentinelResource` 注解和 OpenFeign 的熔断机制是如何整合的？两者同时使用时会冲突吗？请描述整合原理和配置方式。

### 核心答案

**Sentinel 与 Feign 的整合原理：**

Sentinel 通过 `SentinelFeign` 包装 Feign Client，实现了两层保护：

```
调用链路：
Feign Client → SentinelFeign 自动包装 → @SentinelResource 限流/熔断
                  ↓
           底层调用 SphU.entry() / SphAsyncEntry()
```

**整合依赖：**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**配置方式：**

```yaml
# application.yml
feign:
  sentinel:
    enabled: true   # 开启 Sentinel 对 Feign 的支持
```

**@SentinelResource 在 Feign Client 中的使用：**

```java
@FeignClient(name = "order-service", fallback = OrderFeignFallback.class)
public interface OrderFeignClient {

    @GetMapping("/order/{id}")
    @SentinelResource(value = "getOrder", blockHandler = "getOrderBlockHandler",
                      fallback = "getOrderFallback")
    Order getOrder(@PathVariable("id") Long id);
}

// 限流处理（参数必须和原方法一致，最后加 BlockException）
public Order getOrderBlockHandler(Long id, BlockException ex) {
    return Order.builder().id(id).name("限流返回").build();
}

// 熔断降级处理
public Order getOrderFallback(Long id, Throwable throwable) {
    return Order.builder().id(id).name("降级返回").build();
}
```

**Sentinel + Feign 熔断的fallback vs blockHandler：**

| 注解 | 触发条件 | 执行位置 | 用途 |
|------|----------|----------|------|
| `fallback` | 方法抛出异常（业务异常） | 业务线程 | 业务降级 |
| `blockHandler` | Sentinel 限流/熔断/系统规则触发 | Sentinel 回调线程 | 流量控制 |

**两者的优先级关系：**

```
请求进入 → Sentinel 检查限流规则
           ↓
      被限流 → blockHandler 执行
           ↓
      通过检查 → 调用真实方法
                ↓
           方法异常 → fallback 执行
                ↓
           返回结果
```

### 追问方向
- `@SentinelResource` 的 `fallback` 和 Feign Client 的 `fallback` 类有什么区别？优先级谁高？
- Sentinel 的熔断策略（慢调用比例/异常比例/异常数）和 Feign 的超时时间如何配合？
- 如何让 Sentinel 的统计数据暴露到 Actuator 端点？

### 避坑提示
- `blockHandler` 方法必须声明为 `public static`，且参数最后一位必须是 `BlockException`
- Feign 的超时配置（`feign.client.config.default.readTimeout`）要大于 Sentinel 熔断的最大RT，否则可能还没触发熔断就先超时了
- 同时配置 `@SentinelResource(fallback)` 和 Feign `fallback` 类时，注意不要重复降级逻辑

---

## 23. Spring Cloud灰度发布方案，Gateway + Nacos实现实例级权重

### 题目
在实际生产中，如何使用 Spring Cloud Gateway + Nacos 实现灰度发布（实例级权重路由）？请描述完整方案和实现原理。

### 核心答案

**灰度发布的几种常见策略：**

| 策略 | 实现方式 | 适用场景 |
|------|----------|----------|
| 请求头染色 | Gateway 根据 Header 路由 | 内部测试 |
| 权重路由 | 按比例分配流量到不同版本 | 正式发布前验证 |
| 标签路由 | 用户群体标签 + 路由规则 | 特定用户群体 |
| 金丝雀发布 | 渐进式切换流量 | 新版本稳定性验证 |

**Nacos 实例权重配置：**

```yaml
# Nacos 控制台 → 服务详情 → 实例列表 → 编辑 → 权重值（0.0 ~ 1.0）
# 权重为 0 表示暂停服务，但不移除实例
# 权重越高，被选中的概率越高
```

```json
// Nacos 注册的实例元数据中可以携带版本信息
{
  "instanceId": "192.168.1.101#8080#DEFAULT#group#order-service",
  "ip": "192.168.1.101",
  "port": 8080,
  "weight": 0.3,
  "metadata": {
    "version": "v1",
    "gray": "true"
  }
}
```

**Gateway 灰度路由实现原理：**

```java
@Configuration
public class GrayRoutingFilter extends AbstractGatewayFilterFactory {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public GatewayFilter apply(TypedConfig config) {
        return (exchange, chain) -> {
            String version = exchange.getRequest().getHeaders()
                .getFirst("X-Gray-Version");
            
            // 获取所有健康实例
            List<ServiceInstance> instances = discoveryClient
                .getInstances("order-service");
            
            // 过滤符合条件的实例
            List<ServiceInstance> grayInstances = instances.stream()
                .filter(i -> version != null 
                    && version.equals(i.getMetadata().get("version")))
                .collect(Collectors.toList());
            
            // 如果有灰度版本则使用灰度实例，否则走默认
            List<ServiceInstance> targetInstances = grayInstances.isEmpty() 
                ? instances : grayInstances;
            
            // 权重负载均衡选择实例
            ServiceInstance selected = chooseInstance(targetInstances);
            
            // 构建转发地址
            String uri = String.format("http://%s:%d", 
                selected.getHost(), selected.getPort());
            
            return chain.filter(exchange.mutate()
                .request(builder -> builder.uri(uri)).build());
        };
    }
}
```

**基于权重路由的完整配置：**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service  # lb: 启用负载均衡
          predicates:
            - Path=/order/**
          filters:
            - name: GrayWeight
              args:
                serviceId: order-service
                grayHeader: X-Gray-Version
```

**灰度发布的完整流程：**

```
1. 部署新版本（v2）实例 → 注册到 Nacos，metadata.version=v2，weight=0.1
2. 老版本（v1）实例 weight=0.9
3. 用户请求进入 Gateway → 检查 X-Gray-Version Header
4. 有 Header → 路由到对应版本
5. 无 Header → 按权重路由（10% v2，90% v1）
6. 观察监控指标（错误率、响应时间）
7. 逐步提高 v2 权重，直至全量
```

### 追问方向
- 如何避免灰度过程中 Session 一致性问题（用户一会 v1 一会 v2）？
- Nacos 的权重是针对单个实例还是整个服务？权重路由底层用的什么算法？
- 灰度过程中服务发现延迟如何处理？

### 避坑提示
- 权重配置建议从小到大渐进（0.1 → 0.3 → 0.5 → 0.8 → 1.0），每一步观察 5-10 分钟
- 灰度实例 weight=0 时代表"排毒"，不会接收新请求但不会从注册中心删除，避免服务雪崩
- 灰度发布期间，务必监控两个版本的错误率，防止有 Bug 的版本被放大

---

## 24. 服务雪崩的原因，熔断器模式（CircuitBreaker）的状态转换

### 题目
什么是服务雪崩？服务雪崩是如何一步步扩散的？熔断器（CircuitBreaker）模式如何防止雪崩？请详细描述 CircuitBreaker 的三种状态及其转换。

### 核心答案

**服务雪崩的定义：**

服务雪崩是指分布式系统中，**下游服务的故障导致上游服务调用堆积，最终上游服务也宕机，并逐级扩散至整个调用链**的现象。

**服务雪崩的演进链路：**

```
正常状态：
用户 → 服务A → 服务B → 服务C（均正常）

第一步：服务C故障（响应慢/超时）
用户 → 服务A → 服务B → [C超时/报错]
                        ↓
                  B 的线程堆积（等待C返回）
                  B 的资源被耗尽

第二步：B 也开始不可用
用户 → [A调用B超时] → [A的资源也被耗尽]
                        ↓
                  A 开始拒绝请求

第三步：雪崩扩散到整个系统
用户 → [A不可用] → 整个系统宕机
```

**服务雪崩的常见原因：**

| 原因 | 描述 | 示例 |
|------|------|------|
| 硬件故障 | 机房断电、网络抖动 | 服务器宕机 |
| 流量激增 | 促销活动导致请求暴增 | 瞬时 QPS 100倍 |
| 级联失败 | 下游慢查询拖垮上游 | 数据库慢查询 |
| 缓存穿透 | 缓存失效击穿到数据库 | 热点 key 失效 |

**熔断器（CircuitBreaker）模式：**

```
状态转换图：
                    ↓
    ┌──────────────────────────────────┐
    │         CLOSED（闭合）            │ ← 正常状态，所有请求通过
    │   所有请求通过，统计失败次数       │
    └────────────┬─────────────────────┘
                 │ 失败次数/比例超过阈值
                 ↓
    ┌──────────────────────────────────┐
    │         OPEN（断开）              │ ← 快速失败，所有请求直接拒绝
    │   熔断器打开，所有请求直接降级      │
    └────────────┬─────────────────────┘
                 │ 等待时间（sleepWindow）后
                 ↓
    ┌──────────────────────────────────┐
    │      HALF_OPEN（半开）            │ ← 试探恢复，放行部分请求
    │   放行部分请求测试下游是否恢复      │
    └────────────┬─────────────────────┘
         │      │      │
    成功   │   失败   │
         ↓      ↓      ↓
    CLOSED   OPEN   CLOSED（重置计时器）
```

**Resilience4j 熔断器配置示例：**

```java
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        return CircuitBreakerRegistry.of(Map.of(
            "default", CircuitBreakerConfig.custom()
                .failureRateThreshold(50)           // 失败率阈值 50%
                .slidingWindowType(SlidingWindowType.COUNT_BASED)
                .slidingWindowSize(10)             // 统计 10 次请求
                .minimumNumberOfCalls(5)            // 最少 5 次调用才计算
                .waitDurationInOpenState(Duration.ofSeconds(30)) // 30 秒后半开
                .permittedNumberOfCallsInHalfOpenState(3)  // 半开状态放行 3 次
                .build()
        ));
    }
}

// 使用
@Service
public class OrderService {
    
    @Autowired
    private CircuitBreakerRegistry registry;

    public String getOrder(Long id) {
        CircuitBreaker circuitBreaker = registry.circuitBreaker("orderService");
        
        Supplier<String> supplier = () -> orderClient.getOrder(id);
        
        // 降级回调
        Supplier<String> fallback = () -> "降级返回: 订单服务不可用";
        
        return circuitBreaker.executeSupplier(supplier);
    }
}
```

### 追问方向
- 熔断器的滑动窗口是什么？时间窗口和计数窗口的区别？
- 为什么熔断器恢复时要设计成 HALF_OPEN 状态而不是直接 OPEN → CLOSED？
- Resilience4j 和 Spring Cloud CircuitBreaker 的关系是什么？

### 避坑提示
- 熔断阈值设置过小（如 10%）会导致频繁熔断，设置过大则起不到保护作用，建议 50%
- `waitDurationInOpenState` 不宜过短（至少 30s），否则下游还没恢复就又放行请求
- 熔断器统计的是**超时和异常**，不包括业务返回（如空结果），需要确认你的超时配置合理

---

## 25. 什么是服务降级，如何区分熔断、降级、限流

### 题目
服务降级和熔断、限流有什么区别？它们各自的触发条件是什么？三者之间有什么关系？

### 核心答案

**三个概念的定义和区别：**

| 概念 | 定义 | 触发条件 | 目的 | 作用范围 |
|------|------|----------|------|----------|
| 限流 | 控制并发/速率，超过阈值拒绝请求 | QPS/并发数超过设定值 | 保护系统不被冲垮 | 系统入口 |
| 熔断 | 下游故障时快速失败，不再调用下游 | 下游错误率/超时达到阈值 | 防止级联故障扩散 | 调用链中间 |
| 降级 | 返回预设的兜底数据/逻辑 | 系统压力大/依赖不可用 | 保证核心功能可用 | 被调方 |

**三者对比的经典图示：**

```
用户请求
    ↓
[限流] → 超过 QPS 限制 → 直接返回"系统繁忙"（拒绝）
    ↓ 通过
[熔断器] → 下游错误率高 → 直接返回降级结果（不调用下游）
    ↓ 通过
[业务逻辑] → 正常执行 → 返回结果
                ↓ 异常/超时
          [降级逻辑] → 返回兜底数据
```

**限流的具体实现：**

```java
// Sentinel 限流示例
@SentinelResource(value = "getUser", blockHandler = "getUserBlock")
public User getUser(Long id) {
    return userClient.getUser(id);
}

// 限流处理
public User getUserBlock(Long id, BlockException ex) {
    return User.builder().name("限流返回").build();
}

// Sentinel 限流规则配置
List<FlowRule> rules = Collections.singletonList(
    FlowRule.builder()
        .resource("getUser")
        .count(100)  // QPS 不超过 100
        .controlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT)
        .build()
);
```

**降级的具体实现：**

```java
// 业务降级：主动返回兜底数据
@Service
public class ProductService {
    
    public Product getProduct(Long id) {
        try {
            return productDB.getProduct(id);
        } catch (Exception e) {
            // 降级：返回缓存数据
            return productCache.get(id).orElse(
                Product.builder().id(id).name("商品不存在").build()
            );
        }
    }
}

// Feign 降级
@FeignClient(name = "product-service", fallback = ProductFeignFallback.class)
public interface ProductClient {
    @GetMapping("/product/{id}")
    Product getProduct(@PathVariable Long id);
}

@Component
class ProductFeignFallback implements ProductClient {
    @Override
    public Product getProduct(Long id) {
        return Product.builder().id(id).name("降级商品").build();
    }
}
```

**三者关系总结：**

```
限流：保护系统整体入口（防止被冲垮）
   ↓
熔断：保护调用链中间节点（防止级联扩散）
   ↓
降级：保证最终用户有响应（返回兜底数据）

限流是预防措施（事前）
熔断是阻断措施（事中）
降级是补救措施（事后）
```

### 追问方向
- 限流的算法有哪些？令牌桶和漏桶的区别是什么？
- 降级和熔断在代码实现上有什么区别？
- 什么场景下降级比熔断更重要？

### 避坑提示
- 降级不等于熔断：降级是主动返回兜底，熔断是检测到故障后自动触发
- 限流通常在系统入口（如 Gateway）做，而不是每个微服务都做限流
- 降级数据要提前准备（如本地缓存、默认值），不能等到故障时才去查询

---

## 26. Spring Cloud OpenFeign的contract原则

### 题目
Feign 的 contract（契约）是什么意思？Spring Cloud OpenFeign 的默认契约是什么？它和 JAX-RS、Retrofit 的契约有什么区别？

### 核心答案

**Contract（契约）的定义：**

Contract 是 Feign Client 接口上的注解（如 `@RequestMapping`）如何被解析成 HTTP 请求的规则。不同框架定义了不同的注解作为"契约"，Feign 通过切换 Contract 实现对不同注解风格的支持。

**Spring Cloud OpenFeign 的默认契约：**

Spring Cloud OpenFeign 默认使用 **Spring MVC 的注解风格**作为契约：

```java
@FeignClient(name = "user-service")
public interface UserClient {

    @GetMapping("/user/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping("/user")
    User createUser(@RequestBody User user);

    @GetMapping("/user")
    List<User> getUsers(@RequestParam("name") String name);
}
```

**支持的所有契约类型：**

| 契约 | 注解风格 | 引入方式 |
|------|----------|----------|
| Spring MVC Contract（默认） | `@GetMapping`、`@RequestParam` | 默认 |
| JAX-RS Contract | `@GET`、`@Path`、`@QueryParam` | `feign.jaxb` |
| OpenFeign 默认 Contract | `@RequestLine`、`@Param` | 原生 Feign |

**原生 Feign Contract 示例：**

```java
// 使用原生 Feign 契约，不需要 Spring MVC 注解
interface UserClient {
    @RequestLine("GET /user/{id}")
    User getUser(@Param("id") Long id);

    @RequestLine("POST /user")
    User createUser(RequestBody user);
}
```

**Spring MVC Contract 解析原理：**

```
1. SpringMvcContract 继承自 BaseContract
2. processAnnotationsOnClass() 读取 @FeignClient 类上的路径
3. parseAnnotations() 解析方法上的 Spring MVC 注解
4. 构建 RequestTemplate（包含 URL、HTTP 方法、查询参数等）
5. 最终生成 HTTP 请求
```

**自定义 Contract 示例：**

```java
// 自定义注解作为契约
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyGet {
    String value();
}

// 自定义 Contract
public class MyContract implements Contract {
    @Override
    public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
        MethodMetadata metadata = new MethodMetadata();
        metadata.returnType(method.getReturnType());
        
        MyGet myGet = method.getAnnotation(MyGet.class);
        if (myGet != null) {
            metadata.template().method(Request.HttpMethod.GET);
            metadata.template().uri(myGet.value());
        }
        return metadata;
    }
}

// 配置自定义 Contract
@Configuration
public class FeignConfig {
    @Bean
    public Contract feignContract() {
        return new MyContract();
    }
}
```

### 追问方向
- `@RequestMapping` 和 `@GetMapping` 在 Feign 中的解析有什么区别？
- 如何让 Feign 支持 JAX-RS 注解？
- Feign 的 Contract 和 Client 是什么关系？

### 避坑提示
- Spring MVC Contract 只能解析 Spring MVC 注解，不支持原生 Feign 的 `@RequestLine`
- 方法级别的 `@RequestMapping` 会继承类级别的 `@RequestMapping` 路径
- `@RequestParam` 必须指定 name 或 value，否则 Feign 默认使用参数名（需要 `-parameters` 编译参数）

---

## 27. Nacos的配置隔离（namespace、group、dataId三级结构）

### 题目
Nacos 配置中心的三级配置隔离（namespace、group、dataId）各自的用途是什么？为什么需要三级隔离？如何设计一套多环境、多业务、多模块的配置方案？

### 核心答案

**Nacos 配置的三级结构：**

```
Nacos 配置层级：
├── namespace（命名空间）    → 环境隔离
│   ├── group（配置组）      → 业务线/团队隔离
│   │   ├── dataId 1        → 具体配置文件
│   │   ├── dataId 2
│   │   └── dataId 3
│   └── group（另一个业务线）
└── namespace（另一个环境）
```

**三级各自的用途：**

| 级别 | 作用 | 典型值 | 隔离维度 |
|------|------|--------|----------|
| namespace | 环境隔离 | `dev`、`test`、`prod` | 不同环境 |
| group | 业务线/模块隔离 | `order`、`payment`、`user` | 不同业务 |
| dataId | 配置文件名 | `application.yml`、`mybatis-config.xml` | 不同配置项 |

**配置获取方式：**

```java
// 方式1：直接指定
@RestController
public class ConfigController {
    
    @NacosValue(value = "${custom.property:default}", autoRefreshed = true)
    private String property;
}

// 方式2：动态监听
@Autowired
private NacosConfigManager nacosConfigManager;

public void loadConfig() {
    String dataId = "application.yml";
    String group = "DEFAULT_GROUP";
    String namespace = "dev";
    
    String content = nacosConfigManager.getConfigService()
        .getConfig(dataId, group, namespace);
}
```

**application.yml 中的配置：**

```yaml
spring:
  cloud:
    nacos:
      config:
        namespace: ${NACOS_NAMESPACE:dev}           # 命名空间 ID
        group: ${NACOS_GROUP:DEFAULT_GROUP}          # 配置组
        data-id: ${NACOS_DATA_ID:application.yml}   # 配置文件名
        file-extension: yaml                        # 文件扩展名
        refresh-enabled: true                        # 开启自动刷新
```

**多环境、多业务、多模块的配置设计：**

```
Nacos 配置中心设计：
│
├── dev（namespace）
│   ├── order-group（group）
│   │   ├── application.yml          # 主配置
│   │   ├── datasource.yml           # 数据源配置
│   │   └── redis.yml                # Redis 配置
│   ├── payment-group
│   │   ├── application.yml
│   │   └── payment.yml
│   └── user-group
│       ├── application.yml
│       └── user.yml
│
├── test（namespace）
│   └── ...
│
└── prod（namespace）
    └── ...
```

**不同配置文件在 Spring 中的加载：**

```yaml
# 共享配置shared-dataids（多个配置用逗号分隔）
spring:
  cloud:
    nacos:
      config:
        namespace: dev
        group: order-group
        # 加载多个共享配置
        shared-dataids: common.yml,redis.yml
        refreshable-dataids: common.yml
        file-extension: yaml
```

### 追问方向
- namespace 是怎么实现的？不同 namespace 下的服务能否相互发现？
- `shared-dataids` 和 `ext-config` 的区别是什么？
- Nacos 配置变更后，Spring 如何感知并刷新 Bean？

### 避坑提示
- namespace 不是隔离服务注册的，只是隔离配置管理，服务发现还是靠 group 来区分
- `refresh-enabled=true` 只是开启自动刷新功能，真正刷新需要 Bean 标注 `@RefreshScope`
- 配置文件热更新有延迟（默认 3000ms），敏感配置变更建议手动调用 `ConfigController` 刷新

---

## 28. Ribbon的负载均衡策略（RoundRobinRule、RandomRule、WeightedResponseTimeRule）

### 题目
Ribbon 提供了哪些负载均衡策略？请详细描述 RoundRobinRule、RandomRule、WeightedResponseTimeRule 的实现原理，以及如何选择合适的策略。

### 核心答案

**Ribbon 内置负载均衡策略：**

| 策略 | 类名 | 原理 | 特点 |
|------|------|------|------|
| 轮询 | RoundRobinRule | 依次选择每个服务器 | 简单均匀 |
| 随机 | RandomRule | 随机选择一个服务器 | 简单，避免热点 |
| 加权响应时间 | WeightedResponseTimeRule | 响应时间越短权重越高 | 性能导向 |
| 最少连接 | BestAvailableRule | 选择连接数最少的 | 避免过载 |
| 重试 | RetryRule | 轮询 + 失败重试 | 容错 |
| 可用性过滤 | AvailabilityFilteringRule | 过滤熔断/连接数过多的 | 稳定性 |
| 区域感知 | ZoneAvoidanceRule | 优先选择同区域实例（默认） | 降低延迟 |

**RoundRobinRule 实现原理：**

```java
// 伪代码实现
public class RoundRobinRule extends AbstractLoadBalancerRule {
    
    private AtomicInteger nextServerCyclicCounter = new AtomicInteger(0);

    @Override
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        
        List<Server> servers = lb.getReachableServers();
        int total = servers.size();
        if (total == 0) {
            return null;
        }
        
        // 原子递增，然后取模
        int next = nextServerCyclicCounter.incrementAndGet();
        int index = (next - 1) % total;
        
        return servers.get(index);
    }
}
```

**RandomRule 实现原理：**

```java
public class RandomRule extends AbstractLoadBalancerRule {
    
    @Override
    public Server choose(ILoadBalancer lb, Object key) {
        List<Server> servers = lb.getReachableServers();
        if (servers.isEmpty()) {
            return null;
        }
        
        // 简单的随机数
        int randomIndex = ThreadLocalRandom.current().nextInt(servers.size());
        return servers.get(randomIndex);
    }
}
```

**WeightedResponseTimeRule 实现原理：**

```
核心思想：响应时间越短的服务器，被选中的概率越高

权重计算：
- 每隔一段时间（默认 30 秒），收集所有实例的响应时间
- 计算权重公式：weight[i] = MAX(0, (TotalResponseTime - ResponseTime[i])) / TotalResponseTime
- 响应时间越短，TotalResponseTime - ResponseTime[i] 越大，权重越高

选择过程：
- 生成一个 [0, 1) 的随机数
- 遍历所有服务器，累加权重
- 落在哪个区间就选择哪个服务器
```

```java
// 内部维护一个权重列表（每 30 秒更新一次）
private int[] loadBalancerWeights = new int[]{};

@Override
public Server choose(ILoadBalancer lb, Object key) {
    // 初始化时根据响应时间计算权重
    if (initializeEveryCycle()) {
        // 统计响应时间，计算权重
    }
    
    // 权重随机选择
    return chooseByWeight(nextRandomIndex());
}
```

**策略选择建议：**

| 场景 | 推荐策略 | 原因 |
|------|----------|------|
| 服务实例性能一致 | RoundRobinRule | 均匀分布 |
| 服务实例性能不一致 | WeightedResponseTimeRule | 性能好的承担更多流量 |
| 存在慢查询/过载实例 | BestAvailableRule | 找连接数最少的 |
| 需要容错重试 | RetryRule | 失败后尝试下一个 |
| 服务跨多可用区 | ZoneAvoidanceRule（默认） | 减少跨区延迟 |

**配置方式：**

```yaml
# 全局配置
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule

# 或者通过 Java 配置
@Configuration
public class RibbonConfig {
    @Bean
    public IRule ribbonRule() {
        return new WeightedResponseTimeRule();
    }
}
```

### 追问方向
- `ZoneAvoidanceRule` 的区域（Zone）是什么概念？如何配置？
- Ribbon 的负载均衡策略和 Spring Cloud LoadBalancer 的关系是什么？
- 权重更新周期（30秒）如何调优？

### 避坑提示
- WeightedResponseTimeRule 依赖历史响应时间统计，**新上线的实例权重为 0**，会有冷启动问题
- ZoneAvoidanceRule 如果只有一个可用区，会退化成 RoundRobinRule
- Ribbon 8.x 后进入维护模式，Spring Cloud 2020.0 起推荐使用 Spring Cloud LoadBalancer

---

## 29. Spring Cloud LoadBalancer替换Ribbon的原因和实现原理

### 题目
Spring Cloud 2020.0 版本正式移除了 Ribbon，转而使用 Spring Cloud LoadBalancer。为什么会有这个变化？LoadBalancer 的实现原理是什么？

### 核心答案

**Ribbon 被移除的原因：**

| 问题 | 描述 |
|------|------|
| 版本停滞 | Ribbon 2020 年后停止维护，最后版本 2020.0.8 |
| 耦合性高 | Ribbon 与 Netflix 生态深度绑定，无法适应云原生发展 |
| 注解侵入 | `@RibbonClient` 注解分散在业务代码中 |
| 阻塞 IO | 基于同步阻塞的 HTTP 客户端，不适合响应式编程 |
| 重量级 | 功能冗余，很多功能在 Spring Cloud Gateway 中已有替代 |

**Spring Cloud LoadBalancer 的定位：**

Spring Cloud LoadBalancer 是 Spring Cloud 官方推出的轻量级、响应式负载均衡组件，专门为 Spring Cloud 服务调用设计。

**LoadBalancer 实现原理：**

```
调用链路：
RestTemplate / WebClient
    ↓
LoadBalancerInterceptor 拦截请求
    ↓
ServiceInstanceChooser 选择实例
    ↓
实现类（RoundRobinLoadBalancer / RandomLoadBalancer）
    ↓
返回选中的 ServiceInstance
    ↓
构建真实请求 URL
```

**核心接口：**

```java
// 服务实例选择器
public interface ServiceInstanceChooser {
    ServiceInstance choose(String serviceId);
}

// 负载均衡器
public interface ReactorLoadBalancer<T> extends ReactiveLoadBalancer<T> {
    Mono<Response<T>> getInstance();
}

// 默认实现
public class RoundRobinLoadBalancer implements ReactorLoadBalancer<ServiceInstance> {
    @Override
    public Mono<Response<ServiceInstance>> getInstance() {
        // 轮询逻辑，返回 Mono<Response<ServiceInstance>>
    }
}
```

**LoadBalancer 的使用方式：**

```java
// 1. RestTemplate + @LoadBalanced（保持兼容）
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// 2. WebClient（响应式）
@Bean
public WebClient.Builder webClientBuilder(LoadBalancerExchangeFilterFunction function) {
    return WebClient.builder()
        .filter(function);
}

// 3. 显式使用 LoadBalancer
@Service
public class MyService {
    
    @Autowired
    private ReactorLoadBalancer<ServiceInstance> loadBalancer;
    
    public Mono<String> callService() {
        return loadBalancer.getInstance()
            .flatMap(instance -> {
                String url = "http://" + instance.getHost() + ":" + instance.getPort();
                return WebClient.create(url).get().retrieve().bodyToMono(String.class);
            });
    }
}
```

**自定义 LoadBalancer 示例：**

```java
// 基于权重的负载均衡器
@Configuration
public class CustomLoadBalancerConfig {
    
    @Bean
    public ReactorLoadBalancer<ServiceInstance> customLoadBalancer(
            ObjectProvider<ServiceInstances> instancesProvider) {
        return new WeightedLoadBalancer(instancesProvider);
    }
}

// Spring Cloud LoadBalancer 自动装配
@Configuration
@LoadBalancerClient(name = "order-service", configuration = CustomLoadBalancerConfig.class)
public class LoadBalancerConfig {
    // 针对 order-service 使用自定义配置
}
```

**Ribbon vs LoadBalancer 对比：**

| 对比项 | Ribbon | LoadBalancer |
|--------|--------|--------------|
| 编程模型 | 同步阻塞 | 响应式（Reactor） |
| 配置方式 | `@RibbonClient` 注解 | `@LoadBalancerClient` + Java Config |
| 内置策略 | 7 种 | 2 种（RoundRobin、Random） |
| 扩展方式 | 实现 IRule | 实现 ReactorLoadBalancer |
| 维护状态 | 已停止维护 | 活跃维护中 |

### 追问方向
- LoadBalancer 如何实现和 Ribbon 相同的权重负载均衡？
- `@LoadBalanced` 注解的原理是什么？
- LoadBalancer 如何与 Spring Cloud CircuitBreaker 整合？

### 避坑提示
- LoadBalancer 依赖服务发现的 `ServiceInstanceListSupplier`，需要确保引入了对应 discovery 的 starter
- 自定义 LoadBalancer 配置类要避免被全局扫描到，建议放在单独的配置包下
- 响应式编程不熟悉时，建议先从 `@LoadBalanced` RestTemplate 入手

---

## 30. Seata的AT模式原理，两阶段提交和undolog表

### 题目
Seata 的 AT 模式是如何实现分布式事务的？为什么它能实现强一致性又不阻塞业务线程？请详细描述两阶段提交的过程和 undolog 表的作用。

### 核心答案

**Seata AT 模式概述：**

AT 模式是 Seata 最常用的模式，通过**对业务 SQL 的解析**和**undolog 日志**实现分布式事务，全程自动处理，无需人工干预。

**AT 模式的两阶段流程：**

```
第一阶段（Prepare）：
    TM（事务管理器）开启全局事务（XID）
        ↓
    各分支事务（Branch）执行业务 SQL
        ↓
    解析 SQL，生成前后镜像（Before Image / After Image）
        ↓
    写入 undolog 表（记录如何回滚）
        ↓
    向 TC（事务协调者）注册分支，报告完成状态
        ↓
    TC 收到所有分支成功 → 第一阶段完成

第二阶段（Commit/Rollback）：
    所有分支 Prepare 成功 → TC 发 Commit → 异步删除 undolog
    任意分支 Prepare 失败 → TC 发 Rollback → 读取 undolog 回滚
```

**undolog 表的结构和作用：**

```sql
-- undolog 表（Seata 自动创建）
CREATE TABLE `undo_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `branch_id` bigint NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,   -- 存储前后镜像的 JSON
  `log_status` int NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

**前后镜像（Before/After Image）示例：**

```sql
-- 业务 SQL：UPDATE account SET balance = balance - 100 WHERE id = 1

-- Before Image（更新前的数据）
SELECT * FROM account WHERE id = 1;
-- 结果：id=1, balance=1000

-- 执行 UPDATE：balance = 900

-- After Image（更新后的数据）
SELECT * FROM account WHERE id = 1;
-- 结果：id=1, balance=900

-- undolog 记录：{before: {id=1, balance=1000}, after: {id=1, balance=900}}
```

**AT 模式回滚原理：**

```java
// 当第二阶段收到 Rollback 命令时
public class UndoLogManager {
    
    public static void undo(DataSourceProxy dataSourceProxy, String xid, BranchId branchId) {
        // 1. 从 undolog 表读取 rollback_info
        UndoLog undoLog = selectUndoLog(xid, branchId);
        
        // 2. 反序列化得到前后镜像
        BranchUndoLog branchUndoLog = UndoLogSerializer.get()
            .deserialize(undoLog.getRollbackInfo());
        
        // 3. 生成反向 SQL 回滚
        // After Image → Before Image
        // UPDATE account SET balance = 1000 WHERE id = 1
        String reverseSQL = generateReverseSQL(branchUndoLog);
        
        // 4. 执行反向 SQL
        execute(reverseSQL);
        
        // 5. 删除 undolog
        deleteUndoLog(xid, branchId);
    }
}
```

**AT 模式与 XA 模式的区别：**

| 对比项 | AT 模式 | XA 模式 |
|--------|---------|---------|
| 资源锁定 | 无锁（本地事务完成后即可释放） | 全程持有数据库锁 |
| 阻塞 | 非阻塞 | 阻塞（2PC 全程锁） |
| 性能 | 高（接近本地事务） | 低（2PC 开销） |
| SQL 支持 | 仅支持 DML（SELECT/INSERT/UPDATE/DELETE） | 支持所有 SQL |
| 隔离级别 | 读已提交（RC） | 可配置（RC/XA） |

### 追问方向
- AT 模式的"一阶段提交"和普通本地事务有什么区别？
- AT 模式在什么情况下会退化成 TCC 或 XA 模式？
- undolog 表过大的问题如何处理？

### 避坑提示
- AT 模式要求数据库支持本地事务（InnoDB），MyISAM 不支持
- 删除表结构等 DDL 操作不会记录 undolog，需要额外处理
- 全局事务并发控制使用 MVCC，同一个全局事务内对同一行数据的修改会串行执行

---

## 31. Seata的TCC模式、XA模式、Saga模式各自适用场景

### 题目
Seata 除了 AT 模式外，还支持 TCC、XA、Saga 模式，请分别描述这四种模式的特点、适用场景，以及 TCC 模式中 Try/Confirm/Cancel 的实现。

### 核心答案

**四种模式对比：**

| 模式 | 架构 | 特点 | 隔离性 | 性能 |
|------|------|------|--------|------|
| AT | 两阶段（自动） | 自动处理回滚 | 读已提交 | 高 |
| TCC | 两阶段（手动） | 业务代码实现 Try/Confirm/Cancel | 业务可控 | 中 |
| XA | 两阶段（数据库） | 数据库原生支持 | 可配置 | 低 |
| Saga | 长事务 | 编排器驱动 | 最终一致 | 高 |

**TCC 模式（Try-Confirm-Cancel）：**

```
Try 阶段：预留资源（冻结/锁定）
    ↓ 成功
Confirm 阶段：确认使用资源（真正执行）
    ↓
Cancel 阶段：释放预留资源（回滚）
```

```java
// TCC 业务接口
@LocalTCC
public interface TccService {
    
    @TwoPhaseBusinessAction(
        name = "扣减库存",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean try扣减库存(BusinessActionContext context,
                        @BusinessActionContextParameter(paramName = "skuId") Long skuId,
                        @BusinessActionContextParameter(paramName = "count") Integer count);
    
    // Confirm：Try 成功后执行
    boolean confirm(BusinessActionContext context);
    
    // Cancel：Try 失败或超时执行
    boolean cancel(BusinessActionContext context);
}

@Service
public class TccServiceImpl implements TccService {
    
    @Override
    public boolean try扣减库存(BusinessActionContext context, Long skuId, Integer count) {
        // Try：冻结库存（不减实际库存）
        // UPDATE sku SET frozen_stock = frozen_stock + count WHERE id = skuId AND stock >= count
        return frozenStockMapper.freeze(skuId, count) > 0;
    }
    
    @Override
    public boolean confirm(BusinessActionContext context) {
        // Confirm：真正扣减库存
        // UPDATE sku SET stock = stock - count, frozen_stock = frozen_stock - count
        return stockMapper.decreaseActual(context);
    }
    
    @Override
    public boolean cancel(BusinessActionContext context) {
        // Cancel：释放冻结
        // UPDATE sku SET frozen_stock = frozen_stock - count
        return frozenStockMapper.unfreeze(context);
    }
}
```

**XA 模式：**

```
原理：利用数据库的 XA 协议（两阶段提交）
    ↓
阶段一（Prepare）：TM 通知所有 RM 准备提交，RM 锁定资源并写入 redo log
    ↓
阶段二（Commit/Rollback）：TM 通知所有 RM 提交/回滚
    ↓
特点：数据库原生支持，隔离性最强，但性能损耗大
```

```java
// 开启 XA 事务
@GlobalTransactional
public void placeOrder(Order order) {
    // 直接使用 AT 模式，但指定使用 XA
    // 需要 DataSourceProxy 开启 XA 模式
    orderMapper.create(order);
    stockMapper.decrease(order.getSkuId(), order.getCount());
}
```

**Saga 模式：**

```
适用场景：长流程（10+ 步骤），无法接受阻塞的分布式事务
    ↓
编排器驱动：按顺序执行各子事务
    ↓
失败时：执行补偿操作（Compensation）
    ↓
特点：最终一致性，非强一致性
```

```java
// Saga 状态机配置（Spring Statemachine 或 Seata Saga 编排器）
// 每个子事务都有一个补偿方法
public interface SagaAction {
    // 执行业务
    Response execute(Context context);
    
    // 补偿（回滚）
    Response compensate(Context context);
}
```

**四种模式的适用场景总结：**

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 普通业务（库存、订单、账户） | AT | 性能高，自动处理 |
| 跨库操作、强一致性要求 | XA | 数据库原生，隔离性强 |
| 资源有限、不允许锁定 | TCC | 手动控制，可靠性高 |
| 长流程（微服务编排） | Saga | 性能高，最终一致 |
| 非数据库资源（MQ、Redis） | TCC | AT 不支持 |

### 追问方向
- TCC 的空回滚和悬挂问题如何解决？
- Saga 模式的补偿操作失败怎么办？
- AT 和 XA 在 Oracle/PostgreSQL 中的实现有什么区别？

### 避坑提示
- TCC 的 Try/Confirm/Cancel 方法必须幂等，因为网络重试会多次调用
- TCC 空回滚（Try 未执行但 Cancel 执行了）需要记录标记防止悬挂
- Saga 模式的补偿是逆向执行，不是正向重试，所以补偿逻辑要预先设计好

---

## 32. Spring Cloud微服务如何实现分布式事务

### 题目
在 Spring Cloud 微服务架构中，如何实现分布式事务？请列举常见的分布式事务解决方案，并对比它们的优缺点。

### 核心答案

**分布式事务的常见解决方案：**

| 方案 | 代表框架 | 协调模式 | 优点 | 缺点 |
|------|----------|----------|------|------|
| 两阶段提交（2PC） | Seata XA | 同步强一致性 | 强一致 | 阻塞，性能差 |
| 三阶段提交（3PC） | - | 同步强一致性 | 改善阻塞 | 实现复杂 |
| TCC | Seata TCC | 异步最终一致 | 性能中等，无锁 | 业务侵入，幂等难 |
| Saga | Seata Saga | 编排最终一致 | 性能高 | 仅最终一致 |
| 本地消息表 | MQ + Table | 可靠消息最终一致 | 实现简单 | 延迟，耦合 |
| 最大努力通知 | MQ | 定时重试最终一致 | 实现简单 | 不可靠 |
| AT 模式 | Seata AT | 自动补偿最终一致 | 业务无侵入 | 隔离性有限 |

**Seata AT 模式的完整集成：**

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# Seata 配置
seata:
  tx-service-group: my_tx_group
  registry:
    type: nacos
    nacos:
      server-addr: ${NACOS_HOST}:8848
  config:
    type: nacos
    nacos:
      server-addr: ${NACOS_HOST}:8848
```

```java
// 使用 @GlobalTransactional 开启全局事务
@Service
public class OrderService {
    
    @GlobalTransactional(name = "createOrder", rollbackFor = Exception.class)
    public void createOrder(OrderDTO orderDTO) {
        // 1. 创建订单（AT 模式自动回滚）
        Order order = orderMapper.create(orderDTO);
        
        // 2. 扣减库存（AT 模式自动回滚）
        stockFeignClient.decreaseStock(orderDTO.getSkuId(), orderDTO.getCount());
        
        // 3. 扣减余额（AT 模式自动回滚）
        accountFeignClient.decreaseBalance(orderDTO.getUserId(), orderDTO.getAmount());
    }
}
```

**可靠消息最终一致性方案（RocketMQ 事务消息）：**

```java
// 1. Producer 发送半消息（Half Message）
@Transactional
public void createOrder(Order order) {
    // 本地事务：创建订单
    orderMapper.insert(order);
    
    // 发送事务消息（如果本地事务失败，消息不发送）
    rocketMQTemplate.sendMessageInTransaction(
        "order-topic:create",
        MessageBuilder.withPayload(order).build(),
        "order-transaction"
    );
}

// 2. TransactionListener 监听本地事务状态
@RocketMQTransactionListener
public class OrderTransactionListener implements RocketMQLocalTransactionListener {
    
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 本地事务已执行，返回成功
            return RocketMQLocalTransactionState.COMMIT;
        } catch (Exception e) {
            return RocketMQLocalTransactionState.ROLLBACK;
        }
    }
    
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        // 检查本地事务状态（查库）
        Order order = orderMapper.selectById(msg.getKeys());
        if (order != null) {
            return RocketMQLocalTransactionState.COMMIT;
        }
        return RocketMQLocalTransactionState.UNKNOWN;
    }
}
```

**方案选择建议：**

| 业务场景 | 推荐方案 | 原因 |
|----------|----------|------|
| 强一致性要求（转账、支付） | Seata XA | 强一致，数据库原生支持 |
| 高并发下的普通业务 | Seata AT | 性能高，业务无侵入 |
| 非数据库资源 | Seata TCC | 可自定义补偿逻辑 |
| 长流程编排 | Seata Saga | 性能高 |
| 系统解耦、低一致性 | RocketMQ 事务消息 | 异步，性能好 |

### 追问方向
- 分布式事务和本地事务的区别是什么？
- TCC 和 AT 模式在 Spring Cloud 中如何选择？
- 为什么说"尽量避免分布式事务"是更好的设计？

### 避坑提示
- 分布式事务有性能损耗，**优先考虑业务拆分避免跨库事务**
- `@GlobalTransactional` 的 timeout 默认 60 秒，超时全局回滚，但下游超时可能导致数据不一致
- Seata 需要额外部署 TC（Transaction Coordinator）服务，不能忽略运维成本

---

## 33. 为什么Spring Cloud不推荐使用JPA，而推荐MyBatis

### 题目
在 Spring Cloud 微服务项目中，为什么社区更推荐使用 MyBatis 而不是 JPA（Spring Data JPA）？两者在微服务架构下各自有什么优缺点？

### 核心答案

**JPA 与 MyBatis 的本质区别：**

| 对比项 | JPA | MyBatis |
|--------|-----|---------|
| 定位 | ORM 框架（对象-关系映射） | SQL 映射框架 |
| SQL 控制 | 框架自动生成 | 开发者手写 |
| 理念 | 全自动，开发者少写代码 | 半自动，精准控制 SQL |
| 学习曲线 | 低（入门简单） | 中（需要写 SQL） |

**微服务架构下 JPA 的问题：**

1. **SQL 不可控，线上调优困难**
   ```java
   // JPA 自动生成的 SQL（可能很复杂）
   @Entity
   public class Order {
       @ManyToOne(fetch = FetchType.LAZY)
       private User user;
   }
   // JPA 可能生成 N+1 查询：1 条查订单 + N 条查用户
   ```

2. **关联查询性能差**
   - JPA 的关联加载策略（JOIN FETCH）在复杂场景下难以控制
   - 微服务拆库后，跨库关联查询本就不该用 JOIN，而是接口聚合

3. **事务边界与微服务边界冲突**
   - JPA 适合单体应用中跨表的事务
   - 微服务强调**无分布式事务**，每个服务只操作自己的库

4. **难以应对分库分表**
   - ShardingSphere-JDBC 分表后，JPA 很难处理跨分片查询
   - MyBatis 可以通过动态 SQL 精确控制分片键

**MyBatis 的优势：**

1. **SQL 完全可控**
   ```xml
   <!-- MyBatis：精准控制 SQL -->
   <select id="selectOrderDetail" resultMap="OrderDetailMap">
       SELECT o.id, o.create_time,
              u.name as user_name, u.phone
       FROM orders o
       LEFT JOIN user u ON o.user_id = u.id
       WHERE o.id = #{id}
   </select>
   ```

2. **天然支持分库分表**
   ```xml
   <!-- 按 user_id 分片查询 -->
   <select id="selectByUserId" resultType="Order">
       SELECT * FROM orders_${userId % 100}
       WHERE user_id = #{userId}
   </select>
   ```

3. **与微服务架构契合**
   - 每个微服务专注自己的表，不需要复杂的 ORM 关联
   - MyBatis 的接口+XML 分离，SQL 可以独立优化、单独上线

**性能对比（典型场景）：**

| 场景 | JPA 耗时 | MyBatis 耗时 |
|------|----------|--------------|
| 单表 CRUD | 相同 | 相同 |
| 关联查询（1对1） | 略高 | 精准 |
| N+1 查询 | 严重 | 可控 |
| 动态条件查询 | 生成复杂 SQL | 清晰 |
| 分库分表 | 难支持 | 友好 |

**实际项目选择建议：**

```
选择 JPA：内部系统、CRUD 为主、团队 Java ORM 经验不足、快速原型

选择 MyBatis：高并发系统、分库分表、复杂查询、SQL 调优需求、团队有 DBA 支持
```

### 追问方向
- Spring Data JPA 的 `@Query` 和 MyBatis 的 XML 谁更好？
- MyBatis-Plus（MyBatis 的增强）和 JPA 相比如何？
- 分库分表场景下，JPA 是否有替代方案？

### 避坑提示
- 不要把 MyBatis 的 XML 写成"面向 XML 编程"，SQL 应该简洁易维护
- MyBatis 的 N+1 问题同样存在，association/collection 使用不当会导致性能问题
- MyBatis-Plus 是 MyBatis 的增强而非替代，可以保留 MyBatis 的所有特性同时获得增强功能

---

## 34. Spring Cloud微服务的监控方案（Prometheus + Grafana + Micrometer）

### 题目
Spring Cloud 微服务项目通常使用什么方案进行监控？请描述 Prometheus + Grafana + Micrometer 的完整监控体系，以及 Spring Boot Actuator 如何集成 Micrometer。

### 核心答案

**监控体系的整体架构：**

```
微服务集群（Spring Boot 应用）
        ↓ 暴露 metrics（Actuator + Micrometer）
Metrics 数据 → Prometheus Server（采集/存储时序数据）
        ↓ 查询
Grafana（可视化大盘/告警）
```

**Micrometer 的作用：**

Micrometer 是 Spring Boot 2.0 引入的指标门面（类似 SLF4J），它将应用指标抽象为统一的 `Meter` 接口，对下支持多种监控后端（Prometheus、Datadog、InfluxDB 等）：

```
业务代码
    ↓
Micrometer API（Counter、Gauge、Timer、Summary）
    ↓
Micrometer Registry（PrometheusMeterRegistry 等）
    ↓
Prometheus / Datadog / InfluxDB
```

**Spring Boot Actuator + Micrometer 集成：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}   # 带上应用名标签
    distribution:
      percentiles-histogram:
        http.server.requests: true              # 开启请求延迟直方图
```

**自定义业务指标：**

```java
@Service
public class OrderService {

    // 计数器：统计下单数量
    private final Counter orderCounter;
    
    // 计时器：统计下单耗时
    private final Timer orderTimer;
    
    // Gauge：当前在线用户数
    private final AtomicInteger onlineUsers;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("order.created")
            .description("订单创建数量")
            .tag("service", "order-service")
            .register(registry);
        
        this.orderTimer = Timer.builder("order.create.time")
            .description("订单创建耗时")
            .publishPercentiles(0.5, 0.95, 0.99)  // P50、P95、P99
            .register(registry);
        
        this.onlineUsers = registry.gauge("online.users", 
            new AtomicInteger(0));
    }

    public void createOrder(Order order) {
        // 计时器记录执行时间
        orderTimer.record(() -> {
            // 业务逻辑
            orderMapper.insert(order);
            orderCounter.increment();  // 计数器 +1
        });
    }
}
```

**Prometheus 配置：**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['order-service:8080', 'user-service:8080', 'payment-service:8080']
        labels:
          group: 'production'
```

**Grafana 常用监控大盘模板：**

| 指标 | 用途 |
|------|------|
| JVM 内存使用 | 内存是否泄漏 |
| JVM GC 频率/耗时 | GC 是否正常 |
| HTTP 请求 QPS/RT | 服务性能 |
| 数据库连接池 | 连接是否够用 |
| 业务自定义指标 | 订单量、活跃用户 |
| 服务调用成功率 | 熔断/降级情况 |

**Grafana 告警配置：**

```yaml
# Grafana 告警规则示例
- alert: HighOrderErrorRate
  expr: rate(order_create_errors_total[5m]) / rate(order_created_total[5m]) > 0.05
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "订单错误率超过 5%"
    description: "服务 order-service 订单错误率持续 2 分钟超过 5%"
```

### 追问方向
- Micrometer 和 Dropwizard Metrics 有什么关系？
- Prometheus 的 Pull 模式 vs Push 模式，各有什么优缺点？
- 如何监控 MySQL 慢查询和 HTTP 外部调用耗时？

### 避坑提示
- 不要监控太多自定义指标，Prometheus 的 Label 过多会导致存储压力
- `@Timed` 注解比手动 Timer 更简洁，但需要引入 AOP 依赖
- 生产环境的 metrics 端点建议加认证，防止数据泄露

---

## 35. 服务网格（Service Mesh）概念，Istio与Spring Cloud的关系

### 题目
什么是 Service Mesh（服务网格）？它和 Spring Cloud 是什么关系？Istio 作为服务网格的代表框架，它的核心组件和功能是什么？

### 核心答案

**Service Mesh 的定义：**

Service Mesh 是**基础设施层**，用于处理服务间通信。它通常由轻量级网络代理（Sidecar）组成，这些代理与应用代码一起部署，**负责所有网络通信的可靠性、可观察性和安全性**，而应用本身不需要感知网络逻辑。

**核心思想：网络通信从应用层下沉到基础设施层**

```
传统模式（应用自己处理网络）：
应用代码 → Feign Client → 负载均衡 → 超时重试 → 熔断
        ↓ 所有网络逻辑在业务代码中

Service Mesh 模式（Sidecar 代理处理）：
应用代码 → 本地 Sidecar → Sidecar 之间通信
        ↓ 网络逻辑全部由 Sidecar 代理
```

**Spring Cloud vs Service Mesh：**

| 对比项 | Spring Cloud | Service Mesh（Istio） |
|--------|--------------|------------------------|
| 网络代理位置 | 应用进程内（Feign/Hystrix） | 独立 Sidecar 进程 |
| 编程语言限制 | Java | 任何语言 |
| 配置方式 | 代码/注解 | 声明式（K8s CRD） |
| 服务发现 | Eureka/Consul/Nacos | K8s Service + CoreDNS |
| 熔断/限流 | Hystrix/Sentinel | Envoy（原生支持） |
| 路由控制 | Gateway | Envoy |
| 安全性 | Spring Security | mTLS 双向认证 |
| 运维复杂度 | 中 | 高（需要 K8s） |

**Istio 核心组件：**

```
Istio 架构：
┌─────────────────────────────────────────────────────────┐
│                      Control Plane                       │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │    Pilot     │  │   Citadel   │  │   Galley      │  │
│  │  (配置分发)   │  │  (安全管理)  │  │  (配置验证)   │  │
│  └──────────────┘  └─────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────┘
                           ↓ 分发配置
┌─────────────────────────────────────────────────────────┐
│                      Data Plane（Sidecar）                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │ Envoy Pod 1 │  │ Envoy Pod 2 │  │ Envoy Pod 3  │       │
│  │ [App][Side] │  │ [App][Side] │  │ [App][Side] │       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
└─────────────────────────────────────────────────────────┘
```

| 组件 | 作用 |
|------|------|
| Envoy | Sidecar 代理，负责流量拦截、路由、熔断、指标采集 |
| Pilot | 统一配置分发，将路由规则（VirtualService）下发到 Envoy |
| Citadel | 证书管理，实现 mTLS 双向认证 |
| Galley | 配置验证，校验用户编写的 Istio 配置 |
| Mixer | 策略检查和遥测数据收集（已逐步废弃） |

**Istio 的核心功能：**

```yaml
# 流量管理：VirtualService（路由规则）
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90
        - destination:
            host: order-service
            subset: v2
          weight: 10   # 10% 流量到 v2（灰度发布）
---
# 熔断配置：DestinationRule
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s   # 熔断实例剔除时间
```

**Spring Cloud + Istio 的融合部署：**

```
常见方案：
1. Spring Cloud 作为应用层，Istio 处理南北流量（外部 → Gateway → 服务）
2. Spring Cloud 内部用 Feign，Istio 处理服务间的东西流量
3. 纯 Istio：Spring Cloud Netflix 组件全部替换为 Istio

推荐方案：
- 新项目：直接用 Istio 管理流量，Spring Boot 仅作为业务容器
- 存量项目：Spring Cloud + Istio 共存，逐步迁移
```

**Istio 与 Spring Cloud 的关系总结：**

| 维度 | 关系 |
|------|------|
| 定位 | Istio 是基础设施，Spring Cloud 是应用框架 |
| 层级 | Istio 位于网络层，Spring Cloud 位于应用层 |
| 替代 | Istio 可以替代 Spring Cloud Netflix 的 Zuul/Hystrix/Feign |
| 共存 | Spring Cloud 可以保留服务注册/配置管理，Istio 处理流量和安全 |
| 趋势 | 云原生场景下，Istio（Envoy）正在逐步替代 Hystrix/Ribbon/Feign |

### 追问方向
- Istio 的 Sidecar 注入（自动/手动）和性能损耗如何？
- Istio 的 mTLS 是如何工作的？为什么不需要改应用代码？
- Istio 和 Linkerd（另一款 Service Mesh）相比有什么优劣？

### 避坑提示
- Istio 需要 Kubernetes 环境，纯虚拟机部署 Istio 非常复杂
- Sidecar 注入会增加网络延迟（每次请求多一跳），高并发场景需要压测验证
- Istio 版本更新快，生产环境建议锁定版本号，避免升级导致兼容性问题

---

> **下篇预告**：Spring Cloud + 微服务架构面试题（GateWay、Sentinel、Nacos、Feign、Seata、分布式事务、链路追踪、服务网格等）

---

## 微服务架构（25题）

# 微服务架构面试题 Part 2（25题）

---

## 第1题：微服务拆分原则，什么时候该拆，什么时候不该拆

### 核心答案

微服务拆分遵循**高内聚低耦合**原则，具体标准：

**该拆的信号：**
- 独立业务边界清晰，可独立部署发布
- 团队规模超过"2 pizza team"（6~10人），按业务线拆分
- 不同服务有不同资源需求（CPU/内存/存储差异大）
- 故障隔离要求高，单服务故障不能拖垮全局
- 技术栈差异大，需要独立演进（如Java订单服务 + Python推荐服务）

**不该拆的信号：**
- 业务边界模糊，领域模型不清晰（强行拆导致跨服务事务）
- 团队规模小（3人以下），拆分带来运维复杂度翻倍
- 团队没有DevOps能力，拆分后联调成本极高
- 性能敏感场景（服务间通信延迟不可接受）
- 初期业务不确定性高，拆分后频繁重构代价大

**拆分维度：** 按业务能力（Business Capability）或DDD的限界上下文（Bounded Context）拆分，避免按技术层次（Controller层/Service层）或数据库表数量切分。

### 追问方向
- DDD战术设计如何配合拆分？
- 如何处理遗留系统的拆分？（绞杀者模式）
- 拆分后数据库是共享还是独享？

### 避坑提示
- 别说"微服务比单体好所以要拆"，面试官会追问ROI
- 不要把"拆分"等同于"把项目拆成多个Maven模块"
- 提到数据库时别说共享数据库"挺好"，要能解释原因和取舍

---

## 第2题：服务注册与发现的全流程，从应用启动到被其他服务调用

### 核心答案

**全流程分为注册与发现两个阶段：**

**注册阶段（应用启动）：**
1. 服务实例启动，向注册中心（如Nacos/Eureka）发送心跳
2. 注册中心记录：`服务名 + IP + Port + 元数据`（权重、版本、健康状态）
3. 注册中心定时（通常30秒）探测健康状态，超过阈值剔除不可用实例

**发现阶段（Consumer调用Provider）：**
1. Consumer启动时从注册中心拉取全量服务实例列表（Pull模式）
2. 注册中心推送变更通知（Push模式，如Nacos的UDP推送）
3. Consumer本地维护实例列表，结合负载均衡策略（RoundRobin/Random/LeastActive）选择目标实例
4. 发起RPC/HTTP调用

**关键机制：**
- **心跳检测：** 注册中心定时检查服务可用性，不健康实例主动下线
- **本地缓存：** Consumer本地缓存服务列表，防止注册中心故障时完全不可用
- **元数据分组：** 通过namespace/group 实现环境隔离和逻辑分组

### 追问方向
- 注册中心挂了怎么办？服务还能调用吗？
- Nacos的CP/AP模式切换机制？
- 如何实现服务下线的灰度摘流量？

### 避坑提示
- 不要只说"启动时注册"，要覆盖健康检查和剔除机制
- 别说"每次调用都查注册中心"，要提到本地缓存
- Eureka和Nacos的区别要能说清楚

---

## 第3题：分布式事务的解决方案（2PC、TCC、可靠消息最终一致性、最大努力通知）

### 核心答案

**1. 2PC（两阶段提交）**

- **Prepare阶段：** TM向所有参与者发送Prepare请求，参与者锁定资源并写redo log，返回"就绪"
- **Commit阶段：** 所有参与者就绪后，TM发送Commit，提交真正生效
- **缺点：** 同步阻塞、单点TM、数据锁定时间过长、存在数据不一致风险（部分提交）

**2. TCC（Try-Confirm-Cancel）**

- **Try：** 预留资源（冻结库存、预扣金额）
- **Confirm：** 确认执行，使用预留资源
- **Cancel：** 释放预留资源
- 需要业务代码实现三个接口，适合强隔离性场景（银行转账）
- 缺点：业务侵入性大，Cancel接口要能处理空回滚和幂等

**3. 可靠消息最终一致性（事务消息）**

- 利用RocketMQ/Kafka的事务消息机制：
  1. Producer发送半消息（Half Message），本地事务执行
  2. 本地事务成功则Commit，失败则Rollback
  3. Consumer端保证幂等消费，定时对账确保最终一致
- 适合异步场景，最终一致性要求

**4. 最大努力通知**

- 主动方尽最大努力通知被动方，通过定期重试（固定间隔递增）
- 被动方提供查询接口供主动方核对状态
- 适合日志同步、系统回调等非核心场景

### 追问方向
- Seata的AT模式原理？和TCC区别？
- 为什么2PC无法完全保证数据一致性？
- 事务消息的Half Message本地事务失败怎么处理？

### 避坑提示
- 不要无脑推荐TCC，要结合业务场景（强一致性 vs 最终一致）
- 不要说2PC"很好"，它的同步阻塞是经典问题
- 能手写TCC的Try/Confirm/Cancel流程说明理解到位

---

## 第4题：什么是接口幂等性，哪些场景需要保证幂等，Token+Redis方案

### 核心答案

**幂等性：** 同一请求参数，多次执行与一次执行对系统状态的影响完全相同。

**需要保证幂等的场景：**
- 支付下单（重复扣款）
- 订单状态修改（重复状态流转）
- 库存扣减（超卖）
- 消息消费（重复消费）
- 前端重复点击/网络重试导致的重复提交

**天然幂等的操作：**
- SELECT：只读不修改
- UPDATE ... SET status = 'paid'（状态幂等，不依赖原值）
- DELETE（删除id存在则成功，id不存在也成功，某种意义上幂等）

**Token+Redis方案：**
1. 客户端发起操作前，先从后端获取全局唯一Token（UUID）
2. Token存入Redis，设置TTL（如5分钟）
3. 客户端提交请求时携带Token，后端用`SETNX(Token, 1)`原子操作检查并删除
4. 若SETNX返回false，说明已处理过，返回成功或直接拒绝

```
// 伪代码
String token = uuidGenerator.generate();
redis.setex(token, 300, "1"); // 生成Token
// 客户端携带token请求
Boolean success = redis.setnx("order:token:" + token, "1");
if (!success) return "重复请求";
// 业务处理
redis.del("order:token:" + token);
```

### 追问方向
- Token放在Header还是Body？Redis key的TTL设多长？
- 如果Redis挂了怎么办？
- 订单系统的幂等如何结合数据库唯一约束？

### 避坑提示
- 不要说"所有接口都要幂等"，要区分场景
- Token方案要强调SETNX的原子性，不要先查再删（两步操作非原子）
- 能提到数据库唯一索引作为兜底是加分项

---

## 第5题：CORS跨域的原理，JSONP和CORS的区别，Spring Cloud Gateway如何处理跨域

### 核心答案

**CORS（Cross-Origin Resource Sharing）原理：**

浏览器基于**同源策略**（Same-Origin Policy）限制跨域请求。CORS通过HTTP头实现跨域许可：

- **简单请求：** GET/HEAD/POST + 特定Content-Type，直接发送请求，服务器返回`Access-Control-Allow-Origin`
- **预检请求（Preflight）：** PUT/DELETE或复杂Header，浏览器先发`OPTIONS`预检，服务器确认后正式请求
- 关键响应头：
  - `Access-Control-Allow-Origin`：允许的源（`*`或具体域名）
  - `Access-Control-Allow-Methods`：允许的HTTP方法
  - `Access-Control-Allow-Headers`：允许的请求头
  - `Access-Control-Allow-Credentials`：是否允许携带Cookie

**JSONP vs CORS：**

| 维度 | JSONP | CORS |
|------|-------|------|
| 原理 | 利用script标签不受同源策略限制 | 基于HTTP头，标准W3C规范 |
| 方法 | 仅GET | 支持所有HTTP方法 |
| 错误处理 | 无法获取HTTP状态码 | 可获取完整响应状态 |
| 安全性 | 存在XSS注入风险 | 更安全，支持Authorization头 |
| 浏览器支持 | 老旧浏览器也支持 | IE10以下不支持 |

**Spring Cloud Gateway处理跨域：**

```yaml
# application.yml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://example.com"
            allowedMethods: GET,POST,PUT,DELETE
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

原理：Gateway作为全局过滤器，在响应头中追加CORS相关header，所有下游服务无需单独处理。

### 追问方向
- 简单请求和预检请求的判断条件？
- 为什么`Access-Control-Allow-Origin`不能设为`*`同时`allowCredentials=true`？
- OPTIONS请求在Gateway层如何处理以避免穿透到后端？

### 避坑提示
- 不要说JSONP"更好"，它只是历史兼容方案
- 要能说清楚为什么Gateway处理跨域比每个服务单独配置更优
- 提到`maxAge`配置（预检结果缓存时间）是加分项

---

## 第6题：API版本管理策略（URL版本、Header版本、媒体类型版本）

### 核心答案

**1. URL版本（Path Versioning）**
```
GET /api/v1/users/123
GET /api/v2/users/123
```
- 优点：直观、易调试、CDN缓存友好
- 缺点：URL结构变化，破坏REST美感
- 代表：Stripe API、GitHub API

**2. Header版本（Custom Header）**
```
GET /api/users/123
API-Version: 2024-01-01
```
- 优点：URL不变，语义清晰
- 缺点：调试不便（浏览器直接访问看不到效果），需要额外文档
- 代表：Microsoft Azure

**3. 媒体类型版本（Content Negotiation）**
```
GET /api/users/123
Accept: application/vnd.myapp.v2+json
```
- 优点：完全符合REST规范，URL不变
- 缺点：极度不直观，测试和调试困难
- 代表：GitHub API（同时支持Header和URL）

**实践建议：**
- 公开API（对外）用URL版本或Header版本
- 内部微服务间优先Header版本（减少URL复杂度）
- 版本策略要结合API网关做路由转发，Gateway按版本路由到不同服务实例

### 追问方向
- v1和v2服务可以同时运行吗？如何实现平滑过渡？
- 多版本共存时，旧版本何时下线？
- 如何在Gateway层统一做版本路由？

### 避坑提示
- 不要说"无所谓，用哪个都行"——要结合团队规模和技术栈分析
- 提到"蓝绿部署"或"金丝雀发布"配合版本管理是加分项
- 要知道URL版本是业界最常见方案

---

## 第7题：服务间通信的同步 vs 异步，消息队列在微服务中的应用

### 核心答案

**同步通信（Sync）：**
- 调用方阻塞等待响应，如HTTP REST、gRPC
- 优点：实时性强，流程直观，调试方便
- 缺点：调用链长时延迟累加，任何一个环节超时整体失败
- 适用场景：用户请求型的主流程（查订单、查库存）

**异步通信（Async）：**
- 调用方发送消息后立即返回，不阻塞
- 方式：消息队列（Kafka/RocketMQ/RabbitMQ）、事件驱动
- 优点：解耦、削峰填谷、容错性强
- 缺点：事务一致性问题（最终一致）、调试困难

**消息队列在微服务中的典型应用：**
1. **异步解耦：** 订单服务下单后发消息，库存服务、物流服务异步消费
2. **流量削峰：** 秒杀场景，先把请求写入MQ，后端慢慢消费
3. **事件驱动：** 用户注册发消息触发欢迎邮件、积分计算等下游
4. **数据同步：** 跨系统数据同步，通过消息保证最终一致性

**选型参考：**
- Kafka：高吞吐量、日志场景，适合数据管道
- RocketMQ：事务消息、金融级可靠性，阿里系
- RabbitMQ：中小规模，灵活路由（Exchange/Queue/RoutingKey）

### 追问方向
- 如何保证消息不丢失？（生产者确认、Broker持久化、消费者手动ACK）
- 消息重复消费怎么解决？（幂等消费 + 唯一消息ID）
- 消息积压如何处理？

### 避坑提示
- 不要说"异步一定比同步好"，要区分场景
- 能说清楚Kafka和RocketMQ在事务消息上的差异是加分项
- 提到"不要用MQ做同步查询"——这是常见误区

---

## 第8题：什么是Saga模式，对比TCC和2PC

### 核心答案

**Saga模式：** 将一个分布式事务拆分为多个本地事务，每个本地事务有对应的补偿操作（Compensation）。若某一步失败，则按反向顺序执行前面已成功步骤的补偿操作。

**两种编排方式：**
- **Choreography（事件驱动）：** 各服务通过事件协同，无中心编排者
- **Orchestration（编排器模式）：** 中央编排器定义执行顺序和补偿逻辑

**Saga vs TCC：**

| 维度 | Saga | TCC |
|------|------|-----|
| 资源锁定 | 不锁定资源（乐观），高并发友好 | Try阶段锁定资源（悲观） |
| 补偿方式 | 逆向补偿（Compensate） | 预留资源Confirm/Cancel |
| 数据一致性 | 最终一致（非强一致） | 强一致（资源预留） |
| 业务侵入 | 较小（只需写补偿逻辑） | 较大（需实现三阶段接口） |
| 适用场景 | 长流程、跨服务多、并发高 | 强一致性、短流程 |

**Saga vs 2PC：**

| 维度 | Saga | 2PC |
|------|------|-----|
| 阻塞 | 非阻塞（各本地事务独立执行） | 同步阻塞（Prepare阶段锁定资源） |
| 协调者 | 可分布式（编排器） | 单点TM |
| 数据一致性 | 最终一致 | 强一致 |
| 故障恢复 | 补偿逻辑复杂 | 依赖日志恢复 |

### 追问方向
- Saga的补偿失败怎么处理？（重试、人工介入、发送死信队列）
- Saga如何保证各子事务的执行顺序？
- Seata的Saga模式如何实现？

### 避坑提示
- 不要说"Saga比TCC好"——适用场景不同
- 要理解Saga是"乐观补偿"，TCC是"悲观预留"
- 提到"长事务"场景Saga更合适是加分项

---

## 第9题：分布式锁的实现方案（Redis分布式锁、ZooKeeper、数据库），Redis分布式锁的原子性保证

### 核心答案

**1. Redis分布式锁（Redisson实现）：**
```java
RLock lock = redisson.getLock("order:123");
try {
    // 等待时间、持有时间、时间单位
    boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
    if (acquired) {
        // 业务逻辑
    }
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```
- `SET key value NX PX 30000`：原子性保证（NX + PX 合一）
- value用UUID防误删（先比较value再删除）
- 看门狗机制（Watchdog）自动续期

**2. ZooKeeper分布式锁：**
- 临时有序节点 + Watch机制
- 最小节点获得锁，其他节点监听前一个节点
- 优点：可靠性高（ZK保证CP）
- 缺点：性能不如Redis，需要维护ZK集群

**3. 数据库分布式锁：**
- 表中插入唯一索引记录
- 优点：实现简单
- 缺点：性能差、无法解决死锁（依赖超时）

**Redis分布式锁的原子性保证：**
- 不使用`SET + EXPIRE`（非原子）
- 使用`SET key value NX PX milliseconds`一条命令完成加锁
- 释放锁用Lua脚本保证"检查+删除"原子性：
```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 追问方向
- Redis主从切换时分布式锁会失效吗？（RedLock可以但实现复杂）
- 如何防止锁过期但业务还在执行？（看门狗续期）
- Redis锁的value为什么要用UUID？

### 避坑提示
- 不要说`SETNX + EXPIRE`是原子的，这是经典错误
- 不要在生产环境用数据库锁做高性能场景
- 能提到Redisson的看门狗机制说明理解到位

---

## 第10题：接口超时如何处理，Hystrix/Sentinel超时降级的配置

### 核心答案

**超时处理策略：**
1. **设置合理超时时间：** 不要无脑设30秒，结合业务SLA和下游性能
2. **超时后快速失败（Fail Fast）：** 避免资源占用
3. **降级处理（Fallback）：** 返回兜底数据或友好提示
4. **重试（谨慎）：** 读操作可重试，写操作一般不重试（幂等问题）

**Hystrix配置（已停止维护，Spring Cloud 2020版后不再推荐）：**
```java
@HystrixCommand(
    commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000"),
        @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "10")
    },
    fallbackMethod = "fallback"
)
public String getUserInfo() {
    return restTemplate.getForObject("http://user-service/user/1", String.class);
}

public String fallback() {
    return "服务暂不可用，请稍后再试";
}
```

**Sentinel配置（推荐）：**
```java
// 定义资源
@SentinelResource(value = "getUserInfo",
    fallback = "fallbackHandler",
    blockHandler = "blockHandler")
public String getUserInfo() {
    return restTemplate.getForObject("http://user-service/user/1", String.class);
}

// Fallback处理（业务异常、超时）
public String fallbackHandler(Long id, Throwable ex) {
    return "服务暂不可用";
}

// BlockHandler处理（限流、降级）
public String blockHandler(Long id, BlockException ex) {
    return "请求过于频繁";
}
```

```yaml
# Sentinel规则配置
flow:
  resource: getUserInfo
  count: 100
  grade: 1  # QPS
  limitApp: default
```

### 追问方向
- Hystrix和Sentinel的核心区别？（流量控制 vs 熔断降级）
- 如何确定超时时间？（依赖服务P99响应时间 × 1.5）
- 超时降级和限流降级如何区分？

### 避坑提示
- 不要说"超时设越长越安全"，会拖垮调用方资源
- Hystrix已过时，面试说Hystrix要同时提到Sentinel
- 能提到P99响应时间而不是平均响应时间是加分项

---

## 第11题：微服务的健康检查机制，健康检查失败如何处理

### 核心答案

**健康检查机制：**

**1. 进程自检（Actuator Health Indicator）：**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # K8s liveness/readiness探头
```

Spring Boot Actuator提供健康检查端点`/actuator/health`，检查DB连接、Redis、磁盘等。

**2. 注册中心心跳检测：**
- 服务定期向注册中心发送心跳（通常15秒/次）
- 注册中心超过3次未收到心跳，标记为不健康并剔除

**3. K8s的Liveness和Readiness：**
- `livenessProbe`：存活探针，失败会重启容器（`kubectl rollout restart`）
- `readinessProbe`：就绪探针，失败从Service Endpoints摘除，不再接收流量
- 探针类型：HTTP GET（`/actuator/health/liveness`）、TCP Socket、Exec命令

**健康检查失败后的处理流程：**
1. 注册中心标记为UNHEALTHY，停止向Consumer分发该实例流量
2. K8s ReadinessProbe失败：从Service摘除（停止接收新流量），但Pod不重启
3. K8s LivenessProbe失败：重启容器
4. 如果连续失败超过阈值，触发告警（Prometheus AlertManager）

### 追问方向
- 健康检查的检查频率和超时时间如何设置？
- 如何区分"服务过载"和"服务真正宕机"？
- 注册中心的剔除机制和K8s的探针如何配合？

### 避坑提示
- 不要只说"心跳检测"，要区分进程健康和网络可达
- 要理解UNHEALTHY不等于Pod要重启（那是LivenessProbe的事）
- 能提到Spring Boot的`HealthIndicator`自定义实现是加分项

---

## 第12题：服务限流的算法：计数器、滑动窗口、令牌桶、漏桶算法

### 核心答案

**1. 计数器（Fixed Window）：**
- 固定时间窗口内计数，超限则拒绝
- 简单实现，但存在"临界突变"问题（窗口边界QPS翻倍）
- 适用于：对精度要求不高的场景

**2. 滑动窗口（Sliding Window）：**
- 基于时间窗口滚动，将窗口细分为多个小桶（bucket）
- 精度比计数器高，Redis的ZSet可实现
- 解决了临界问题，但实现复杂度稍高

**3. 令牌桶（Token Bucket）：**
- 以固定速率往桶里放令牌，桶满则丢弃
- 请求来时从桶取令牌，取到则通过，否则拒绝/等待
- **允许突发流量**（桶满时一次性取出多个令牌）
- 代表实现：Guava RateLimiter

```java
RateLimiter limiter = RateLimiter.create(100); // QPS=100
limiter.acquire(); // 获取令牌
```

**4. 漏桶（Leaky Bucket）：**
- 请求以任意速率进入桶，桶以固定速率漏出
- 平滑输出，**不允许突发流量**（强行整形）
- 实现：FIFO队列 + 固定速率消费线程

**对比：**

| 算法 | 精确度 | 突发流量 | 实现难度 |
|------|--------|----------|----------|
| 计数器 | 较低 | 不允许（临界问题） | 简单 |
| 滑动窗口 | 较高 | 不允许 | 中等 |
| 令牌桶 | 高 | 允许 | 中等 |
| 漏桶 | 高 | 不允许 | 简单 |

### 追问方向
- 令牌桶和漏桶的选择场景？
- 如何实现分布式限流？（Redis + Lua脚本）
- Sentinel用的是哪种算法？

### 避坑提示
- 不要说"漏桶比令牌桶好"——场景不同，令牌桶更常用
- 要知道Guava RateLimiter是令牌桶实现
- 能提到Sentinel基于滑动窗口计算器是加分项

---

## 第13题：微服务网关的职责，为什么需要网关，Gateway在架构中的位置

### 核心答案

**网关的核心职责：**
1. **统一入口：** 所有客户端请求从网关进入，不用暴露每个微服务地址
2. **路由转发：** 根据URL路径、Header等条件动态路由到下游服务
3. **跨域处理：** 统一处理CORS，避免每个服务重复配置
4. **限流熔断：** 流量控制、熔断降级保护后端
5. **认证鉴权：** JWT Token验证、黑白名单
6. **日志监控：** 统一埋点traceId、记录请求日志
7. **协议转换：** HTTP → Dubbo/gRPC协议转换

**为什么需要网关：**
- 避免后端服务直接暴露在公网
- 统一安全策略（认证、限流）不用在每个服务重复实现
- 减少客户端复杂度（不需要维护服务地址清单）

**Gateway在架构中的位置：**
```
Client → Gateway → 防火墙/负载均衡 → 微服务集群
```

所有流量必须经过网关，网关是架构中的"单点入口"，也是全局策略的"切面层"。

**Spring Cloud Gateway核心配置：**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1  # 去掉前缀
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```

### 追问方向
- Gateway如何实现负载均衡？（结合Ribbon/LB）
- Gateway和Nginx的区别？何时用Gateway何时用Nginx？
- Gateway如何做动态路由？

### 避坑提示
- 不要说"网关是可选的"——生产环境必须有
- 要区分Spring Cloud Gateway（Java）vs Kong vs Envoy等不同网关
- 网关不做业务逻辑，只做路由和策略

---

## 第14题：如何设计一个高可用的微服务架构，避免单点故障

### 核心答案

**高可用设计原则：**

**1. 消除单点（No Single Point of Failure）：**
- 每个组件至少2个实例（服务、注册中心、数据库、MQ）
- 注册中心：Nacos集群（至少3节点）
- 数据库：主从复制 + 读写分离
- MQ：集群部署

**2. 流量高可用：**
- 网关层：多实例 + 负载均衡
- 服务层：多实例注册到注册中心，Consumer自动感知
- 数据库：主从切换，连接池熔断

**3. 降级与熔断：**
- 配置Hystrix/Sentinel熔断规则
- 非核心服务故障时自动降级，保留核心链路

**4. 超时与重试：**
- 合理设置超时时间
- 重试策略（指数退避 + 最大重试次数）防止雪崩
- Ribbon重试配置：`MaxAutoRetriesNextServer=2`

**5. 灾备与多活：**
- 同城双活：主机房+备机房
- 异地多活：多地域部署，数据同步

**6. 监控与告警：**
- Prometheus + Grafana监控QPS/响应时间/错误率
- 告警阈值：CPU > 80%、P99延迟 > 500ms
- 日志聚合：ELK/Graylog

**架构图视角：**
```
            LB（SLB）
              │
    ┌─────────┼─────────┐
  Gateway   Gateway   Gateway
    │         │          │
  User      Order      Pay     ← 各服务多实例
    │         │          │
  MySQL     MySQL     MQ       ← 主从/集群
    │         │          │
  Nacos（3节点集群）
```

### 追问方向
- 注册中心挂了怎么办？服务还能互相调用吗？
- 如何做全链路的故障演练？（Chaos Engineering）
- 什么场景需要同城双活vs异地多活？

### 避坑提示
- 不要说"加机器就高可用了"，要说明架构层面的冗余设计
- 要理解CAP定理：高可用 + 分区容错 vs 一致性
- 提到"监控+告警+快速恢复"闭环是加分项

---

## 第15题：什么是服务网格（Service Mesh），Sidecar模式，Envoy

### 核心答案

**服务网格（Service Mesh）：**
- 基础设施层，用于处理服务间通信
- 将网络通信、安全、监控等能力从应用代码下沉到基础设施
- 核心目标：**让开发者只关注业务逻辑，不关注通信基础设施**

**Sidecar模式（边车模式）：**
- 为每个服务实例部署一个独立代理（Sidecar）
- 服务所有对外通信都经过Sidecar
- Sidecar负责：路由、限流、熔断、认证、追踪
- 服务本身不感知这些横切关注点

```
服务A → Sidecar A → 网络 → Sidecar B → 服务B
```

**Envoy：**
- 开源高性能代理，Service Mesh的数据平面（Data Plane）
- 支持动态配置（LDS/RDS/CDS/EDS）、透明流量拦截
- 提供指标（StatsD/Prometheus）、分布式追踪（Zipkin/Jaeger）
- 被Istio采用作为默认Sidecar

**Istio：**
- 控制平面（Control Plane），管理Envoy Sidecar
- Pilot：配置路由规则
- Mixer：策略检查 + Telemetry收集
- Citadel：安全（mTLS证书管理）

### 追问方向
- Service Mesh和Spring Cloud的区别？能否共存？
- Sidecar模式的性能开销如何？（延迟 + 资源消耗）
- Envoy的xDS协议是什么？

### 避坑提示
- 不要说"Service Mesh是微服务的替代品"
- 要理解数据平面和控制平面的区别
- 能提到Envoy的热加载配置（无需重启）是加分项

---

## 第16题：配置中心选型：Apollo vs Nacos vs Spring Cloud Config

### 核心答案

**1. Apollo（阿波罗）：**
- 携程开源，支持多环境（DEV/FAT/UAT/PRO）
- 配置管理功能完善：灰度发布、配置回滚、权限管理、审计日志
- 客户端：Java/Go/Python/.NET多语言
- 支持命名空间（Namespace）隔离
- **优点：** 功能最完善，适合大规模配置管理
- **缺点：** 部署复杂，学习成本高

**2. Nacos：**
- 阿里开源，同时支持**服务发现 + 配置管理**
- 支持AP（最终一致）和CP（强一致）模式切换
- 部署简单，中文文档友好
- 集成Spring Cloud Alibaba生态
- **优点：** 二合一（注册中心+配置中心），轻量易用
- **缺点：** 权限管理功能较弱

**3. Spring Cloud Config：**
- Spring官方配置中心
- 支持Git后端（配置版本化）
- 配合Bus（消息总线）实现配置刷新
- **优点：** 天然集成Spring Cloud，适合Spring体系
- **缺点：** 不支持配置变更推送（需配合Bus）、多语言支持差

**选型建议：**
- Spring Cloud体系 + 快速启动 → Nacos
- 企业级配置管理 + 多环境 + 灰度 → Apollo
- 纯Spring项目 + Git工作流 → Config + Bus

### 追问方向
- Nacos如何实现配置变更的实时推送？（长轮询 + UDP）
- Apollo的多环境隔离是如何实现的？
- 配置中心挂了，应用能正常启动吗？

### 避坑提示
- 不要说"Nacos功能最全所以最好"——Apollo的权限管理更强
- 要理解Nacos的服务发现和配置管理是独立的
- 能提到Spring Cloud Config的Git后端配置版本化是加分项

---

## 第17题：分布式追踪中，traceId如何生成，如何在MDC中传递

### 核心答案

**traceId生成规则：**
- 格式：64位或128位全局唯一ID
- 常用算法：
  - Snowflake（时间戳 + 机器ID + 序列号）
  - UUID（简单但不友好，不利于链路查询）
  - CAT的MessageId（业务相关）
- 通常在网关层生成（如Sentinel或Zuul Filter），注入到Header中

**MDC（Mapped Diagnostic Context）传递：**
- MDC是Logback/Log4j的线程级上下文存储
- 核心思路：filter拦截请求，从Header提取traceId放入MDC，所有日志自动携带

**全链路传递流程：**
```
请求进入网关
  → 生成traceId（UUID/Snowflake）
  → 放入MDC（Logback MDC.insertProp("traceId", traceId)）
  → 通过HTTP Header传递给下游（X-Trace-Id: traceId）
  → 下游服务Filter从Header提取traceId，放入当前线程MDC
  → 日志输出自动包含traceId
  → 异步线程：InheritableThreadLocal 或 TransmittableThreadLocal
```

**实现示例（Gateway）：**
```java
@Component
public class TraceFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String traceId = UUID.randomUUID().toString().replace("-", "");
        exchange.getAttributes().put("traceId", traceId);
        
        // 通过Header传递
        ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
            .header("X-Trace-Id", traceId)
            .build();
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
}
```

**MDC配置（Logback）：**
```xml
<pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] [traceId:%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
```

### 追问方向
- ThreadLocal在异步场景下如何传递？（TransmittableThreadLocal）
- traceId如何在MQ消息中传递？（消息Header）
- 全链路追踪系统（SkyWalking/Jaeger）的数据采集原理？

### 避坑提示
- 不要只说"用UUID生成"，要能说清整条链路的传递机制
- 要知道异步线程池场景下ThreadLocal会丢失（需要InheritableThreadLocal或TTL）
- 能提到MDC是Logback特有、ThreadLocal是Java原生是加分项

---

## 第18题：什么是断路器，状态机转换（CLOSED、OPEN、HALF_OPEN）

### 核心答案

**断路器（Circuit Breaker）模式：**
- 当下游服务故障时，切断调用链，防止故障蔓延导致雪崩
- 类似电路保险丝：电流过大时保险丝熔断，保护电器

**状态机三态转换：**

```
CLOSED（闭合）正常 → OPEN（断开）故障触发
    ↑                      ↓
    │                      ↓（半开定时器）
    └────── HALF_OPEN ←────┘
         （半开，恢复探测）
```

**1. CLOSED（闭合状态）：**
- 断路器关闭，所有请求正常通过
- 统计请求成功/失败数量和比例
- 当失败率达到阈值，触发断路器OPEN

**2. OPEN（断开状态）：**
- 所有请求直接返回降级响应（Fail Fast）
- 不向故障服务发起实际调用
- 经过一个"熔断窗口"（如30秒）后，转换到HALF_OPEN

**3. HALF_OPEN（半开状态）：**
- 允许有限数量的探测请求通过（如1个）
- 如果探测成功 → 恢复到CLOSED
- 如果探测失败 → 重新回到OPEN，重新计时

**Sentinel配置示例：**
```java
@SentinelResource(value = "getUser",
    blockHandler = "blockHandler")
public String getUser(Long id) { ... }

// 熔断规则
DegradeRule rule = new DegradeRule("getUser")
    .setGrade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType())
    .setCount(0.5)  // 比例阈值
    .setSlowRatioThreshold(1000)  // 慢调用阈值(ms)
    .setMinRequestAmount(5)  // 最小请求数
    .setStatInterval(10)  // 统计窗口(秒)
    .setTimeWindow(10);  // 熔断窗口时长(秒)
```

### 追问方向
- 熔断和降级的区别？
- 失败率阈值怎么定？（基于历史P99响应时间）
- 为什么需要HALF_OPEN状态？（避免故障恢复时的瞬时冲击）

### 避坑提示
- 不要把"熔断"和"限流"混为一谈
- 要理解CLOSED→OPEN的触发条件和时间窗口的关系
- 能说清Sentinel和Hystrix熔断策略的差异是加分项

---

## 第19题：线程池隔离 vs 信号量隔离，线程池调优

### 核心答案

**线程池隔离：**
- 每个依赖服务分配独立的线程池，互不影响
- 线程池隔离后，一个服务慢不会占满所有线程
- 但线程切换有开销，适合并发量大、延迟敏感场景

**信号量隔离：**
- 使用JDK的`Semaphore`（计数器），不创建真实线程
- 请求竞争信号量许可证，获得到执行，否则快速失败
- 开销极低，适合轻量级、并发极高的场景
- 但无法区分任务优先级，不适合慢调用

**对比：**

| 维度 | 线程池隔离 | 信号量隔离 |
|------|-----------|-----------|
| 线程创建 | 真实线程池 | 无线程（Semaphore） |
| 开销 | 线程创建/切换开销 | 极低 |
| 隔离性 | 强（独立线程池） | 一般（共享线程） |
| 支持异步 | 支持（可返回Future） | 不支持 |
| 适用场景 | 慢调用、低并发 | 快调用、高并发 |

**线程池调优参数：**
```java
// Hystrix线程池配置
@HystrixCommand(
    threadPoolProperties = {
        @HystrixProperty(name = "coreSize", value = "20"),        // 核心线程数
        @HystrixProperty(name = "maxQueueSize", value = "100"),   // 队列长度
        @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"), // 空闲回收
        @HystrixProperty(name = "queueSizeRejectionThreshold", value = "80") // 队列满后阈值
    }
)
```

**调优策略：**
- `coreSize`：参考每秒请求数和平均响应时间（throughput = QPS × RT）
- `maxQueueSize`：队列缓冲能力，-1用SynchronousQueue（无缓冲）
- 不要盲目增大线程池，过大反而增加上下文切换开销

### 追问方向
- 线程池隔离下如何处理返回值？（Future / RxJava）
- 线程池的拒绝策略如何选？（CallerRunsPolicy vs AbortPolicy）
- 信号量隔离如何实现超时控制？

### 避坑提示
- 不要说"线程池隔离一定比信号量好"
- 要知道Hystrix默认用线程池隔离，Sentinel用信号量隔离
- 提到"根据下游服务特性选择隔离策略"是加分项

---

## 第20题：为什么需要分布式事务，MySQL的ACID能否保证跨服务

### 核心答案

**为什么需要分布式事务：**

单体系统中，所有操作在一个数据库事务内，ACID由数据库本地事务保证。

微服务架构下，一次业务操作可能涉及多个服务：
- 下单服务：插入订单记录
- 库存服务：扣减库存
- 支付服务：扣减余额
- 积分服务：增加积分

这4个操作跨越4个服务进程、4个数据库实例，无法用本地事务保证一致性。

**MySQL的ACID能否保证跨服务：**

**不能。** MySQL的ACID事务只能保证**单个数据库实例内**的ACID特性：
- **Atomicity（原子性）：** 依赖数据库Redo Log，只作用于本地事务
- **Consistency（一致性）：** 数据库约束只作用于本地表
- **Isolation（隔离性）：** MVCC只作用于本地连接
- **Durability（持久性）：** Redo Log只作用于本地磁盘

跨服务场景下，需要**分布式事务协议**（2PC/TCC/Saga）来协调多个参与者的提交和回滚。

**分布式事务的挑战：**
- 跨网络通信，可能出现消息丢失、网络分区
- 各节点无法感知彼此的锁状态
- 协调者本身可能是单点故障

### 追问方向
- CAP定理如何理解？分布式事务选CP还是AP？
- Seata的AT模式是什么？和MT模式区别？
- 为什么2PC在分布式环境下不完美但仍在用？

### 避坑提示
- 不要说"MySQL支持分布式事务"，这指的是XA协议（2PC），不是本地ACID
- 要区分"分布式事务协议"和"数据库本地事务"
- 能提到Seata是当前主流分布式事务解决方案是加分项

---

## 第21题：什么是幂等性，SELECT和UPDATE天然幂等的场景

### 核心答案

**幂等性定义：** 对同一资源的多次相同请求，其结果与一次请求的结果相同。

**天然幂等的读操作：**
- `SELECT`：只读取数据，不修改，无任何副作用
- `UPDATE ... SET status = 'CANCELLED' WHERE id = 1`：状态设置为一个固定值，无论执行多少次结果一样
- `UPDATE ... SET count = 100 WHERE id = 1`：直接赋值，不是增量操作
- `DELETE ... WHERE id = 1`：删除已删除的资源，返回"成功"（视具体实现而定）

**非幂等的写操作：**
- `UPDATE ... SET count = count - 1`：扣减库存，执行多次结果不同（超卖）
- `UPDATE ... SET balance = balance - 100`：扣款，执行多次结果不同
- `INSERT`：重复插入会产生多条记录

**天然幂等的关键判断：**
- 操作结果是否依赖当前状态？
- 是否会产生副作用（Side Effect）？

**实际工程中的幂等保证：**
- 唯一索引 + INSERT防重
- 状态机 + WHERE条件防重
- 分布式锁 + 去重表

```sql
-- 支付幂等示例：使用唯一键 + 幂等Key
INSERT INTO payment_log (idempotency_key, order_id, amount, status)
VALUES ('order_123_pay_001', 'order_123', 100, 'PAID')
ON DUPLICATE KEY UPDATE status = status; -- 已存在则忽略
```

### 追问方向
- 为什么幂等性在MQ消费中尤为重要？
- 全局唯一幂等Key的设计？
- 前端防重复提交和后端幂等如何配合？

### 避坑提示
- 不要说"SELECT也是修改数据"
- UPDATE时区分"赋值型"和"运算型"是加分项
- 能提到"先查再改"不是原子操作，不算幂等

---

## 第22题：服务降级演练，如何进行全链路压测

### 核心答案

**服务降级演练：**

**1. 手动触发降级（功能演练）：**
- 关闭非核心服务（如推荐服务、积分服务）
- 验证核心链路（下单+支付+物流）仍可用
- 检查降级后的系统行为（返回兜底数据/友好提示）

**2. 自动化降级（Chaos Engineering）：**
- 使用ChaosBlade/Goreplay注入故障（延迟、异常、宕机）
- 验证熔断器是否触发、降级策略是否生效

**3. 降级策略配置：**
```java
// Sentinel降级规则
DegradeRule rule = new DegradeRule("getUserInfo")
    .setGrade(CircuitBreakerStrategy.ERROR_COUNT.getType())
    .setCount(5)  // 5次错误触发降级
    .setTimeWindow(10);  // 10秒后恢复
```

**全链路压测：**

**核心挑战：**
- 生产环境真实压测，不能污染真实数据
- 需要流量标记（traceId标记压测流量）
- 下游依赖需要Mock或降级

**压测步骤：**

1. **压测流量标记：** 请求头携带`X-PTEST-ID`或类似标识
2. **网关识别：** Gateway识别压测流量，打标签透传到下游
3. **数据隔离：**
   - 写入压测表（ptest_xxx）而非真实表
   - 或使用影子库（shadow_db）策略
4. **依赖Mock：** 对非核心依赖（短信、推送）Mock处理
5. **监控告警：** 实时监控QPS/延迟/错误率/P999
6. **流量切换：** 从正常流量逐步加压到目标QPS

**压测工具：**
- JMeter：批量发请求，适合HTTP
- wrk2：固定QPS压测
- Gatling：脚本化压测场景

### 追问方向
- 压测数据如何脱敏？
- 如何判断系统性能瓶颈？（CPU/内存/IO/连接池）
- 压测结果的指标如何定义SLA？

### 避坑提示
- 不要说"直接生产环境压测"，数据隔离是核心问题
- 要知道压测前必须做容量规划，不能盲目加压
- 能提到"全链路灰度"配合压测是加分项

---

## 第23题：Spring Cloud Alibaba生态在国内的优势

### 核心答案

**Spring Cloud Alibaba（SCA）核心组件：**

| 组件 | 功能 | 对应Spring Cloud组件 |
|------|------|---------------------|
| Nacos | 注册中心+配置中心 | Eureka + Config |
| Sentinel | 流控降级熔断 | Hystrix |
| Seata | 分布式事务 | 无（自研） |
| Dubbo | RPC框架 | OpenFeign |
| RocketMQ | 消息队列 | 无（Kafka备选） |
| Gateway | API网关 | Spring Cloud Gateway |
| Seata | 分布式事务 | 无 |

**国内优势：**

1. **中文社区活跃：** 文档完善，B站/掘金有大量实践分享
2. **全家桶开箱即用：** 注册中心+配置中心二合一，比Eureka+Config组合更轻量
3. **国产化适配：** 适配阿里云/华为云/腾讯云等国内云厂商
4. **Sentinel更现代：** Hystrix已停止维护，Sentinel功能更完善（流量控制、系统自适应、黑白名单）
5. **Seata生态完善：** AT/TCC/Saga/XA多种模式，Seata社区活跃度超过其他分布式事务方案
6. **RocketMQ原生集成：** 国内业务场景MQ是刚需，RocketMQ在事务消息上有优势
7. **国内大厂背书：** 淘宝/双十一验证过

**对比Spring Cloud Netflix：**
- Netflix Eureka 2.x胎死腹中，Eureka 1.x不再维护
- Hystrix停止维护，Spring Cloud官方推荐Sentinel
- Config + Bus复杂度高，Nacos一站式解决

### 追问方向
- Nacos和Sentinel的生产部署注意事项？
- Seata的AT模式对业务代码有侵入吗？
- Dubbo和OpenFeign如何选型？

### 避坑提示
- 不要说"Spring Cloud Alibaba可以完全替代Spring Cloud"
- 要知道某些场景Spring Cloud原生组件仍更合适（如对接Netflix生态）
- 能提到"国内云原生落地选SCA"是加分项

---

## 第24题：Dubbo和Spring Cloud的区别，Dubbo的RPC序列化协议

### 核心答案

**Dubbo vs Spring Cloud：**

| 维度 | Dubbo | Spring Cloud |
|------|-------|--------------|
| 通信协议 | Dubbo协议（RPC），默认TCP | HTTP REST |
| 性能 | 高（二进制序列化，网络开销小） | 较低（JSON序列化） |
| 耦合性 | 强依赖（Provider-Consumer接口需共享） | 松耦合（HTTP JSON弱依赖） |
| 生态 | 通信框架，生态靠扩展 | 全家桶（600+项目，含安全/配置/链路追踪） |
| 服务发现 | ZooKeeper/Nacos/Redis | Eureka/Nacos/Consul |
| 适用场景 | 内部微服务，高性能要求 | 面向外部API，多语言异构 |

**Dubbo协议工作原理：**
1. Provider暴露服务接口，Consumer通过代理（Proxy）调用
2. 代理层将调用序列化（通过Filter链），发送到Dubbo协议栈
3. 基于TCP的NIO（非阻塞IO），单连接多请求复用
4. Consumer端负载均衡（Random/RoundRobin/LeastActive）

**Dubbo序列化协议：**
| 协议 | 特点 |
|------|------|
| Hessian | 二进制序列化，性能好，跨语言 |
| Dubbo | 阿里巴巴自研二进制协议 |
| JSON | 可读性好，性能差 |
| Kryo | Java原生，性能好，不支持跨语言 |
| FST | 快速序列化，性能接近Kryo |

```xml
<!-- Dubbo协议配置 -->
<dubbo:protocol name="dubbo" port="20880" serialization="hessian2"/>
```

**选型建议：**
- 高并发内部服务 → Dubbo
- 面向外部/多语言/B2C场景 → Spring Cloud/OpenFeign

### 追问方向
- Dubbo的Filter链是什么？有哪些内置Filter？
- Dubbo的负载均衡策略有哪些？如何配置？
- 为什么Dubbo性能比HTTP REST高？

### 避坑提示
- 不要无脑推荐Dubbo，要说明"内部高并发场景"这个前提
- 要知道Dubbo接口耦合是双刃剑（类型安全但版本管理复杂）
- 能提到gRPC作为高性能替代方案是加分项

---

## 第25题：未来微服务架构演进方向：Serverless、Service Mesh、网格计算

### 核心答案

**1. Serverless（无服务器架构）：**
- 开发者不管理服务器，只编写函数（Function）
- 云厂商负责扩缩容、运维、计费（按调用次数/执行时长）
- 代表：AWS Lambda、阿里云函数计算、华为云FunctionGraph
- **优势：** 极致弹性、零运维、按需付费
- **挑战：** 冷启动延迟、厂商锁定、本地调试困难、长时间任务不友好
- **适用场景：** 事件驱动型任务（图片处理、消息处理）、突发流量处理

**2. Service Mesh（服务网格）：**
- 数据平面和控制平面分离
- Sidecar代理所有流量（如Envoy/Istio）
- 开发者专注业务逻辑，通信基础设施下沉到Mesh层
- **优势：** 多语言支持、细粒度流量管理、可观测性内置
- **趋势：** Istio社区演进、轻量化Mesh（Linkerd）、eBPF + Mesh融合

**3. 网格计算（Grid Computing）：**
- 将计算任务分布到大量计算节点
- 节点协同完成超大规模计算（如科学计算、AI训练）
- 代表：Hadoop/YARN、Spark、Kubernetes联邦（Federation）
- **趋势：** 云原生与高性能计算融合（HPC on K8s）

**三者关系：**
```
单体 → 微服务 → Service Mesh（通信基础设施下沉）
微服务 → Serverless（极致抽象，函数即服务）
Service Mesh + Serverless → 融合（Mesh管理函数间通信）
```

**架构演进路径：**
1. 当前：Spring Cloud / Dubbo 微服务
2. 中期：引入Service Mesh（Istio/Linkerd），减轻业务代码负担
3. 远期：部分场景迁移到Serverless，非核心业务函数化

### 追问方向
- Serverless和FaaS的关系？
- Service Mesh 2.0有哪些新特性？
- 如何评估业务是否适合Serverless改造？

### 避坑提示
- 不要说"Serverless会取代微服务"——它们是互补关系
- 不要说"Service Mesh是微服务的银弹"——增加复杂度和运维成本
- 能提到"渐进式演进"而非"推翻重来"是加分项

---

*文档版本：Part 2 - 微服务架构进阶*
*共计25题，覆盖微服务核心架构、分布式事务、弹性机制、可观测性等核心领域*
