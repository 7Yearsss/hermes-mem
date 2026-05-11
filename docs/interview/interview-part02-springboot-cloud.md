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

> **下篇预告**：Spring Cloud + 微服务架构面试题（GateWay、Sentinel、Nacos、Feign、Seata、分布式事务、链路追踪、服务网格等）
