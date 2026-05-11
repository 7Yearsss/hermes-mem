# Vue3 + TypeScript 前端面试题（第14部分）

> **面试方向**: Vue 3 深度 + TypeScript 进阶 + 项目实践  
> **项目背景**: WMS 仓库管理系统（`/usr/src/wms-web`），技术栈 Vue 3 + TypeScript + Vite + Pinia + Element Plus + VueUse  
> **适配层级**: 3-5 年前端工程师 / Java 转前端全栈

---

## 第 1 题：Vue3 新特性——Composition API / setup 语法 / 响应式系统 / ref vs reactive

### 题目

你之前做 Java，现在转前端，问几个 Vue 3 的核心变化。**Vue 3 相比 Vue 2 最核心的新特性是什么？setup 语法糖怎么用？响应式系统从 Object.defineProperty 换成了什么？`ref` 和 `reactive` 有什么区别，各自适合什么场景？**

> 提示：WMS 前端项目 `wms-web` 中，组件大量使用 `const xxx = ref<T>(...)` 和 `const state = reactive<T>({ ... })` 模式。

### 核心答案

**1. Composition API 核心目的**：解决 Options API 中"逻辑关注点分散"的问题——同一个功能的代码（data / computed / methods / watch）散落在不同选项里，Composition API 允许按逻辑功能组织代码（叫"逻辑关注点组合"）。

**2. setup 语法糖**：
```typescript
// 完整写法
export default defineComponent({
  setup(props, context) {
    const count = ref(0)
    const doubled = computed(() => count.value * 2)
    function add() { count.value++ }
    return { count, doubled, add }
  }
})

// <script setup> 语法糖（wms-web 通用写法）
<script setup lang="ts">
import { ref, computed } from 'vue'
const count = ref(0)          // 自动暴露到模板
const doubled = computed(() => count.value * 2)
</script>
```

`<script setup>` 编译后等价于上述完整写法，但更简洁——所有顶层变量/函数直接暴露给模板，**不需要 return**。

**3. 响应式系统变化**：
- Vue 2：`Object.defineProperty`，需要递归遍历对象每个属性，对数组需要覆盖原生方法（如 `push`/`splice`）实现响应式
- Vue 3：`Proxy` 代理整个对象，惰性递归（只有访问深层属性时才递归代理），性能更好，支持数组下标监听

**4. ref vs reactive 对比**：

| | `ref` | `reactive` |
|---|---|---|
| 适用类型 | 基础类型 + 对象 | 对象/数组（proxy） |
| 访问方式 | `.value`（模板中自动解包） | 直接访问（不需要 `.value`） |
| 解构行为 | 解构后**丢失响应式**（需 `toRefs`） | 解构后**仍保留响应式**（proxy 特性） |
| 重新赋值 | `ref` 重新赋值`= ref(newVal)` 保留响应式 | `reactive` 重新赋值 **丢失响应式**，只能 `Object.assign` |
| 泛型支持 | `ref<T>(initialValue)` | `reactive<T>(obj)` |

```typescript
// ref 适合：基础类型、需要重新赋值、需要泛型参数
const name = ref<string>('')
const list = ref<Item[]>([])         // Item 来自 wms-web 的类型定义
const loading = ref(false)

// reactive 适合：多个相关状态组成一个"状态对象"
const filters = reactive({
  warehouseId: '',
  status: '',
  dateRange: [] as Date[],
})
// 修改单个属性保持响应式：filters.status = 'PENDING'

// 在 WMS 中的实际用法（参考 wms-web/src/app/views/inventory/**/*.vue）
const tableData = ref<InventoryRecord[]>([])
const searchForm = reactive({
  itemCode: '',
  locationCode: '',
  quantity: undefined as number | undefined,
})
```

### 追问方向

1. `ref` 的 `.value` 在模板中为什么不用写？——编译器自动解包
2. `toRefs` 的作用是什么？——把 `reactive` 对象转成纯 `ref` 集合，防止解构丢失响应式
3. `shallowRef` 和 `ref` 的区别？——`shallowRef` 只追踪顶层 `.value` 变化，不追踪深层（性能优化用）

### 避坑提示

- 不要说"ref 用于基础类型，reactive 用于对象"——两者对象都支持，只是风格和 API 语义不同
- `reactive` 解构问题是高频踩坑点：很多人以为解构 reactive 后还保留响应式，实际上解构出的是普通值，需要用 `toRefs` 包一层

---

## 第 2 题：Vue3 响应式原理——Proxy vs Object.defineProperty，Proxy 的 Handler 方法

### 题目

你说 Vue 3 用 Proxy 代替了 Object.defineProperty。**Proxy 的基本语法是什么？它的 handler（拦截器）方法有哪些？Vue 3 实际用了哪几个？相对于 Object.defineProperty，Proxy 解决了哪些问题？**

### 核心答案

**1. Proxy 基本语法**：
```typescript
const target = { name: 'WMS', quantity: 100 }
const proxy = new Proxy(target, {
  get(target, key, receiver) { ... },      // 读取属性
  set(target, key, value, receiver) { ... }, // 写入属性
  deleteProperty(target, key) { ... },    // delete proxy.name
  has(target, key) { ... },               // 'name' in proxy
  ownKeys(target) { ... },                // Object.keys/proxy.forEach
  getOwnPropertyDescriptor(target, key) { ... },
  defineProperty(target, key, descriptor) { ... },
  preventExtensions(target) { ... },
  getPrototypeOf(target) { ... },
  setPrototypeOf(target, proto) { ... },
  isExtensible(target) { ... },
})
```

**2. Vue 3 实际用到的 handler（核心 3 个 + 辅助）**：
- `get`：依赖收集。访问 `obj.foo` 时，Vue 的响应式系统记录"哪个 effect 依赖了这个属性"
- `set`：触发更新。赋值 `obj.foo = 'bar'` 时，通知所有依赖的 effect 重新执行
- `deleteProperty`：删除属性时也要触发更新和清理依赖

**3. 解决的问题**：

| 问题 | Object.defineProperty | Proxy |
|---|---|---|
| 数组下标监听 | 需要覆盖数组方法，不完美 | 直接代理，可监听 `arr[0] = x` |
| 新增属性 | 需要 `Vue.set` | 自动拦截，天然支持 |
| 删除属性 | 需要 `Vue.delete` | 自动拦截 |
| 性能 | 递归遍历所有属性初始化 |惰性代理，访问时才递归 |
| 多层嵌套 | 每一层都要 defineProperty | Proxy 递归代理整条路径 |
| Map/Set/WeakMap/WeakSet | 不支持 | 完全支持 |

**4. Vue 3 的实现要点**：
```typescript
// Vue 3 响应式的核心结构（简化）
function reactive<T extends object>(obj: T): T {
  return new Proxy(obj, {
    get(target, key, receiver) {
      const result = Reflect.get(target, key, receiver)
      // 依赖收集：track(target, key)
      track(target, key)  // 记录当前 activeEffect
      // 如果是对象，递归返回代理（深度响应式）
      if (isObject(result)) {
        return reactive(result)
      }
      return result
    },
    set(target, key, value, receiver) {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      if (oldValue !== value) {
        // 触发更新：trigger(target, key)
        trigger(target, key)  // 通知所有依赖的 effect
      }
      return result
    },
    deleteProperty(target, key) {
      const hadKey = hasOwn(target, key)
      const result = Reflect.deleteProperty(target, key)
      if (hadKey && result) {
        trigger(target, key)  // 删除也触发
      }
      return result
    }
  })
}
```

### 追问方向

1. `Reflect` 是什么？为什么要用 `Reflect.get/set` 而不是直接操作 target？——`Reflect` 是 ES6 提供的操作对象的标准 API，和 Proxy 配套使用，保证 `this` 绑定正确（receiver 是 proxy 实例本身，用于继承场景）
2. Vue 3 中 `readonly` 和 `shallowReactive` 怎么用 Proxy 实现？——`readonly` 的 handler 不提供 set/delete；`shallowReactive` 在 get 时不递归调用 `reactive()`
3. `Proxy` 能完全替代 `Object.defineProperty` 吗？——是的，Vue 3 已经完全抛弃 Object.defineProperty

### 避坑提示

- 手写响应式系统时，容易忽略 `receiver` 参数——它是代理对象本身，用于处理继承场景（如原型链上有代理对象）
- Vue 3 中 `reactive()` 传入已代理对象会直接返回同一个代理，**不会重复代理**

---

## 第 3 题：setup 函数——执行时机、computed/watch/watchEffect 在 setup 中使用

### 题目

你说用 `<script setup>`。**setup 函数什么时候执行？它和 beforeCreate/created 是什么关系？computed、watch、watchEffect 在 setup 中如何使用？它们三个有什么区别？**

> 提示：WMS 前端项目中，表单校验、列表搜索、分页等逻辑大量在 setup 中用 watch 处理副作用。

### 核心答案

**1. setup 执行时机**：
- `setup` 在 **beforeCreate 之前**执行（组件实例创建前），此时 `this` 是 `undefined`
- 等价于 Vue 2 的 `beforeCreate + created` 两个钩子合并的阶段
- setup 中没有 `this`，所有响应式和生命周期都是通过导入的函数获取

```typescript
export default defineComponent({
  beforeCreate() {
    console.log('beforeCreate - this:', this)  // Vue 组件实例
  },
  setup() {
    console.log('setup - this:', this)  // undefined
    // 此时 data/computed/methods 都还没初始化
    return {}
  }
})
```

**2. computed**：
```typescript
// 只读计算属性
const doubled = computed(() => count.value * 2)

// 可写计算属性（WMS中少见）
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (val) => { [firstName.value, lastName.value] = val.split(' ') }
})

// 带泛型（类型安全）
interface InventoryItem { id: string; name: string; quantity: number }
const lowStockItems = computed<InventoryItem[]>(() =>
  inventoryList.value.filter(item => item.quantity < item.threshold)
)
```

**3. watch**：
```typescript
// 监听单个 ref
watch(searchKeyword, (newVal, oldVal) => {
  fetchData()
})

// 监听多个 ref
watch([warehouseId, status], ([newWh, newSt], [oldWh, oldSt]) => {
  fetchData()
})

// 监听 reactive 对象某个属性（必须用函数）
watch(() => filters.status, (newVal) => {
  fetchData()
})

// 深度监听
watch(() => deepObj, (newVal) => { ... }, { deep: true })

// 立即执行（Immediate）
watch(source, cb, { immediate: true })  // 组件创建时立即执行一次
```

**4. watchEffect**：
```typescript
// 自动追踪所有依赖，执行时机：初始化 + 依赖变化时
watchEffect(async () => {
  // 自动追踪 this.items.value 的读取
  const res = await fetchInventory(this.items.value)
  this.tableData = res.data
})

// 清除副作用（如取消请求）
watchEffect(async (onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())  // 组件卸载或下次重新执行前调用
  const data = await fetchData(controller.signal)
})
```

**5. computed vs watch vs watchEffect 对比**：

| | computed | watch | watchEffect |
|---|---|---|---|
| 用途 | 计算派生值（同步纯函数） | 监听特定数据变化执行副作用（异步/有副作用） | 任何副作用，自动追踪依赖 |
| 返回值 | 新 ref（可复用） | 不返回（用于副作用） | 不返回 |
| 执行时机 | 懒执行（被访问时） | 依赖变化时（默认懒） | 初始化立即执行一次 |
| 适用场景 | 格式转换、过滤、统计 | 异步请求、条件触发 | 任何需要立即执行的副作用 |
| WMS 示例 | `lowStockItems = computed(() => ...)` | `watch(keyword, fetchData)` | `watchEffect(() => fetchUserInfo(userId.value))` |

