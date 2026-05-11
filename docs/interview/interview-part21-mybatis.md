# MyBatis 持久层框架面试题

> 本文覆盖 20 道 MyBatis 高频面试题，每题包含题目、核心答案、追问方向、避坑提示。建议配合 MyBatis 源码（3.5.x）阅读。

---

## 第 1 题：MyBatis 整体架构

### 题目
请描述 MyBatis 的整体架构，并说明 `SqlSessionFactoryBuilder`、`SqlSessionFactory`、`SqlSession`、`Executor` 各自的职责与关系。

### 核心答案

MyBatis 整体分为三层架构：

```
┌─────────────────────────────────────────────────┐
│                   API 层                        │
│  SqlSessionFactoryBuilder → SqlSessionFactory   │
│                               ↓                 │
│                          SqlSession             │
│                               ↓                 │
│                         Executor                │
│                               ↓                 │
│              StatementHandler / ResultSetHandler │
│                    ↓            ↓               │
│              JDBC Layer (DataSource)            │
└─────────────────────────────────────────────────┘
```

| 组件 | 职责 | 生命周期 |
|------|------|----------|
| `SqlSessionFactoryBuilder` | 读取配置文件，构建 `SqlSessionFactory` | 方法级（build 后即可销毁） |
| `SqlSessionFactory` | 生产 `SqlSession`，是工厂单例 | 应用级（整个应用生命周期） |
| `SqlSession` | 典型 CRUD 入口，非线程安全 | 请求级（每个线程独享，用完需关闭） |
| `Executor` | 执行 SQL 的核心调度器，负责缓存管理 | 请求级 |

`SqlSession` 是门面，内部委托给 `Executor`，`Executor` 再通过 `StatementHandler` 操作 JDBC。

### 追问方向
- `SqlSession` 为什么不设计成线程安全？
- `SqlSessionFactory` 的实现是单例还是多例？
- `openSession()` 的几个重载方法区别是什么？

### 避坑提示
- 面试时常混淆 `SqlSession` 和 `Connection` 的生命周期——`SqlSession` 是逻辑会话，`Connection` 是真实数据库连接，两者一一对应但并不等价。
- 不要说 "SqlSessionFactoryBuilder 一直存在"，它是方法级临时对象，用完即弃。

---

## 第 2 题：MyBatis 缓存

### 题目
MyBatis 一级缓存和二级缓存的区别是什么？什么情况下缓存会失效？

### 核心答案

| 维度 | 一级缓存（Local Cache） | 二级缓存（Second Level Cache） |
|------|------------------------|-------------------------------|
| 作用域 | `SqlSession` 级别，同一个 session 内共享 | `Mapper` 级别，多个 `SqlSession` 共享 |
| 存储介质 | `HashMap`（内存） | `HashMap` / 可插拔（EhCache、Redis 等） |
| 缓存实现 | `PerpetualCache` | `CachingExecutor` 装饰器 |
| 默认开启 | 是 | 否（需在 mapper.xml 中配置 `<cache/>`） |
| 失效时机 | `SqlSession` close 或 clearCache | `commit` / `close` 时写入 |

**一级缓存失效场景：**
1. `SqlSession` 调用 `close()` → 缓存清空
2. 调用 `clearCache()` → 手动清空
3. 执行 `update/delete/insert` DML 语句 → 当前 statement 的缓存失效（非全局清空）
4. 查询条件不同（key 不同）

**二级缓存失效场景：**
- 同一 namespace 下执行 `insert/update/delete` 并 `commit` → 该 namespace 所有缓存清空

### 追问方向
- 二级缓存的实现原理是什么？（`CachingExecutor` 装饰模式）
- 为什么说二级缓存慎用？它有什么坑？
- `useCache=false` 和 `flushCache=true` 的区别是什么？

### 避坑提示
- 回答时务必说清"一级缓存是 SqlSession 级别"，说成"一级缓存是 mybatis 默认开启"不够精确。
- 多 SqlSession 场景下，一级缓存不共享；二级的共享也仅限同一个 mapper namespace。

---

## 第 3 题：`#{}` vs `${}`

### 题目
MyBatis 中 `#{}` 和 `${}` 的区别是什么？为什么优先使用 `#{}`？

### 核心答案

| 特性 | `#{}` | `${}` |
|------|-------|-------|
| 底层行为 | JDBC `PreparedStatement` 占位符参数绑定 | 字符串拼接，直接拼入 SQL |
| 预编译 | 是（参数进入预编译阶段） | 否（参数不参与预编译） |
| SQL 注入 | 安全 | 危险 |
| 类型处理 | 自动类型转换 + `TypeHandler` | 无类型处理，原样拼接 |
| 适用场景 | 普通参数 | 动态表名/列名、order by 子句 |

**SQL 注入示例对比：**

```java
// 使用 #{} — 安全
SELECT * FROM user WHERE name = #{name}
// 实际执行: SELECT * FROM user WHERE name = ?
// 参数: "Jack"  → 预编译后执行

// 使用 ${} — 危险
SELECT * FROM user WHERE name = '${name}'
// 参数: "Jack' OR '1'='1" → SELECT * FROM user WHERE name = 'Jack' OR '1'='1'
```

### 追问方向
- 为什么 `${orderColumn}` 不能用 `#{}` 替代？
- MyBatis 中 `#{}` 的参数解析流程是什么？
- `TypeHandler` 在其中起到什么作用？

### 避坑提示
- 永远不要用 `${}` 接收用户直接输入的内容，这是 SQL 注入的经典入口。
- 动态表名可以用 `bind` 标签或 ` scripting` 来规避 `${}` 的风险。
- 面试时若被追问 "预编译的原理"，需要答出 `PreparedStatement` 的参数绑定流程。

