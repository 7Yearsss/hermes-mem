# Maven/Gradle 构建工具面试题

---

## 1. Maven 坐标是什么？各元素的作用是什么？

**题目**：Maven 坐标（Coordinates）由哪些元素组成？`groupId`、`artifactId`、`version`、`packaging` 各自的作用是什么？为什么坐标能唯一标识一个构件？

**核心答案**：

Maven 坐标是 Maven 仓库中定位任意构件的唯一地址，由四个元素构成：

- **groupId**：组织或项目的命名空间，通常是反向域名，如 `com.alibaba`、`org.springframework`。作用类似 Java 包名，用于对构件进行逻辑分组。
- **artifactId**：构件在 `groupId` 内的唯一标识名称，即项目的实际模块名或 JAR 包名，如 `druid`、`spring-core`。
- **version**：构件的版本号，支持语义化版本（`1.0.0-SNAPSHOT`、`2.3.1.RELEASE`）和快照版本（`SNAPSHOT`）。
- **packaging**：打包类型，默认为 `jar`，还可以是 `war`、`pom`、`ear`、`maven-plugin` 等。它决定了构建产物的类型。

四个元素组合在一起（`groupId:artifactId:packaging:version`）构成构件的唯一标识坐标，类似于 Maven 世界的"坐标系统"。

**追问方向**：

- `SNAPSHOT` 版本和正式版本有什么区别？Maven 如何处理 SNAPSHOT 的更新？
- 如果两个依赖使用了相同的 `groupId:artifactId:version` 但 `packaging` 不同（比如一个是 `jar`，一个是 `war`），Maven 如何处理？
- Maven 坐标中的 `classifier` 是什么？什么场景下会用到？

**避坑提示**：

- 不要把 `packaging` 和文件扩展名混为一谈，`packaging` 是构建类型，不是文件后缀。
- `groupId` 不是必须和 Java 包名一致，但实践中强烈建议保持一致，否则 IDE 的包名组织会很混乱。

---

## 2. Maven 仓库有哪几种？各自的职责是什么？

**题目**：请描述 Maven 仓库的分类体系——本地仓库、中央仓库、私服，它们各自的位置、作用和工作流程是怎样的？

**核心答案**：

Maven 仓库分为三级：

- **本地仓库（Local Repository）**：位于用户机器上，路径默认为 `~/.m2/repository`。所有被项目引用过的依赖都会被下载缓存到此处，供后续项目复用。
- **中央仓库（Central Repository）**：Maven 社区托管的公共仓库，URL 为 `https://repo.maven.apache.org/maven2`。当本地仓库不存在某个依赖时，Maven 会自动去中央仓库下载。
- **私服（Private Repository）**：企业或团队自建的仓库服务器，常用产品有 **Nexus** 和 **Artifactory**。私服可以代理中央仓库缓存依赖、托管内部私有构件（公司内部 JAR、Spring Boot 私有化部署等）、提供 Maven 仓库的访问权限控制与审计。

**工作流程**：本地仓库 → 私服（若有） → 中央仓库。Maven 按照这个顺序逐级查找，命中即停。

**追问方向**：

- 如果私服宕机了，本地仓库也没有缓存，项目还能正常构建吗？
- 如何配置 Maven 优先从私服下载，而不从中央仓库下载？
- 私服的 SNAPSHOT 仓库和 Release 仓库有什么区别？为什么需要分开？

**避坑提示**：

- 私服不是必须的，但对企业项目来说是标准配置，不要在生产环境直接依赖中央仓库（网络延迟和不稳定是主要风险）。
- 本地仓库路径可以通过 `settings.xml` 中的 `<localRepository>` 自定义，但很少需要改。

---

## 3. Maven 依赖传递是什么？如何解决依赖冲突？

**题目**：什么是 Maven 的依赖传递（Transitive Dependency）？当同一个依赖被多个路径引入导致版本冲突时，Maven 采用什么策略解决？

**核心答案**：

**依赖传递**：当项目引入依赖 A，而 A 本身依赖 B 时，B 会自动被引入到项目中，这就是依赖传递。Maven 自动解析传递性依赖，大大减少了 `pom.xml` 中手动声明的工作量。

**依赖冲突解决策略**：

Maven 采用 **最短路径优先（Nearest Definition）** 原则，也叫"最近定义胜出"：

- 如果同一个依赖（如 `commons-lang`）通过两条不同的依赖路径被引入，Maven 会选择路径最短的那个版本。
- 例：`project → A(v1.2) → B(v1.3)` 与 `project → C(v1.4)` 同时引入 `commons-lang`，则 `v1.4` 被使用（因为直接依赖路径更短）。

**当路径长度相同时**：采用 **先声明优先（First Declaration）** 原则，即 `pom.xml` 中靠前的 `<dependency>` 声明优先。

```
A 和 B 路径长度相同
→ 谁先声明（谁在 pom.xml 中靠上）谁获胜
```

**追问方向**：