### 追问方向

1. `watch` 和 `watchEffect` 的根本区别？——`watch` 显式指定监听源，`watchEffect` 自动追踪所有依赖
2. 如何停止一个 watch？——`const unwatch = watch(...)` 然后 `unwatch()`，或用 `onUnmounted` 中调用
3. 监听 `reactive` 对象时，用函数和不用的区别？——`watch(obj, ...)` 监听整个对象（deep），`watch(() => obj.prop, ...)` 精确监听某个属性

### 避坑提示

- `watchEffect` 初始化时**立即执行一次**——很多人以为它是懒的，忘了在 onCleanup 清理副作用（如取消请求），导致竞态条件
- 监听 `reactive` 对象属性时**必须用 getter 函数** `() => obj.prop`，否则拿到的不是响应式的
- `computed` 内部不要有副作用（如 `fetchData`），只做纯计算

---

## 第 4 题：生命周期钩子——Vue2 vs Vue3 生命周期对比，onMounted/onUpdated/onUnmounted

### 题目

**Vue 2 和 Vue 3 的生命周期钩子有什么区别？Vue 3 新增了哪些钩子？onMounted、onUpdated、onUnmounted 分别在什么时候触发？组件卸载时 onUnmounted 适合做什么清理操作？**

> 提示：WMS 前端中，onMounted 用于初始加载数据，onUnmounted 用于销毁 WebSocket 连接、清除定时器。

### 核心答案

**1. 生命周期图对比**：

```
Vue 2:
beforeCreate → created → beforeMount → mounted → beforeUpdate → updated → beforeDestroy → destroyed

Vue 3 (Composition API):
setup (beforeCreate/created 同级)
→ onBeforeMount → onMounted
→ onBeforeUpdate → onUpdated
→ onBeforeUnmount → onUnmounted
新增：onErrorCaptured / onRenderTracked / onRenderTriggered / onActivated / onDeactivated
```

**2. 关键区别**：
- `beforeCreate`/`created` → 被 `setup()` 替代（setup 在它们之前执行）
- 销毁阶段：`beforeDestroy`/`destroyed` → `onBeforeUnmount`/`onUnmounted`（语义更明确）
- Vue 3 新增 **调试钩子**：`onRenderTracked`（首次渲染收集依赖）、`onRenderTriggered`（重新渲染触发点）——用于性能分析

**3. 各钩子触发时机**：

```typescript
import { onMounted, onUpdated, onUnmounted, onBeforeMount, onBeforeUpdate, onBeforeUnmount } from 'vue'

// beforeMount / onBeforeMount：DOM 挂载之前，组件已解析完模板
onBeforeMount(() => {
  // 可以访问 this（如果用 Options API）
  console.log('DOM 即将挂载')
})

// mounted / onMounted：DOM 挂载完成，可访问 $el/$refs
onMounted(() => {
  // WMS 实际用法：初始化图表、请求数据、绑定事件
  initChart()
  fetchInventoryList()
  window.addEventListener('resize', handleResize)
})

// beforeUpdate / onBeforeUpdate：状态变更后，DOM 更新前
onBeforeUpdate(() => {
  // 可以获取更新前的 DOM 状态（如果需要）
})

// updated / onUpdated：DOM 更新完成后
onUpdated(() => {
  // 避免在此更新状态（可能引起无限循环）
  // 适合：DOM 依赖的第三方插件 resize
})

// beforeUnmount / onBeforeUnmount：组件卸载前
onBeforeUnmount(() => {
  // 清理准备工作
})

// unmounted / onUnmounted：组件卸载完成，实例销毁
onUnmounted(() => {
  // WMS 实际用法：
  // 1. 清除定时器
  clearInterval(timerId)
  clearTimeout(timeoutId)
  // 2. 取消请求（AbortController）
  controller.abort()
  // 3. 移除事件监听（防止内存泄漏）
  window.removeEventListener('resize', handleResize)
  // 4. 关闭 WebSocket
  ws.close()
  // 5. 取消订阅
  eventBus.off('user:login', handler)
})
```

**4. 父子组件生命周期执行顺序**：
```
父 beforeCreate → 父 setup → 父 onBeforeMount
  → 子 beforeCreate → 子 setup → 子 onBeforeMount → 子 mounted
→ 父 mounted
```
先子后父（子挂载完成，父才挂载完成）。

### 追问方向

1. `onUpdated` 中能改状态吗？——可以但不推荐，容易引起无限循环（状态变→更新→updated→再变→更新）
2. `keep-alive` 缓存组件的生命周期变化？——激活时 `onActivated`，停用时 `onDeactivated`，跳过 `onMounted`/`onUnmounted`
3. `setup` 中可以直接用 `onMounted` 多次注册吗？——可以，多次调用会按注册顺序执行

### 避坑提示

- **内存泄漏是 Vue 常见 bug**：onUnmounted 中忘记清除定时器、事件监听、WebSocket 等，导致组件销毁后仍在运行
- WMS 中如果用 `setInterval` 做轮询，**必须在 onUnmounted 中 clearInterval**，否则即使切换路由组件销毁了，定时器还在跑
- 不要在 `onUpdated` 里调用 `nextTick` ——它本身就是更新完成后的钩子，DOM 已经是最新的

---

## 第 5 题：TypeScript 类型系统——基础类型 / 接口 / 类型别名 / 联合类型 / 交叉类型

### 题目

你之前做 Java，Java 的类型系统你熟悉。**TypeScript 的基础类型有哪些？接口（interface）和类型别名（type alias）有什么区别？联合类型（union）和交叉类型（intersection）分别用在什么场景？WMS 项目中如何定义一个物料的 TypeScript 类型？**

> 提示：WMS 前端在 `/usr/src/wms-web/src/app/apis/` 下有大量 API 类型定义。

### 核心答案

**1. 基础类型**：
```typescript
// 基础类型
const name: string = 'WMS'
const quantity: number = 100
const active: boolean = true
const list: null = null
const u: undefined = undefined
const sym: symbol = Symbol('id')
const big: bigint = 100n

// 数组
const items: string[] = ['A', 'B']
const counts: Array<number> = [1, 2, 3]  // 泛型语法

// 元组（固定长度+固定类型）
const record: [string, number, boolean] = ['SKU001', 100, true]

// 枚举
enum OrderStatus { PENDING = 'PENDING', PROCESSING = 'PROCESSING', COMPLETED = 'COMPLETED' }
const status: OrderStatus = OrderStatus.PENDING

// any / unknown / never
const unknownVal: unknown = JSON.parse(input)  // 安全的 any，比 any 更严格
const neverVal: never = (() => { throw new Error() })()
```

**2. 接口 vs 类型别名**：

```typescript
// 接口（interface）
interface Item {
  id: string
  name: string
  quantity: number
  category?: string  // 可选属性
  readonly sku: string  // 只读属性
}
interface ItemWithPrice extends Item {
  price: number
}

// 类型别名（type alias）
type ItemType = {
  id: string
  name: string
  quantity: number
}

// 关键区别：
// 1. interface 可以声明合并（同名 interface 自动合并），type 不能
// 2. interface 描述对象结构更直观；type alias 更灵活（可联合、可原始类型）
// 3. class 实现接口：class MyClass implements Item { ... }
```

**3. 联合类型（Union）**：
```typescript
// 某个字段可以是多种类型
type stringOrNumber = string | number
type loadingState = 'idle' | 'loading' | 'success' | 'error'
type responseData = InventoryItem | null

// 函数参数联合
function process(id: string | number): void {
  if (typeof id === 'string') {
    console.log(id.toUpperCase())  // TS 知道是 string
  } else {
    console.log(id.toFixed(2))     // TS 知道是 number
  }
}
```

**4. 交叉类型（Intersection）**：
```typescript
// 合并多个类型（AND 关系）
type Employee = { name: string } & { age: number } & { role: string }
type ExtendedItem = Item & { price: number } & { supplier: Supplier }

// 实际 WMS 用法：合并基础字段 + 业务扩展字段
interface BaseEntity {
  id: string
  createdAt: Date
  createdBy: string
}
interface InventoryEntity extends BaseEntity {
  itemCode: string
  locationCode: string
  quantity: number
}
```

**5. WMS 项目中的实际类型定义**（参考 `wms-web/src/app/apis/`）：
```typescript
// 物料主数据 - 定义在 api 文件中
export interface ItemMaster {
  id: string
  itemCode: string           // 物料编码
  itemName: string           // 物料名称
  categoryId: string
  uomId: string              // 单位
  grossWeight: number
  netWeight: number
  hazardFlag: boolean        // 危险品标识
  sku: string                // 库存单位
  status: 'ACTIVE' | 'INACTIVE' | 'ARCHIVED'
  // 动态自定义字段
  dynTxtProperty01?: string
  dynTxtProperty02?: string
}

// API 统一响应结构
export interface ApiResponse<T> {
  code: number
  message: string
  data: T
  timestamp: number
}

// 分页参数
export interface PageQuery {
  page: number
  pageSize: number
  sortBy?: string
  sortOrder?: 'asc' | 'desc'
}

export interface PageResult<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}
```

### 追问方向

1. `interface` 能用 `extends` 继承，type alias 用什么？——用交叉类型 `&`；`type A = B & C`
2. `void` 和 `undefined` 有什么区别？——`void` 表示不返回值的函数，实际返回 `undefined`；`undefined` 是具体的类型值
3. `type` 和 `interface` 哪个更好？——没有绝对答案；`interface` 适合描述对象结构和声明合并；`type` 适合联合类型、原始类型别名、条件类型

### 避坑提示

- 避免滥用 `any`——WMS 中间层规范明确要求"禁止 `any` 逃逸到 Service 层"
- 联合类型常用字符串字面量联合（`'PENDING' | 'COMPLETED'`）而不是枚举，更轻量
- `readonly` 只防止直接赋值，不能防止 `Object.freeze` 之外的深层修改

---

## 第 6 题：TypeScript 泛型——泛型函数 / 泛型接口 / 泛型约束，`<T>` 语法

### 题目

Java 也有泛型。**TypeScript 泛型的 `<T>` 语法和 Java 类似吗？泛型函数、泛型接口怎么定义？泛型约束（extends）有什么用？请结合 WMS 的实际场景举个例子。**

### 核心答案

**1. 泛型函数**：
```typescript
// 泛型函数：函数名后加 <T>，参数和返回值用 T
function identity<T>(arg: T): T {
  return arg
}

// 调用时可以显式指定类型
const result = identity<string>('hello')
// 也可以让 TS 自动推断
const result2 = identity(123)  // T 推断为 number

// 多个类型参数
function pair<K, V>(key: K, value: V): [K, V] {
  return [key, value]
}
```

**2. 泛型接口**：
```typescript
// API 响应泛型接口
interface ApiResponse<T> {
  code: number
  message: string
  data: T
  timestamp: number
}

// WMS 中的实际用法
interface InventoryApiResponse extends ApiResponse<InventoryItem[]> {
  traceId: string  // 扩展字段
}

// 泛型接口约束
interface HasId {
  id: string
}

// 泛型约束：T 必须有 id 属性
function getById<T extends HasId>(list: T[], id: string): T | undefined {
  return list.find(item => item.id === id)
}

const item = getById(inventoryList, 'SKU001')  // item 类型是 InventoryItem
```