---

## 第 4 题：MyBatis 分页

### 题目
MyBatis 有哪几种分页方式？`RowBounds` 物理分页的原理是什么？PageHelper 插件的原理是什么？

### 核心答案

**方式一：RowBounds 逻辑分页（内存分页）**
```java
List<User> users = sqlSession.selectList("selectUser", null, new RowBounds(offset, limit));
```
- 原理：MySQL 下 `RowBounds` 会被忽略（需 SQL 层面处理），Oracle 下会加 `ROWNUM` 限制
- 问题：先查全部数据到内存，再截取，大数据量极不推荐

**方式二：SQL 层面物理分页（推荐）**
```xml
<select id="selectUser" resultType="User">
  SELECT * FROM user LIMIT #{offset}, #{limit}
</select>
```
手动在 SQL 中写 `LIMIT`。

**方式三：PageHelper 插件（最常用）**
```java
PageHelper.startPage(pageNum, pageSize);
List<User> users = userMapper.selectUsers();
// 使用 PageInfo 或 IPage 接收
PageInfo<User> pageInfo = new PageInfo<>(users);
```

PageHelper 原理：
1. `PageHelper.startPage()` 将分页参数存入 `ThreadLocal`（`PageHelper` 线程变量）
2. MyBatis 插件拦截 `Executor` 的 `query()` 方法
3. 从 ThreadLocal 取出分页参数，改写 SQL 为 `SELECT * FROM t LIMIT offset, limit`
4. 最后将分页数据包装成 `Page` 对象

### 追问方向
- PageHelper 的 `PageInfo` 和 `IPage` 区别是什么？
- 为什么 PageHelper 要用 `ThreadLocal` 而不是参数传入？
- Oracle 数据库下 PageHelper 如何处理 ROWNUM？

### 避坑提示
- 绝对不要在生产环境用 `RowBounds` 做大数据量分页，它是逻辑分页，先查全表再截取。
- PageHelper 只对紧跟其后的第一条 SQL 生效，多次查询时注意调用顺序。
- 多表关联 JOIN 时，分页插件可能失效，需要手动写分页 SQL。

---

## 第 5 题：动态 SQL

### 题目
MyBatis 的 `<if>`、`<where>`、`<set>`、`<trim>`、`<foreach>`、`<choose>` 标签各自用途是什么？

### 核心答案

| 标签 | 用途 | 示例 |
|------|------|------|
| `<if test="...">` | 条件判断，test 中写 OGNL 表达式 | `<if test="name != null">AND name = #{name}</if>` |
| `<where>` | 智能包裹 WHERE，自动去除多余 AND/OR | `<where><if test="...">AND ...</if></where>` |
| `<set>` | 智能包裹 SET，自动去除末尾逗号 | 用于 UPDATE 语句 |
| `<trim>` | 精确控制前后缀/前后缀的去除 | 可替代 `<where>` 和 `<set>` 的行为 |
| `<foreach>` | 遍历集合，常用于 IN 子句、批量插入 | `collection="list" item="item" open="(" close=")" separator=","` |
| `<choose>` | 相当于 if-else，只能匹配一个 | `<choose><when>...</when><otherwise>...</otherwise></choose>` |

**完整示例：**
```xml
<update id="updateUser" parameterType="User">
  UPDATE user
  <set>
    <if test="name != null">name = #{name},</if>
    <if test="age != null">age = #{age},</if>
  </set>
  WHERE id = #{id}
</update>

<select id="selectByIds" resultType="User">
  SELECT * FROM user
  <where>
    <if test="ids != null and ids.size > 0">
      id IN
      <foreach item="id" collection="ids" open="(" separator="," close=")">
        #{id}
      </foreach>
    </if>
  </where>
</select>
```

### 追问方向
- `<where>` 标签的源码实现原理？
- `<trim prefix="WHERE" prefixOverrides="AND|OR">` 和 `<where>` 的等价关系？
- `<foreach>` 的 `collection` 参数有哪些合法写法？

### 避坑提示
- `<if test="ids.size > 0">` 中直接写 `ids.size` 可能因空指针抛异常，标准写法是 `test="ids != null and !ids.isEmpty()"` 或 `test="@Ognl@isEmpty(ids)"`。
- MyBatis 3.5.x 中 `<where>` 只能处理 AND 开头，OR 开头需要配合 `<trim>`。

---

## 第 6 题：Mapper 接口绑定

### 题目
MyBatis 中 Mapper 接口与 SQL 的绑定原理是什么？为什么 Mapper 接口不需要实现类？

### 核心答案

MyBatis 使用 **JDK 动态代理** 实现 Mapper 接口绑定。

**绑定流程：**
```
1. 解析 mapper.xml，记录 namespace + statementId → MappedStatement
2. 解析 Mapper 接口，使用 SqlSession.getMapper(Mapper.class)
3. SqlSession 内部使用 MapperProxyFactory 生成 JDK 动态代理对象
4. 代理对象拦截方法调用：
   methodName → statementId（methodName）
   参数 → 执行 args
5. 通过 SqlSession 执行查询
```

**源码关键类：**
- `MapperProxyFactory` — 创建代理工厂
- `MapperProxy` — `InvocationHandler` 实现类
- `MapperMethod` — 封装 Mapper 方法的执行逻辑（CRUD）

```java
// 底层原理
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
// 等价于
UserMapper mapper = new MapperProxyFactory<>(UserMapper.class).newInstance(sqlSession);
```

### 追问方向
- 如果 namespace 与接口全限定名不匹配会怎样？
- Mapper 方法重载时 MyBatis 如何处理？
- `@Param` 注解在多参数场景下起什么作用？