- 如何强制指定使用某个特定版本的依赖（排除传递依赖）？
- `<exclusions>` 标签的作用是什么？如何在父 POM 中统一管理排除规则？
- 如果两个版本的依赖路径长度不同，但手工指定了长路径的版本，Maven 会听谁的吗？

**避坑提示**：

- 不要依赖"先声明优先"来控制依赖版本——这是隐式规则，容易被团队其他成员意外破坏。可维护的做法是使用 `dependencyManagement` 显式锁定版本。
- 路径长度计算是从项目到依赖的**声明链路**，而不是依赖的声明顺序。

---

## 4. Maven 依赖作用域有哪些？各自的使用场景是什么？

**题目**：Maven 依赖作用域（Scope）用于控制依赖在什么 classpath 下有效，请列举所有作用域并说明其区别和使用场景。

**核心答案**：

Maven 提供 5 种作用域（默认是 `compile`）：

| 作用域 | 编译时有效 | 测试时有效 | 运行时有效 | 打包时包含 | 典型使用场景 |
|--------|-----------|-----------|-----------|-----------|-------------|
| **compile** | ✅ | ✅ | ✅ | ✅ | 默认值，绝大多数依赖 |
| **provided** | ✅ | ✅ | ❌ | ❌ | 容器已提供，如 `servlet-api`、`jsp-api` |
| **runtime** | ❌ | ✅ | ✅ | ✅ | 编译不需要，运行时需要，如 JDBC 驱动实现 |
| **test** | ❌ | ✅ | ❌ | ❌ | 仅测试代码使用，如 JUnit、Mockito |
| **system** | ✅ | ✅ | ❌ | ❌ | 系统路径依赖，不推荐使用 |

- **compile**（默认）：所有阶段都有效，会被打包。
- **provided**：编译和测试有效，但**不打包**到产物中。典型场景是 Servlet 容器（Tomcat）已提供 `servlet-api.jar`，若打包进去会与容器中的版本冲突。
- **runtime**：编译时不需要，运行时需要。典型场景是 JDBC 驱动——编译时只需要 `java.sql` 接口（`compile` 作用域的 `javax.sql`），但运行需要具体的驱动实现（MySQL/PostgreSQL 的 driver jar）。
- **test**：仅在 `src/test/java` 下编译和运行有效，不参与打包。JUnit、Spring Test、Mockito 等测试框架使用此作用域。
- **system**：从本地系统路径加载 JAR，类似于 `provided` 但需要通过 `<systemPath>` 显式指定路径。**不推荐使用**，因为可移植性极差，CI/CD 环境中路径很可能不存在。

**追问方向**：

- `runtime` 作用域的依赖在 `mvn compile` 阶段会不会被加入 classpath？为什么？
- 如果一个 `provided` 依赖在运行时也用了（MVC 框架的 `@Autowired` 等），会不会有问题？
- `scope` 为 `test` 的依赖会被其他模块通过传递性依赖引入吗？

**避坑提示**：

- 混淆 `provided` 和 `runtime` 是高频错误。如果依赖在编译时需要 import 其类，就不能用 `runtime`。
- `system` 作用域基本等同于"自找麻烦"，99% 的场景可以用本地 Maven 仓库或私服来替代。

---

## 5. 描述 Maven 的构建生命周期及主要阶段。

**题目**：Maven 有哪几套生命周期？每个生命周期包含哪些主要阶段（phase）？执行 `mvn clean package` 和 `mvn install` 有什么区别？

**核心答案**：

Maven 内置三套独立的生命周期：**clean**、**default**、**site**。

**clean 生命周期**（构建前清理）：

- `pre-clean` → `clean` → `post-clean`

**default 生命周期**（核心构建）：

- `validate` → `compile` → `test` → `package` → `verify` → `install` → `deploy`

**site 生命周期**（生成项目站点文档）：

- `pre-site` → `site` → `post-site` → `site-deploy`

常用的阶段顺序：

```
clean      → 清理输出目录
compile    → 编译源代码（src/main/java）
test       → 运行单元测试（src/test/java）
package    → 打包成 JAR/WAR
install    → 安装到本地仓库（供其他本地项目引用）
deploy     → 推送到远程仓库（私服/中央仓库）
```

**`mvn clean package`**：先清理，再执行 default 的 `package` 阶段（及其之前所有阶段）。
**`mvn install`**：执行 default 的 `install` 阶段（包含 `package`），但不执行 `deploy`。

| 命令 | 执行了哪些阶段 |
|------|---------------|
| `mvn clean package` | clean + compile + test + package |
| `mvn install` | compile + test + package + install（无 clean） |
| `mvn clean install` | clean + compile + test + package + install |

**追问方向**：

- 如果只想运行测试但不打包，该用什么命令？
- `mvn test` 会不会自动先执行 `compile`？为什么？
- 每个阶段绑定哪些插件目标（goal）？比如 `package` 阶段默认绑定什么？

**避坑提示**：

- `clean` 是独立的生命周期，和 `default` 完全分开，所以 `mvn clean install` 中 `clean` 和 `install` 属于不同生命周期，可以一起执行。
- 执行 `mvn package` 后没有 `clean`，旧的编译 class 文件可能残留，导致"明明改了代码但运行结果没变"的困惑。

