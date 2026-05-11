# 数据结构与算法 · 高频面试题（第8部分）

> 本系列完整覆盖 25 道大厂高频考点，每题包含：**题目**、**核心答案**、**追问方向**、**避坑提示**。

---

## 第 1 题 · 数组 vs 链表

### 题目

**请从时间复杂度、空间成本、CPU 缓存三个维度，对比数组和链表的查找、插入、删除操作，并说明 Java 中 ArrayList 和 LinkedList 的适用场景。**

### 核心答案

| 操作 | 数组 | 链表 |
|------|------|------|
| 查找（下标访问） | **O(1)**（随机寻址） | **O(n)**（顺序遍历） |
| 插入（头部） | O(n)（需后移元素） | **O(1)** |
| 删除（头部） | O(n)（需前移元素） | **O(1)** |
| 插入（尾部） | Amortized O(1)（扩容均摊） | O(1) 已知尾指针时 |
| 插入（中间） | O(n) | O(1) 已知位置，但需先遍历 |

**CPU 缓存层面**：数组是连续内存空间，**缓存命中率远高于链表**。访问数组元素时 CPU 预取机制（prefetch）可以将附近元素一起加载到 L1/L2 缓存，而链表节点分散在堆内存各处，每次跳转都可能触发一次 cache miss，开销往往是内存访问延迟的数十倍。

**Java 中的选择**：

- **ArrayList**：底层 Object[] 数组，适用于频繁随机访问（`get(int index)`）的场景。尾部插入/删除 amortized O(1)，但中间插入/删除需要 O(n) 元素移动。
- **LinkedList**：双向链表 (`Node<E> first, last`)，适用于**频繁的头部/尾部插入删除**、不需要随机访问的场景。遍历时使用 `Iterator` 可避免 O(n) 的 `get(index)`。

> **口诀**：查多改少用数组，改多查少用链表。

### 追问方向

- ArrayList 扩容机制是怎样的？`grow()` 方法每次扩展多少？
- 链表在 Java 中真的完全没有数组参与吗？（答案：每个 `Node` 对象本身占用堆内存，也有 GC 开销）
- 什么是 CPU 缓存行（cache line）？为什么数组遍历比链表快与之相关？

### 避坑提示

- 不能仅背复杂度表，要主动提到**缓存行**和**预取机制**——这是面试官区分"背题"和"理解"的杀手锏。
- 提到 ArrayList 扩容时，要说清楚** amortized O(1)** 而不是简单说 O(1)，体现你对均摊分析的理解。

---

## 第 2 题 · 栈和队列

### 题目

**栈和队列分别可以用哪些数据结构实现？并分别举三个实际应用场景。**

### 核心答案

**栈的实现**：`ArrayDeque`（推荐，效率最高）、`LinkedList`、数组。

**队列的实现**：循环数组（`ArrayDeque` 的循环实现）、`LinkedList`（双向链表头尾操作都是 O(1)）、优先队列（`PriorityQueue` — 特殊队列）。

**应用场景**：

| 场景 | 用栈还是队列 | 说明 |
|------|------------|------|
| **括号匹配**（判断 `(){ }[]` 是否合法） | 栈 | 左括号入栈，右括号匹配栈顶 |
| **单调栈**（Next Greater Element） | 栈 | 维护递增/递减栈，求每个元素右边第一个比它大的元素 |
| **约瑟夫环**（Josephus Problem） | 队列（模拟） | 每次从队头出一个人报数，报到 m 出队，最终幸存者 |
| **函数调用栈**（递归原理） | 栈 | 方法调用栈帧入栈，返回时出栈 |
| **BFS 遍历图/树** | 队列 | 每次弹出节点，将子节点入队 |
| **撤销/重做** | 栈 | 撤销栈 + 重做栈 |
| **表达式求值**（中缀转后缀） | 栈 | 运算符优先级处理 |

**约瑟夫环关键**：用队列模拟时，每轮将队首元素出队再插回队尾（报到 m 的前 m-1 人），报到 m 时出队不插回。也可数学递推：`f(n,m) = (f(n-1,m) + m) % n`。

### 追问方向

- 单调栈的时间复杂度是多少？为什么是 O(n)？
- 约瑟夫环如果不用队列，用数学递推怎么做？边界条件是什么？
- 如何用两个栈实现一个队列？（"栈逆序"原理）

### 避坑提示

- 约瑟夫环是高频手写题，**不要一上来就写队列模拟**，要能推导出数学递推式 `f(n,m) = (f(n-1,m) + m) % n`。
- 单调栈是算法题高频考点，必须能清晰解释"维护单调性"和"出栈时决策"的过程。

---

## 第 3 题 · 哈希表

### 题目

**什么是哈希冲突？有哪些解决方式？Java 的 HashMap 和 ConcurrentHashMap 分别是如何解决冲突的？扩容因子控制在多少比较合理，为什么？**

### 核心答案

**哈希冲突**：两个不同 key 通过哈希函数计算出相同的桶下标。

**解决方式**：

| 方式 | 原理 | 优缺点 |
|------|------|--------|
| **拉链法（Separate Chaining）** | 每个桶存放链表头指针（或红黑树），冲突元素挂在链上 | 实现简单；链表过长退化为 O(n)，Java 8 对链表>8转为红黑树 |
| **开放寻址法（Open Addressing）** | 冲突时探测下一个空槽：线性探测 `h(k)+i`、二次探测、`Double Hashing` | 不用额外指针，适合装载因子低的场景；删除操作复杂，容易产生"聚集" |
| **再哈希法** | 使用第二个哈希函数计算备用槽 | 探测序列固定时仍可能冲突 |

**Java HashMap**：

- JDK 8 前：拉链法 + 链表
- JDK 8+：链表长度 > **8** 且桶数组长度 >= **64** 时，链表转为**红黑树**（TreeNode）
- 扩容时 rehash（重新计算每个元素的桶位置）

**ConcurrentHashMap**：

- JDK 7：Segment 分段锁，每个 Segment 持有一把锁（`ReentrantLock`），默认 16 个 Segment
- JDK 8+：**CAS + Synchronized**（Node 的 `volatile` + CAS + `synchronized`），锁的粒度细化到每个桶，不再用 Segment
- 读操作完全无锁（volatile），写操作锁单个桶

**扩容因子**：`loadFactor = 0.75`

- 太小（如 0.5）：频繁扩容，空间浪费
- 太大（如 1.0）：冲突增多，链表/红黑树变长，查找退化
- **0.75 是时间空间 trade-off**：哈希表 put 时会先检查 `size > threshold (capacity * loadFactor)`，超过触发 `resize()` 扩容 2 倍。

### 追问方向

- HashMap 在并发下可能出现什么问题？（死循环、数据丢失 — JDK 7 的头插法会导致环形链表）
- 红黑树退化到链表的阈值为什么是 8？
- 如何自己实现一个简易哈希表？key 的 `hashCode()` 方法有什么要求？

### 避坑提示

- 面试官常追问"JDK 7 vs JDK 8 的 ConcurrentHashMap 区别"，要区分清楚**分段锁 vs CAS+Synchronized**。
- 说扩容因子时，不仅要报 0.75，还要解释**均摊扩容成本**的概念，体现你理解均摊分析。

---

## 第 4 题 · 二叉树遍历

### 题目

**请写出二叉树前序、中序、后序遍历的递归和迭代实现，并说明如何用层序遍历求二叉树深度。**

### 核心答案