### 避坑提示
- Mapper 接口的命名空间（namespace）必须与接口全限定名一一对应，否则 `getMapper()` 会抛出 `BindingException`。
- Mapper 接口方法不能重载，因为 SQL 的 statementId 是 `namespace + 方法名`，重载会导致 statementId 冲突。
- 接口方法返回如果是 `void`，结果通过 Out 参数回填（不推荐）。

---

## 第 7 题：MyBatis 初始化

### 题目
MyBatis 启动时如何解析配置文件？XML 解析用了什么技术？

### 核心答案

MyBatis 初始化核心流程：

```
SqlSessionFactoryBuilder.build(inputStream)
    ↓
XMLConfigBuilder.parse()           ← XPath 解析配置文件
    ↓
Configration 对象（所有配置的内存模型）
    ↓
SqlSessionFactory
```

**XPath 解析细节：**
- 使用 Java `XPathFactory` + `XPath` 解析 `mybatis-config.xml`
- `<settings>`、`<typeAliases>`、`<environments>` 等标签分别由对应的 `XxxBuilder` 处理
- `XMLMapperBuilder` 解析 `<mapper>` 标签（import、resource、url、class 四种方式）
- 使用 `XPathExpression` 匹配节点，遍历构建 `MappedStatement`

**具体解析流程：**
```
mybatis-config.xml
  ├── <properties>    → ParameterMap
  ├── <settings>      → Settings（缓存、懒加载等开关）
  ├── <typeAliases>   → TypeAliasRegistry
  ├── <typeHandlers>  → TypeHandlerRegistry
  ├── <environments>  → DataSource + TransactionFactory
  └── <mappers>
        └── <mapper resource="xxx.xml">
              → XMLMapperBuilder
              → MappedStatement（CRUD SQL）
```

### 追问方向
- `XPath` 和 `DOM` 解析的区别是什么？为什么选 XPath？
- MyBatis 如何处理 mapper xml 的 import 依赖？
- `SqlSessionFactory` 一旦 build 后，配置还能改吗？

### 避坑提示
- MyBatis 配置对象是线程安全的，但一旦 build 后修改 `Configuration` 对象（如添加 mapper）是危险操作。
- mapper xml 文件解析失败时，错误信息可能只显示 "invalid bound statement"，需要加 log 排查是否 namespace 写错。

---

## 第 8 题：Executor 执行器

### 题目
MyBatis 的 `Executor` 有哪几种？他们之间有什么区别？

### 核心答案

`Executor` 是 MyBatis SQL 执行的核心，所有 SqlSession 的 CRUD 操作最终都委派给 Executor。

| 执行器 | 类名 | 特点 |
|--------|------|------|
| SIMPLE | `SimpleExecutor` | 每条 SQL 创建一个 `PreparedStatement`，用完关闭（默认） |
| REUSE | `ReuseExecutor` | 缓存同 SQL 的 `PreparedStatement`，复用减少创建开销 |
| BATCH | `BatchExecutor` | 批量执行同 SQL，合并提交（用于批量插入/更新） |

**源码结构：**
```java
// BaseExecutor（模板方法模式，缓存、一级缓存、事务封装）
public abstract class BaseExecutor {
    protected PerpetualCache localCache;    // 一级缓存
    protected PerpetualCache localOutputParameterCache;
    protected Transaction transaction;
    // query() → doQuery() 抽象方法
}

// SimpleExecutor extends BaseExecutor
// ReuseExecutor extends BaseExecutor（内部 Map<String, PreparedStatement> 缓存）
// BatchExecutor extends BaseExecutor（jdbc batch 操作）
```

**配置方式：**
```xml
<!-- mybatis-config.xml -->
<settings>
    <setting name="defaultExecutorType" value="REUSE"/>
</settings>
```

### 追问方向
- `BatchExecutor` 的 `flushStatements()` 何时调用？
- `BaseExecutor` 中一级缓存的实现细节？
- `CachingExecutor` 与上述三种 Executor 的关系是什么？

### 避坑提示
- `BatchExecutor` 执行 `update()` 时返回的是 `int[]`，需要遍历获取每条影响行数。
- `BatchExecutor` 多语句拼接时注意 Oracle 需设置 `allowMultiQueries=true`。
- SIMPLE 模式并不意味着每次都创建新连接，是创建新的 `PreparedStatement`。

---

## 第 9 题：StatementHandler

### 题目
`StatementHandler` 的职责是什么？它如何完成 SQL 预编译和参数绑定的？

### 核心答案

`StatementHandler` 是 JDBC 层面的 SQL 执行门户，负责：

1. **创建 `Statement` / `PreparedStatement`**（由 `RoutingStatementHandler` 路由到具体实现）
2. **SQL 预编译**（`PreparedStatement.prepare()`）
3. **参数绑定**（`PreparedStatement.setObject()`）
4. **结果集 `ResultSet` 的获取**

**类层次结构：**
```
StatementHandler（接口）
  ├── RoutingStatementHandler（路由，根据配置选择）
  │     ├── SimpleStatementHandler      (Statement)
  │     ├── PreparedStatementHandler     (PreparedStatement) ← 常用
  │     └── CallableStatementHandler     (CallableStatement)
```

**参数绑定源码流程：**
```java
// PreparedStatementHandler.query()
ps = prepareStatement(executor, ms);
→ handler.bindParameters(ps);      // 参数绑定
→ rs = ps.executeQuery();          // 执行
→ return handler.<ResultSet>wrap(rs);  // 结果集处理
```

### 追问方向
- `ParameterMapping` 是如何构建的？参数Mapping的来源？
- `ResultSetWrapper` 在结果集处理中起什么作用？
- `RoutingStatementHandler` 的路由策略是什么？