---

## 6. Maven 插件机制是如何工作的？插件目标（goal）是什么？

**题目**：Maven 插件（Plugin）机制是什么？插件目标（goal）和生命周期阶段（phase）是如何绑定的？请举例说明。

**核心答案**：

Maven 本身不执行具体的编译、打包操作，这些全部由**插件**来完成。插件是一组 `goal`（目标）的集合，每个 goal 是插件中的最小执行单元。

**核心概念**：

- **插件（Plugin）**：一组相关 goal 的打包，如 `maven-compiler-plugin`、`maven-surefire-plugin`。
- **插件目标（Goal）**：插件中的具体功能，如 `compiler:compile`（编译 Java 源码）、`surefire:test`（运行测试）。
- **绑定（Binding）**：将插件的 goal 绑定到生命周期的特定阶段，使该阶段被触发时自动执行对应 goal。

**典型绑定示例**：

```
compile  阶段 → maven-compiler-plugin:compile
test     阶段 → maven-surefire-plugin:test
package  阶段 → maven-jar-plugin:jar (或 maven-war-plugin:war)
install  阶段 → maven-install-plugin:install
deploy   阶段 → maven-deploy-plugin:deploy
```

**命令行直接调用 goal**：`mvn compiler:compile`（不经过生命周期，直接运行插件目标）

**追问方向**：

- 如何自定义一个插件并在 pom.xml 中使用？
- 同一个阶段可以绑定多个 goal 吗？执行顺序是什么？
- 如何查看某个插件有哪些可用的 goal？

**避坑提示**：

- 插件的 `<executions>` 配置中可以设置 `<phase>` 来指定绑定阶段，设置 `<goals>` 来指定具体 goal。
- 不是所有 goal 都需要绑定到生命周期，有些 goal 适合直接命令行调用（如 `mvn versions:set`）。

---

## 7. Maven pom.xml 的解析顺序是什么？

**题目**：Maven 如何解析 pom.xml？描述 parent POM、dependencyManagement、pluginManagement 的继承和解析顺序。

**核心答案**：

Maven 采用 **最小优先级原则（Least Important Wins）** 进行属性覆盖，解析顺序从父到子、从全局到局部：

**解析顺序（后声明/后配置的值覆盖先前的）**：

1. **Super POM**（Maven 内置的父 POM，最顶层）
2. **父 POM**（`<parent>` 声明的项目 POM）
3. **当前项目 POM**
4. **activeProfiles**（profile 会进一步覆盖）

**dependencyManagement 解析规则**：

- 子 POM **不会自动继承** `dependencyManagement`，必须显式在子 POM 中声明需要的依赖（不需要指定版本，版本由父 POM 管理）。
- 如果父 POM 和子 POM 都声明了同一个依赖的不同版本，**子 POM 版本优先**。
- `dependencyManagement` 只声明依赖的**版本和作用域**，不实际引入依赖。

**pluginManagement 解析规则**：

- 和 `dependencyManagement` 类似，`pluginManagement` 只声明插件的**版本和默认配置**，不实际执行插件。
- 子模块需要显式在 `<plugins>` 中声明才会使用 `pluginManagement` 中的配置。

**优先级**：子 POM 的直接声明 > 父 POM 的 dependencyManagement / pluginManagement > Super POM。

**追问方向**：

- 如果父 POM 的 dependencyManagement 中声明了依赖，但子 POM 没有声明，会被引入吗？
- 如何在子 POM 中覆盖父 POM 插件的配置（如修改 `maven-compiler-plugin` 的 Java 版本）？
- `mvn help:effective-pom` 命令能看到什么？

**避坑提示**：

- 混淆 `dependencyManagement`（只声明不引入）和普通 `dependencies`（直接引入）是常见错误。
- 不要在父 POM 的 `<dependencies>` 中放具体依赖，这会导致所有子模块强制依赖。正确做法是放在 `<dependencyManagement>` 中。

---

## 8. Maven 多模块项目如何决定构建顺序？

**题目**：Maven 多模块项目中，reactor 构建顺序是如何确定的？如果模块间存在循环依赖会怎样？

**核心答案**：

Maven 多模块项目通过 **reactor** 机制自动分析模块间的依赖关系，并按照**依赖顺序进行拓扑排序**来决定构建顺序。

**reactor 构建顺序原则**：

- 被依赖的模块**先构建**，依赖方**后构建**。
- 例如：`parent` → `common` → `service` → `web`，reactor 保证 `common` 在 `service` 之前构建完成并 install 到本地仓库。
- 使用 `mvn install` 而非 `mvn package`，因为后续模块依赖的是本地仓库中的 artifact。

**声明式依赖控制**：