```java
// 前序遍历（根-左-右）
void preOrder(TreeNode root) {
    if (root == null) return;
    visit(root.val);          // 根
    preOrder(root.left);      // 左
    preOrder(root.right);     // 右
}

// 中序遍历（左-根-右）
void inOrder(TreeNode root) {
    if (root == null) return;
    inOrder(root.left);
    visit(root.val);          // 根
    inOrder(root.right);
}

// 后序遍历（左-右-根）
void postOrder(TreeNode root) {
    if (root == null) return;
    postOrder(root.left);
    postOrder(root.right);
    visit(root.val);          // 根
}

// 层序遍历（队列）
void levelOrder(TreeNode root) {
    if (root == null) return;
    Queue<TreeNode> q = new LinkedList<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = q.poll();
            visit(node.val);
            if (node.left != null)  q.offer(node.left);
            if (node.right != null) q.offer(node.right);
        }
    }
}

// 求最大深度
int maxDepth(TreeNode root) {
    if (root == null) return 0;
    int left = maxDepth(root.left);
    int right = maxDepth(root.right);
    return Math.max(left, right) + 1;
}
```

**迭代遍历核心技巧**：前序/中序/后序用**栈**模拟递归，手动维护节点访问顺序。

### 追问方向

- 前序遍历的迭代写法中，入栈顺序和出栈（访问）顺序是什么关系？
- 后序遍历的迭代实现有什么特殊之处？（需要记录上一个访问的节点，避免重复访问右子树）
- 如何用 Morris 遍历实现 O(1) 空间复杂度的中序遍历？

### 避坑提示

- 后序遍历迭代版是**高频手写题**，容易写错。要说清楚如何用 `prev` 指针判断右子树是否已访问。
- 层序遍历时 `for (int i = 0; i < size; i++)` 中的 `size` 很重要——它是当前层的节点数，确保每层节点正确分离。

---

## 第 5 题 · 二叉搜索树

### 题目

**二叉搜索树（BST）的查找、插入、删除操作时间复杂度是多少？什么是平衡二叉树？AVL 和红黑树有什么区别？**

### 核心答案

**BST 复杂度**：平均 O(log n)，最坏 O(n)（退化为链表，顺序插入时）。

**查找**：从根开始，比根大往右，比根小往左，最坏 O(n)。

**插入**：先查找确定位置（找到 null 叶子），插入新节点，最坏 O(n)。

**删除**（三种情况）：

1. **叶子节点**：直接删
2. **只有左/右子树**：用唯一子树顶替
3. **左右子树都有**：用**前驱**（左子树最大）或**后继**（右子树最小）节点值替换，转为情况 1 或 2

**平衡二叉树**：任意节点的左右子树高度差不超过 1（AVL）或满足红黑约束（红黑树）。

| 对比 | AVL 树 | 红黑树 |
|------|--------|--------|
| 平衡标准 | **严格平衡**（左右子树高度差 ≤1） | **近似平衡**（最长路径 ≤ 2×最短路径） |
| 插入/删除旋转次数 | 最多 O(log n) 次旋转 | 通常 **最多 2-3 次旋转** |
| 查找效率 | 更高（更平衡） | 略低（允许一定不平衡） |
| 应用 | 数据库（查找多） | Java `TreeMap`/`HashMap` 桶、Linux 进程调度 |

### 追问方向

- BST 删除时，什么情况下选前驱节点而不是后继节点？
- 为什么 MySQL 索引用 B+ 树而不是 AVL 或红黑树？（分支因子、多路平衡、磁盘友好）
- BST 如何支持"第 K 大"查询？

### 避坑提示

- BST 删除的三种情况必须能**画图说明**，纯背文字容易乱。
- AVL 和红黑树的区别是经典比较题，不能只说"AVL 更严格"，要量化：AVL 旋转次数是 O(log n)，红黑树是 O(1)。

---

## 第 6 题 · 红黑树

### 题目

**红黑树有哪五大性质？插入操作可能触发哪些旋转？请描述旋转类型（LL/RR/LR/RL）。红黑树为什么比 AVL 应用更广泛？**

### 核心答案

**红黑树五大性质**：

1. 每个节点非红即黑
2. 根节点是黑色
3. 叶子节点（NIL/NULL）是黑色
4. **红节点的子节点必须是黑色**（即不存在两个相邻的红节点）
5. 对每个节点，从该节点到其所有后代叶子节点的路径上，**黑色节点数量相同**（黑高相同）

**插入旋转**（叔叔节点颜色决定策略）：

| 情况 | 旋转类型 | 说明 |
|------|---------|------|
| 叔叔是红节点 | **只变色**（父、叔变黑，祖父变红） | 不需要旋转 |
| 叔叔是黑节点 + LL 型（父是祖父左孩子，当前是父左孩子） | **右单旋** + 变色 |  |
| 叔叔是黑节点 + RR 型（父是祖父右孩子，当前是父右孩子） | **左单旋** + 变色 |  |
| 叔叔是黑节点 + LR 型（父是祖父左孩子，当前是父右孩子） | **先左旋（变成LL）再右旋** |  |
| 叔叔是黑节点 + RL 型（父是祖父右孩子，当前是父左孩子） | **先右旋（变成RR）再左旋** |  |

**为什么比 AVL 应用广**：

- AVL 插入/删除时可能触发 **O(log n) 次旋转**，而红黑树最多 **2-3 次**（颜色调整）
- 插入/删除操作远多于查找操作时（如 TreeMap 的 put/remove），红黑树性能更好
- **实际工程**：Java `TreeMap`/`HashMap`、C++ `std::map`、Linux 内核调度器都用红黑树

### 追问方向

- 红黑树删除操作有哪些情况？删除黑色节点如何保持黑高不变？
- 为什么红黑树新插入的节点默认是红色而不是黑色？（避免破坏黑高，增加调整成本）
- 红黑树的高度范围是多少？（最深 ≤ 2log₂(n+1)，但实际高度 ≈ 2log₂n）

### 避坑提示

- 面试时能**画出四种旋转的图**比背文字更重要。建议用铅笔纸笔配合描述。
- 很多候选人背了旋转类型但说不出**为什么变色**：因为红色破坏"无相邻红节点"规则，黑色破坏"黑高相同"规则，调整方向不同。

---

## 第 7 题 · 堆

### 题目

**什么是堆（大根堆/小根堆）？如何用数组实现？TopK 问题如何用堆解决？Java 中的 PriorityQueue 是小根堆还是大根堆？**

### 核心答案

**堆**：完全二叉树，任意节点值 >=（或 <=）其子节点值，分为大根堆（max heap）和小根堆（min heap）。

**数组实现**（下标从 1 开始更方便计算）：

```
parent(i) = i / 2
left(i)   = 2 * i
right(i)  = 2 * i + 1
```

用数组存储完全二叉树，不需要指针，节省空间。

**核心操作**：

- `heapifyUp`：新插入元素上浮（比较父节点，上浮直到满足堆性质）
- `heapifyDown`：堆顶出堆后，用最后一个元素补上堆顶，然后下沉（与较大的子节点比较，下沉直到满足堆性质）
- 两个操作都是 **O(log n)**

**TopK 问题**：

- 建立大小为 K 的**小根堆**（堆顶是 K 个中最小的）
- 遍历剩余元素，若 `> heapTop`，则替换堆顶并 `heapifyDown`
- 时间复杂度：**O(n log K)**，适合 n >> K 的场景
- 为什么不建大根堆？因为大根堆需要保存全部 n 个元素

**Java PriorityQueue**：

- **默认是小根堆**（`Comparator.reverseOrder()` 转大根堆）
- 底层是**对象数组**实现的**二叉堆**（不是红黑树）

### 追问方向