**3. 泛型约束（extends）**：
```typescript
// 基本约束：T 必须是某种结构
function process<T extends object>(obj: T): void {
  console.log(Object.keys(obj))
}

// 约束为特定类型
function merge<T extends Item, U extends Item>(a: T, b: U): T & U {
  return { ...a, ...b }
}

// keyof 约束：K 必须是 T 的键
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const itemName = getProperty(item, 'itemName')  // string 类型
// getProperty(item, 'nonExist')  // 编译报错
```

**4. WMS 实际场景**：
```typescript
// 泛型分页查询
interface PageQuery {
  page: number
  pageSize: number
}

function fetchPaged<T>(api: () => Promise<ApiResponse<T[]>>, query: PageQuery) {
  return api().then(res => ({
    data: res.data,
    total: res.total,
    ...query
  }))
}

// 泛型 Form 提交（工具函数）
async function submitForm<T extends object, R>(
  formData: T,
  submitFn: (data: T) => Promise<ApiResponse<R>>
): Promise<R> {
  const res = await submitFn(formData)
  if (res.code !== 200) throw new Error(res.message)
  return res.data
}

// Vue 3 defineComponent 泛型
defineComponent<{
  items: InventoryItem[]
  loading: boolean
}>()
```

### 追问方向

1. `<T extends unknown>` 和 `<T>` 有什么区别？——前者约束 T 为非空，后者不约束（`unknown` 是 Top Type，任何类型都可赋值给 unknown）
2. 泛型可以用默认值吗？——可以，`function fn<T = string>(arg: T)`
3. 运行时泛型还存在吗？——泛型是**编译时**的概念，编译后全部被擦除（Type Erasure），JS 运行时没有 T

### 避坑提示

- 泛型约束不要太宽泛（`<T extends object>` 基本够用），也不要太严格（除非确实需要特定字段）
- WMS 前端中，API 层的类型定义是泛型应用最密集的地方——所有请求/响应都应围绕 `ApiResponse<T>` 展开，避免每个接口单独定义

---

## 第 7 题：TypeScript 装饰器——@Component / @Prop / @Watch，装饰器编译阶段

### 题目

Java 注解你熟悉。**TypeScript 装饰器（Decorator）和 Java 注解类似吗？Vue 3 的 defineComponent 中 @Prop、@Watch 怎么用？装饰器在编译后是什么形式？wms-web 项目用的是 Options API 装饰器写法还是 Composition API？**

### 核心答案

**1. TypeScript 装饰器 vs Java 注解**：

| | Java 注解 | TypeScript 装饰器 |
|---|---|---|
| 标准版本 | Java 5+ | ESstage 3（目前仍是 TC39 提案） |
| 执行时机 | 运行时（反射） | 编译时生成代码（类创建时） |
| 签名 | `@Annotation` | `@decorator`（本质是函数） |
| 参数 | 注解可以有属性 | 装饰器工厂接收参数返回装饰器 |

**2. Vue 3 Options API 装饰器写法（需要 `vue-property-decorator`）**：
```typescript
import { Component, Prop, Watch, Vue } from 'vue-property-decorator'
import { Emit } from 'vuex-class'

@Component({
  components: { ChildComponent }
})
export default class InventoryList extends Vue {
  // @Prop - 父组件传入的属性
  @Prop({ type: String, required: true })
  public warehouseId!: string

  @Prop({ type: Number, default: 20 })
  public pageSize: number = 20

  // @Watch - 监听属性变化
  @Watch('warehouseId', { immediate: true, deep: true })
  onWarehouseIdChange(newVal: string, oldVal: string) {
    this.fetchData()
  }

  // @Emit - 触发父组件事件
  @Emit('update:visible')
  public closeDialog() {
    return false  // 返回值作为事件 payload
  }

  // data
  public tableData: InventoryItem[] = []

  // computed
  get filteredData() {
    return this.tableData.filter(...)
  }

  // methods
  public async fetchData() { ... }
}
```

**3. defineComponent 中装饰器的编译结果**：
```typescript
// 源代码
@Component({ components: { ChildComponent } })
class MyComponent extends Vue {
  @Prop({ type: String }) name!: string
}

// 编译后（伪代码，vue-property-decorator 转换结果）
const MyComponent = Vue.extend({
  components: { ChildComponent },
  props: {
    name: { type: String, required: true }
  },
  // @Prop 转换成 props 选项
})

// 装饰器本质：@decorator 语法只是下面的语法糖
// @Component(options)  =>  Component(options)(MyClass)
```

**4. wms-web 的实际情况**：
```typescript
// wms-web 使用 Composition API（<script setup>），不需要装饰器
// <script setup lang="ts">
import { ref, computed } from 'vue'
import { defineComponent } from 'vue'

// defineComponent 本身不是装饰器，但可以传入泛型 props 类型
defineComponent<{
  title: string
  data: InventoryItem[]
}>()
// 实际 wms-web 用 <script setup> + defineProps<{...}>() 替代
```

**5. defineProps 与 TypeScript 集成**（wms-web 实际用法）：
```typescript
// <script setup> 中的 defineProps 是编译时宏，不是装饰器
const props = defineProps<{
  title: string
  items: InventoryItem[]
  loading?: boolean
}>()

// withDefaults 提供默认值（编译时）
const props = withDefaults(defineProps<{
  title?: string
}>(), {
  title: 'Default Title'
})
```

### 追问方向

1. 装饰器需要开启什么编译选项？——`tsconfig.json` 中 `"experimentalDecorators": true`（目前仍是实验性功能）
2. Vue 3 推荐用装饰器还是 `<script setup>`？——**推荐 `<script setup>`**，Vue 3 官方已明确 Composition API 是未来方向
3. `@Watch` 和 `watch()` 函数有什么区别？——功能相同，`@Watch` 是 Options API 风格的装饰器，`watch()` 是 Composition API 的函数式写法

### 避坑提示

- `vue-property-decorator` 是第三方库，依赖 `Vue.extend`，Vue 3 兼容性需要确认版本（>= 0.10.0）
- wms-web 如果已全面采用 `<script setup>`，面试中应重点展示 `defineProps` 和 `withDefaults` 的用法，而非装饰器
- `!`（非空断言）在 `@Prop` 参数后表示"必定有值"，是 TypeScript 的类型断言，和 Vue 无关

---

## 第 8 题：Vue3 Teleport——模态框 / 弹窗渲染到 body 的原理

### 题目

WMS 有很多弹窗（Modal）和消息通知（Toast）。**Vue 3 的 Teleport 是什么？解决什么问题？它的 `to` 属性怎么用？为什么说 Teleport 能解决 Modal 默认渲染在当前组件层级的问题？**

### 核心答案

**1. 问题背景**：
```html
<!-- 组件层级：App → Layout → ModalContent -->
<!-- 组件样式有 overflow: hidden，Modal 会被截断 -->
<template>
  <div class="layout">
    <div class="content">
      <Modal />
    </div>
  </div>
</template>
```

**2. Teleport 基本语法**：
```html
<!-- to 目标选择器 -->
<Teleport to="body">
  <div class="modal-overlay">
    <div class="modal-content">
      <slot />
    </div>
  </div>
</Teleport>

<!-- 也可以用 :to 动态绑定 -->
<Teleport :to="containerSelector">
  <div>...</div>
</Teleport>

<!-- disabled 条件渲染 -->
<Teleport to="body" :disabled="!isOpen">
  <div>...</div>
</Teleport>
```

**3. 原理**：
Teleport 将内容**物理移动**到 DOM 树的指定位置（`document.body`），但**逻辑上仍是 Vue 组件树的一部分。

```html
<!-- 编译后（简化）的实际 DOM 结构 -->
<!-- Vue 组件树不变 -->
<App>
  <Layout>
    <Modal>
      <Teleport to="body">  ← 组件树中仍在 Modal 内
        <div class="modal-content">...</div>
      </Teleport>

<!-- 真实 DOM 结构 -->
<!-- body 下直接挂载 -->
<div class="modal-content">...</div>
```

**4. WMS 中的实际用法**：
```typescript
// WMS 的 ConfirmDialog 组件
<template>
  <Teleport to="body">
    <Transition name="modal-fade">
      <div v-if="visible" class="confirm-overlay" @click.self="handleCancel">
        <div class="confirm-dialog">
          <h3>{{ title }}</h3>
          <p>{{ message }}</p>
          <div class="confirm-actions">
            <el-button @click="handleCancel">{{ cancelText }}</el-button>
            <el-button type="primary" @click="handleConfirm">{{ confirmText }}</el-button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup lang="ts">
// Teleport 的 visible 控制仍由父组件管理
// DOM 物理移动不影响事件冒泡和组件通信
defineProps<{
  visible: boolean
  title: string
  message: string
}>()
</script>
```

### 追问方向

1. Teleport 可以 teleport 到哪些位置？——任何有效的 CSS 选择器（`'body'`、`'#app'`、`'.modal-container'`），必须是已存在的 DOM 元素
2. 组件卸载时，Teleport 的内容会被销毁吗？——是的，Teleport 内容是组件模板的一部分，组件卸载时内容一并销毁（即使它物理上在 body 下）
3. `disabled` 属性有什么用？——临时禁用 teleport，内容会渲染在原位置（不移动到 to 目标），适合 SSR 或条件切换场景

### 避坑提示

- Teleport 的 `to` 目标元素必须存在——如果目标选择器找不到元素，Teleport 会报错
- `z-index` 问题：Teleport 到 body 后，弹窗的层级和 body 下其他元素的 `z-index` 比较，需要手动调整（如加 overlay 背景层）
- 滚动锁定：Modal 打开时通常需要 `document.body.style.overflow = 'hidden'`，Teleport 解决不了这个，需要自己处理

---

## 第 9 题：Vue3 Suspense——异步组件加载，#default / #fallback 插槽

### 题目

WMS 页面加载有 Loading 状态。**Vue 3 Suspense 是什么？它和异步组件（defineAsyncComponent）有什么区别？Suspense 的 #default 和 #fallback 插槽分别什么时候渲染？Suspense 和 Error Boundaries 有什么关系？**

### 核心答案

**1. Suspense 用途**：
Suspense 让异步组件（通常配合 `async setup()`）的加载/等待状态声明式管理，不需要手动管理 `v-if/v-show` + `loading` 状态。

**2. defineAsyncComponent**：
```typescript
import { defineAsyncComponent } from 'vue'

// 简单写法
const AsyncInventoryList = defineAsyncComponent(() =>
  import('./components/InventoryList.vue')
)

// 带配置
const AsyncInventoryList = defineAsyncComponent({
  loader: () => import('./components/InventoryList.vue'),
  loadingComponent: LoadingSpinner,  // 加载中显示
  errorComponent: ErrorDisplay,       // 错误时显示
  delay: 200,                          // 延迟显示 loading（避免闪烁）
  timeout: 3000                        // 超时显示 error
})
```

**3. Suspense + async setup**：
```html
<!-- 父组件 -->
<Suspense>
  <template #default>
    <!-- 异步组件加载完成后显示 -->
    <AsyncInventoryList :warehouse-id="whId" />
  </template>

  <template #fallback>
    <!-- 加载中显示 -->
    <div class="loading-wrapper">
      <el-skeleton :rows="10" animated />
    </div>
  </template>
</Suspense>
```