```xml
<!-- service/pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.example</groupId>
        <artifactId>common</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

**循环依赖（Circular Dependency）**：
如果 A 依赖 B，B 依赖 A，Maven 会抛出异常：

```
[ERROR] [ERROR] The projects in the reactor contain a cycle:
com.example:service:jar:1.0.0 -> com.example:common:jar:1.0.0 ->
com.example:service:jar:1.0.0
```

**`--also-make` 和 `--also-make-dependents` 参数**：

- `mvn install -pl module-name --also-make`：构建指定模块及其依赖的模块。
- `mvn install --also-make-dependents`：构建指定模块及其**依赖该模块**的模块。

**追问方向**：

- 如果两个模块之间没有依赖关系但在同一 reactor 中，构建顺序由什么决定？
- 如何跳过某个子模块不参与构建？
- `mvn -pl` 和 `mvn -am` 参数的作用是什么？

**避坑提示**：

- 多模块项目构建时不要用 `mvn package`，必须用 `mvn install`，否则后续模块找不到依赖的 artifact。
- 父子模块之间尽量避免双向依赖——父子应该是纯粹的继承关系，依赖应该发生在同级的兄弟模块之间。

---

## 9. Gradle 相比 Maven 有哪些核心优势和区别？

**题目**：Gradle 和 Maven 的核心区别是什么？从 DSL、构建速度、灵活性、缓存机制等维度进行对比。

**核心答案**：

| 维度 | Maven | Gradle |
|------|-------|--------|
| **构建语言（DSL）** | XML（`pom.xml`），声明式但表达力有限 | Groovy/Kotlin DSL（`build.gradle` / `build.gradle.kts`），是真正的编程语言 |
| **构建速度** | 相对较慢，每次构建完整执行 | 增量构建、智能构建缓存，比 Maven 快 2~10 倍 |
| **灵活性** | 插件机制固定，扩展能力受限 | Task 机制灵活，几乎可以自定义任何构建行为 |
| **缓存机制** | 依赖缓存（本地仓库），但每次全量下载 SNAPSHOT | Gradle 依赖缓存 + 构建缓存（可共享，支持远程缓存服务） |
| **多模块管理** | 通过 parent POM 继承，支持有限 | 通过 `settings.gradle` 声明项目结构，更灵活 |
| **增量构建** | 不支持，Maven 每次重新执行所有目标 | 原生支持输入输出追踪，仅重新执行变化的 Task |
| **并行执行** | 通过 `mvn -T` 支持有限并行 | 原生支持并行执行 Task 和模块 |

**Gradle 的核心优势**：

- **Build Cache**：相同输入的 Task 输出可以跨机器复用。
- **Configuration Cache**：配置阶段结果可缓存，进一步加速（现代 Gradle 支持）。
- **增量编译**：Java 增量编译比 Maven 的全量 javac 快很多。

**Maven 的优势**：

- XML 简单易懂，团队学习曲线低。
- 生态成熟，企业级支持广泛。
- 生命周期概念清晰，约定优于配置。

**追问方向**：

- Gradle 的 DSL 为什么比 Maven 的 XML 更强大？
- 如果项目已经用 Maven，迁移到 Gradle 的一般步骤是什么？
- Gradle 的 daemon 机制是什么？有什么优缺点？

**避坑提示**：

- 不要把 Gradle 和 Maven 的"哪个更好"当成非此即彼的选择——很多大型项目同时使用二者（Maven 管理发布，Gradle 处理特定构建任务）。
- Gradle 的灵活性是双刃剑：没有约束的 `build.gradle` 很容易变成难以维护的"构建代码"。

---

## 10. Gradle 依赖配置 `implementation`、`api`、`compileOnly` 的区别是什么？

**题目**：在 Gradle（尤其是 Android 和 Spring Boot 项目）中，`implementation`、`api`、`compileOnly` 三种依赖配置有何区别？各自的使用场景是什么？

**核心答案**：

这三种配置来自 Java 插件（`java-library` 插件提供 `api`）：

- **`implementation`**（替代旧版 `compile`）：依赖仅在**当前模块内部**可见，不会传递到依赖该模块的其他模块。
  - 使用场景：模块内部使用的工具类、HTTP 客户端等。
  - 优势：减少编译时的 transitive dependency，提升编译速度。

- **`api`**（原 `compile`，需要 `java-library` 插件）：依赖会**传递**给下游模块，就像 Maven 的 `compile` scope。
  - 使用场景：当你写的是一个库模块，API 中使用了某个依赖，且下游模块也需要用到该依赖。
  - 优势：保持与 Maven `compile` scope 等价的传递性。

- **`compileOnly`**：仅在**编译时**需要，运行时不需要，不参与打包。
  - 使用场景：注解处理器（如 Lombok）、仅编译时使用的生成工具。

**对比表**：

| 配置 | 编译时 | 运行时 | 传递到下游 | 打包 |
|------|--------|--------|-----------|------|
| `implementation` | ✅ | ✅ | ❌ | ✅ |
| `api` | ✅ | ✅ | ✅ | ✅ |
| `compileOnly` | ✅ | ❌ | ❌ | ❌ |

**追问方向**：

- 为什么 `implementation` 能提升编译速度？
- 如果我用的是 `java` 插件而不是 `java-library`，还能用 `api` 配置吗？
- `runtimeOnly`（旧版 `runtime'`）和 `implementation` 的区别是什么？