### 避坑提示
- `StatementHandler` 与 `Executor` 的关系容易混淆——`Executor` 是更高层的调度，`StatementHandler` 是 JDBC 层面的执行器。
- `ParameterHandler` 负责设置参数，`StatementHandler` 负责使用参数执行。

---

## 第 10 题：ResultSetHandler

### 题目
`ResultSetHandler` 如何将 JDBC `ResultSet` 映射为 Java 对象？嵌套结果映射的原理是什么？

### 核心答案

`ResultSetHandler` 是 MyBatis 将查询结果映射回 Java 对象的核心组件。

**接口方法：**
```java
public interface ResultSetHandler {
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

**映射流程：**
```
ResultSet → ResultSetWrapper（包装元数据：列名、类型）
    ↓
MetaObject（通过反射构建目标对象）
    ↓
自动映射（列名 → 属性名）+ TypeHandler 类型转换
    ↓
嵌套映射（association / collection）
    ↓
List<E> / 单对象 / 标量值
```

**嵌套结果映射示例：**
```xml
<resultMap id="orderMap" type="Order">
    <id property="id" column="id"/>
    <association property="user" javaType="User"
        resultMap="userMap" column="user_id"/>
</resultMap>
```

**嵌套映射两种策略：**
1. **嵌套 Select（N+1 问题根源）**：通过额外 SQL 查关联数据
2. **嵌套结果集**：一次 SQL JOIN，内存中分组映射（`NestedResultSetHandler` 处理）

### 追问方向
- `handleResultSets` 如何处理多 ResultSet（多结果集）？
- `ResultSetWrapper` 中 `typeHandlerRegistry` 的作用是什么？
- 如何自定义映射规则（比如列名下划线转驼峰）？

### 避坑提示
- 嵌套 Select（N+1）在大数据量场景下是性能杀手，优先使用嵌套结果集。
- MyBatis 默认开启自动映射（`automappingBehavior`），但嵌套 resultMap 需要显式配置。
- `columnPrefix` 在嵌套映射中可以避免列名冲突。

---

## 第 11 题：MyBatis 插件机制

### 题目
MyBatis 插件的实现原理是什么？`@Intercepts`、`@Signature`、`Plugin.wrap` 的作用是什么？

### 核心答案

MyBatis 采用 **责任链 + 装饰器模式** 实现插件机制。

**核心原理：**
1. 配置 `Interceptor` 类，使用 `@Intercepts` + `@Signature` 标注拦截点
2. `Configuration` 在初始化时调用 `InterceptorChain.pluginAll()` 为目标对象创建代理
3. `Plugin.wrap()` 使用 `InvocationHandler` 包装目标对象

```java
@Intercepts({
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class,
                       RowBounds.class, ResultHandler.class})
})
public class MyPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 前置处理
        Object proceed = invocation.proceed(); // 继续执行
        // 后置处理
        return proceed;
    }
}
```

**可拦截的四大对象：**
| 组件 | 可拦截方法 | 说明 |
|------|-----------|------|
| `Executor` | `update`, `query`, `flushStatements`, `commit`, `rollback` | 全生命周期 |
| `StatementHandler` | `prepare`, `parameterize`, `batch`, `query`, `update` | SQL prepare 阶段 |
| `ParameterHandler` | `getParameterObject`, `setParameters` | 参数处理 |
| `ResultSetHandler` | `handleResultSets`, `handleCursorResultSets` | 结果集处理 |

**执行顺序（责任链）：**
```
Executor (plugin) → CachingExecutor
    → StatementHandler (plugin) → RoutingStatementHandler
        → ParameterHandler (plugin)
        → PreparedStatement (JDBC)
        → ResultSetHandler (plugin)
```

### 追问方向
- `@Signature` 的 `args` 参数为什么必须完整写？缺少会怎样？
- 如何在插件中判断当前 SQL 是 SELECT 还是 UPDATE？
- 插件拦截顺序是否可以控制？

### 避坑提示
- 插件会对所有匹配的 method 拦截，在高并发 SQL 执行路径上添加插件要慎重，避免性能问题。
- `@Signature` 的 `args` 顺序必须与被拦截方法的参数签名完全一致，这是最常见的插件配置错误。
- MyBatis 插件是嵌套代理，不是链式调用，`Invocation.proceed()` 递归进入下一层。

---

## 第 12 题：插件应用——分页插件与乐观锁插件

### 题目
请分别说明分页插件和乐观锁插件的实现思路。

### 核心答案

**分页插件实现思路：**

```java
@Intercepts({
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class,
                       RowBounds.class, ResultHandler.class})
})
public class PageHelper implements Interceptor {
    // ThreadLocal 存储分页参数（pageNum, pageSize）
    private static final ThreadLocal<Page> LOCAL_PAGE = new ThreadLocal<>();

    public static void startPage(int pageNum, int pageSize) {
        LOCAL_PAGE.set(new Page(pageNum, pageSize));
    }

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Page page = LOCAL_PAGE.get();
        if (page == null) {
            return invocation.proceed(); // 无分页参数，直接执行
        }

        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        RowBounds rowBounds = (RowBounds) args[2];

        // 改写 RowBounds 为 RowBounds.DEFAULT（禁用内存分页）
        args[2] = RowBounds.DEFAULT;

        // 获取原始 SQL，改写为带 LIMIT 的 SQL
        BoundSql boundSql = ms.getBoundSql(parameter);
        String originalSql = boundSql.getSql();
        String pageSql = originalSql + " LIMIT " + page.getOffset() + "," + page.getPageSize();