- 堆排序的完整过程是什么？建堆 O(n) 怎么来的？（从最后一个非叶子节点下沉，`∑_{i=1}^{log n} (n/2^i) * i = O(n)`）
- 如果数据流式输入，无法一次性加载全部数据，如何求 TopK？（借 Extrabody 概念，维护固定大小的堆）
- PriorityQueue 的 `remove(Object o)` 是什么时间复杂度？（O(n)，需要线性查找）

### 避坑提示

- 有人误以为 TopK 用大根堆——这是错的，大根堆保存的是最大的 K 个，但每次替换都要比较堆顶，如果堆顶不是最小的就需要重新调整。
- 建堆时间复杂度 O(n) 的推导要能说清楚——不是 O(n log n)，这是很多候选人的盲区。

---

## 第 8 题 · 图

### 题目

**图的存储方式有哪些？邻接矩阵和邻接表各有什么优缺点？请描述 BFS 和 DFS 遍历图的完整过程，以及 DAG 的判定方法。**

### 核心答案

**存储方式**：

| 方式 | 结构 | 适用场景 |
|------|------|----------|
| **邻接矩阵** | `int[][] matrix`，`matrix[i][j]=1` 表示 i→j 有边 | 稠密图（`|E| ≈ |V|²`），O(1) 判定两点是否有边 |
| **邻接表** | `List<Integer>[] adj`，`adj[i]` 存所有从 i 出发的邻居 | 稀疏图（`|E| << |V|²`），省空间，遍历邻居方便 |

**BFS 遍历**（队列）：

```java
void bfs(int start) {
    boolean[] visited = new boolean[V];
    Queue<Integer> q = new LinkedList<>();
    visited[start] = true;
    q.offer(start);
    while (!q.isEmpty()) {
        int v = q.poll();
        visit(v);
        for (int u : adj[v]) {
            if (!visited[u]) {
                visited[u] = true;
                q.offer(u);
            }
        }
    }
}
```

**DFS 遍历**（栈或递归）：

```java
void dfs(int start) {
    boolean[] visited = new boolean[V];
    dfsHelper(start, visited);
}
void dfsHelper(int v, boolean[] visited) {
    visited[v] = true;
    visit(v);
    for (int u : adj[v]) {
        if (!visited[u]) dfsHelper(u, visited);
    }
}
```

**DAG 判定**：用 **Kahn 算法**（拓扑排序 BFS 版）：统计每个节点的入度，将入度为 0 的节点加入队列，逐一删去。如果最终所有节点都被删除，则是 DAG；如果删不掉（有环），则不是 DAG。

### 追问方向

- BFS 和 DFS 的空间复杂度分别是多少？（BFS O(V)，DFS 最坏 O(V)，但实际递归深度为 O(h)，h 是树高）
- 拓扑排序的 DFS 版怎么实现？（在 DFS 完成后按逆序记录节点）
- 如果图中有负权边用什么算法？（Bellman-Ford）没有负权呢？（Dijkstra）

### 避坑提示

- BFS 用**队列**（先入先出），DFS 用**栈**或递归（先入后出）。面试时不要把两者搞混。
- DAG 判定是独立考点，Kahn 算法的"删入度为 0 的节点"过程要能说清楚。

---

## 第 9 题 · 排序算法总览

### 题目

**请列出常见排序算法的时间复杂度、空间复杂度、稳定性，并说明哪些是原地排序。**

### 核心答案

| 算法 | 时间（平均） | 时间（最坏） | 空间 | 稳定 | 原地 |
|------|------------|------------|------|------|------|
| 冒泡排序 | O(n²) | O(n²) | O(1) | ✅ 稳定 | ✅ |
| 选择排序 | O(n²) | O(n²) | O(1) | ❌ 不稳定 | ✅ |
| 插入排序 | O(n²) | O(n²) | O(1) | ✅ 稳定 | ✅ |
| 希尔排序 | O(n^1.3) ~ O(n^1.5) | O(n²) | O(1) | ❌ 不稳定 | ✅ |
| 归并排序 | O(n log n) | O(n log n) | O(n) | ✅ 稳定 | ❌（需要 O(n) 辅助） |
| 快速排序 | O(n log n) | O(n²) | O(log n)（递归栈） | ❌ 不稳定 | ✅（原地 partition） |
| 堆排序 | O(n log n) | O(n log n) | O(1) | ❌ 不稳定 | ✅ |

**稳定性定义**：排序后相等的两个元素相对顺序保持不变。

### 追问方向

- 为什么希尔排序不稳定？（间隔插入可能打乱相同元素的相对顺序）
- 既然冒泡/选择/插入都是 O(n²)，为什么插入排序在实际中比选择排序表现好？（因为插入排序的常数因子更小，且对近乎有序的数据可接近 O(n)）
- 排序算法在 JVM 中哪些用了？（`Arrays.sort()` 对基本类型用 Dual-Pivot QuickSort（不稳定），对对象用 TimSort（归并变体，稳定））

### 避坑提示

- 快速排序最坏 O(n²) 的触发条件（**近乎有序的数组 + 固定 pivot 选择策略**）要说清楚，这也是面试官追问优化的切入点。
- 空间复杂度不要搞混：归并排序 O(n) 是**真的需要额外数组**，快排 O(log n) 是**递归栈深度**，堆排序 O(1) 是因为完全原地。

---

## 第 10 题 · 快速排序

### 题目

**快速排序的 partition 思路是什么？请写出单边循环法和双边循环法的核心逻辑。快排的时间复杂度如何分析？有哪些优化手段？**

### 核心答案

**partition 核心**：将数组分为两部分，`pivot` 左边的元素都 `<= pivot`，右边的元素都 `>= pivot`。

**单边循环法（Lomuto 划分）**：

```java
int partition(int[] arr, int low, int high) {
    int pivot = arr[high];           // 选最右为基准
    int i = low - 1;                  // i 是小于区的边界
    for (int j = low; j < high; j++) {
        if (arr[j] <= pivot) {
            swap(arr, ++i, j);       // 元素进入小于区
        }
    }
    swap(arr, ++i, high);             // pivot 放到中间
    return i;
}
```

**双边循环法（Hoare 划分）**：从左右向中间扫描，i 从左找第一个 `> pivot` 的，j 从右找第一个 `< pivot` 的，交换。最后 i == j 为基准位置。

**时间复杂度分析**：

- **最好情况**（每次 pivot 恰好是中位数）：`T(n) = 2T(n/2) + O(n)` → **O(n log n)**（递归树高度 log n，每层 partition O(n)）
- **最坏情况**（每次 pivot 都是最小/最大）：`T(n) = T(n-1) + O(n)` → **O(n²)**
- **随机化 pivot** 可以避免最坏情况，期望 O(n log n)

**优化手段**：

1. **三数取中（Median-of-Three）**：取 `low`、`mid`、`high` 三个位置的中位数作为 pivot，避免固定 pivot 在有序数组上的退化
2. **小区间优化**：当 `high - low < 某个阈值`（如 10）时，改用**插入排序**（常数因子小，递归深度也减小）
3. **随机化 pivot**：`swap(arr, low, random(low, high))`

### 追问方向

- Lomuto 和 Hoare 两种 partition 哪个更好？（Hoare 通常 partition 次数更少，效率略高；但 Lomuto 更易写对，是面试常选）
- 快排是原地排序，但实际空间复杂度为什么是 O(log n)？（递归栈），如何做到真正的 O(1) 空间？（用栈模拟递归）
- Dijkstra 的三路划分（Three-Way Partition）是什么？解决了什么问题？（处理大量重复元素的数组，避免退化）

### 避坑提示