```typescript
// 子组件（异步 setup）
<script setup lang="ts">
// setup 可以是 async 函数
const props = defineProps<{ warehouseId: string }>()

// 异步请求会在 setup 期间触发 Suspense 等待
const inventoryList = ref<InventoryItem[]>([])
const inventoryList = await fetchInventory(props.warehouseId)
</script>
```

**4. 执行流程**：
```
Suspense 渲染
  → 尝试渲染 #default（异步组件）
  → 异步组件 setup() 开始执行（可能 await fetchData）
  → Suspense 显示 #fallback（loading）
  → setup 完成
  → Suspense 切换到 #default 显示真实内容
  → （如果超时报错，显示 errorComponent 或 #fallback）
```

**5. 错误处理**：
```typescript
// setup 中抛出 Promise 即可触发 Suspense 等待
// 错误通过 errorCaptured 或 onErrorCaptured 处理
import { onErrorCaptured } from 'vue'

onErrorCaptured((err, instance, info) => {
  console.error('Child component error:', err, info)
  return false  // 返回 false 阻止错误传播
})
```

### 追问方向

1. Suspense 和 `v-if + loading` 相比的优势？——声明式，模板更清晰；不需要在每个用到异步组件的地方都加 loading 判断
2. Suspense 可以嵌套吗？——可以，多层异步组件嵌套时，内层 Suspense 的 #fallback 在其 #default 加载期间显示
3. `defineAsyncComponent` 和 `suspense` 哪个更好？——两者可以组合使用；`defineAsyncComponent` 控制组件级加载，`Suspense` 控制整个组件树的异步状态

### 避坑提示

- Suspense 的 `v-if` 控制放在**父组件**而非 Suspense 上——`Suspense` 本身的 `v-if` 无意义
- async setup 中如果有多个 `await`，Suspense 会在**所有 await 都完成后**才切换到 #default（整个 setup 是一个大 Promise）
- Suspense **目前没有官方 loadingComponent 的 props**，只能用 `#fallback` 插槽模拟

---

## 第 10 题：Pinia vs Vuex——Pinia 的优势，storeToRefs 解构

### 题目

WMS 前端用 Pinia 做状态管理。**Pinia 相比 Vuex 有什么优势？Vuex 4 和 Pinia 的核心区别是什么？`storeToRefs` 怎么用，为什么不能直接解构 store？**

> 提示：`/usr/src/wms-web/src/app/stores/` 下有多个 Pinia store 定义。

### 核心答案

**1. Pinia 核心优势**：

| 对比项 | Pinia | Vuex |
|---|---|---|
| API 复杂度 | 极简，概念少 | 4 个核心概念（state/getters/mutations/actions） |
| TypeScript 支持 | 天然支持，好 | 需要装饰器或手动类型标注 |
| 模块（Module） | 不用按模块注册，store 就是独立模块 | 必须按模块注册 |
| 命名空间 | 可选（`defineStore('id', ...)`） | 必须有 |
| 异步处理 | actions 直接支持 async/await | mutation 同步，action 异步，概念多 |
| 热更新 | 支持（修改 store 不刷新页面） | 有限支持 |
| 体积 | 更小（~1KB） | 较大 |
| 调试工具 | Vue 3 DevTools 插件 | Vue DevTools |

**2. Pinia 核心用法**（wms-web 实际写法）：
```typescript
// stores/inventory.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { fetchInventoryList, adjustInventory } from '@/apis/inventoryApi'
import type { InventoryItem } from '@/apis/types'

export const useInventoryStore = defineStore('inventory', () => {
  // state
  const list = ref<InventoryItem[]>([])
  const loading = ref(false)
  const total = ref(0)

  // getters（computed）
  const lowStockItems = computed(() =>
    list.value.filter(item => item.quantity < item.threshold)
  )

  const warehouseMap = computed(() => {
    const map = new Map<string, InventoryItem>()
    list.value.forEach(item => map.set(item.warehouseId, item))
    return map
  })

  // actions
  async function loadInventory(query: InventoryQuery) {
    loading.value = true
    try {
      const res = await fetchInventoryList(query)
      list.value = res.data.list
      total.value = res.data.total
    } finally {
      loading.value = false
    }
  }

  async function adjust(id: string, quantity: number) {
    await adjustInventory({ id, quantity })
    const item = list.value.find(i => i.id === id)
    if (item) item.quantity = quantity
  }

  return { list, loading, total, lowStockItems, warehouseMap, loadInventory, adjust }
})
```

**3. storeToRefs**：
```typescript
import { storeToRefs } from 'pinia'
import { useInventoryStore } from '@/stores/inventory'

const inventoryStore = useInventoryStore()

// ❌ 直接解构 - 丢失响应式
const { list, loading } = inventoryStore
// list 变成普通数组，loading 变成普通 boolean

// ✅ storeToRefs 解构 - 保持响应式
const { list, loading } = storeToRefs(inventoryStore)
// list 是 ref<InventoryItem[]>，loading 是 ref<boolean>

// 访问值需要 .value
console.log(list.value)
console.log(loading.value)

// ✅ computed/getters 解构用 storeToRefs
const { lowStockItems } = storeToRefs(inventoryStore)
// lowStockItems 是 ComputedRef

// ✅ actions 直接从 store 解构（actions 本身是函数，不需要响应式）
const { loadInventory, adjust } = inventoryStore
```

**4. 在组件中使用**：
```typescript
// views/inventory/InventoryList.vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useInventoryStore } from '@/stores/inventory'

const store = useInventoryStore()
const { list, loading, total } = storeToRefs(store)
const { loadInventory, adjust } = store

// 页面加载时调用
loadInventory({ warehouseId: 'WH001', page: 1, pageSize: 20 })
</script>
```

### 追问方向

1. Pinia 中如何调试（Vue DevTools）？——`pinia.use(piniaPlugin)` 与 Vue DevTools 集成，可查看 state 变化历史
2. Pinia 的 `$reset` 方法如何使用？——只在 Options API 风格的 store 中可用（`defineStore('id', { state: () => ({}) })`），Composition API 风格的 store 需要手动重置
3. Pinia 持久化怎么做？——用 `pinia-plugin-persistedstate` 插件，`persist: true` 开启

### 避坑提示

- **直接解构 store 是高频踩坑点**——很多人以为 `const { list } = inventoryStore` 能保持响应式，实际丢失了
- computed 风格 getter 如果返回函数，不要用 `storeToRefs`，直接解构即可（`const { getItemById } = store`）
- store 文件应按领域/模块划分（inventory、user、settings），不要把所有状态塞进一个 store

---

## 第 11 题：Vue Router4——动态路由 / 路由守卫 / 导航解析流程

### 题目

WMS 有多个页面（入库、出库、库存查询）。**Vue Router 4 的动态路由怎么配置？路由守卫有哪些？全局守卫、前置守卫（beforeEach）怎么用？导航解析的完整流程是什么？**

> 提示：`/usr/src/wms-web/src/app/` 下应有 router 配置。

### 核心答案

**1. 动态路由（addRoute）**：
```typescript
import { createRouter, createWebHistory, RouteRecordRaw } from 'vue-router'

const routes: RouteRecordRaw[] = [
  {
    path: '/inventory',
    name: 'InventoryList',
    component: () => import('@/views/inventory/InventoryList.vue'),
    meta: { title: '库存查询', requiresAuth: true }
  },
  // 动态路由 - 用户权限不同，加载不同菜单
  {
    path: '/admin/:pathMatch(.*)*',
    name: 'Admin',
    component: () => import('@/views/admin/AdminPanel.vue'),
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

// 动态添加路由（通常用于权限控制）
router.addRoute({
  path: '/inbound',
  name: 'Inbound',
  component: () => import('@/views/inbound/InboundList.vue')
})

// 动态添加子路由
router.addRoute('admin', {
  path: 'users',
  name: 'AdminUsers',
  component: () => import('@/views/admin/Users.vue')
})
```

**2. 路由守卫（Navigation Guards）**：

```typescript
// 全局前置守卫（最常用）
router.beforeEach((to, from, next) => {
  // to: 目标路由对象
  // from: 来源路由对象
  // next: 放行函数

  // 设置页面标题
  document.title = to.meta.title as string || 'WMS'

  // 权限检查
  if (to.meta.requiresAuth && !isLoggedIn()) {
    next({ name: 'Login', query: { redirect: to.fullPath } })
  } else {
    next()  // 放行
  }
})

// 全局后置守卫（很少用）
router.afterEach((to, from) => {
  console.log(`导航从 ${from.path} 到 ${to.path}`)
})

// 路由独享守卫
const routes = [
  {
    path: '/inbound',
    component: InboundList,
    beforeEnter: (to, from, next) => {
      // 只在进入 /inbound 时触发
      if (hasPermission('inbound:read')) next()
      else next('/403')
    }
  }
]

// 组件内守卫（Options API 风格）
export default {
  beforeRouteEnter(to, from, next) {
    // 组件创建前调用，this 不可用
    next(vm => { /* vm 是组件实例 */ })
  },
  beforeRouteUpdate(to, from, next) {
    // 路由参数变化，但组件被复用时调用
    next()
  },
  beforeRouteLeave(to, from, next) {
    // 离开组件时调用
    next()
  }
}
```

**3. 导航解析完整流程**：
```
1. 导航触发（router.push / 点击链接）
2. 调用全局 beforeEach 守卫链
3. 复用组件？ → 是 → 调用 beforeRouteUpdate 守卫
4. 查找目标路由的 beforeEnter 独享守卫
5. 解析异步组件（router.getRoutes 刷新）
6. 调用组件内 beforeRouteEnter 守卫
7. 全局 afterEach 守卫
8. DOM 更新（beforeCreate → created → beforeMount → mounted）
9. beforeRouteEnter 的 next(vm => { /* 回调 */ }) 触发
```

**4. WMS 实际权限守卫示例**：
```typescript
// router/guards.ts
import { useUserStore } from '@/stores/user'

export function setupRouterGuards(router: Router) {
  router.beforeEach(async (to, from, next) => {
    const userStore = useUserStore()

    // 首次访问时获取用户信息
    if (!userStore.profile && to.meta.requiresAuth) {
      try {
        await userStore.fetchProfile()
      } catch {
        next({ name: 'Login' })
        return
      }
    }

    // 菜单权限控制
    const requiredPermissions = to.meta.permissions as string[] | undefined
    if (requiredPermissions?.length && !userStore.hasAllPermissions(requiredPermissions)) {
      next({ name: 'Forbidden' })
      return
    }

    next()
  })
}
```

### 追问方向

1. `next()` / `next(false)` / `next('/')` / `next(Error('xxx'))` 区别？——放行/中止/重定向到其他路由/报错触发错误处理
2. 路由变了但组件没复用（如同级 Tab 切换）用哪个钩子？——`beforeRouteUpdate`
3. `router.push` 和 `router.replace` 区别？——`push` 会产生历史记录，`replace` 不会

### 避坑提示

- `beforeEach` 是异步的——可以 `await next()` 等待异步操作（如获取用户信息）完成后再放行
- 路由 `meta` 类型需要扩展 Module 类型声明，否则 `to.meta.permissions` 会报 TS 错误
- `beforeRouteEnter` 中 `this` 不可用，需要通过 `next(vm => { this = vm })` 访问