**避坑提示**：

- 优先使用 `implementation`，避免不必要的依赖传递——这是 Gradle 最佳实践。
- 不要在库模块中使用 `implementation` 发布 API 类，否则下游模块编译时会找不到类。

---

## 11. Gradle Task 机制：任务依赖、任务跳过、条件执行是什么？

**题目**：描述 Gradle 中 Task（任务）的概念、如何声明任务依赖、如何跳过任务、以及条件执行的实现方式。

**核心答案**：

**Task 是 Gradle 的基本工作单元**，相当于 Maven 中的插件 goal。

**声明任务**：

```groovy
task hello {
    doLast {
        println 'Hello World'
    }
}
```

**任务依赖**：通过 `dependsOn` 声明任务间的依赖关系：

```groovy
task loadData {
    doLast { println 'loading data...' }
}

task processData {
    dependsOn loadData  // processData 会在 loadData 之后执行
    doLast { println 'processing...' }
}
```

**任务跳过**：

- **`enabled = false`**：完全禁用任务（不执行，也不产生输出）：
  ```groovy
  task myTask {
      enabled false  // 跳过该任务
  }
  ```

- **`stopExecutionIf()`**：条件性跳过：
  ```groovy
  task myTask {
      doLast {
          if (project.hasProperty('skipTask')) return
          println 'executing...'
      }
  }
  ```

- **`onlyIf {}`**：更现代的条件执行方式：
  ```groovy
  task hello {
      doLast { println 'hello' }
  }
  hello.onlyIf { !project.hasProperty('skipHello') }
  ```

**增量构建条件**：`TaskInputs` 和 `TaskOutputs` 追踪输入输出的变化，自动判断是否需要重新执行。

**追问方向**：

- `doFirst` 和 `doLast` 的区别是什么？哪个先执行？
- 如何让一个 Task 同时依赖多个其他 Task？
- Gradle 的 Task 如何与 Maven 的生命周期阶段对应？

**避坑提示**：

- `doLast` 内部使用 `return` 只是跳过当前任务的剩余逻辑，`onlyIf` 才是正确跳过整个任务的方式。
- 任务依赖声明的顺序不影响实际执行顺序——执行顺序由依赖图决定，不是声明顺序。

---

## 12. Gradle 生命周期的三个阶段是什么？

**题目**：描述 Gradle 构建的三个阶段：配置阶段（Configuration）、生成阶段（Execution）、依赖解析（Dependency Resolution）。它们分别做了什么？

**核心答案**：

Gradle 构建分为三个阶段：

**1. 配置阶段（Configuration）**：

- Gradle 解析所有 `build.gradle`（或 `build.gradle.kts`）脚本。
- 执行脚本中的代码，**为每个 Project 创建 Task 图谱**（Task 之间的依赖关系）。
- 这个阶段**所有 Project 都会参与配置**（即使某些项目最终不会被构建）。
- 特点：配置代码会立即执行（不像 Maven 的 lifecycle 是延迟绑定的）。

**2. 依赖解析阶段（Dependency Resolution）**：

- Gradle 解析 Task 的依赖图，确定需要执行哪些 Task。
- 解析 `dependencies` 配置，从配置的仓库（本地、Maven、Ivy）中下载依赖。
- 确定每个依赖的具体版本（遵循版本冲突解决策略）。

**3. 生成阶段（Execution）**：

- 按照拓扑排序顺序**执行**需要运行的 Task。
- 每个 Task 根据其输入输出的变化决定是否真正执行（增量构建）。
- 生成构建产物（class 文件、JAR 包等）。

**配置阶段的特点**：配置代码中的 `println` 会**立即输出**，即使你只是运行 `gradle tasks`。这是调试 Gradle 构建问题的重要观察点。

```
初始化 → 配置 → 依赖解析 → 执行
```

**追问方向**：

- 为什么配置阶段的代码要尽量轻量？过度复杂的配置代码会带来什么问题？
- `settings.gradle` 的配置在哪个阶段生效？
- Gradle 的 "configuration on demand" 是什么意思？

**避坑提示**：

- 不要在 `build.gradle` 的顶层写耗时的网络请求或文件 I/O——它们在**配置阶段**就会执行，即使只是 `gradle tasks`。
- 在 `doLast` / `doFirst` 闭包外写的代码属于配置阶段代码，不属于执行阶段。

---

## 13. Gradle 增量构建和构建缓存是如何工作的？

**题目**：Gradle 的增量构建（Incremental Build）机制是什么？输入输出哈希和构建缓存（Build Cache）是如何协同工作的？

**核心答案**：

**增量构建（Incremental Build）**：

Gradle 为每个 Task 定义了**输入（Inputs）**和**输出（Outputs）**。构建时，Gradle 计算输入的哈希值，与上次构建时记录的输出哈希对比：

- 如果输入哈希**没变化** → 跳过 Task，直接复用上次的输出。
- 如果输入哈希**变化了** → 重新执行 Task，生成新输出。
- 如果输出哈希变化了 → 通知依赖此 Task 的下游 Task 重新执行。