- 手写 partition 时，**边界条件**（`low`、`high`、`i`、`j` 的更新）是出错重灾区。建议先说清楚变量含义再写代码。
- 说"快排比归并排序快"时要补充前提：平均情况下；最坏情况归并更稳定。两者不是绝对的谁更好。

---

## 第 11 题 · 归并排序

### 题目

**归并排序的分治思想是什么？请描述合并两个有序数组的过程。链表的归并排序有什么优势？**

### 核心答案

**分治三步**：

1. **分解**：将数组从中间分成两半，递归处理左右半部分
2. **解决**：递归排序左右两个子数组
3. **合并**：将两个有序子数组合并为一个有序数组

**合并两个有序数组**：

```java
void merge(int[] arr, int low, int mid, int high) {
    int[] tmp = new int[high - low + 1];
    int i = low, j = mid + 1, k = 0;
    while (i <= mid && j <= high) {
        tmp[k++] = arr[i] <= arr[j] ? arr[i++] : arr[j++];
    }
    while (i <= mid)  tmp[k++] = arr[i++];
    while (j <= high) tmp[k++] = arr[j++];
    System.arraycopy(tmp, 0, arr, low, tmp.length);
}
```

**时间复杂度**：每层 merge O(n)，共 log n 层 → **O(n log n)**，且是**稳定**的 O(n log n)。

**链表归并排序的优势**：

- 找链表中间节点：只需一次遍历（快慢指针），**不需要额外数组**
- 合并链表：不需要开辟额外空间（节点指针重连即可），**空间复杂度 O(1)**
- 综合：链表归并排序可以做到 **O(n log n) 时间 + O(1) 空间**（数组归并排序需要 O(n) 辅助空间）

**数组归并排序的问题**：需要 O(n) 辅助空间，这是它的最大缺点。

### 追问方向

- 归并排序的递归深度是多少？（log₂n），递归栈空间呢？（O(log n)）
- 如果要实现"外部排序"（数据无法一次性加载到内存），归并排序有什么优势？（分块排序 + 多路归并，适合磁盘读写）
- 自底向上的归并排序（Bottom-Up Merge Sort）怎么写？（从小数组开始，逐步扩大归并区间，不需要递归）

### 避坑提示

- 合并时**先复制到临时数组**再合并回原数组，这是标准做法，不要在合并时直接在原数组上移动指针（容易出错）。
- 链表归并的优势是**高频追问点**，很多候选人知道归并排序但说不出链表版的 O(1) 空间优势。

---

## 第 12 题 · 排序稳定性

### 题目

**什么是排序的稳定性？请列出常见稳定排序和不稳定排序，并说明在什么场景下稳定性至关重要。**

### 核心答案

**稳定性**：排序后相等的两个元素**相对顺序**与排序前一致。

**稳定排序**：冒泡排序、插入排序、归并排序、TimSort、二叉树排序（按中序遍历时）。

**不稳定排序**：选择排序、希尔排序、快速排序、堆排序。

**不稳定原因举例**：

- **选择排序**：在交换 `swap(arr, i, minIdx)` 时，`minIdx` 位置的元素可能被交换到前面，打乱相同元素的相对顺序
- **快速排序**：partition 时相等元素可能被分到不同侧

**稳定性至关重要的场景**：

1. **多级排序**：先按成绩排序，再按学号排序。如果第一次排序不稳定，第二次排序会破坏第一次的结果
2. **电商排序**：按销量排序后，再按价格排序。不稳定排序会导致同一价格的商品销量顺序错乱
3. **金融数据**：按时间戳排序后再按金额排序，不稳定会破坏时间顺序

**Java 中的体现**：`Arrays.sort()` 对**基本类型**用快速排序（不稳定），对**对象数组**用 TimSort（稳定，归并变体）。

### 追问方向

- 为什么很多语言对基本类型用不稳定排序？（基本类型无身份信息，稳定与否无意义，不稳定排序通常更快）
- 希尔排序为什么不稳定？（间隔 g 的插入排序会打乱相同元素的相对顺序）
- 如何将不稳定排序"改造"为稳定？（给每个元素加一个原始索引作为第二排序键）

### 避坑提示

- 不要说"不稳定排序就是不好的"——稳定性只是取舍之一。快排、堆排虽然不稳定但平均更快。
- 实际回答时能用**多级排序**举例会让面试官印象深刻，比如"先按部门排序，再按工号排序"。

---

## 第 13 题 · 二分查找

### 题目

**请实现一个标准的二分查找，并写出其变体：查找左边界、查找右边界、查找第一个大于目标值的元素、查找最后一个小于等于目标值的元素。**

### 核心答案

**标准二分**（查找目标值，返回下标或 -1）：

```java
int binarySearch(int[] arr, int target) {
    int lo = 0, hi = arr.length - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target)  return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

**左边界**（查找第一个 >= target 的位置）：

```java
int leftBound(int[] arr, int target) {
    int lo = 0, hi = arr.length;  // hi = length（不是 length-1）
    while (lo < hi) {              // lo == hi 时退出
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid;             // arr[mid] >= target，缩小右边界
    }
    return lo;  // lo 即为左边界
}
```

**右边界**（查找最后一个 <= target 的位置）：

```java
int rightBound(int[] arr, int target) {
    int lo = 0, hi = arr.length;
    while (lo < hi) {
        int mid = lo + (hi - lo + 1) / 2;  // 上取整，避免死循环
        if (arr[mid] > target) hi = mid - 1;
        else lo = mid;                     // arr[mid] <= target
    }
    return lo;
}
```

**关键区别**：左边界用 `mid = lo + (hi - lo) / 2`（下取整），右边界用**上取整**（`+1`），否则 `lo == hi` 死循环。

### 追问方向

- 二分查找的复杂度为什么是 O(log n)？（每次将搜索区间缩小一半）
- 二分查找适用于什么数据结构？（**有序数组**，链表二分是 O(n) 因为无法随机访问）
- 如果数据是流式的（持续新增），二分查找还能用吗？（可以用"二分搜索 BST"或"跳表"来维护有序性）

### 避坑提示

- `mid = (lo + hi) / 2` 可能**整数溢出**，面试时用 `lo + (hi - lo) / 2` 更安全，体现工程意识。
- 左边界代码中 `hi = arr.length` 而不是 `arr.length - 1`，以及 `while (lo < hi)` 而不是 `lo <= hi`——这是区分左边界和普通二分的关键。

---

## 第 14 题 · 动态规划

### 题目

**请用动态规划的思想分析斐波那契数列的求解，并扩展到：0-1 背包问题、最长递增子序列（LIS）、编辑距离。**

### 核心答案

**斐波那契数列**：

```java
// 递归 + 记忆化（自顶向下）
int fibMemo(int n, int[] memo) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n];
    return memo[n] = fibMemo(n-1, memo) + fibMemo(n-2, memo);
}

// 迭代 + 空间优化（自底向上）
int fibDP(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        int c = a + b;
        a = b; b = c;
    }
    return b;
}
```

**0-1 背包问题**：

```java
// dp[i][w] = 前 i 个物品，容量 w 的最大价值
int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n+1][capacity+1];
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            dp[i][w] = dp[i-1][w];  // 不选第 i 个
            if (w >= weights[i-1])
                dp[i][w] = Math.max(dp[i][w],
                    dp[i-1][w - weights[i-1]] + values[i-1]);
        }
    }
    return dp[n][capacity];
}
// 空间优化：一维 dp，从后往前遍历
```

**LIS（最长递增子序列）**：

```java
// O(n²) DP版
int lisDP(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    for (int i = 1; i < n; i++)
        for (int j = 0; j < i; j++)
            if (nums[j] < nums[i])
                dp[i] = Math.max(dp[i], dp[j] + 1);
    return Arrays.stream(dp).max().orElse(0);
}