---

## 第 12 题：TypeScript 高级类型——Utility Types（Partial/Required/Pick/Omit/Record）

### 题目

TypeScript 内置了很多 Utility Types。**Partial、Required、Pick、Omit、Record 各自的作用是什么？请结合 WMS 场景举例：表单编辑时从完整数据中提取部分字段，用哪个 Utility Type？表单提交后需要所有字段必填，用哪个？**

### 核心答案

**1. 各 Utility Type 详解**：

```typescript
// Partial<T> - T 的所有属性变为可选
interface InventoryItem {
  id: string
  itemCode: string
  quantity: number
}
type InventoryUpdate = Partial<InventoryItem>
// 等价于
type InventoryUpdate = {
  id?: string
  itemCode?: string
  quantity?: number
}

// Required<T> - T 的所有属性变为必选
type RequiredItem = Required<InventoryUpdate>  // 全必填

// Pick<T, K> - 从 T 中挑选 K 个属性
type ItemBasicInfo = Pick<InventoryItem, 'id' | 'itemCode'>
// 等价于
type ItemBasicInfo = {
  id: string
  itemCode: string
}

// Omit<T, K> - 从 T 中排除 K 个属性（取反 Pick）
type ItemWithoutId = Omit<InventoryItem, 'id'>
// 等价于
type ItemWithoutId = {
  itemCode: string
  quantity: number
}

// Record<K, V> - K 作为 key，V 作为 value 的对象类型
type WarehouseStatus = Record<'WH001' | 'WH002' | 'WH003', boolean>
// WMS 实际用法：仓库 ID → 库存列表
type InventoryByWarehouse = Record<string, InventoryItem[]>

// Exclude<T, U> - 从 T 中排除可分配给 U 的类型
type A = string | number | boolean
type B = Exclude<A, boolean>  // string | number

// Extract<T, U> - 从 T 中提取可分配给 U 的类型
type C = Extract<A, string | object>  // string

// NonNullable<T> - 排除 null 和 undefined
type D = NonNullable<string | null | undefined>  // string

// ReturnType<F> - 获取函数返回值类型
function getInventory() { return { id: '1' } }
type Inventory = ReturnType<typeof getInventory>  // { id: string }

// Parameters<F> - 获取函数参数类型
type GetInventoryParams = Parameters<typeof getInventory>  // []
```

**2. WMS 实际场景应用**：

```typescript
// 场景1：表单编辑 - 部分字段更新（Partial）
const form = reactive<Partial<InventoryItem>>({
  quantity: 100,
  // 其他字段不用填
})

// 场景2：表单提交 - 所有字段必填（Required）
interface InventoryCreateForm {
  itemCode: string
  itemName: string
  warehouseId: string
  quantity: number
}
// 创建时用 Required（如果表单初始有空值）
const submitForm = (data: Required<InventoryCreateForm>) => { ... }

// 场景3：API 响应中只取部分字段（Pick）
// 后端返回完整 Item，但前端只需要这几个字段展示列表
type ItemListVO = Pick<ItemMaster, 'id' | 'itemCode' | 'itemName' | 'uomId'>

// 场景4：排除敏感字段（Omit）
// 后端返回包含 createdBy 等审计字段的完整对象
type ItemApiResponse = Omit<ItemMaster, 'createdBy' | 'updatedBy'>

// 场景5：多仓库库存 Map（Record）
type WarehouseInventoryMap = Record<string, InventoryItem[]>
const inventoryByWh: WarehouseInventoryMap = {
  'WH001': [...],
  'WH002': [...],
}

// 场景6：合并 Omit + Partial + Pick
// 编辑时用：排除 id，其他可选
type InventoryEditForm = Omit<InventoryItem, 'id'>  // id 不可编辑
```

### 追问方向

1. `Partial<T>` 嵌套到深层怎么做？——`Partial` 只对顶层生效，深层需要 `DeepPartial` 自定义（递归应用 Partial）
2. `Pick` 和 `Omit` 的 key 约束？——key 必须在 T 的键中，否则 TS 报错
3. 自己实现 `Partial`？——`type Partial<T> = { [P in keyof T]?: T[P] }`，它是映射类型的语法糖

### 避坑提示

- `Partial` 默认只处理**第一层**，嵌套对象内部不会自动变可选——需要自定义 `DeepPartial<T>` 递归处理
- `Record<string, T>` 的 key 是 string 时，key 类型是字符串索引签名，所有字符串都可以作为 key

---

## 第 13 题：TypeScript 类型推断——infer 关键字，条件类型

### 题目

TypeScript 的类型推断和 `infer` 关键字比较高级。**什么是类型推断？infer 在什么场景下用？条件类型（Conditional Types）怎么理解？请写一个从 Promise 中提取 resolved 类型的工具类型。**

### 核心答案

**1. 类型推断（Type Inference）**：
```typescript
// TypeScript 自动推断类型，不需要显式标注
const name = 'WMS'  // TS 自动推断为 string
const list = [1, 2, 3]  // number[]
const obj = { id: '1', quantity: 100 }  // { id: string; quantity: number }

// 函数返回值推断
function process(x: number, y: number) {
  return x + y  // TS 推断返回类型为 number
}

// 泛型推断：从参数推断泛型类型
function identity<T>(arg: T): T {
  return arg
}
identity('hello')  // T 推断为 string
```

**2. 条件类型（Conditional Types）**：
```typescript
// 基本语法：T extends U ? X : Y
type IsString<T> = T extends string ? true : false
type A = IsString<'hello'>  // true
type B = IsString<123>       // false

// 实际应用：从联合类型中提取某一部分
type Direction = 'north' | 'south' | 'east' | 'west'
type Cardinal = Direction extends 'north' | 'south' | 'east' | 'west' ? Direction : never
// Cardinal = 'north' | 'south' | 'east' | 'west'

// 分布式条件类型：应用于联合类型时自动"分发"
type Without<T, U> = T extends U ? never : T
type C = Without<'a' | 'b' | 'c', 'a' | 'b'>  // 'c'
```

**3. infer 关键字**：
`infer` 在条件类型中使用，用于**从类型中提取部分**并命名，供后面使用。

```typescript
// infer 用法：提取 Promise 的 resolved 类型
type Awaited<T> = T extends Promise<infer U> ? U : T
type ResolvedString = Awaited<Promise<string>>  // string
type ResolvedNumber = Awaited<number>           // number（不是 Promise，直接返回）

// 提取数组元素类型
type ElementType<T> = T extends (infer E)[] ? E : never
type Num = ElementType<number[]>  // number

// 提取函数返回值类型（内建 ReturnType 就是这么实现的）
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : any

// 提取函数第一个参数类型
type FirstArg<T> = T extends (first: infer F, ...rest: any[]) => any ? F : never
type Str = FirstArg<(first: string, second: number) => void>  // string

// 提取构造函数的实例类型
type InstanceType<T> = T extends new (...args: any[]) => infer R ? R : any
```

**4. WMS 实际应用场景**：
```typescript
// 从 API 响应中提取 data 字段类型
type ApiData<T> = T extends { data: infer D } ? D : never
type InventoryData = ApiData<{ data: InventoryItem[]; code: number }>  // InventoryItem[]

// 从仓库列表响应中提取仓库数组的类型
type WarehouseListType = ApiData<ApiResponse<Warehouse[]>>  // Warehouse[]

// 提取数组 Promise 的 resolved 结果
async function fetchInventoryList(): Promise<ApiResponse<InventoryItem[]>> {
  return { code: 200, data: [], message: '' }
}
type InventoryResult = Awaited<ReturnType<typeof fetchInventoryList>>  // ApiResponse<InventoryItem[]>
```

### 追问方向

1. `infer` 只能在条件类型中使用吗？——是的，`infer` 是条件类型的一部分，不能单独使用
2. 分布式条件类型是什么？——联合类型触发条件类型时，会逐个应用于每个联合成员再合并，如 `string | number extends T ? ... : ...` 变成 `(... extends T ? ... : ...) | (... extends T ? ... : ...)`
3. 内建类型 `Awaited<T>` 和自写的 `Awaited<T>` 有什么区别？——TS 内置 `Awaited<T>` 处理嵌套 Promise 递归解析，比简单实现的更完善

### 避坑提示

- `infer` 提取的变量只能在条件类型的 `true` 分支使用，false 分支不可用
- 条件类型的泛型约束不要太宽——如 `T extends object` 会导致非对象类型直接走 false 分支，可能丢失有用信息

---

## 第 14 题：Vue3 组件通信——props / emit / expose / ref / attrs，provide / inject 跨层级

### 题目

WMS 有很多父子组件、跨层级通信场景。**Vue 3 组件通信的方式有哪些？props 和 emit 是如何配合 TypeScript 做到类型安全的？attrs 和 $attrs 有什么区别？provide/inject 适合什么场景？expose 和 ref 怎么用？**

> 提示：WMS 前端的库存组件 `/usr/src/wms-web/src/app/views/inventory/` 中应有大量父子组件通信实践。

### 核心答案

**1. props + emit（父子通信）**：
```typescript
// 父组件
<script setup lang="ts">
import { ref } from 'vue'
const currentItem = ref<InventoryItem | null>(null)

function handleUpdate(id: string, quantity: number) {
  // 处理子组件触发的更新事件
  updateItem(id, quantity)
}
</script>

<template>
  <!-- 传入 props，监听 emit -->
  <InventoryEditModal
    :item="currentItem"
    :visible="modalVisible"
    @update="handleUpdate"
    @close="modalVisible = false"
  />
</template>
```

```typescript
// 子组件（defineProps + defineEmits - 编译时宏，无需导入）
<script setup lang="ts">
// defineProps - 编译时宏，自动暴露给模板
const props = defineProps<{
  item: InventoryItem
  visible: boolean
}>()

// defineEmits - 编译时宏，类型安全的 emit
const emit = defineEmits<{
  (e: 'update', id: string, quantity: number): void
  (e: 'close'): void
}>()

// 使用 emit
function onConfirm() {
  emit('update', props.item.id, newQuantity.value)
  emit('close')
}
</script>
```

**2. expose / ref（父调用子方法）**：
```typescript
// 子组件 - 暴露给父组件的方法/属性
<script setup lang="ts">
function validate() { ... }
function reset() { ... }

// expose - 明确暴露哪些给父组件
defineExpose({
  validate,
  reset,
  // 父组件可以通过 ref.current.validate() 调用
})

// 不在 setup return 中的内容，父组件通过 ref 无法访问
const internalState = ref(0)  // 不会被暴露
</script>
```

```html
<!-- 父组件 -->
<template>
  <InventoryEditModal ref="modalRef" />
</template>
<script setup>
const modalRef = ref<InstanceType<typeof InventoryEditModal> | null>(null)

function onEdit() {
  modalRef.value?.validate()  // 调用子组件暴露的方法
}
</script>
```

**3. attrs（透传属性）**：
```html
<!-- $attrs：父组件传入的非 props 属性，会自动传递给根元素或指定元素 -->
<!-- 场景：封装 el-input，自动传入所有 attrs（class/style/placeholder/disabled 等） -->

<!-- 子组件 -->
<template>
  <!-- v-bind="$attrs" 将所有 attrs 传递给 div -->
  <div class="base-input">
    <label>{{ label }}</label>
    <input v-bind="$attrs" class="input-field" />
  </div>
</template>

<script setup lang="ts">
defineProps<{ label: string }>()
// defineProps 不包含的 attrs 会进入 $attrs
</script>

<!-- 父组件 -->
<BaseInput
  label="物料编码"
  v-model="itemCode"
  placeholder="请输入"
  class="custom-input"   <!-- 这些都会透传到子组件的根 div -->
/>
```