```groovy
task compileJava(type: JavaCompile) {
    source = fileTree('src/main/java')
    // inputs 和 outputs 由 JavaCompile 插件自动追踪
}
```

**构建缓存（Build Cache）**：

- **本地构建缓存**：默认启用，Task 输出缓存在 `~/.gradle/caches/build-cache-1/`。
- **远程构建缓存（Shared Cache）**：可配置连接到 Gradle Cloud Services 或自定义的缓存服务器，**不同机器之间共享**相同的 Task 输出。
- 远程缓存通过输入哈希作为 key，相同输入在不同机器上产生相同的输出，可以直接复用。

**配置构建缓存**：

```groovy
// build.gradle
buildCache {
    local { enabled = true }
    remote {
        enabled = true
        url = 'https://cache.example.com/'
        credentials {
            username = 'user'
            password = 'pass'
        }
    }
}
```

**追问方向**：

- 如果一个 Task 没有声明 inputs 或 outputs，Gradle 会怎么处理？
- SNAPSHOT 依赖在 Gradle 缓存机制下会有什么表现？
- `gradle --rerun-tasks` 和 `gradle --build-cache` 的区别是什么？

**避坑提示**：

- Task 的输入不仅包括文件内容，还包括环境变量、系统属性等——如果这些被错误纳入了哈希计算，会导致意外的 Task 重执行。
- 使用 `@SkipWhenEmpty` 注解可以在输入目录为空时自动跳过 Task。

---

## 14. Maven 跳过测试的两种方式有什么区别？

**题目**：`mvn -DskipTests` 和 `mvn -Dmaven.test.skip=true` 都能跳过测试，它们有什么区别？各会导致什么后果？

**核心答案**：

| 参数 | 编译测试源码 | 运行测试 | 打包测试 JAR |
|------|------------|---------|-------------|
| `-DskipTests` | ✅ 编译 | ❌ 跳过运行 | ✅ 打包 |
| `-Dmaven.test.skip=true` | ❌ 跳过编译 | ❌ 跳过运行 | ❌ 不打包 |

**`mvn -DskipTests`**：

- 编译测试源代码（`src/test/java` → `target/test-classes`）。
- **跳过测试执行**（不运行 JUnit/TestNG 测试）。
- 测试 JAR 会被正常打包（`xxx-tests.jar`）。

**`mvn -Dmaven.test.skip=true`**：

- **完全跳过测试相关的编译和执行**。
- 测试代码不会被编译，测试 JAR 也不会被打包。
- 这个参数通过 `maven-compiler-plugin` 和 `maven-surefire-plugin` 的配置来实现。

**使用场景**：

- 紧急 hotfix 场景：`mvn -DskipTests`（测试太慢，先打包上线，之后再跑测试）。
- 代码没改，只改了资源文件：`mvn -Dmaven.test.skip=true`（测试没必要重跑）。

**追问方向**：

- 如果测试代码编译失败但加了 `-DskipTests`，能成功打包吗？
- 如何通过 pom.xml 永久配置跳过测试？
- `mvn package -DskipTests` 和 `mvn install -DskipTests` 行为一致吗？

**避坑提示**：

- 不要在 CI 中长期使用 `-Dmaven.test.skip=true`——测试编译失败的问题会被掩盖。
- `-DskipTests` 跳过的是测试执行，但测试源码仍然被编译，这个差异在排查编译错误时很重要。

---

## 15. Maven Profiles 如何实现多环境配置？

**题目**：Maven Profiles 是什么？如何使用 Profiles 实现 dev/test/prod 多环境配置？

**核心答案**：

Maven Profiles 允许在同一份 `pom.xml` 中定义多套配置，通过 `-P` 参数激活对应的 Profile。

**定义 Profile**：

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>development</env>
            <db.url>jdbc:mysql://localhost:3306/dev</db.url>
        </properties>
    </profile>
    <profile>
        <id>test</id>
        <properties>
            <env>testing</env>
            <db.url>jdbc:mysql://test-server:3306/test</db.url>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <env>production</env>
            <db.url>jdbc:mysql://prod-server:3306/prod</db.url>
        </properties>
        <dependencies>
            <!-- prod 独有的依赖，如商业数据库驱动 -->
        </dependencies>
    </profile>