// O(n log n) 二分版：维护一个有序的"尾部数组"
int lisBinary(int[] nums) {
    int[] tail = new int[nums.length];
    int len = 0;
    for (int num : nums) {
        int i = Arrays.binarySearch(tail, 0, len, num);
        if (i < 0) i = -(i + 1);
        tail[i] = num;
        if (i == len) len++;
    }
    return len;
}
```

**编辑距离**（LeetCode 72）：

```java
int editDistance(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m+1][n+1];
    for (int i = 0; i <= m; i++) dp[i][0] = i;  // 删除 i 个字符
    for (int j = 0; j <= n; j++) dp[0][j] = j;  // 插入 j 个字符
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i-1) == s2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1];
            else
                dp[i][j] = 1 + Math.min(
                    Math.min(dp[i-1][j],   // 删除
                             dp[i][j-1]),  // 插入
                             dp[i-1][j-1]); // 替换
        }
    return dp[m][n];
}
```

**DP 套路**：状态定义 → 初始化 → 状态转移方程 → 遍历方向 → 返回值。

### 追问方向

- 0-1 背包的空间优化为什么要从后往前遍历？（避免同一物品被重复选取）
- LIS 二分版中 `tail[i]` 的含义是什么？（长度为 i+1 的递增子序列的最小尾部值）
- 编辑距离的应用场景？（拼写纠错、DNA 序列比对、差量更新算法）

### 避坑提示

- DP 问题最重要的是**状态定义**和**状态转移方程**，这两个说清楚代码实现只是翻译。
- LIS 的 O(n log n) 二分法是高频进阶题，很多候选人只知道 O(n²) DP 版本，要能推导 O(n log n) 版本的正确性。

---

## 第 15 题 · 贪心算法

### 题目

**什么是贪心算法？它和动态规划的核心区别是什么？请用贪心算法解决：区间调度问题、哈夫曼编码、钱币找零。**

### 核心答案

**贪心算法**：每一步选择都取当前状态下的**最优解**（局部最优），期望最终获得全局最优解。

**贪心 vs 动态规划**：

| | 贪心 | 动态规划 |
|--|------|----------|
| 决策时机 | **先做选择**（选择的解空间缩小） | **最后做选择**（比较所有子问题的最优） |
| 最优子结构 | 不一定满足 | 必须满足 |
| 子问题重叠 | 不需要 | 需要（DP 利用重叠子问题） |
| 正确性证明 | 需要证明**贪心选择性质** | 需要证明**最优子结构 + 状态转移** |

**区间调度**（最多安排多少不重叠的区间）：

```java
// 按结束时间排序，贪心选最早结束的
int intervalScheduling(int[][] intervals) {
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[1]));
    int count = 0, lastEnd = Integer.MIN_VALUE;
    for (int[] interval : intervals) {
        if (interval[0] >= lastEnd) {
            count++;
            lastEnd = interval[1];
        }
    }
    return count;
}
```

**哈夫曼编码**：使用**小根堆**构造最优前缀码。每次从堆中取出权重最小的两个节点合并，生成新节点放回堆中。证明：哈夫曼树是最优二叉树（**贪心选择**——最小权重节点深度最深）。

**钱币找零**：给定面额 `{1, 5, 10, 20, 50, 100}`，用最少数目找零。策略：**贪心选最大面额**（因为面额是 1 的倍数）。注：某些面额组合下贪心失效（如 `{1, 3, 4}` 找 6 元，贪心 4+1+1=3 枚，但最优是 3+3=2 枚）。

### 追问方向

- 区间调度的正确性如何证明？（交换论证——如果选择了结束时间更晚的区间，可以替换为更早结束的区间，不会减少可选区间数）
- 哈夫曼编码为什么用小根堆？（每次需要快速获取最小权重的两个节点）
- 什么情况下贪心算法会失效？（不满足贪心选择性质，如硬币面额 {1,3,4}）

### 避坑提示

- 贪心算法必须**能证明正确性**，只说"我觉得这样选最优"没有说服力。要掌握"交换论证"和"数学归纳法"两种证明方式。
- 钱币找零的"面额体系"是判断贪心是否有效的关键：若面额是**canonical**（如美元体系），贪心有效；否则需要 DP。

---

## 第 16 题 · 回溯算法

### 题目

**请用回溯算法解决：全排列、组合求和（LeetCode 39）、N 皇后问题。说明什么是剪枝，以及剪枝的常见策略。**

### 核心答案

**全排列**（LeetCode 46）：

```java
void backtrack(int[] nums, List<Integer> path, boolean[] used) {
    if (path.size() == nums.length) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;            // 已选过的跳过
        used[i] = true;
        path.add(nums[i]);
        backtrack(nums, path, used);
        path.remove(path.size() - 1);     // 回溯
        used[i] = false;
    }
}
```

**组合求和**（LeetCode 39，数组无重复，每个元素可重复选）：

```java
void backtrack(int[] candidates, int target, int start, List<Integer> path) {
    if (target < 0) return;
    if (target == 0) {
        result.add(new ArrayList<>(path));
        return;
    }
    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > target) continue; // 剪枝：提前终止
        path.add(candidates[i]);
        backtrack(candidates, target - candidates[i], i, path); // i 不是 i+1，允许重复使用
        path.remove(path.size() - 1);
    }
}
```

**N 皇后**（LeetCode 51）：

```java
void solve(int row) {
    if (row == n) {
        result.add(boardToString());
        return;
    }
    for (int col = 0; col < n; col++) {
        if (isValid(row, col)) {
            placeQueen(row, col);
            solve(row + 1);
            removeQueen(row, col);
        }
    }
}
// 剪枝：在放置前检查列、主对角线、副对角线是否已有皇后
boolean isValid(int row, int col) {
    for (int i = 0; i < row; i++)
        if (board[i] == col) return false;
    for (int i = 0; i < row; i++)
        if (Math.abs(board[i] - col) == Math.abs(i - row)) return false;
    return true;
}
```

**剪枝策略**：

- **可行性剪枝**：当前状态已经不满足约束条件，提前返回（如组合求和 `target < 0`）
- **已访问剪枝**：避免重复选择（如全排列的 `used[]`）
- **对称剪枝**：避免等价的重复搜索（如数独中对称填法）

### 追问方向

- 回溯的时间复杂度为什么是 O(n!) 或更高？（因为搜索树很大，实际用剪枝降低）
- N 皇后如何用位运算优化？（利用整数的位表示列、主对角线、副对角线的占用状态）
- 全排列 II（LeetCode 47，有重复数字）如何去重？（先排序，相邻相同且未使用时跳过）

### 避坑提示

- 回溯的**撤销选择**（回溯）步骤是高频出错点，漏写或写错顺序会导致结果错误。养成习惯：`path.add()` 后紧跟 `backtrack()`，再紧跟 `path.remove()`。
- N 皇后必须能用 `boolean[]` 或 `int[]` 表示棋盘并写出合法性检查——这是面试手写题的经典。

---

## 第 17 题 · BFS 和 DFS

### 题目

**BFS 和 DFS 各自适合什么场景？DFS 的递归实现和栈实现有什么区别？什么是回溯和剪枝？**

### 核心答案

| | BFS | DFS |
|--|-----|-----|
| 数据结构 | **队列**（先入先出） | **栈**或递归（先入后出） |
| 搜索特点 | **层层推进**，先访问近的节点 | **先深入一条路**，再回溯 |
| 最短路径 | **天然支持**（第一次到达即最短） | 不能直接用于无权最短路（需要记录层信息） |
| 空间复杂度 | O(V)（最宽层的节点数） | O(h)（递归深度/栈高度） |
| 适用场景 | 最短路径、层次遍历、扩散问题 | 连通分量、拓扑排序、找所有路径 |

**BFS 适用场景**：

- 无权图的最短路径（迷宫最少步数）
- 二叉树的层次遍历
- 扩散问题（岛屿数量、腐烂橘子）

**DFS 递归 vs 栈**：

- 递归：系统栈自动管理参数传递，实现简单，但递归深度受栈大小限制（默认约 1000 层）
- 显式栈：手动模拟，自己控制，适合深度不受限制或需要记录中间状态

**回溯**：DFS 遍历时，当一条路走到死胡同，**撤销最后的选择**（状态恢复），回到上一步尝试其他分支。

**剪枝**：在搜索过程中，**提前跳过**明显不可能产生最优解的分支，减少搜索空间。

### 追问方向

- 为什么 BFS 能找到无权图的最短路径？（BFS 按层次扩展，第一次访问到某个节点时经过了最短路径）
- 递归太深会怎样？（StackOverflowError，Linux 默认栈大小约 8MB）
- 如何用 BFS 判断二分图？（染色法：相邻节点不同色，遇到同色说明不是二分图）

### 避坑提示

- BFS 的最短路径正确性容易被质疑——要强调"第一次访问"这个条件，不是每次都取最短。
- 递归和栈实现的 DFS 本质相同，递归只是系统帮你用栈。要能互相转换。

---

## 第 18 题 · 字符串匹配：KMP 和 BM

### 题目

**请描述 KMP 算法的核心思想，以及如何构造 next 数组（部分匹配表）。BM 算法和 KMP 有什么区别？**

### 核心答案

**KMP 核心思想**：当发生不匹配时，**利用已匹配的信息**，将模式串向右滑动"最长可匹配前缀"的长度，避免重复比较。

**next 数组构造**（PMT/部分匹配表）：

```java
void buildNext(String pattern) {
    int[] next = new int[pattern.length()];
    next[0] = 0;  // 无真前缀时为 0
    int j = 0;    // j 是当前匹配的最长前缀长度
    for (int i = 1; i < pattern.length(); i++) {
        while (j > 0 && pattern.charAt(i) != pattern.charAt(j))
            j = next[j - 1];  // 回退到次长匹配前缀
        if (pattern.charAt(i) == pattern.charAt(j))
            j++;
        next[i] = j;
    }
}
```

`next[i]` 的含义：`pattern[0..i]` 中，**最长相等前后缀的长度**。

**KMP 匹配**：

```java
int kmp(String text, String pattern) {
    int[] next = buildNext(pattern);
    int j = 0;
    for (int i = 0; i < text.length(); i++) {
        while (j > 0 && text.charAt(i) != pattern.charAt(j))
            j = next[j - 1];
        if (text.charAt(i) == pattern.charAt(j)) j++;
        if (j == pattern.length())
            return i - j + 1;  // 找到匹配
    }
    return -1;
}
```

**BM 算法**（Boyer-Moore）：从模式串**尾部向前**匹配，遇到不匹配时根据"坏字符规则"和"好后缀规则"选择最大滑动距离。最好情况 O(n/m)，远超 KMP。

| 对比 | KMP | BM |
|------|-----|-----|
| 匹配方向 | 从前向后 | 从后向前 |
| 滑动依据 | next 数组（已匹配前缀） | 坏字符 + 好后缀 |
| 最优复杂度 | O(n+m) | O(n/m)（实践更快） |

### 追问方向

- next 数组中 `j = next[j - 1]` 的回退机制是什么原理？（当前字符失配时，找到次长的可匹配前缀）
- BM 的"坏字符规则"如何计算滑动距离？（模式串中从后往前找失配字符在模式串中的位置）
- KMP 和 BM 在什么场景下表现差异最大？（模式串有大量重复字符时，BM 滑动更激进）

### 避坑提示

- next 数组是 KMP 的核心，必须能**手写构造过程**，不是只会调用函数。
- 面试常问：KMP 为什么能保证不回退文本指针 i？（因为利用了已匹配的信息，前面的比较结果没有浪费）

---

## 第 19 题 · 布隆过滤器

### 题目

**布隆过滤器的原理是什么？如何计算误判率？Redis 6 如何实现布隆过滤器？布隆过滤器在亿级数据去重场景下如何使用？**

### 核心答案

**原理**：用一个长度为 m 的位数组和 k 个独立哈希函数。将 key 的 k 个哈希值映射到位数组的 k 个位置，置为 1。查询时，k 个位置全为 1 则**可能存在**（误判），任一位为 0 则**一定不存在**（不会漏判）。

```
插入：h1(key) % m = 2, h2(key) % m = 5, h3(key) % m = 8  →  bit[2]=bit[5]=bit[8]=1
查询：bit[2]=1 && bit[5]=1 && bit[8]=1  →  可能存在（会误判）
```

**误判率公式**：

```
p = (1 - e^(-kn/m))^k