        // 重新构建 MappedStatement（懒复制）
        // 执行分页查询
        // 将结果包装成 Page 对象返回
        return page;
    }
}
```

**乐观锁插件实现思路：**

```java
@Intercepts({
    @Signature(type = Executor.class, method = "update", args = {
        MappedStatement.class, Object.class})
})
public class OptimisticLockerInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
        Object parameter = invocation.getArgs()[1];

        // 判断是更新操作
        if (ms.getSqlCommandType() == SqlCommandType.UPDATE) {
            MetaObject metaObject = SystemMetaObject.forObject(parameter);
            Integer version = (Integer) metaObject.getValue("version");
            if (version != null) {
                // 改写 SQL 追加 WHERE version = #{version}
                // version 值自增
                metaObject.setValue("version", version + 1);
            }
        }
        return invocation.proceed();
    }
}
```

配合实体类注解：
```java
@Version
private Integer version;
```

### 追问方向
- 分页插件如何兼容 Oracle 的 ROWNUM？
- 乐观锁插件如何处理更新失败（版本冲突）抛出的异常？
- 插件间执行顺序如何保证？

### 避坑提示
- 分页插件改写 SQL 时，不能破坏原有 SQL 的语法，需要正则解析 FROM/WHERE 等关键词位置。
- 乐观锁插件需要在 update 时捕获 `OptimisticLockException` 并转为业务异常。
- 两者共存在同一插件链中，需注意 `proceed()` 调用层级。

---

## 第 13 题：MyBatisGenerator

### 题目
`MyBatis Generator`（MBG）的作用是什么？它能生成哪些文件？

### 核心答案

MBG 是 MyBatis 官方提供的**逆向工程**工具，根据数据库表结构自动生成：

| 生成文件 | 说明 |
|---------|------|
| Model（实体类） | 包含表字段的 JavaBean，自动处理驼峰命名 |
| Mapper 接口 | CRUD 方法定义（`selectByExample` 等） |
| Mapper XML | 对应 CRUD 的 SQL 映射（Example 条件构造） |
| Example 类 | 用于构建动态查询条件 |

**使用方式：**
```bash
# 命令行
java -jar mybatis-generator-core-x.x.x.jar -configfile generatorConfig.xml
```

```xml
<!-- generatorConfig.xml -->
<generatorConfiguration>
    <context id="Mysql" targetRuntime="MyBatis3">
        <jdbcConnection driverClassName="com.mysql.cj.jdbc.Driver"
            connectionURL="jdbc:mysql://localhost:3306/test"
            userId="root" password="root"/>
        <javaModelGenerator targetPackage="model" targetProject="src"/>
        <sqlMapGenerator targetPackage="mapper" targetProject="src"/>
        <javaClientGenerator type="XMLMAPPER" targetPackage="mapper" targetProject="src"/>
        <table tableName="user"/>
    </context>
</generatorConfig>
```

**MBG 生成方法示例：**
- `int insert(User record)`
- `User selectByPrimaryKey(Long id)`
- `List<User> selectByExample(UserExample example)` — 条件查询
- `int updateByExample(User record, UserExample example)` — 条件更新

### 追问方向
- `targetRuntime="MyBatis3DynamicSql"` 和 `"MyBatis3"` 区别？
- MBG 生成的 Example 类如何用于条件构造？Example 的局限性是什么？
- MBG 如何处理联合主键、Blob 字段？

### 避坑提示
- MBG 生成的 XML 中的列名使用全小写，而 Java 属性是驼峰，需要在 generatorConfig.xml 中配置 `enableCountByExample="false"` 等开关减少不需要的方法。
- 每次重新生成会覆盖已有文件，**自定义 SQL 需要写在单独的 XML 中**，不要写在 MBG 生成的文件里。
- 建议使用 `<plugin type="org.mybatis.generator.plugins.EqualsHashCodePlugin"/>` 等插件补充生成代码。

---

## 第 14 题：MyBatis 与 Spring 集成

### 题目
MyBatis 与 Spring 集成时，`SqlSessionFactoryBean` 的作用是什么？Mapper 是如何被注入到 Spring 容器的？

### 核心答案

`SqlSessionFactoryBean` 是 MyBatis-Spring 集成的核心，它让 MyBatis 融入 Spring IoC 容器。

**核心配置：**
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"/>
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>
```

**`SqlSessionFactoryBean` 职责：**
1. 实现 `FactoryBean<SqlSessionFactory>`，Spring 初始化时回调 `getObject()` 创建 `SqlSessionFactory`
2. 加载 MyBatis 配置文件（`configLocation`）和 mapper xml（`mapperLocations`）
3. 注册 `typeAliases`、`plugins`、`typeHandlers` 等

**Mapper 扫描注册原理：**
- `MapperScannerConfigurer`（`BeanDefinitionRegistryPostProcessor`）扫描 `basePackage` 下所有 Mapper 接口
- 为每个接口创建 `MapperFactoryBean`（`SqlSessionDaoSupport` 子类）的 `BeanDefinition`
- Spring 注入时，通过 `SqlSession` 获取 Mapper 代理对象

```java
// 源码简化
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }
}
```

### 追问方向
- `SqlSessionTemplate` 和 `SqlSessionFactory` 的区别？
- `@MapperScan` 注解和 `MapperScannerConfigurer` XML 配置的优劣？
- Mapper 方法上是否可以加 `@Transactional`？

### 避坑提示
- `SqlSessionFactoryBean` 必须注入 `dataSource`，否则启动报错。
- `MapperScannerConfigurer` 的 `sqlSessionFactoryBeanName` 和 `sqlSessionFactoryBean` 二选一，前者更灵活（避免早期初始化问题）。
- Spring 管理的事务与 MyBatis 一级缓存的交互：`commit()` 时清缓存，`rollback()` 不清。

---

## 第 15 题：懒加载与 N+1 问题

### 题目
MyBatis 懒加载的原理是什么？它如何引发 N+1 问题？`fetchType=Lazy` 的使用场景是什么？

### 核心答案