</profiles>
```

**使用过滤资源**：

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

在 `application.properties` 中使用占位符 `${db.url}`，Maven 在打包时自动替换为 Profile 中定义的值。

**激活 Profile 的方式**：

- 命令行激活：`mvn package -P dev`
- 默认激活：`<activeByDefault>true</activeByDefault>`
- 基于 OS/文件存在自动激活：`<activation>`
- Maven settings 中激活：`settings.xml` 里配置 `<activeProfiles>`

**追问方向**：

- 如果同时激活了多个 Profile，Maven 以哪个为准？
- 如何让不同 Profile 使用不同的依赖（而不是仅替换 properties）？
- Spring Boot 的 profile 机制和 Maven Profile 有什么区别？实际项目中推荐哪种？

**避坑提示**：

- 不要把 Profile 当成代码逻辑分支——Profile 只应该用于**构建时**的差异化配置，运行时差异化用 Spring Profile。
- 大量 Profile 会导致 `pom.xml` 膨胀，建议通过父 POM 统一管理公共 Profile，具体的差异放在子模块。

---

## 16. Maven BOM 是什么？它如何管理依赖版本？

**题目**：Maven BOM（Bill of Materials）是什么？Spring Boot 如何通过 `spring-boot-dependencies` 管理依赖版本？

**核心答案**：

**BOM 本质上是一个特殊的 POM**，其 `<packaging>` 为 `pom`，`<dependencyManagement>` 中声明了大量依赖的版本信息。BOM 的作用是**统一管理依赖版本**，下游项目引入依赖时可以省略 `<version>`。

**Spring Boot 的 BOM**：

```xml
<!-- spring-boot-dependencies-3.2.0.pom -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>  <!-- 关键：类型为 import -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

**`import` scope 的作用**：将 BOM 中的 `dependencyManagement` 列表**合并（扁平化）**到当前项目的 `dependencyManagement` 中，类似于继承但更灵活。

**子项目使用**：

```xml
<dependencies>
    <!-- 不需要指定 version，版本由 spring-boot-dependencies 统一管理 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.spring.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

**自定义 BOM**：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-project-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**追问方向**：

- `scope=import` 和 `scope=compile` 在 BOM 中的区别是什么？
- 如果子项目显式指定了版本号，会覆盖 BOM 中的版本吗？
- 多 BOM 之间有冲突的依赖版本，Maven 如何处理？

**避坑提示**：

- BOM 的 `<dependencyManagement>` 不会自动引入依赖，只是声明版本——真正引入还需要在 `<dependencies>` 中声明。
- 一个项目最多只能有一个 `scope=import` 的 BOM（因为顺序依赖会导致不可预测的结果）。

---

## 17. Gradle Wrapper 是什么？它如何管理 Gradle 版本？

**题目**：Gradle Wrapper（gradle wrapper）是什么？它包含哪些文件？团队如何通过它统一 Gradle 版本？

**核心答案**：

Gradle Wrapper 是一组脚本和配置文件，允许开发者在没有预装 Gradle 的环境中执行 Gradle 构建，同时**锁定项目使用的 Gradle 版本**。

**核心文件结构**：

```
project/
├── gradlew          # Unix 平台的启动脚本
├── gradlew.bat      # Windows 平台的启动脚本
└── gradle/wrapper/
    ├── gradle-wrapper.jar
    └── gradle-wrapper.properties  # 指定 Gradle 版本
```

**`gradle-wrapper.properties` 内容**：

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

**如何生成 Wrapper**：

```bash
# 项目根目录执行
gradle wrapper --gradle-version 8.5

# 或在 build.gradle 中声明后运行
gradle wrapper
```

**版本升级**：修改 `gradle-wrapper.properties` 中的 `distributionUrl` 即可。

**团队协作**：所有团队成员只需要有 JDK，**不需要安装 Gradle**——运行 `gradlew` 会自动下载指定版本。

**追问方向**：

- `gradlew` 是如何找到并下载正确版本的 Gradle 的？
- 离线环境下 `gradlew` 能工作吗？
- Wrapper 的 `all` 发行版和 `bin` 发行版有什么区别？

**避坑提示**：

- 不要把 `gradle-wrapper.properties` 放在 `.gitignore` 中——它是项目基础设施，团队所有成员都需要。
- 升级 Gradle 版本前建议先在本地测试，通过 `gradlew --version` 验证下载是否正常。

---

## 18. 如何使用 `mvn dependency:tree` 分析依赖冲突？

**题目**：`mvn dependency:tree` 命令的作用是什么？如何分析依赖冲突？如果看到某个依赖有两个不同版本同时出现，应该如何处理？

**核心答案**：

**`mvn dependency:tree`** 递归显示项目中所有直接依赖和传递依赖的完整树状结构。

```bash
mvn dependency:tree
# 输出示例
[INFO] com.example:myapp:jar:1.0.0
[INFO] +- org.springframework:spring-core:jar:6.1.0:compile
[INFO] |  \- org.springframework:spring-jcl:jar:6.1.0:compile
[INFO] +- commons-lang:commons-lang:jar:2.5:compile
[INFO] \- com.google.guava:guava:jar:31.0-jre:compile
```

**分析依赖冲突**：

1. **查看完整树**：`mvn dependency:tree` 找到冲突的依赖。
2. **加上 `-Dverbose`**：显示被排除的依赖（`omitted for conflict`）。
   ```bash
   mvn dependency:tree -Dverbose
   # 会显示：omitted for conflict、omitted for duplicate
   ```
3. **查看特定依赖**：
   ```bash
   mvn dependency:tree -Dincludes=commons-lang
   ```

**解决冲突**：

在 `pom.xml` 中使用 `<exclusions>` 排除不需要的传递依赖，或在父 POM 的 `dependencyManagement` 中显式锁定版本。

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>module-A</artifactId>
    <exclusions>
        <exclusion>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

**追问方向**：

- `dependency:tree` 显示的 `compile`、`runtime` 等 scope 和依赖声明中的 scope 有什么关系？
- 如果依赖树很长，有什么技巧快速定位冲突？
- 如何将 `dependency:tree` 的输出保存到文件供团队分享？

**避坑提示**：

- `dependency:tree` 默认不显示 `provided` 和 `test` 作用域的传递依赖，需要加 `-Dscope=compile`（默认 scope）或者用 `-Dverbose` 查看完整范围。
- 不要在找到冲突后直接用 `<exclusions>` 删除，而是先评估被排除的传递依赖是否在项目中真的不需要。

---

## 19. 如何使用 Maven 构建 Docker 镜像？

**题目**：在 Maven 项目中，如何使用 `dockerfile-maven-plugin` 将应用打包为 Docker 镜像？请描述配置和构建流程。

**核心答案**：

**添加插件配置**：

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>dockerfile-maven-plugin</artifactId>
    <version>1.4.13</version>
    <configuration>
        <repository>registry.example.com/${project.artifactId}</repository>
        <tag>${project.version}</tag>
        <buildArgs>
            <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
        </buildArgs>
    </configuration>
    <executions>
        <execution>
            <id>build-image</id>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
        <execution>
            <id>push-image</id>
            <phase>deploy</phase>
            <goals>
                <goal>push</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

**对应的 Dockerfile**：

```dockerfile
FROM openjdk:17-slim
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**构建命令**：