其中：
n = 已插入元素数量
m = 位数组长度
k = 哈希函数数量
```

**最优哈希函数数量**：`k = (m/n) * ln2`

**Redis 实现**（RedisBloom 模块 / Redis 6 `BF.ADD` / `BF.EXISTS`）：

```
BF.ADD bloom myitem
BF.EXISTS bloom myitem  →  0（不存在）或 1（可能存在）
BF.MADD bloom item1 item2 item3
```

**亿级数据去重使用方式**：

1. **数据先落库**：布隆过滤器放在数据库前层，先过滤掉明显不存在的请求
2. **分片**：多个布隆过滤器分片存储，降低单表误判影响
3. **不支持删除**：标准布隆过滤器无法删除（计数布隆过滤器可以但占用更多空间）
4. **结合 Redis**：Redis 支持分布式布隆过滤器，多实例水平扩展

### 追问方向

- 布隆过滤器如何删除数据？（不支持，用"计数布隆过滤器"或"分层布隆过滤器"）
- 布隆过滤器和哈希表相比空间节省多少？（约 10 倍，取决于误判率和空间设置）
- 实际工程中误判率一般设置多少？（通常 1%~5%）

### 避坑提示

- 面试中必须强调：**布隆过滤器说"存在"不一定存在（误判），说"不存在"一定不存在（不会漏判）**——这是核心特性。
- 计算最优 k 时用 `(m/n) * ln2`，不是说出来的，是用数学推导（对 p 求导取极值）。

---

## 第 20 题 · 一致性哈希

### 题目

**什么是一致性哈希？它解决了什么问题？Hash 环、虚拟节点分别是什么？如何解决 Hash 倾斜问题？**

### 核心答案

**问题背景**：普通哈希 `nodeIndex = hash(key) % N`，当节点数 N 变化时，**几乎所有 key 都需要重新映射**（缓存雪崩）。

**一致性哈希**：将哈希值空间组织成一个环（0 ~ 2³²-1），每个节点映射到环上的一个位置。key 顺时针找到第一个节点。

```
环：0 ──────────────────────────────── 2³²-1
        nodeA              nodeB
           ↑                 ↑
        hash(nodeA)     hash(nodeB)
        key顺时针找到第一个节点
```

**虚拟节点**：每个物理节点对应多个"虚拟节点"（`nodeA#1`, `nodeA#2`, ...），均与在环上不同位置。解决**数据倾斜**问题。

**Hash 倾斜**：当节点较少时，key 很容易集中在少数节点上（环上间隔不均匀）。

**解决方案**：引入虚拟节点，使物理节点的映射更均匀，通常每个物理节点映射 **150~200 个虚拟节点**。

**节点增删**：只影响相邻节点的数据，其他节点不受影响。