**懒加载原理（MyBatis 3 以后使用 CGlib）：**

MyBatis 在结果集映射时，对 `<association>` 和 `<collection>` 使用 `LazyLoadDecorator` 包装。

```
ResultSetHandler.handleResultSets()
    ↓
MetaObject.getValue("orders")  // 触发懒加载
    ↓
LazyLoader.load()
    ↓
再次执行 SELECT 查询关联数据
```

**触发时机：** 访问懒加载对象的 getter 方法时，触发 `LazyLoader.load()`，通过动态代理执行额外 SQL。

**N+1 问题示例：**
```xml
<!-- 一条查询获取 10 个 User -->
SELECT * FROM user;

<!-- 每个 User 触发一次懒加载获取 Orders -->
SELECT * FROM orders WHERE user_id = 1;  -- N=1
SELECT * FROM orders WHERE user_id = 2;  -- N=2
...
SELECT * FROM orders WHERE user_id = 10; -- N=10
```
共执行 **1 + 10 = 11 条 SQL**。

**解决方案：**
```xml
<!-- 方式一：嵌套结果集，一次查询 -->
<association property="orders" resultMap="orderMap"
    fetchType="eager"/>
<!-- 或 -->
<association property="orders" column="id"
    select="com.xxx.OrderMapper.selectByUserId"/>
```

**`fetchType` 属性：**
- `fetchType="lazy"`：使用全局懒加载策略（默认）
- `fetchType="eager"`：立即加载，忽略全局设置

### 追问方向
- 懒加载的动态代理是 `Javassist` 还是 `Cglib`？如何配置切换？
- 懒加载在 Spring Boot 中如何开启/关闭？
- 如何利用 `groupBy` 避免嵌套查询的 N+1？

### 避坑提示
- 懒加载在 Web 场景下（JSON 序列化）访问 getter 时触发 SQL，若序列化工具（如 Jackson）未正确配置，可能导致懒加载失效或懒加载未触发。
- 在事务内懒加载触发 N+1 会产生大量额外 SQL，**高并发场景下慎用懒加载**。
- `fetchType` 只在嵌套 select 场景下生效，嵌套结果集始终是立即加载。

---

## 第 16 题：MyBatis Plus

### 题目
MyBatis Plus 相比 MyBatis 有哪些核心增强？`QueryWrapper` 和 `ActiveRecord` 的用法是什么？

### 核心答案

MyBatis Plus（MP）是 MyBatis 的增强工具，**不改变 MyBatis 原有机制**，提供以下核心增强：

| 增强点 | 说明 |
|--------|------|
| CRUD 通用接口 | `IService`、`ServiceImpl`，无需写 CRUD SQL |
| `QueryWrapper` | 条件构造器，告别 XML 中的动态 SQL |
| ActiveRecord | 继承 `Model<T>`，实体自带 CRUD 方法 |
| 自动填充 | `create_time` / `update_time` 自动填充 |
| 逻辑删除 | `@TableLogic` 注解，删除变更为更新 |
| 分页插件 | 内置分页，无需手动处理 `RowBounds` |
| 性能分析 | SQL 执行耗时日志 |

**QueryWrapper 示例：**
```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.lambda()
    .select(User::getName, User::getAge)
    .ge(User::getAge, 18)
    .likeRight(User::getName, "张")
    .orderByDesc(User::getCreateTime)
    .last("LIMIT 10");

List<User> users = userMapper.selectList(wrapper);
```

**ActiveRecord 示例：**
```java
// 继承 Model<User>
public class User extends Model<User> {
    private Long id;
    private String name;
}

// 使用
User user = new User();
user.setName("张三");
user.insert();   // 相当于 INSERT
user.update();   // 相当于 UPDATE WHERE id = ?
user.deleteById(); // 相当于 DELETE
User u = user.selectById(1L); // 相当于 SELECT
```

### 追问方向
- MyBatis Plus 分页插件的原理与 PageHelper 有何不同？
- MP 的 `IService` 和 `Mapper` 的关系是什么？
- MP 如何与 MyBatis 原生 XML 共存？

### 避坑提示
- MP 的 `QueryWrapper` 查询条件很多，面试时至少能写出 `eq`、`ne`、`like`、`in`、`orderBy` 等常见用法。
- MP 是 MyBatis 的增强层，不是替代品，原生 XML 写法与 MP 完全兼容。
- MP 分页查询**必须配置分页插件**，否则 `Page` 对象不会拦截 SQL 而变成普通查询。

---

## 第 17 题：批量操作

### 题目
MyBatis 批量插入的几种实现方式？`BatchExecutor` 和 `foreach` 批量INSERT的区别？

### 核心答案

**方式一：`BatchExecutor` 批量执行**
```java
// 必须使用 SimpleExecutor / BatchExecutor，默认 REUSE 不支持 batch
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
    UserMapper mapper = session.getMapper(UserMapper.class);
    for (User user : users) {
        mapper.insertUser(user);
    }
    session.flushStatements(); // 立即发送
    session.commit();
}
```

对应 Mapper XML：
```xml
<insert id="insertUser" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO user (name, age) VALUES (#{name}, #{age})
</insert>
```

**方式二：`insertBatch` 单条 SQL，foreach 展开**
```xml
<insert id="insertBatch" parameterType="list">
    INSERT INTO user (name, age) VALUES
    <foreach collection="list" item="u" separator=",">
        (#{u.name}, #{u.age})
    </foreach>
</insert>
```

**方式三：MyBatis Plus 的 `saveBatch`**
```java
userService.saveBatch(userList); // 内部使用 BATCH executor + 分批
```

**三者对比：**