**4. provide / inject（跨层级通信）**：
```typescript
// 祖先组件（通常在 setup 顶层）
import { provide, ref } from 'vue'
const theme = ref('light')
const user = reactive({ id: 'U001', name: '张三' })

// provide(key, value)
provide('theme', theme)                  // ref（保持响应式）
provide('currentUser', toRef(user, 'name'))  // 单个属性
provide('appVersion', '2.0')            // 原始值（无响应式）

// 后代组件
import { inject } from 'vue'

// 注入（类型安全写法）
const theme = inject<Ref<string>>('theme', ref('light'))
const version = inject<string>('appVersion', '1.0')

// 如果不提供默认值，且可能不存在
const maybeValue = inject<string>('optional')  // 类型是 string | undefined
```

```typescript
// WMS 实际场景：主题、当前用户信息、全局配置
// 在 App.vue 中 provide
export default {
  setup() {
    const globalConfig = reactive({
      warehouseId: 'WH001',
      warehouseName: '主仓',
      permissions: [] as string[],
    })
    provide('globalConfig', globalConfig)
  }
}

// 在任意深层组件中 inject
const config = inject<GlobalConfig>('globalConfig')
console.log(config?.warehouseName)
```

**5. 通信方式对比**：

| 方式 | 方向 | 响应式 | 适用场景 |
|---|---|---|---|
| props | 父→子 | ✅ | 常规数据传递 |
| emit | 子→父 | ✅ | 事件通知、回调 |
| expose + ref | 父→子（方法） | N/A | 父调用子方法 |
| attrs | 父→子（透传） | ✅ | 透传 HTML 属性 |
| provide/inject | 祖先→后代 | ✅ | 全局状态、配置 |
| Pinia | 任意 | ✅ | 全局状态共享 |
| eventBus | 任意 | ✅ | 跨组件事件（非推荐） |

### 追问方向

1. `v-model` 在 Vue 3 中的实现原理？——`:modelValue` + `@update:modelValue`，一个组件可以支持多个 `v-model`
2. `provide/inject` 是响应式的吗？——`provide` 的 ref/reactive 在后代组件中保持响应式；原始值则不会
3. `defineProps` 返回的对象能解构吗？——不能直接解构（会丢失响应式），需要用 `toRefs` 或 `toRef`

### 避坑提示

- `provide/inject` 不是响应式的——除非 `provide` 的是 `ref` 或 `reactive` 对象，直接 provide 原始值后代拿到的不会响应
- 循环依赖的 provide/inject 会导致空对象——大型项目中建议用 Pinia 替代深层 inject
- `expose` 只能暴露函数和原始类型值，暴露的 ref 在父组件中会自动解包

---

## 第 15 题：TypeScript 类型守卫——typeof / instanceof / in / is，类型收窄

### 题目

Java 中 instanceof 你熟悉。**TypeScript 的类型守卫（Type Guard）是什么？typeof、instanceof、in、is 各自怎么用于类型收窄？TS 怎么在条件分支中自动收窄类型？请结合 WMS 的场景举例。**

### 核心答案

**1. 类型守卫的作用**：类型守卫是在条件语句中，TS 能自动缩小（narrow）联合类型范围的功能，让分支内的类型更精确。

**2. typeof 类型守卫**：
```typescript
function processValue(val: string | number | boolean) {
  if (typeof val === 'string') {
    // TS 知道 val 是 string，自动获得 string 方法提示
    console.log(val.toUpperCase())
  } else if (typeof val === 'number') {
    console.log(val.toFixed(2))
  } else {
    // val 是 boolean
    console.log(val ? 'true' : 'false')
  }
}
```

**3. instanceof 类型守卫**：
```typescript
class InboundOrder { type = 'INBOUND' as const }
class OutboundOrder { type = 'OUTBOUND' as const }

function processOrder(order: InboundOrder | OutboundOrder) {
  if (order instanceof InboundOrder) {
    console.log('入库单:', order.type)
  } else {
    console.log('出库单:', (order as OutboundOrder).type)
  }
}
```

**4. in 操作符类型守卫**：
```typescript
interface HasQuantity { quantity: number }
interface HasPrice { price: number }

function calc(item: HasQuantity | HasPrice) {
  if ('quantity' in item) {
    console.log(item.quantity * 10)  // item 是 HasQuantity
  } else {
    console.log(item.price * 1.1)    // item 是 HasPrice
  }
}
```

**5. 自定义类型守卫（is 关键字）**：
```typescript
// is 关键字：让 TS 在条件分支中收窄类型
function isInventoryItem(obj: any): obj is InventoryItem {
  return obj && typeof obj.id === 'string' && typeof obj.itemCode === 'string'
}

// 使用
function processUnknown(obj: any) {
  if (isInventoryItem(obj)) {
    console.log(obj.itemCode)  // TS 知道 obj 是 InventoryItem
  }
}

// 数组类型守卫
function isNonEmpty<T>(arr: T[]): arr is NonNullable<T>[] {
  return arr.length > 0
}
```

**6. WMS 实际场景**：
```typescript
// 场景：处理 API 响应，可能返回 Error 或 Data
type ApiResult<T> = { success: true; data: T } | { success: false; error: string }

function handleResult<T>(result: ApiResult<T>) {
  if (result.success) {
    console.log(result.data)  // data: T
  } else {
    console.error(result.error)  // error: string
  }
}

// 场景：处理多种业务单据
type Document = InboundDoc | OutboundDoc | TransferDoc

function getDocSummary(doc: Document) {
  if ('inboundNo' in doc) {
    return `入库单: ${doc.inboundNo}`
  } else if ('outboundNo' in doc) {
    return `出库单: ${doc.outboundNo}`
  } else {
    return `调拨单: ${(doc as TransferDoc).transferNo}`
  }
}
```

**7. 断言函数与类型谓词**：
```typescript
// 类型谓词（Type Predicate）：返回 boolean，但告诉 TS "is X"
function isString(val: unknown): val is string {
  return typeof val === 'string'
}

// 使用
const maybeStr: unknown = 'hello'
if (isString(maybeStr)) {
  console.log(maybeStr.toUpperCase())  // TS 知道是 string
}

// 区分：类型断言（as）和类型守卫（is）
// as：强制告诉 TS "这个值是 X"，不验证
// is：返回 boolean，TS 会根据返回值自动收窄
```

### 追问方向

1. 可辨识联合类型（Discriminated Unions）是什么？——用一个共有字段（如 `type: 'A' | 'B'`）作为"标签"，TS 可通过 `switch(type)` 自动收窄
2. `never` 类型在收窄中的作用？——联合类型所有分支都处理后，TS 能推到出 `never`，用于检测是否遗漏分支
3. `== null` 和 `=== null` 对收窄的影响？——`== null` 同时覆盖 `undefined` 和 `null`，常用于"是否存在"的判断

### 避坑提示

- 自定义类型守卫返回 `boolean` 而不是 `is 类型`，TS 不会自动收窄——必须用 `is X` 语法
- 在 Vue 组件中，`instanceof` 适合判断 JS 类实例；判断普通对象类型需要自定义 `is X` 函数
- `in` 操作符收窄后，如果对象有可选属性（如 `quantity?: number`），`in` 判断仍然有效，但要注意 `undefined` 情况

---

## 第 16 题：Vue3 自定义指令——v-focus / v-permission，directive 钩子函数

### 题目

WMS 有很多表单，有些字段需要权限控制。**Vue 3 的自定义指令怎么定义？directive 的生命周期钩子有哪些？如何在指令中操作 DOM？怎么实现一个权限控制指令 `v-permission`？**

### 核心答案

**1. Vue 3 指令生命周期钩子**：
```typescript
// 完整签名
const myDirective: Directive = {
  // 指令绑定到元素时调用（类似 mounted）
  beforeMount(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode | null) {},
  // 元素插入父 DOM 时调用（DOM 已存在）
  mounted(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode | null) {},
  // 组件更新前调用
  beforeUpdate(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode) {},
  // 组件更新后调用
  updated(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode) {},
  // 指令卸载前调用
  beforeUnmount(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode | null) {},
  // 指令卸载后调用（类似 unmounted）
  unmounted(el: HTMLElement, binding: DirectiveBinding, vnode: VNode, prevVnode: VNode | null) {},
}
```

**2. binding 对象结构**：
```typescript
binding = {
  value: any,           // 指令绑定的值（v-focus="true" 中的 true）
  oldValue: any,        // 上一个值（用于 updated/onUpdated）
  arg: string | null,   // 指令参数（v-focus:arg 中的 arg）
  modifiers: {},        // 修饰符对象（v-focus.foo.bar 中的 { foo: true, bar: true })
  instance: ComponentPublicInstance | null,  // 指令所属组件实例
}
```

**3. v-focus 自动聚焦指令**：
```typescript
// directives/vFocus.ts
const vFocus: Directive<HTMLElement, boolean | undefined> = {
  mounted(el, binding) {
    // 聚焦逻辑
    if (binding.value !== false) {
      el.focus()
    }
  },
  updated(el, binding) {
    // 如果绑定值变为 true，重新聚焦
    if (binding.value === true) {
      el.focus()
    }
  }
}

export default vFocus
```

```html
<!-- 使用 -->
<input v-focus="!isReadonly" placeholder="自动聚焦" />
```

**4. v-permission 权限指令**：
```typescript
// directives/vPermission.ts
import { Directive } from 'vue'
import { useUserStore } from '@/stores/user'

const vPermission: Directive<HTMLElement, string | string[]> = {
  mounted(el, binding) {
    const userStore = useUserStore()
    const required = Array.isArray(binding.value) ? binding.value : [binding.value]

    // 没有权限，移除元素
    if (!userStore.hasAllPermissions(required)) {
      el.style.display = 'none'
      // 或 el.parentNode?.removeChild(el)
    }
  },
  updated(el, binding) {
    // 权限变化时重新检查
    const userStore = useUserStore()
    const required = Array.isArray(binding.value) ? binding.value : [binding.value]

    if (!userStore.hasAllPermissions(required)) {
      el.style.display = 'none'
    } else {
      el.style.display = ''
    }
  }
}

export default vPermission
```

```html
<!-- 使用 -->
<el-button v-permission="'inventory:create'" type="primary">
  新增库存
</el-button>

<el-button v-permission="['inbound:approve', 'inbound:read']" type="success">
  审批入库
</el-button>
```

**5. 全局注册指令**：
```typescript
// main.ts
import { createApp } from 'vue'
import vFocus from '@/directives/vFocus'
import vPermission from '@/directives/vPermission'

const app = createApp(App)
app.directive('focus', vFocus)
app.directive('permission', vPermission)  // 自动转为 'permission'
app.mount('#app')
```

### 追问方向

1. 指令中的 `el` 是什么类型？——`HTMLElement`，是指令绑定到的原生 DOM 元素（不是 Vue 组件）
2. 组件上使用指令时，`el` 是什么？——组件的根元素
3. 指令能访问组件的响应式状态吗？——`binding.instance` 可以访问组件实例，进而访问 `binding.instance.proxy`（组件的响应式代理）