**应用场景**：Redis Cluster、Memcached、DynamoDB、Cassandra 的数据分片。

### 追问方向

- 一致性哈希如何保证负载均衡？（虚拟节点数量足够多时，环上间隔近似均匀）
- 如果三个节点环上分布非常不均匀（比如都在 30°~120° 区间），怎么处理？（增加虚拟节点，或用"槽(slot)"机制）
- Cassandra 的一致性哈希和 Dynamo 系的有什么区别？（都有一致性哈希，但 Dynamo 有" hinted handoff"和"read repair"等容错机制）

### 避坑提示

- 一致性哈希的**核心价值**是减少节点变更时的数据迁移量，不是"保证均衡"——实际均衡靠虚拟节点。
- 很多人忽视"顺时针查找"这个细节，面试时能画出环并说明查找方向会加分。

---

## 第 21 题 · LRU Cache

### 题目

**如何实现一个 LRU Cache？请用 LinkedHashMap 实现。Redis 的 LRU 是如何实现的？LFU 和 LRU 有什么区别？**

### 核心答案

**LRU（Least Recently Used）**：最近最少使用，淘汰最久未访问的数据。

**LinkedHashMap 实现**（Java）：

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);  // accessOrder=true（按访问顺序）
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > capacity;  // 超过容量时删除最老的
    }
}
```

核心：`accessOrder = true` 使得每次 `get()/put()` 将访问的节点移到链表尾部。

**手写双链表 + HashMap 实现**（更常见于面试）：

```java
class LRUCache {
    private Map<Integer, Node> map = new HashMap<>();
    private Node head = new Node(0, 0), tail = new Node(0, 0);
    private int capacity;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail; tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        moveToTail(node);  // 访问后移到尾部
        return node.val;
    }

    public void put(int key, int val) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.val = val;
            moveToTail(node);
        } else {
            Node node = new Node(key, val);
            map.put(key, node);
            addToTail(node);
            if (map.size() > capacity) {
                Node removed = removeHead();
                map.remove(removed.key);
            }
        }
    }
    // 双链表操作：addToTail, moveToTail, removeHead
}
```

**Redis LRU**：

- Redis 用 `maxmemory-policy allkeys-lru` 或 `volatile-lru` 开启 LRU
- Redis 并不精确记录每个 key 的访问时间（太占内存），而是**近似 LRU**：随机采样 5（或可配置 `maxmemory-samples`）个 key，选择最久未使用的淘汰
- Redis 3.0+ 引入了**LFU**（Least Frequently Used）

**LFU vs LRU**：

| | LRU | LFU |
|--|-----|-----|
| 淘汰策略 | 最近最少使用（时间维度） | 最不经常使用（频率维度） |
| 优点 | 实现简单，对热点数据友好 | 区分高频和低频数据 |
| 缺点 | 一次性的热点数据会被误删 | 不关注访问时间，低频数据可能占据缓存 |

### 追问方向

- LinkedHashMap 的 `removeEldestEntry` 是在什么时候被调用的？（`put()` 时检查）
- Redis LRU 的"近似"是怎么近似的？（随机采样 + 比较 idle time）
- 如果缓存满了之后立即 get 一个刚淘汰的 key 会发生什么？（缓存穿透，DB 压力）

### 避坑提示

- LinkedHashMap 实现是 Java 专属，但双链表 + HashMap 是跨语言的通用实现，两者都要会。
- Redis LRU 的"近似"是一个重要工程权衡：精确 LRU 需要维护访问时间排序（O(n log n) 或 O(n) 空间），近似只需随机采样（O(1)），Redis 选择近似是合理的工程 trade-off。

---

## 第 22 题 · 并查集

### 题目

**什么是并查集（Union-Find）？它如何解决连通性问题？请描述路径压缩和按秩合并的原理。**

### 核心答案

**并查集**：支持**合并（Union）**和**查找（Find）**两种操作的数据结构，用于判断元素是否在同一集合。

**核心思想**：每个元素有一个父指针，根节点的父指针指向自己。查找时返回根节点。

```java
class UnionFind {
    int[] parent;
    int[] rank;  // 树高（按秩合并）

    UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x)
            parent[x] = find(parent[x]);  // 路径压缩：直接指向根
        return parent[x];
    }

    void union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return;
        // 按秩合并：小秩的根挂到大秩的根
        if (rank[px] < rank[py]) px = px ^ py ^ (px = py);
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
    }
}
```

**路径压缩**（Find 优化）：在 find 过程中，将路径上所有节点的父指针直接指向根节点，使树变扁平。

**按秩合并**（Union 优化）：合并时，将**较短的树**接到**较长的树**下面，避免树退化成链表。

**时间复杂度**：接近 O(1)（Ackermann 函数的反函数 α(n)，实际视为常数）

**应用场景**：

- 岛屿数量（LeetCode 200）：每次发现陆地，union 周围四个方向的陆地
- 检测图中的环（最小生成树 Kruskal）
- 社交网络中的好友关系判断

### 追问方向

- 路径压缩和按秩合并哪个更重要？（两者结合最优，但路径压缩对单链情况的改善更显著）
- 并查集能解决什么问题而 BFS/DFS 不能？（动态维护**等价关系**的连通分量，支持高效的合并操作）
- 如果要支持删除操作怎么办？（并查集本身不支持，通常用"时间戳"或重建）

### 避坑提示

- `find(x)` 中路径压缩的递归写法 `parent[x] = find(parent[x])` 是标准写法，面试时直接写。
- 按秩合并的"秩"可以是树高，也可以是集合大小，原理相同。

---

## 第 23 题 · 线段树

### 题目

**什么是线段树？它如何支持区间查询和区间更新？区间合并是什么场景？**

### 核心答案

**线段树**：一种二叉树结构，将数组区间 `[l, r]` 递归划分为左右子区间，每个节点存储一个区间的统计信息（和、最大值、最小值等）。适用于**区间查询 + 区间更新**场景。

**结构**：数组实现时，根节点存 `[0, n-1]`，左孩子存 `[l, mid]`，右孩子存 `[mid+1, r]`。

```
数组: [1, 3, 5, 7, 9, 11]
线段树:
              [0,5] (sum=36)
           /          \
       [0,2](9)      [3,5](27)
      /    \        /     \
  [0,1]  [2,2]  [3,4]   [5,5]
   (4)    (5)    (16)    (11)