| 方式 | SQL 数量 | 网络开销 | 主键回填 | 适用场景 |
|------|---------|---------|---------|---------|
| BatchExecutor | N 条（预编译复用） | 低 | 支持 | 大批量插入（千级+） |
| foreach INSERT | 1 条 | 极低 | 部分（需配置） | 少量数据（百级以下） |
| saveBatch(MP) | 自动分批 | 自动优化 | 支持 | 推荐使用 |

### 追问方向
- `BatchExecutor` 的 `flushStatements()` 和 `commit()` 区别？
- 批量插入时 `useGeneratedKeys` 和 `<selectKey>` 同时配置会冲突吗？
- Oracle 下如何实现批量插入（语法差异）？

### 避坑提示
- MySQL 批量插入单条 SQL 过长（超过 `max_allowed_packet`）会失败，需要控制 batch 大小或切换为 BatchExecutor。
- `BatchExecutor` 在 `commit()` 前不会真正执行，`flushStatements()` 可以手动触发。
- JDBC 的 `addBatch()` 也有 SQL 长度限制，不要无限制扩大 batch size。

---

## 第 18 题：自定义 TypeHandler

### 题目
如何自定义 `TypeHandler` 处理 JSON 字段（如实体类中有 `Map<String, Object>` 属性）？

### 核心答案

`TypeHandler` 是 MyBatis 类型转换的核心接口，`JdbcType` ↔ `JavaType` 双向转换。

**自定义 JSON TypeHandler 步骤：**

```java
// Step 1：定义 Handler，实现 TypeHandler 或继承 BaseTypeHandler
public class JsonTypeHandler<T> extends BaseTypeHandler<T> {
    private Class<T> type;
    private ObjectMapper objectMapper = new ObjectMapper();

    public JsonTypeHandler(Class<T> type) {
        this.type = type;
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                     T parameter, JdbcType jdbcType) throws SQLException {
        // Java → JDBC（写入数据库）
        ps.setString(i, objectMapper.writeValueAsString(parameter));
    }

    @Override
    public T getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        // JDBC → Java
        String json = rs.getString(columnName);
        return parseJson(json);
    }

    @Override
    public T getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return parseJson(rs.getString(columnIndex));
    }

    @Override
    public T getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        return parseJson(cs.getString(columnIndex));
    }

    private T parseJson(String json) {
        if (json == null || json.isEmpty()) return null;
        return objectMapper.readValue(json, type);
    }
}
```

**Step 2：注册 Handler**
```xml
<!-- 方式一：mapper xml 中指定 -->
<result column="extra" property="extra"
    typeHandler="com.example.handler.JsonTypeHandler"/>

<!-- 方式二：mybatis-config.xml 全局注册 -->
<typeHandlers>
    <typeHandler handler="com.example.handler.JsonTypeHandler"
                 javaType="java.util.Map"/>
</typeHandlers>
```

**Step 3：实体类使用**
```java
public class Order {
    private Long id;
    private String name;
    private Map<String, Object> extra; // JSON 字段
}
```

### 追问方向
- `BaseTypeHandler` 和 `TypeHandler` 的区别？为何优先继承 `BaseTypeHandler`？
- 如何让 TypeHandler 同时支持 `List<User>` 和 `Map<String, Object>` 等多种类型？
- JSON 解析时遇到未知属性如何处理（`ObjectMapper` 配置）？

### 避坑提示
- Handler 注册后不会自动扫描，需在 `mybatis-config.xml` 或注解中显式配置。
- JSON 字段在 MySQL 中通常用 `TEXT` 或 `JSON` 类型存储，Oracle 用 `CLOB`。
- `ObjectMapper` 要配置 `configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)` 防止字段不匹配抛异常。

---

## 第 19 题：MyBatis 配置

### 题目
MyBatis 配置文件中 `<settings>` 和 `<typeAliases>` 的常见配置项有哪些？

### 核心答案

**`<settings>` 核心配置：**

```xml
<settings>
    <!-- 开启驼峰命名自动映射（a_column → aColumn） -->
    <setting name="mapUnderscoreToCamelCase" value="true"/>

    <!-- 全局懒加载（association/collection 默认懒加载） -->
    <setting name="lazyLoadingEnabled" value="true"/>

    <!-- 积极加载（调用任意方法都触发懒加载）vs 消极加载（调用特定方法触发） -->
    <setting name="aggressiveLazyLoading" value="false"/>

    <!-- 二级缓存开关（默认 true，一级缓存不可关闭） -->
    <setting name="cacheEnabled" value="true"/>

    <!-- 延时加载触发方法（默认 "equals,clone,hashCode,toString"） -->
    <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode"/>

    <!-- 日志实现（SLF4J, LOG4J, LOG4J2, JDK_LOGGING, COMMONS_LOGGING, NO_LOGGING, ...） -->
    <setting name="logImpl" value="SLF4J"/>

    <!-- 全局映射器（决定 JDBC null 值如何处理） -->
    <setting name="jdbcTypeForNull" value="NULL"/>

    <!-- 超过此时间（秒）抛出异常，0 不限 -->
    <setting name="defaultStatementTimeout" value="25000"/>

    <!-- fetchSize（每次从数据库读取的行数，Oracle 常用） -->
    <setting name="defaultFetchSize" value="100"/>
</settings>
```

**`<typeAliases>` 配置：**

```xml
<!-- 方式一：单个注册 -->
<typeAliases>
    <typeAlias type="com.example.model.User" alias="User"/>
    <typeAlias type="com.example.model.Order" alias="Order"/>
</typeAliases>

<!-- 方式二：包扫描（所有类自动注册为类名小写的别名） -->
<typeAliases>
    <package name="com.example.model"/>
</typeAliases>

<!-- 方式三：注解别名 -->
@Alias("User")
public class User { ... }
```

**MyBatis 内置类型别名（不需配置）：**