### 避坑提示

- 指令的 `mounted`/`unmounted` 对应组件的 `onMounted`/`onUnmounted`，但组件卸载不等于指令卸载（keep-alive 缓存时差异明显）
- 权限指令隐藏元素用 `display: none` 而非 `v-if`，因为权限检查发生在 DOM 渲染后；更优方案是路由级别的权限控制
- Vue 3 中 `inserted` 钩子已改名为 `mounted`，别用 Vue 2 的 API 名称

---

## 第 17 题：TypeScript 泛型约束——extends 关键字约束泛型类型

### 题目

你说用 `<T extends SomeType>` 约束泛型。**TypeScript 中 extends 约束泛型的用法有哪些？`<T extends keyof U>`、`<T extends object>`、`<T extends number>` 各自表示什么？为什么需要泛型约束？请结合 WMS 写一个例子。**

### 核心答案

**1. 泛型约束基本语法**：
```typescript
// 约束 T 必须有某些属性
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const item = { id: '001', itemCode: 'SKU001', quantity: 100 }
const id = getProperty(item, 'id')      // 类型是 string
const qty = getProperty(item, 'quantity')  // 类型是 number
// getProperty(item, 'nonExist')  // 编译报错：类型 "nonExist" 不能赋值给 keyof T
```

**2. 常见约束形式**：

```typescript
// 约束为特定结构（T 必须有 id）
interface HasId { id: string }
function findById<T extends HasId>(list: T[], id: string): T | undefined {
  return list.find(item => item.id === id)
}

// 约束为对象（非原始类型）
function process<T extends object>(obj: T): void {
  // T 一定是 object，可以用 Object.keys
}

// 约束为非 null 非 undefined
function process<T extends {}>(obj: T): void { }

// 约束为数组
function first<T extends any[]>(arr: T): T[number] {
  return arr[0]
}
first([1, 2, 3])  // number

// 多重约束（交叉类型）
function merge<T extends BaseEntity & HasId>(entity: T): string {
  return entity.id
}
```

**3. WMS 实际场景**：

```typescript
// 场景1：通用表单校验
interface Validatable {
  validate(): boolean
}

// 约束表单元素必须实现校验方法
async function validateForm<T extends Validatable>(form: T): Promise<boolean> {
  return form.validate()
}

// 场景2：通用 ID 查找
function findEntityById<T extends { id: string }>(
  entities: T[],
  id: string
): T | undefined {
  return entities.find(e => e.id === id)
}

// 场景3：通用深拷贝（约束为可序列化对象）
function deepClone<T extends object>(obj: T): T {
  return JSON.parse(JSON.stringify(obj))
}

// 场景4：API 响应处理
type ApiResponse<T> = {
  code: number
  message: string
  data: T
}

// 约束 T 必须有某些字段（用于列表查询）
interface ListQuery {
  page: number
  pageSize: number
}

function fetchList<T extends object>(
  query: T & ListQuery
): Promise<ApiResponse<any[]>> {
  return request.post('/list', query)
}

// 场景5：泛型约束 + 默认类型
function getValue<T extends object, K extends keyof T>(
  obj: T,
  key: K,
  defaultValue: T[K]
): T[K] {
  return obj[key] ?? defaultValue
}

const defaultQty = getValue(item, 'quantity', 0)  // number
```

### 追问方向

1. `T extends SomeType` 和 `T = SomeType`（默认类型参数）区别？——前者约束 T 的形状，后者只是默认值；默认类型参数不约束 T
2. `T extends never` 有什么用？——`never` 是底类型，`T extends never` 在大多数情况下 `T` 只能是 `never`，常用于类型推导的边界检查
3. `keyof T` 返回什么类型？——返回 T 的所有 key 组成的联合类型，如 `keyof { id: string; name: string }` → `'id' | 'name'`

### 避坑提示

- 泛型约束过于宽松（如 `<T extends object>`）会丢失类型信息，过于严格会降低通用性
- 多重约束用 `&`（交叉类型）而非 `extends extends` 语法

---

## 第 18 题：Vue3 性能优化——v-memo / shallowRef / 组件懒加载 / v-once

### 题目

WMS 的库存列表页面数据量大，容易卡顿。**Vue 3 有哪些性能优化手段？v-memo、shallowRef、组件懒加载（defineAsyncComponent）、v-once 分别在什么场景使用？虚拟滚动了解吗？**

### 核心答案

**1. v-memo**：
缓存模板的子节点，只有依赖项变化时才重新渲染。
```html
<!-- 只有 warehouseId 或 status 变化时才重新渲染此 tr -->
<tr v-memo="[warehouseId, status]">
  <td>{{ item.itemCode }}</td>
  <td>{{ item.quantity }}</td>
</tr>

<!-- 用在长列表中：只更新数量变化的那一行 -->
<div v-for="item in inventoryList" :key="item.id" v-memo="[item.quantity]">
  <InventoryRow :item="item" />
</div>
```

**2. shallowRef**：
只追踪 `.value` 变化，不追踪深层（对象内部）变化。
```typescript
// ref：深度响应式
const list = ref<InventoryItem[]>([])
list.value[0].quantity = 100  // 触发更新

// shallowRef：只追踪顶层 .value 替换
const list = shallowRef<InventoryItem[]>([])
list.value[0].quantity = 100  // ❌ 不触发更新
list.value = newArray         // ✅ 触发更新

// 适用场景：超大数组/对象，只需要整体替换的场景
// WMS 场景：列表整体刷新时用 shallowRef，局部更新用 ref
```

**3. 组件懒加载（defineAsyncComponent）**：
```typescript
import { defineAsyncComponent } from 'vue'

// 路由懒加载（最常用）
const routes = [
  {
    path: '/inventory',
    component: () => import('./views/inventory/InventoryList.vue')
    // 访问时才加载这个 JS chunk
  }
]

// 非路由组件懒加载
const AsyncInventoryModal = defineAsyncComponent({
  loader: () => import('./InventoryModal.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorBoundary,
  delay: 200,
  timeout: 10000
})

// Suspense 配合异步组件
<Suspense>
  <template #default>
    <AsyncInventoryModal />
  </template>
  <template #fallback>
    <LoadingSpinner />
  </template>
</Suspense>
```

**4. v-once**：
```html
<!-- 只渲染一次，静态内容用 v-once，不会重新渲染 -->
<div v-once>
  <p>系统版本: {{ version }}</p>
  <p>版权信息: © 2024</p>
</div>

<!-- 结合 v-if，条件渲染一次后不变 -->
<el-card v-if="isAdmin" v-once>
  <AdminPanel />
</el-card>
```

**5. 其他 Vue 3 优化手段**：
```typescript
// 1. computed 缓存（比 methods 更高效）
// ❌ method：每次调用都重新计算
const formatted = getFormattedList(list)
// ✅ computed：依赖不变时不重新计算
const formatted = computed(() => list.value.map(...))

// 2. 合理使用 v-if / v-show
// v-if：条件不常切换（切换开销大）
// v-show：频繁切换（初始渲染开销小，切换用 display 控制）

// 3. 虚拟滚动（vue-virtual-scroller / vxe-table 虚拟滚动）
// WMS 库存列表如果有 10000+ 条，用虚拟滚动只渲染可见行
import { VirtualScroller } from 'vue-virtual-scroller'
<VirtualScroller :items="inventoryList" :item-height="50">
  <template #default="{ item }">
    <InventoryRow :item="item" />
  </template>
</VirtualScroller>

// 4. keep-alive 缓存不销毁
<keep-alive :include="['InventoryList', 'InboundList']">
  <router-view />
</keep-alive>

// 5. 合理使用 reactive（避免深度响应式开销）
const form = reactive({ ... })  // 深层响应式
const form = shallowReactive({ ... })  // 浅层响应式
```

### 追问方向

1. `v-memo` 和 `computed` 的区别？——`computed` 是 JS 级别的响应式缓存，`v-memo` 是模板级别的渲染缓存
2. `shallowRef` 想要更新深层怎么办？——配合 `triggerRef()` 强制触发更新，或使用 `reactive` 的 `shallowReactive`
3. 虚拟滚动的原理？——只渲染可视区域内的 DOM 节点，通过监听 scroll 事件动态计算可见范围，更新时复用已创建的 DOM 节点

### 避坑提示

- `v-memo` 误用会导致渲染不更新——依赖数组必须是真正影响渲染结果的变量
- 组件懒加载会显示 loading，但首屏 SSR 不支持 `defineAsyncComponent`
- `shallowRef` 适合"整个列表替换"场景，不适合"列表内单项修改"场景——用错会导致数据改了 UI 没更新

---

## 第 19 题：TypeScript 映射类型——`{[K in keyof T]: T[K]}`，keyof 操作符

### 题目

TypeScript 的映射类型语法比较抽象。**`{[K in keyof T]: T[K]}` 是什么意思？keyof 操作符返回什么？映射类型在 Vue 3 / TypeScript 中有哪些实际应用？请写一个把所有属性变为只读（Readonly）的映射类型。**

### 核心答案

**1. keyof 操作符**：
```typescript
interface InventoryItem {
  id: string
  itemCode: string
  quantity: number
}

type Keys = keyof InventoryItem  // 'id' | 'itemCode' | 'quantity'
```

**2. 映射类型基础**：
```typescript
// {[K in keyof T]: T[K]} - 遍历 T 的所有 key，生成相同类型的属性
type Clone<T> = {
  [K in keyof T]: T[K]
}
type ClonedItem = Clone<InventoryItem>  // 等价于 InventoryItem

// 所有属性变为只读
type Readonly<T> = {
  readonly [K in keyof T]: T[K]
}
type ReadonlyItem = Readonly<InventoryItem>
// 等价于
// { readonly id: string; readonly itemCode: string; readonly quantity: number }
```

**3. 内置映射类型源码解析**：
```typescript
// Partial<T>
type Partial<T> = {
  [K in keyof T]?: T[K]
}

// Required<T>
type Required<T> = {
  [K in keyof T]-?: T[K]  // -? 表示移除可选性
}

// Pick<T, K>
type Pick<T, K extends keyof T> = {
  [K2 in K]: T[K2]  // K 是 keyof T 的子集
}

// Omit<T, K>
type Omit<T, K extends keyof T> = {
  [K2 in Exclude<keyof T, K>]: T[K2]
}

// Record<K, V>
type Record<K extends keyof any, V> = {
  [P in K]: V
}
```

**4. 实际应用场景**：

```typescript
// 场景1：API 请求参数（所有字段可选）
type InventoryQuery = Partial<InventoryItem>
// WMS 中：{ id?: string; itemCode?: string; quantity?: number }

// 场景2：表单默认值生成
type DefaultForm<T> = {
  [K in keyof T]?: T[K]
}
const defaultItem = {} as DefaultForm<InventoryItem>
// { id?: string; itemCode?: string; quantity?: number }

// 场景3：深只读（递归映射）
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K]
}

// 场景4：深可选
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K]
}

// 场景5：函数参数类型映射（每个参数加修饰）
type Arguments<T extends (...args: any) => any> = {
  [K in keyof Parameters<T>]: Parameters<T>[K]
}

// 场景6：属性的值类型变换
type Stringify<T> = {
  [K in keyof T]: string
}
type ItemStringified = Stringify<InventoryItem>
// { id: string; itemCode: string; quantity: string }
```