```

**区间查询**（O(log n)）：

```java
int query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return identity;  // 无交集
    if (ql <= l && r <= qr) return tree[node]; // 完全覆盖
    int mid = (l + r) / 2;
    return merge(query(leftChild), query(rightChild));
}
```

**区间更新**（O(log n)）：找到完全覆盖的区间，更新该节点，然后沿路径向上 `merge` 更新父节点。

**区间合并**：当查询的多个子区间需要**合并**结果时（如求最大子段和），`merge` 操作就是合并两个子区间的统计信息。典型问题：区间染色、最大子序列和（LeetCode 53）。

**复杂度**：

| 操作 | 朴素数组 | 线段树 |
|------|---------|--------|
| 单点更新 | O(1) | O(log n) |
| 区间查询 | O(n) | O(log n) |
| 区间更新 | O(n) | O(log n) |
| 空间 | O(n) | O(4n) |

### 追问方向

- 线段树和树状数组（Fenwick Tree）有什么区别？（树状数组只支持前缀和，线段树支持任意区间；树状数组空间 O(n)，实现更简单）
- 如果查询的是"区间众数"（出现次数最多的数），线段树能合并吗？（不能直接合并，需要更复杂的数据结构，如主席树）
- 线段树如何处理"区间加"和"区间乘"同时存在？（懒传播 + 双标记下推）

### 避坑提示

- 线段树是"树"类型题目中的高级考点，关键是**递归分治**和**merge 操作**的思想。理解后可以用同一套框架解决多类问题。
- 如果面试中线段树写起来太复杂，可以先描述思路，代码实现中重点写 `query` 和 `update` 的递归结构。

---

## 第 24 题 · 滑动窗口

### 题目

**滑动窗口算法适用于什么场景？请用单调队列实现滑动窗口最大值（LeetCode 239），并扩展到"字符串最小覆盖子串"问题。**

### 核心答案

**适用场景**：子串/子数组问题，**连续区间**内的最优解查找。

**滑动窗口最大值**（LeetCode 239）：

```java
int[] maxSlidingWindow(int[] nums, int k) {
    if (nums == null || k <= 0) return new int[0];
    int n = nums.length;
    int[] res = new int[n - k + 1];
    LinkedList<Integer> deque = new LinkedList<>(); // 存索引

    for (int i = 0; i < n; i++) {
        // 移除窗口左边界外的元素
        while (!deque.isEmpty() && deque.peekFirst() <= i - k)
            deque.pollFirst();

        // 移除小于当前元素的索引（它们不可能是最大值）
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i])
            deque.pollLast();

        deque.offerLast(i);

        // 窗口形成后记录最大值
        if (i >= k - 1)
            res[i - k + 1] = nums[deque.peekFirst()];
    }
    return res;
}
```

**核心思想**：维护一个**单调递减的双端队列**（队首是当前窗口最大值的索引）。入队时移除比当前元素小的（无用）；出队时移除窗口外的。

**字符串最小覆盖子串**（LeetCode 76）：

```java
String minWindow(String s, String t) {
    int[] need = new int[128];  // t 中字符计数
    for (char c : t.toCharArray()) need[c]++;

    int left = 0, right = 0, count = t.length(), minLen = Integer.MAX_VALUE;
    int minL = 0;

    while (right < s.length()) {
        char rc = s.charAt(right);
        if (need[rc] > 0) count--;
        need[rc]--;  // 计入窗口

        while (count == 0) {  // 窗口已覆盖 t
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minL = left;
            }
            char lc = s.charAt(left);
            if (need[lc] >= 0) count++;  // 即将移出需要的字符
            need[lc]++;  // 从窗口移除
            left++;
        }
        right++;
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(minL, minL + minLen);
}
```

**复杂度**：两个问题都是 **O(n)**，每个字符最多入队出队各一次。

### 追问方向

- 滑动窗口的"左闭右开"和"左闭右闭"区间定义有什么区别？（通常用左闭右开 `[left, right)` 更容易处理边界）
- 如果窗口大小是固定的（如求长度为 K 的最大平均值），如何改？（固定窗口，用同样框架但出队条件改为 `i - deque.peekFirst() >= k`）
- 单调队列中维护的是值还是索引？（索引——方便判断是否在窗口内，且可以取到对应的值）

### 避坑提示

- 单调队列的"移除窗口外元素"和"移除小于当前元素的元素"顺序很重要：先判断窗口外，再移除无用小元素。顺序错了逻辑就不对了。
- 最小覆盖子串的 `need[rc]--` 在 `if (need[rc] > 0) count--` 之后执行——因为 need 可能已经负值（窗口内该字符已够用），两者逻辑要分清。

---

## 第 25 题 · TopK 问题

### 题目

**TopK 问题有哪些主流解法？请对比堆排序法和快速排序 partition 法，并说明分布式环境下如何求 TopK。**

### 核心答案

**TopK 问题**：从 n 个数中找出最大的（或最小的）K 个数。

**方法一：堆排序法**（第 7 题已详述）

- 建大小为 K 的**小根堆**（找最大的 K 个）
- 遍历剩余元素，若 `> heapTop`，替换堆顶并 `heapifyDown`
- 时间：O(n log K)，空间：O(K)
- 适合：**n 远大于 K**，流式数据

**方法二：快速排序 partition 法**

```java
int findTopK(int[] arr, int k) {  // 找第 k 大的数（0-indexed）
    int low = 0, high = arr.length - 1;
    while (true) {
        int pivot = partition(arr, low, high);
        if (pivot == k) return arr[pivot];
        else if (pivot < k) low = pivot + 1;
        else high = pivot - 1;
    }
}
```

- 利用快排的 partition，将数组分为 `>= pivot` 和 `<= pivot` 两部分
- 如果 pivot 恰好在第 K 位置，左边就是 TopK
- 时间：平均 O(n)，最坏 O(n²)（有序数组 + 固定 pivot）
- 适合：**需要一次性求固定 K**，n 不太大的场景

**两种方法对比**：

| | 堆排序法 | 快排 partition |
|--|---------|----------------|
| 时间复杂度 | O(n log K) | 平均 O(n) |
| 最坏情况 | O(n log K)（不会退化） | O(n²)（有序数组） |
| 是否需要完全排序 | 否 | 否（部分排序） |
| 适用 K 大小 | K 较小时优 | K 接近 n/2 时也优 |
| 数据要求 | 只需知道 TopK | 需要能随机访问 |

**分布式 TopK**（MapReduce 思想）：

```
Stage 1（Map）：每个 Mapper 对本地数据求本地 TopK（堆，K 较小）
Stage 1（Reduce）：合并所有 Mapper 的 TopK，得到全局候选集（size = Mappers × K）
Stage 2（Map）：对全局候选集再次求 TopK
Stage 2（Reduce）：得到最终 TopK
```

关键点：MapReduce 的 shuffle 阶段天然将相同 key 的数据汇聚，只需要对每个 Mapper 输出的局部 TopK 再做一次全局汇总。

**实际工程**：

- 数据量大用**堆排序法**（内存友好，可流式）
- 单机求固定 K 且 K 不太接近 n 用**快排 partition**（更快）
- 分布式用 **MapReduce 两阶段**（先局部后全局）

### 追问方向

- 如果 K = 1（求最大值/最小值），有什么 O(n) 的方法？（扫描一遍，用两个变量记录最大最小）
- 如果内存极其有限（无法存 K 个元素）怎么办？（可用"桶排序"思想，按值域分桶）
- 快排 partition 求 TopK 的最坏情况如何优化？（随机化 pivot，或用"BFPRT"算法保证最坏 O(n)）

### 避坑提示

- TopK 用**小根堆**而不是大根堆，这是最容易搞混的点：小根堆的堆顶是 K 个中最小的，只要新元素比堆顶大就替换——最终 K 个就是 TopK。
- 分布式 TopK 的两阶段 MapReduce 是经典框架，回答时不要说"把所有数据发到一个节点排序"——那不是分布式思维。

---

## 附录 · 面试 Checklist

### 每题必过三问

1. **复杂度**：时间 + 空间，最好/平均/最坏三种情况
2. **边界**：空输入、单元素、全部相同、已排序、倒序等边界 case
3. **扩展**：JDK/Redis/实际应用中的对应实现

### 手写题高频陷阱

- **二分查找**：`lo + (hi - lo) / 2` 防止溢出；左边界 `while (lo < hi)` 退出条件
- **快排 partition**：边界处理（i 和 j 的更新顺序）；递归出口 `low >= high`
- **归并排序**：临时数组大小 + 复制回原数组
- **回溯**：撤销选择（`removeLast`）不能忘
- **堆**：从 0 还是 1 开始建堆决定父子关系公式不同

---

*本篇共 25 题，覆盖大厂算法面试 80%+ 高频考点。建议配合《剑指 Offer》、LeetCode Hot 100 进行专项练习。*