```bash
# 打包并构建镜像（自动执行 package 阶段）
mvn clean package dockerfile:build

# 推送到远程仓库
mvn clean deploy dockerfile:push
```

**插件自动行为**：

- `dockerfile:build` 在 `package` 阶段绑定时，会在 `mvn package` 时自动构建 Docker 镜像。
- 镜像名称格式：`repository/artifactId:version`。

**追问方向**：

- `dockerfile-maven-plugin` 和 `jib-maven-plugin` 有什么区别？
- 如果没有安装 Docker，插件能正常工作吗？
- 如何在多模块项目中为每个子模块构建独立的 Docker 镜像？

**避坑提示**：

- `dockerfile-maven-plugin` 依赖宿主机上已安装 Docker 且 daemon 运行中。如果在 CI/CD 环境中使用，需要确保 Docker-in-Docker（DinD）或 Docker socket 挂载正确。
- 新项目推荐考虑 Google 的 `jib-maven-plugin`，它不需要 Docker daemon，可以直接构建镜像并推送到 registry。

---

## 20. Maven 私有仓库认证如何配置？

**题目**：当 Maven 需要从私有仓库（Nexus/Artifactory）下载依赖或上传构件时，如何在 `settings.xml` 中配置认证信息？

**核心答案**：

Maven 认证信息配置在 `~/.m2/settings.xml` 的 `<servers>` 标签中，与 `pom.xml` 分离以保证安全性。

**settings.xml 配置**：

```xml
<settings>
    <servers>
        <!-- 私服认证 -->
        <server>
            <id>my-nexus-releases</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>my-nexus-snapshots</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
        <!-- 私有中央仓库镜像认证 -->
        <server>
            <id>aliyun-mirror</id>
            <username>mirror-user</username>
            <password>mirror-pass</password>
        </server>
    </servers>
</settings>
```

**ID 必须匹配**：`settings.xml` 中的 `<id>` 必须与 `pom.xml` 中 `<repository>` 或 `<mirror>` 的 `<id>` 一致，Maven 根据 ID 进行匹配。

```xml
<!-- pom.xml 中 -->
<repositories>
    <repository>
        <id>my-nexus-releases</id>
        <url>https://nexus.example.com/repository/releases/</url>
    </repository>
</repositories>
```

**密码加密**：Maven 2.1.0+ 支持密码加密（通过 `mvn --encrypt-password`），但需配置 Master Password：

```bash
# 加密仓库密码
mvn --encrypt-password admin123
# 将输出添加到 settings.xml
<settings>
    <security>
        <masterPassword>{某种加密后的密码}</masterPassword>
    </security>
</settings>
```

**追问方向**：

- 如果密码中有特殊字符（如 `$`、`@`），应该如何处理？
- `settings.xml` 中的 `<localRepository>` 配置优先级高于什么？
- CI/CD 环境中如何避免明文密码？有哪些替代方案（如环境变量、密钥管理服务）？

**避坑提示**：

- **永远不要把认证信息写在 `pom.xml` 中**——`pom.xml` 通常被提交到版本库，密码会泄露。
- `settings.xml` 本身可以加入 `.gitignore`，确保不意外提交到代码仓库。
- Maven 的密码加密是**可逆**的，不要依赖它作为真正的安全措施——真正的安全靠 Vault/密钥管理服务实现。

---

*文档版本：2025 | 面试题系列 Part 5 - Maven/Gradle 构建工具*