| Java 类型 | MyBatis alias |
|-----------|--------------|
| `String` | string |
| `Integer` | int / integer |
| `Long` | long |
| `Boolean` | boolean |
| `Map` | map |
| `List` | list / arraylist |
| `Date` | date |

### 追问方向
- `mapUnderscoreToCamelCase` 设置后还需要 `resultMap` 吗？
- `lazyLoadingEnabled` 和 `fetchType="lazy"` 的关系？
- `jdbcTypeForNull` 默认值是什么？什么场景需要改成 `VARCHAR`？

### 避坑提示
- `mapUnderscoreToCamelCase` 只对自动映射有效，手写 `resultMap` 仍需显式映射。
- `aggressiveLazyLoading` 在 MyBatis 3.x 默认为 `true`（调用对象任意方法都会触发懒加载），新版已改为 `false`，面试时注意版本区分。
- `typeAliases` 包扫描时，若多个类同名（不同 package），别名会冲突，需用 `@Alias` 注解区分。

---

## 第 20 题：常见问题——UUID 主键与 Oracle 序列

### 题目
使用 MyBatis 生成主键时，UUID 和 Oracle 序列的处理方式有何不同？

### 核心答案

**UUID 主键处理：**

```java
// 方式一：Java 代码生成（实体类中处理）
public class User {
    private String id;

    @PrePersist
    public void generateId() {
        this.id = UUID.randomUUID().toString().replace("-", "");
    }
}
```

```xml
<!-- 方式二：INSERT 时由 MySQL 函数生成 -->
<insert id="insertUser" parameterType="User">
    INSERT INTO user (id, name) VALUES (REPLACE(UUID(), '-', ''), #{name})
</insert>

<!-- 方式三：JDBC 全局唯一 ID -->
<insert id="insertUser" useGeneratedKeys="false" keyProperty="id">
    <selectKey resultType="string" order="BEFORE">
        SELECT REPLACE(UUID(), '-', '')
    </selectKey>
    INSERT INTO user (id, name) VALUES (#{id}, #{name})
</insert>
```

**Oracle 序列处理：**

```xml
<!-- 方式一：selectKey 获取序列 -->
<insert id="insertUser" parameterType="User">
    <selectKey resultType="long" order="BEFORE" keyProperty="id">
        SELECT SEQ_USER.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO user (id, name) VALUES (#{id}, #{name})
</insert>

<!-- 方式二：JDBC 3.0 自增（Oracle 12c+ 支持 IDENTITY） -->
<!-- Oracle 12c 之前的版本不支持自增主键，必须用序列 -->
```

**两者对比：**

| 特性 | UUID | Oracle 序列 |
|------|------|------------|
| 类型 | 字符串（36字符）/ 128位 | 数字 |
| 存储空间 | MySQL CHAR(36) 或 VARCHAR(32) | NUMBER(10-19) |
| 生成方式 | 应用层或 DB 函数 | 数据库序列对象 |
| 索引效率 | 字符串无序，插入热点分散 | 有序数字，范围扫描效率高 |
| 主键查询性能 | 较差（字符串比较） | 优 |
| 分页影响 | UUID 无序导致严重页分裂 | 序列有序，插入连续 |

### 追问方向
- 为什么推荐使用数字型主键而非字符串型？
- MySQL 的 `auto_increment` 与 Oracle 序列如何配置 `useGeneratedKeys`？
- MyBatis Plus 的 `IdType.AUTO` 和 `IdType.ASSIGN_ID` 分别对应什么策略？

### 避坑提示
- MyBatis 的 `useGeneratedKeys` 只对支持自增的数据库（MySQL、SQL Server）有效，Oracle 不支持，需用 `<selectKey>`。
- UUID 字符串主键在索引页分裂上比数字型严重得多，高并发写入场景下性能差距明显。
- Oracle 序列使用 `@SelectKey` 时注意 `order="BEFORE"`，在 INSERT 之前执行；若用触发器生成，则 `order="AFTER"`。

---

## 附录：面试速查索引

| 序号 | 主题 | 核心关键词 |
|------|------|-----------|
| 1 | 整体架构 | SqlSessionFactoryBuilder / SqlSessionFactory / SqlSession / Executor |
| 2 | 缓存 | 一级 SqlSession 级 / 二级 Mapper 级 / CachingExecutor |
| 3 | #{} vs ${} | 预编译 vs 字符串拼接 / SQL注入 |
| 4 | 分页 | RowBounds内存分页 / PageHelper拦截器 |
| 5 | 动态SQL | if/where/set/trim/foreach/choose |
| 6 | Mapper绑定 | JDK动态代理 / MapperProxyFactory |
| 7 | 初始化 | XPath / XMLConfigBuilder / MappedStatement |
| 8 | Executor | Simple/Reuse/Batch 三种执行器 |
| 9 | StatementHandler | prepare / parameterize / query |
| 10 | ResultSetHandler | 嵌套结果映射 / NestedResultSetHandler |
| 11 | 插件机制 | @Intercepts / @Signature / Plugin.wrap |
| 12 | 插件应用 | 分页 / 乐观锁 |
| 13 | MyBatisGenerator | 逆向工程 / Model + Mapper + Example |
| 14 | Spring集成 | SqlSessionFactoryBean / MapperFactoryBean |
| 15 | 懒加载 | CGlib代理 / N+1问题 / fetchType |
| 16 | MyBatis Plus | QueryWrapper / ActiveRecord / IService |
| 17 | 批量操作 | BatchExecutor / foreach INSERT |
| 18 | TypeHandler | BaseTypeHandler / JSON序列化 |
| 19 | 配置 | mapUnderscoreToCamelCase / typeAliases |
| 20 | UUID与序列 | selectKey / useGeneratedKeys / order |