**5. 条件映射类型**：
```typescript
// 只映射值为函数类型的属性
type FunctionOnly<T> = {
  [K in keyof T as T[K] extends Function ? K : never]: T[K]
}

// 场景：只读方法，保留方法类型
type ExcludeReadonly<T> = {
  [K in keyof T as {} extends Pick<T, K> ? never : K]: T[K]
}
```

### 追问方向

1. `[K in keyof T]?:` 中的 `?` 是条件运算符还是可选修饰符？——是**可选修饰符**，表示该属性变为可选；条件运算符是 `extends`
2. `as` 关键字在映射类型中干什么？——重映射 key，如 `[K in keyof T as \`${K}Id\`]: T[K]` 把所有 key 加 `Id` 后缀
3. 内置的 `Readonly<T>` 和手写的 `Readonly<T>` 哪个更好？——内置的更完整（处理索引签名），但手写能理解原理

### 避坑提示

- 映射类型默认处理所有 key，包括索引签名（`[key: string]: any`）
- 映射类型返回的是新对象类型，不影响原类型（TS 类型是不可变的）
- `Exclude`和 `Extract` 作用于联合类型时是分布式的，作用于映射类型的 key 时需要用 `as` 重映射

---

## 第 20 题：Vue3 + TypeScript 项目实践——defineComponent 泛型、类型推断最佳实践

### 题目

最后一道，综合题。结合 WMS 项目（`/usr/src/wms-web`）。**在 Vue 3 + TypeScript 项目中，如何用 `defineComponent` 配合泛型做到完整的类型推断？请写出 WMS 库存列表组件从 props → computed → API 调用 → 渲染的全链路类型安全代码。defineComponent 中 defineProps 和 defineEmits 的泛型写法？有哪些最佳实践和常见错误？**

### 核心答案

**1. defineComponent + defineProps + defineEmits（TypeScript 泛型写法）**：

```typescript
// views/inventory/InventoryList.vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import type { InventoryItem, InventoryQuery, PageResult } from '@/apis/types'
import { fetchInventoryList, exportInventoryExcel } from '@/apis/inventoryApi'

// ==================== Props 定义 ====================
const props = defineProps<{
  warehouseId?: string
  autoLoad?: boolean
}>()

// withDefaults：提供默认值的编译时写法
const defaultProps = withDefaults(defineProps<{
  warehouseId?: string
  autoLoad?: boolean
}>(), {
  warehouseId: '',
  autoLoad: true,
})

// ==================== Emits 定义 ====================
const emit = defineEmits<{
  (e: 'row-click', row: InventoryItem): void
  (e: 'selection-change', rows: InventoryItem[]): void
  (e: 'export-complete', filename: string): void
}>()

// ==================== 响应式数据 ====================
const tableData = ref<InventoryItem[]>([])
const loading = ref(false)
const total = ref(0)

// 分页
const pagination = reactive({
  page: 1,
  pageSize: 20,
})

// 搜索表单
const searchForm = reactive({
  itemCode: '',
  itemName: '',
  locationCode: '',
  status: '' as 'ALL' | 'IN_STOCK' | 'LOW_STOCK' | 'OUT_OF_STOCK',
})

// ==================== Computed ====================
// 低库存统计（类型自动推断）
const lowStockCount = computed(() =>
  tableData.value.filter(item => item.quantity < item.threshold).length
)

// 分页后的数据
const paginatedData = computed(() => {
  const start = (pagination.page - 1) * pagination.pageSize
  return tableData.value.slice(start, start + pagination.pageSize)
})

// 搜索参数（合并 props 和表单）
const queryParams = computed<InventoryQuery>(() => ({
  page: pagination.page,
  pageSize: pagination.pageSize,
  itemCode: searchForm.itemCode || undefined,
  itemName: searchForm.itemName || undefined,
  warehouseId: props.warehouseId || undefined,
  status: searchForm.status === 'ALL' ? undefined : searchForm.status,
}))

// ==================== API 调用 ====================
async function loadData() {
  loading.value = true
  try {
    const res = await fetchInventoryList(queryParams.value)
    // 类型安全：res.data 是 InventoryItem[]，total 是 number
    tableData.value = res.data
    total.value = res.total
  } catch (err: any) {
    ElMessage.error(err.message || '加载失败')
  } finally {
    loading.value = false
  }
}

async function handleExport() {
  try {
    const blob = await exportInventoryExcel(queryParams.value)
    const filename = `库存报表_${Date.now()}.xlsx`
    // 下载文件...
    emit('export-complete', filename)
  } catch {
    ElMessage.error('导出失败')
  }
}

// ==================== 事件处理 ====================
function handlePageChange(page: number) {
  pagination.page = page
  loadData()
}

function handleRowClick(row: InventoryItem) {
  emit('row-click', row)
}

// ==================== 生命周期 ====================
onMounted(() => {
  if (defaultProps.autoLoad) {
    loadData()
  }
})

// 监听搜索条件变化，重新加载
watch(
  () => [searchForm.itemCode, searchForm.itemName, searchForm.status],
  () => {
    pagination.page = 1
    loadData()
  }
)
</script>
```

**2. defineComponent（Options API 风格）泛型**：

```typescript
// components/InventoryCard.vue - Options API 风格
import { defineComponent, PropType, computed } from 'vue'
import type { InventoryItem } from '@/apis/types'

export default defineComponent({
  name: 'InventoryCard',
  props: {
    item: {
      type: Object as PropType<InventoryItem>,
      required: true,
    },
    editable: {
      type: Boolean,
      default: false,
    },
  },
  emits: ['update:item', 'delete'],
  setup(props, { emit }) {
    const isLowStock = computed(() => props.item.quantity < props.item.threshold)

    function handleUpdate(quantity: number) {
      emit('update:item', { ...props.item, quantity })
    }

    return { isLowStock, handleUpdate }
  },
})
```

**3. 最佳实践**：

```typescript
// ✅ 最佳实践1：props 使用泛型定义而非 propType 字符串
defineProps<{ item: InventoryItem }>()           // ✅
defineProps({ item: Object as PropType<InventoryItem> })  // ❌ 老写法

// ✅ 最佳实践2：emit 使用函数重载签名
const emit = defineEmits<{
  (e: 'update', item: InventoryItem): void
  (e: 'delete', id: string): void
}>()

// ✅ 最佳实践3：computed 返回类型尽量让 TS 自动推断
const count = computed(() => tableData.value.length)  // 自动推断为 number

// ✅ 最佳实践4：ref 泛型明确指定
const list = ref<InventoryItem[]>([])  // 明确类型，避免 any 逃逸

// ✅ 最佳实践5：event handler 类型
function handleClick(event: MouseEvent) {  // 原生事件加类型
  console.log(event.clientX)
}

// ✅ 最佳实践6：reactive 对象类型
const form = reactive<InventoryQuery>({
  page: 1,
  pageSize: 20,
  itemCode: '',
})

// ❌ 常见错误1：props 不标注类型，any 逃逸
const props = defineProps({ item: Object })  // ❌ 应该是 defineProps<{item: Item}>()

// ❌ 常见错误2：ref 不给泛型
const list = ref([])  // ❌ 应该是 ref<InventoryItem[]>([])

// ❌ 常见错误3：computed 内部修改状态
const doubled = computed(() => {
  count.value++  // ❌ 不要在 computed 中产生副作用
  return count.value * 2
})

// ❌ 常见错误4：watch 监听响应式对象属性不用函数
watch(searchForm.status, ...)  // ❌ 监听的是整个 searchForm，应该用 () => searchForm.status
```

**4. WMS 项目类型定义规范（参考 `wms-web/src/app/apis/types.ts`）**：
```typescript
// 所有 API 类型统一放在 apis/types.ts 或各模块的 types.ts 中
// 避免 any 交叉污染

export type ItemStatus = 'ACTIVE' | 'INACTIVE' | 'ARCHIVED'
export type InventoryStatus = 'IN_STOCK' | 'LOW_STOCK' | 'OUT_OF_STOCK'

// 请求参数
export interface InventoryQuery {
  page: number
  pageSize: number
  itemCode?: string
  itemName?: string
  warehouseId?: string
  status?: InventoryStatus
}

// 响应数据
export interface InventoryRecord {
  id: string
  itemCode: string
  itemName: string
  warehouseId: string
  warehouseName: string
  locationCode: string
  quantity: number
  threshold: number
  status: InventoryStatus
  updatedAt: string
}

// 统一分页结构
export interface PagedResponse<T> {
  list: T[]
  total: number
  page: number
  pageSize: number
}
```

### 追问方向

1. Vue 3.3+ 新增的 `defineOptions` 有什么用？——可以用 `<script setup>` 定义 `name`、`inheritAttrs`、`components` 等，不需要单独的 `<script>` 块
2. `InstanceType<typeof xxx>` 获取组件实例类型有什么用？——用于 `ref` 变量的类型标注，如 `const modalRef = ref<InstanceType<typeof InventoryModal> | null>(null)`
3. TypeScript `strict` 模式下有哪些常见报错？——`undefined is not a function`、`Cannot read property 'xxx' of null`、`Argument of type 'xxx' is not assignable to parameter of type 'yyy'`

### 避坑提示

- **TS 类型不要逃逸到模板**：defineProps 的泛型定义在编译时宏中处理，模板中的类型提示依赖这里写得对
- **循环依赖的类型**：两个文件相互 import types 可能导致 TS 无法解析——用 `import type` 打破循环，或抽取公共类型到 third.d.ts
- **defineEmits 的类型不能自动从 props 继承**：如果 emit 的 payload 包含 props 的子类型，需要手动对齐

---

## 附录：Vue3 + TypeScript 面试速查表

| 概念 | 关键点 |
|---|---|
| `<script setup>` | 顶层变量自动暴露给模板，不需要 return |
| `ref` vs `reactive` | ref 适合基础类型+重新赋值；reactive 适合对象+批量状态 |
| `computed` | 纯计算，缓存，依赖变化才重新执行 |
| `watch` vs `watchEffect` | watch 显式监听指定源；watchEffect 自动追踪依赖 |
| `Proxy` vs defineProperty | Proxy惰性、数组友好、新增/删除属性自动拦截 |
| `defineProps<T>()` | 编译时宏，泛型定义 props，模板类型安全 |
| `defineEmits<T>()` | 编译时宏，泛型定义 emit，类型安全的 event |
| `storeToRefs` | Pinia store 解构时保持响应式，actions 不用 |
| `Teleport` | DOM 物理移动到 body，组件树逻辑不变 |
| `Suspense` | async setup + #default/#fallback 插槽 |
| `defineAsyncComponent` | 组件懒加载，配合 loadingComponent/errorComponent |
| `v-memo` | 模板级渲染缓存，依赖数组变化才重渲染 |
| `shallowRef` | 只追踪 .value 替换，适合大数组整体替换场景 |
| `keyof T` | 获取 T 的所有 key 联合类型 |
| `Partial<T>` | `{ [K in keyof T]?: T[K] }` |
| `infer` | 条件类型中提取子类型，如 `T extends Promise<infer U> ? U : T` |
| `is` 类型守卫 | `function isX(val: any): val is X` 让 TS 收窄类型 |
| `as` 重映射 key | `[K in keyof T as \`${K}Id\`]: T[K]` 重命名 key |
