# Java SE 核心面试题 - 第一部分

## 集合框架（Collection Framework）

### ArrayList（动态数组）

**Q1. ArrayList的内部数据结构是什么？它如何处理动态扩容？**

**难度：** 中级

**答案：** ArrayList内部使用动态数组，会自动增长和收缩。创建ArrayList时，它使用默认容量10进行初始化。当元素数量超过当前容量时，ArrayList会创建一个更大的新数组（通常是旧容量的1.5倍），并使用`Arrays.copyOf()`将所有现有元素复制到新数组中。这种扩容操作的时间复杂度为O(n)，因为需要复制所有元素。扩容的开销被分摊到所有添加操作上，使每个添加操作的均摊时间复杂度为O(1)。ArrayList通过索引进行随机访问的时间复杂度为O(1)，因为它使用连续的数组结构。

**追问方向：** ArrayList与LinkedList在不同操作上的性能对比如何？
**WMS Reference:** 在WMS `InventoryService`中用于存储活动订单的行项目，在生成拣货单期间频繁进行随机访问。

---

**Q2. 解释ArrayList和LinkedList在时间复杂度方面的差异。**

**难度：** 中级

**答案：** ArrayList由于其连续的内存布局，提供O(1)的随机访问；而LinkedList需要O(n)的遍历才能到达任意索引。对于在开头或中间的插入和删除操作，如果已有节点引用，LinkedList提供O(1)，而ArrayList需要O(n)因为可能需要移动元素。在列表末尾，ArrayList是O(1)（均摊），而LinkedList使用尾引用添加时也是O(1)。LinkedList由于存储前驱和后继指针，每个元素消耗更多内存。选择取决于使用模式：频繁随机访问时ArrayList更好，频繁在已知位置插入/删除时LinkedList更好。

**追问方向：** 在WMS场景中何时会选择LinkedList而不是ArrayList？
**WMS Reference:** LinkedList可能用于WMS `RouteOptimizationService`中管理动态的路径点序列，其中频繁进行插入和删除操作。

---

**Q3. ArrayList如何确保线程安全，有什么替代的线程安全列表？**

**难度：** 高级

**答案：** ArrayList本身不是线程安全的；它允许并发修改，这可能导致`ConcurrentModificationException`或数据损坏。对于线程安全的替代方案，Java提供了`Collections.synchronizedList()`，它用同步方法包装ArrayList，但在迭代期间由于锁定整个集合而性能较差。`CopyOnWriteArrayList`设计用于读多写少的场景，每次修改都创建一个新的副本；它允许并发读取而无需锁定，但写操作成本较高。`Vector`是一个遗留的线程安全变体，但由于每个方法都同步而性能较差，通常不推荐使用。最佳选择取决于工作负载是读密集型还是写密集型。

**追问方向：** 为什么`Collections.synchronizedList()`在迭代期间仍然会抛出ConcurrentModificationException？
**WMS Reference:** `CopyOnWriteArrayList`用于WMS `AlertNotificationService`中管理监听器列表，其中通知读取频繁但监听器变化很少。

---

**Q4. ArrayList迭代器的fail-fast（快速失败）行为是什么？何时会触发？**

**难度：** 高级

**答案：** ArrayList使用fail-fast迭代器，如果检测到在迭代器创建后集合被结构性修改，它会立即抛出`ConcurrentModificationException`。这是通过维护一个`modCount`变量来实现的，该变量跟踪结构性修改；每个迭代器在创建时会对这个计数进行快照，并在每次迭代调用时进行比较。结构性修改包括添加、删除或更改列表大小。这种机制不能保证检测到所有并发修改，但能识别大多数意外修改。fail-fast行为是有意设计的，旨在防止数据损坏和不可预测的行为。一旦迭代器检测到修改，无论哪个线程执行了修改，它都会在下一次操作时抛出异常。

**追问方向：** CopyOnWriteArrayList与常规ArrayList在fail-fast行为方面有何不同？
**WMS Reference:** 理解fail-fast在WMS `OrderProcessingService`中至关重要，因为多个线程可能同时迭代同一个订单项目列表。

---

**Q5. ArrayList的初始容量是多少？设置初始容量如何影响性能？**

**难度：** 初级

**答案：** ArrayList的默认初始容量为10个元素，在添加第一个元素时分配。如果已知或可以估计初始大小，在构造函数中指定初始容量可以避免中间扩容操作及其相关的内存和CPU开销。每次扩容操作都涉及分配一个新的更大的数组并复制所有现有元素，这是O(n)的并会产生临时垃圾。适当设置容量可以减少内存碎片和GC压力，避免多次数组分配。`ensureCapacity()`方法也可用于在批量添加之前预分配容量。

**追问方向：** 如果将初始容量设置为Integer.MAX_VALUE会发生什么？
**WMS Reference:** 在WMS `BatchProcessingService`中，加载数千个订单进行批处理时，设置适当的初始容量可以提高性能。

---

### HashMap（哈希映射）

**Q6. 解释HashMap的内部结构以及它如何处理哈希冲突。**

**难度：** 高级

**答案：** HashMap内部使用桶数组（Node/Entry对象），每个桶可以容纳单个键值对，或在发生冲突时容纳一个键值对链表。添加键值对时，HashMap计算键的hashCode，应用一个补充哈希函数来减少冲突，然后使用结果来确定桶索引。当多个键映射到同一个桶（冲突）时，它们在该桶内形成一个链表（Java 8+中对于长链会使用平衡树）。`put()`方法检查桶列表中每个节点是否存在现有键匹配，要么更新值要么添加新节点。负载因子（默认0.75）决定何时调整数组容量，通常是双倍扩容。

**追问方向：** HashMap中的补充哈希函数是什么？为什么需要它？
**WMS Reference:** HashMap在WMS `LocationService`中广泛用于将位置条码映射到Location对象，以实现O(1)的查找性能。

---

**Q7. HashMap操作的时间复杂度是多少？哪些因素会影响它？**

**难度：** 中级

**答案：** 在理想条件下（完美哈希且无冲突），HashMap为`get()`和`put()`操作都提供O(1)的时间复杂度。然而，在最坏情况下，当所有键都哈希到同一个桶形成长链表时，操作会退化为O(n)。Java 8+通过在桶超过8个节点时将长链表转换为平衡树来缓解这个问题，将最坏情况降低到O(log n)。0.75的负载因子在空间和时间之间提供了良好的平衡；较低的负载因子减少冲突但使用更多内存。HashMap的性能也取决于键的`hashCode()`实现质量。

**追问方向：** Java 8+中的树化（treeification）如何改善HashMap性能？
**WMS Reference:** 在WMS `SkuService`中，HashMap对SKU数据的查找必须是O(1)才能满足订单拣货期间的实时性能要求。

---

**Q8. HashMap和Hashtable有什么区别？**

**难度：** 初级

**答案：** HashMap和Hashtable都实现了Map接口，但有根本区别。Hashtable在每个方法上都同步，使其线程安全但在单线程或并发场景中性能较差。HashMap不同步，提供更好的性能，但需要外部同步才能用于线程安全场景。Hashtable不允许空键或空值，而HashMap允许一个空键和无限个空值。Hashtable是从Java 1.0以来的遗留类，而HashMap是一种利用改进哈希算法的现代实现。此外，HashMap使用Iterator（fail-fast）而Hashtable使用Enumeration（不是fail-fast）。

**追问方向：** 在新的WMS实现中你会选择哪个？为什么？
**WMS Reference:** 现代WMS实现更喜欢HashMap与`ConcurrentHashMap`一起用于并发访问，而不是较旧的同步Hashtable。

---

**Q9. 当HashMap中两个不同的键有相同的hashCode时会发生什么？**

**难度：** 中级

**答案：** 当两个不同的键产生相同的hashCode时，它们被放在同一个桶中，造成哈希冲突。HashMap通过将该桶中冲突的条目存储为链表节点（Java 8+中为树）来处理。在`get()`操作期间，HashMap使用哈希找到正确的桶，然后遍历链表，使用`hashCode()`和`equals()`方法比较键以找到精确匹配。许多键映射到同一个桶的糟糕hashCode分布会导致性能下降。良好的`hashCode()`实现应该在哈希空间中均匀分布键以最小化冲突。

**追问方向：** 如何为WMS Location类实现一个好的hashCode()方法？
**WMS Reference:** 在具有warehouseId、zone、aisle、rack和shelf字段的WMS `Location`类中，糟糕的hashCode()可能导致性能问题。

---

**Q10. 在迭代HashMap时可以修改它吗？可能发生什么异常？**

**难度：** 中级

**答案：** 在使用迭代器迭代HashMap时对其进行结构性修改会抛出`ConcurrentModificationException`（fail-fast行为）。结构性修改包括添加或删除条目；更新现有条目的值通常是可以的。这种保护即使修改是在不同线程中执行的也会生效。要在迭代期间安全修改，请使用迭代器自己的`remove()`方法，它会删除`next()`返回的最后一个元素。或者，迭代地图键集或条目集的副本。对于并发修改，考虑使用`ConcurrentHashMap`，它提供弱一致性迭代器，不会抛出此异常。

**追问方向：** 如何实现安全的"移除匹配条件的条目"操作？
**WMS Reference:** 在WMS `InventoryReservationService`中，安全地移除过期的预订，同时其他线程可能正在读取同一个映射。

---

**Q11. HashMap中负载因子的作用是什么？当超过它时会发生什么？**

**难度：** 中级

**答案：** 负载因子是一个阈值，决定HashMap何时应该增加容量以保持性能。默认负载因子0.75意味着当75%的桶被填充时，HashMap创建一个双倍容量的新数组，并将所有现有条目重新哈希到新数组中。较高的负载因子更节省内存，但会增加冲突概率从而降低性能。较低的负载因子减少冲突但浪费内存。重新哈希过程是O(n)的，在插入操作超过阈值时发生。在批量插入时调整负载因子很重要，可以最小化重新哈希开销。

**追问方向：** 如何为一个已知数量的条目优化HashMap？
**WMS Reference:** 在WMS `WarehouseInitializationService`中，根据预期的库存计数预先调整HashMap大小可以提高启动性能。

---

**Q12. 解释HashMap、LinkedHashMap和TreeMap之间的区别。**

**难度：** 高级

**答案：** HashMap为get/put操作提供O(1)的平均性能，但不维护条目的任何顺序。LinkedHashMap扩展了HashMap并维护插入顺序（或者通过accessOrder=true维护访问顺序），在保持O(1)基本操作性能的同时允许可预测的迭代顺序，外加O(1)用于维护链表。TreeMap实现了SortedMap并根据自然顺序或提供的Comparator维护键，提供O(log n)的get/put操作性能，并支持范围查询和有序遍历。TreeMap要求键是可比较的，或者必须提供Comparator。当需要O(1)操作和插入顺序时，LinkedHashMap是理想选择；而当需要排序键或导航方法如`subMap()`、`headMap()`和`tailMap()`时需要TreeMap。

**追问方向：** 在WMS场景中何时会使用accessOrder=true的LinkedHashMap？
**WMS Reference:** accessOrder=true的LinkedHashMap可以使用LRU淘汰策略缓存最近访问的WMS `ItemMaster`数据。

---

### ConcurrentHashMap（并发哈希映射）

**Q13. ConcurrentHashMap如何在不同步所有操作的情况下实现线程安全？**

**难度：** 高级

**答案：** ConcurrentHashMap使用一种称为锁分段（lock striping）的技术，将内部数组分割成可以独立锁定的段。默认情况下，有16个段，每个段作为一个独立的哈希表。当修改一个条目时，只锁定包含该条目的段，允许其他段被其他线程并发修改。读取不需要锁定，可以使用volatile读取并发进行，因为值通过volatile语义可见。这种设计为读写都提供了高并发性，不像Hashtable或synchronized Map锁定整个集合。段数可以在构造时配置以调整并发访问。

**追问方向：** 锁分段与在整个集合上使用单一锁有什么区别？
**WMS Reference:** ConcurrentHashMap用于WMS `SharedInventoryState`，其中多个拣货线程需要并发访问库存计数。

---

**Q14. ConcurrentHashMap和Collections.synchronizedMap()有什么区别？**

**难度：** 高级

**答案：** Collections.synchronizedMap()用同步方法包装一个常规Map，意味着一次只有一个线程可以访问Map进行任何操作，包括迭代和get操作。ConcurrentHashMap通过使用锁分段和非阻塞读取语义允许多个线程并发读写。ConcurrentHashMap的迭代器是弱一致性的，允许在迭代期间并发修改而不会抛出ConcurrentModificationException，而synchronizedMap的迭代器是fail-fast的。ConcurrentHashMap提供原子复合操作如`putIfAbsent()`、`replace()`和`remove()`，这些在synchronizedMap中不可用。对于读密集型工作负载，ConcurrentHashMap明显优于synchronizedMap，因为读取不会阻塞。

**追问方向：** ConcurrentHashMap提供哪些在WMS场景中特别有用的原子操作？
**WMS Reference:** ConcurrentHashMap的`compute()`和`merge()`方法用于WMS `InventoryTrackingService`中的原子库存计数更新。

---

**Q15. 当在ConcurrentHashMap的get操作中找不到键时它如何处理？**

**难度：** 初级

**答案：** 当在ConcurrentHashMap中找不到键时，`get()`方法只返回null，表示不存在该键值映射。与某些实现不同，ConcurrentHashMap不会为找不到的键抛出异常；对于找不到的键和值为null的键都返回null（因为ConcurrentHashMap不允许空键或值）。如果需要区分"键不存在"和"键有null值"，调用者应该使用`containsKey()`方法。这种行为与某些其他Map实现不同，在那些实现中null可能是找不到的键的有效返回值。

**追问方向：** 为什么ConcurrentHashMap不允许空键或值？
**WMS Reference:** 在WMS `OrderLineItemService`中，在`get()`之前使用`containsKey()`检查可以区分"SKU不在订单中"和"数量为零"。

---

**Q16. 解释ConcurrentHashMap迭代器的弱一致性。**

**难度：** 高级

**答案：** ConcurrentHashMap迭代器是弱一致性的，意味着它们可能会或可能不会反映迭代器创建后所做的修改，但它们不会抛出ConcurrentModificationException。迭代器可以遍历迭代器创建时存在的元素，也可能反映创建后所做的修改，但永远不会因并发修改而抛出异常。这与HashMap的fail-fast迭代器不同，后者会在发生任何结构性修改时立即抛出。弱一致性保证允许更高的并发性，因为在迭代期间集合不需要被锁定。迭代器的remove()方法移除的元素会从底层Map中真正移除。

**追问方向：** 在WMS订单处理场景中弱一致性的实际影响是什么？
**WMS Reference:** 在WMS `RealTimeInventoryService`中，弱一致性允许并发库存更新而不会中断活动监控线程。

---

**Q17. ConcurrentHashMap中`compute()`、`merge()`和`computeIfAbsent()`方法的用途是什么？**

**难度：** 高级

**答案：** 这些方法提供原子复合操作，将检索和更新组合在单个原子操作中，无需外部同步。`compute(key, remappingFunction)`对当前值应用重新映射函数并返回新值，原子地更新条目。`merge(key, defaultValue, remappingFunction)`使用提供的函数将现有值与新值组合，如果不存在条目则使用默认值。`computeIfAbsent(key, mappingFunction)`仅在键不存在时计算并插入，避免对已存在值的冗余计算。这些方法防止了一个线程在更新时另一个线程正在读取值的竞态条件，消除了"检查然后行动"这种容易受到并发修改影响的模式的需要。

**追问方向：** 如何使用这些方法实现原子的"增加库存计数"操作？
**WMS Reference:** 在WMS `InventoryCountService`中，使用`compute()`进行原子计数器更新确保当多个拣货员同时修改库存时计数准确。

---

### LinkedHashSet（链接哈希集合）

**Q18. 什么是LinkedHashSet？它与HashSet有什么区别？**

**难度：** 中级

**答案：** LinkedHashSet是一个Set实现，扩展了HashSet但使用双向链表维护插入顺序。虽然HashSet为基本操作提供O(1)时间且没有顺序保证，但LinkedHashSet保留元素被插入的顺序，允许可预测的迭代。顺序是通过LinkedHashMap内部结构维护的，LinkedHashSet通过组合使用它。LinkedHashSet允许空元素且没有同步。由于维护链表，内存开销略高于HashSet，遍历的迭代性能为O(n)。当需要唯一性和插入顺序保留时，它特别有用。

**追问方向：** 如果在WMS接收过程中使用LinkedHashSet而不是HashSet会发生什么？
**WMS Reference:** LinkedHashSet可以维护WMS `ReceivingProcess`中项目被扫描的顺序，用于审计跟踪。

---

**Q19. LinkedHashSet如何在维护顺序的同时实现Set接口？**

**难度：** 高级

**答案：** LinkedHashSet内部使用LinkedHashMap将元素存储为键，使用一个虚拟值对象。LinkedHashMap重写了`newNode()`和相关方法来维护一个双向链表，按插入顺序贯穿所有条目。当通过`add()`添加元素时，它作为键存储在LinkedHashMap中，附带一个共享的虚拟值，链表被更新以将新条目包含在尾部。LinkedHashSet返回的迭代器使用这个链表进行有序遍历，通过`LinkedHashMap.LinkedHashIterator`。这种设计利用了HashSet通过哈希的高效存储和LinkedHashMap的顺序维护，而无需LinkedHashSet直接实现所有复杂性。

**追问方向：** LinkedHashSet可以维护访问顺序而不是插入顺序吗？
**WMS Reference:** 访问顺序模式可以为频繁访问的WMS `UserSession`对象实现LRU缓存。

---

**Q20. 使用LinkedHashSet而不是HashSet的性能影响是什么？**

**难度：** 中级

**答案：** LinkedHashSet由于维护链表的额外开销，对于添加、删除和包含操作比HashSet性能稍差。HashSet操作平均为O(1)，而LinkedHashSet在修改期间为更新前驱/后继指针添加了一个小常数因子。内存消耗也更高，因为每个条目存储两个额外的指针（before和after）用于链表。迭代LinkedHashSet是O(n)并保持插入顺序，当顺序重要时这可能是一个优势。如果不需要顺序，由于较低的内存和稍好的性能，HashSet是更好的选择。对于有频繁修改的大型集合，LinkedHashSet的常数因子开销变得更加明显。

**追问方向：** 何时会在WMS应用程序中支付LinkedHashSet的开销？
**WMS Reference:** LinkedHashSet的顺序在WMS `PickSequenceGenerator`中很有价值，其中维护扫描仪输入顺序对合规性很重要。

---

### PriorityQueue（优先级队列）

**Q21. 什么是PriorityQueue？它与常规队列有什么区别？**

**难度：** 中级

**答案：** PriorityQueue是一个队列，其中元素根据其自然顺序或提供的Comparator排序，而不是FIFO顺序。队列头部是根据自然顺序具有最小值的元素（默认是最小堆），而不是最老的元素。元素以任意顺序插入但以排序顺序检索。与强制插入顺序的常规队列不同，PriorityQueue使用二叉堆数据结构，提供O(log n)的插入和删除，以及O(1)的查看。如果使用自然顺序，PriorityQueue不允许空元素。它是无界的，但可以用初始容量构造，并根据需要自动增长。它不是线程安全的；要并发访问，请使用PriorityBlockingQueue。

**追问方向：** 如何使用PriorityQueue实现最大堆？
**WMS Reference:** PriorityQueue在WMS `TaskSchedulerService`中是理想的，用于根据订单紧急程度和 proximity（接近度）对拣货任务进行优先级排序。

---

**Q22. 解释PriorityQueue内部使用的堆数据结构。**

**难度：** 高级

**答案：** PriorityQueue内部使用二叉最小堆实现，这是一个完整的二叉树，其中每个节点的值小于或等于其子节点的值。堆存储为数组，其中对于索引i处的节点，其左子节点在2i+1，右子节点在2i+2，父节点在(i-1)/2。当添加元素时，它被放在末尾并通过与父节点比较进行"上浮"（sift up），如果较小则交换，保持堆属性为O(log n)。当移除最小元素时，根节点被最后一个元素替换，通过与子节点比较进行"下沉"（sift down）并与较小的交换，也是O(log n)。堆结构确保在根节点处O(1)访问最小元素。每次操作后堆属性自动维护。

**追问方向：** 在大小为n的PriorityQueue中找到第k个最小元素的时间复杂度是多少？
**WMS Reference:** 在WMS `BatchOrderProcessor`中，堆操作通过始终首先挑选最紧急的订单来高效处理数千个订单。

---

**Q23. 当尝试将不可比较的元素添加到PriorityQueue时会发生什么？**

**难度：** 中级

**答案：** 当元素无法根据PriorityQueue的顺序进行比较时，会在运行时抛出`ClassCastException`，当堆尝试对元素排序时。对于自然顺序，元素必须实现Comparable；对于自定义Comparator，Comparator必须能够比较元素。异常发生在第一次不兼容比较尝试时，通常是在插入或删除操作期间。队列可能包含元素的混合，其中一些比较在遇到不兼容对之前成功。这是一个运行时问题，如果在测试中未执行特定排序，可能不会暴露。

**追问方向：** 如何处理WMS任务具有不同优先级类型（紧急、正常、低）的情况？
**WMS Reference:** 在WMS `TaskPriorityService`中，使用处理不同任务类型的自定义Comparator确保正确的优先级排序。

---

### CopyOnWriteArrayList（写时复制数组列表）

**Q24. 什么是CopyOnWriteArrayList？何时应该使用它？**

**难度：** 高级

**答案：** CopyOnWriteArrayList是一个线程安全的List实现，其中所有可变操作都创建底层数组的新副本。它设计用于读远多于写的场景，例如维护监听器列表或很少修改的缓存数据。迭代器对迭代器创建时的数组快照进行操作，提供一致的遍历而无需同步或ConcurrentModificationException风险。写操作如add、set和remove创建新数组副本，复制旧数据，应用更改，然后原子地替换引用。代价是写操作昂贵（O(n)），并且可能不会反映早期创建的迭代器的后续修改。频繁写入时内存消耗可能很高。

**追问方向：** 当多个线程同时尝试修改CopyOnWriteArrayList时会发生什么？
**WMS Reference:** CopyOnWriteArrayList用于WMS `EventDispatcherService`中管理事件监听器列表，其中监听器添加/删除不频繁但事件触发频繁。

---

**Q25. 为什么CopyOnWriteArrayList的迭代器是弱一致性的？**

**难度：** 高级

**答案：** CopyOnWriteArrayList迭代器对数组快照而不是实时数组进行操作，这本质上防止了ConcurrentModificationException，因为修改会创建新数组而不是修改正在被迭代的数组。迭代器持有快照数组的引用，因此它不会看到迭代器创建后添加的元素，如果在快照拍摄后发生修改，对现有元素的修改也不会可见。这种弱一致性意味着迭代器可能会或可能不会反映最新状态，但它们保证不会抛出异常并遍历一致的快照。代价是读者可能使用过时的数据，但作为交换，他们获得无锁读取和一致的内部迭代状态。

**追问方向：** 弱一致性何时会在WMS场景中成为问题？
**WMS Reference:** 弱一致性在WMS `RealTimeDashboardService`中很重要，其中显示监听器绝对当前的状态至关重要。

---

**Q26. CopyOnWriteArrayList各种操作的性能特征是什么？**

**难度：** 高级

**答案：** CopyOnWriteArrayList提供O(1)的读取性能，因为读取直接访问数组而无需锁定。写操作如add、set和remove是O(n)，因为必须复制整个数组。内存开销很大：每次写入创建一个新的数组副本，而旧数组可能仍被迭代器引用，导致临时重复存储。迭代是对快照大小的O(n)，但不会阻止写入者或读取者。总成本包括新数组的分配开销。在高写入频率下，此实现性能很差并消耗过多内存；仅适用于写入很少且读取非常频繁的情况。批量操作特别昂贵，因为每个操作都会触发完整复制。

**追问方向：** 对于99%读取、1%写入的工作负载，CopyOnWriteArrayList与同步List相比如何？
**WMS Reference:** 对于WMS `ConfigurationService`，配置被频繁读取但很少更改，CopyOnWriteArrayList是理想的。

---

### Fail-Fast机制

**Q27. 解释Java集合中的fail-fast迭代器机制。**

**难度：** 高级

**答案：** Fail-fast迭代器通过维护一个修改计数（modCount）来检测并发修改，该计数在每次集合结构性更改时递增。当创建迭代器时，它会快照当前的modCount，并在返回每个元素之前检查它。如果modCount已更改，则抛出ConcurrentModificationException。这种机制不能保证检测到所有并发修改，因此称为"fail-fast"而不是"fail-safe"；它旨在防止由并发修改引起不可预测的行为。Fail-fast行为适用于大多数JDK集合类，包括ArrayList、HashSet、HashMap及其变体。该机制仅防止同一JVM中的意外并发修改；它不能帮助处理来自多个线程的真正并发访问。

**追问方向：** Fail-fast与fail-safe在并发编程上下文中的区别是什么？
**WMS Reference:** 理解fail-fast在WMS `ConcurrentOrderProcessor`中至关重要，其中多个线程可能访问相同的订单数据结构。

---

**Q28. 为什么ConcurrentHashMap在迭代期间不会抛出ConcurrentModificationException？**

**难度：** 高级

**答案：** ConcurrentHashMap不会抛出ConcurrentModificationException，因为它的迭代器被设计为弱一致性而不是fail-fast的。迭代器在迭代开始时对表状态进行快照操作，或者可能遍历当前状态同时容忍不影响当前被访问条目的并发修改。ConcurrentHashMap不维护被迭代器检查的modCount变量，避免了保持其一致性所需的同步开销。这个设计选择支持ConcurrentHashMap旨在的高并发用例。代价是迭代器可能反映迭代器创建后所做的修改。

**追问方向：** 弱一致性对使用ConcurrentHashMap迭代器的应用程序逻辑有什么影响？
**WMS Reference:** 在WMS `ConcurrentInventoryTracker`中，弱一致性允许在 其他线程进行预订时迭代库存。

---

**Q29. 在增强for循环迭代期间可以修改集合而不会ConcurrentModificationException吗？**

**难度：** 中级

**答案：** 不行，在常规集合上使用增强for循环（内部使用迭代器）在集合在迭代期间被结构性修改时会抛出ConcurrentModificationException。在迭代期间安全修改的唯一方法是：使用迭代器自己的`remove()`方法，它会删除`next()`调用返回的元素；迭代集合的副本；或使用为并发修改设计的线程安全集合如ConcurrentHashMap或CopyOnWriteArrayList。在迭代期间通过集合自己的方法（而不是迭代器的方法）删除元素总是会触发异常。这适用于所有fail-fast集合。

**追问方向：** 如何安全地过滤列表以移除匹配条件的元素？
**WMS Reference:** 在WMS `OrderCleanupService`中，使用iterator.remove()或分别收集要移除的元素来安全地移除取消的行项目。

---

## 泛型（Generics）

**Q30. 什么是Java泛型？为什么引入它们？**

**难度：** 初级

**答案：** Java泛型在Java 5中引入，通过允许类型（类和接口）被参数化，提供编译时类型安全。泛型之前，集合持有Object引用，需要手动转换并在运行时冒着ClassCastException风险；泛型使编译器能够强制执行类型正确性。像`ArrayList<String>`这样的泛型类告诉编译器只能添加String对象，消除了检索时转换的需要。泛型还通过类型无关算法实现更好的代码重用。编译器应用类型擦除，将泛型转换为其边界（如果是无界的则为Object）并插入转换，因此在运行时字节码类似于泛型之前的代码。

**追问方向：** 什么是类型擦除？它如何影响泛型类型的运行时行为？
**WMS Reference:** WMS `InventoryService`使用`List<SKU>`确保只有SKU对象可以被添加到库存列表中。

---

**Q31. Java泛型中无界通配符和原始类型有什么区别？**

**难度：** 高级

**答案：** 无界通配符（`<?>`）表示未知类型并且是类型安全的；你可以从`<?>`集合中读取，但只能作为Object引用，并且不能写入任何东西（除了null）。原始类型（`List`）是参数化类型的非泛型等价物，为了与泛型之前的代码向后兼容而存在；它们完全绕过类型检查并生成编译器警告。使用原始类型，你可以向`List`添加任何对象，而使用`List<?>`你不能添加任何对象（除了null）。原始类型在运行时通过泛型类字面量保留一些泛型类型信息，而无界通配符擦除到Object。在现代Java代码中应该避免使用原始类型。

**追问方向：** 何时使用`List<?>`而不是`List<Object>`？
**WMS Reference:** 在WMS `DataExportService`中，使用`List<?>`允许统一处理任何特定类型的集合。

---

**Q32. 解释上界通配符及其使用场景。**

**难度：** 中级

**答案：** 上界通配符（`<? extends Type>`）表示类型必须是Type或Type的子类型。当你只需要从集合中读取（消费-out）时使用它们，因为你可以安全地将元素读取为声明的类型，但由于对实际子类型的不确定性，不能添加元素。例如，`List<? extends Number>`可以接受`List<Integer>`、`List<Double>`或`List<Number>`。PECS原则（Producer Extends, Consumer Super）指导它们的使用：如果集合为你的代码生成元素，使用extends。不能添加元素是因为列表可能是更具体的类型如`List<Integer>`，添加`Double`会违反类型安全。

**追问方向：** 你可以向`List<? extends Number>`添加null吗？
**WMS Reference:** 在WMS `ReportingService`中，使用`List<? extends Order>`允许处理`List<RegularOrder>`和`List<RushOrder>`。

---

**Q33. 解释下界通配符及其使用场景。**

**难度：** 中级

**答案：** 下界通配符（`<? super Type>`）表示类型必须是Type或Type的超类型。当你只需要写入集合（消费-in）时使用它们，因为你可以安全地添加Type或其子类型，但读取只能得到Object引用。例如，`List<? super Integer>`可以接受`List<Integer>`、`List<Number>`或`List<Object>`。PECS原则适用：如果集合消费元素（你的代码写入其中），使用super。读取的限制是因为实际列表可能是`List<Object>`，不能保证返回超出Object的类型的元素。

**追问方向：** 如何结合上界和下界通配符创建像`<? extends Comparable<? super T>>`这样的有界通配符？
**WMS Reference:** 在WMS `SortingService`中，有界通配符允许编写接受任何可比较集合进行排序的方法。

---

**Q34. 什么是PECS原则？如何应用它？**

**难度：** 高级

**答案：** PECS代表"Producer Extends, Consumer Super"，是决定使用extends和super通配符的记忆法。如果你的代码从集合中读取元素（集合为你的代码生成元素），使用`extends`。如果你的代码写入集合（集合消费你代码中的元素），使用`super`。例如，一个对`List<Number>`中的数字求和的方法应该接受`List<? extends Number>`来读取数字。一个用整数填充列表的方法应该接受`List<? super Integer>`以允许将整数写入Number或Object列表。当同时需要生产和消费时，使用没有通配符的精确类型。正确应用PECS可以最大化灵活性同时保持类型安全。

**追问方向：** 为什么你不能将Double放入`List<? extends Number>`即使Double扩展了Number？
**WMS Reference:** 在WMS `InventoryCountAggregator`中，PECS有助于设计接受各种数值集合类型的灵活方法。

---

**Q35. 什么是泛型方法？它与泛型类有什么区别？**

**难度：** 中级

**答案：** 泛型方法是在返回类型之前声明类型参数的方法，适用于静态和非静态方法。与类型参数适用于整个类的泛型类不同，泛型方法的类型参数作用域仅限于方法本身。类型推断允许编译器从方法参数或上下文推断类型参数。泛型方法能够编写泛型类无法实现的代码，例如创建泛型静态实用方法。它们遵循与泛型类相同的类型擦除规则。泛型方法在实用类中特别有用，当泛型性特定于方法的行为而不是类时。

**追问方向：** 编写一个在WMS PriorityQueue中查找最大元素的泛型方法。
**WMS Reference:** 在WMS `UtilityService`中，泛型方法如`findMax(Collection<T>, Comparator<T>)`提供可重用的功能。

---

**Q36. 什么是类型令牌？它们与运行时泛型有什么关系？**

**难度：** 高级

**答案：** 类型令牌指的是在运行时传递Class<T>对象以保留在编译时由于类型擦除而被擦除的泛型类型信息。由于泛型类型参数在编译时被移除并替换为其擦除（通常是Object或第一个边界），运行时代码无法直接访问泛型类型。通过传递Class<T>令牌如`String.class`或`List<Order>.class`，方法可以在运行时接收类型信息。此模式用于Jackson等框架的反序列化以及泛型工厂模式。当编译时类型安全不再可用时，令牌允许创建类型化实例、执行类型化转换或在运行时验证类型兼容性。

**追问方向：** 你如何在WMS中使用类型令牌实现创建实例的泛型方法？
**WMS Reference:** 在WMS `ObjectFactoryService`中，使用`Class<T>`令牌能够从配置创建正确类型的对象。

---

**Q37. 什么是有界类型参数？何时使用它？**

**难度：** 中级

**答案：** 有界类型参数限制可用作类型实参的类型，将它们限制为指定类型的子类。声明为`<T extends Number>`，它允许类型参数为Number或任何子类如Integer或Double。有界类型支持调用特定于边界类型的方法而无需转换，因为编译器知道该类型是子类型。当类型必须实现多个接口时，可以使用`&`指定多个边界。没有边界，类型参数被擦除到Object，需要转换才能使用任何特定方法。有界类型对于编写使用数字、可比较或其他约束类型的泛型算法至关重要。

**追问方向：** 如果你指定`T extends Comparable & Serializable`会发生什么？
**WMS Reference:** 在WMS `SearchService`中，具有Comparable的有界类型确保项目可以排序，无论具体类型如何。

---

**Q38. 方法签名中`<T>`和`<?>`有什么区别？**

**难度：** 高级

**答案：** `<T>`声明一个新的类型参数，作用域限于该方法，允许编译器根据参数推断并强制执行实际类型，从而能够读取和写入正确类型的元素。`<?>`不声明类型参数，表示未知类型，允许作为Object读取并且除了null外不允许写入。使用`<T>`，方法可以用任何类型调用，返回类型将正确类型化为T；使用`<?>`，返回类型是Object，你不能添加元素。当方法不需要知道确切类型时使用通配符，而当类型对操作的语义重要时使用显式类型参数。

**追问方向：** 为什么不能从具有`List<?>`参数的方法返回正确类型的值？
**WMS Reference:** 在WMS `ValidationService`中，具有`<T>`的泛型方法能够返回正确类型的验证结果。

---

## 异常系统（Exception System）

**Q39. 解释Java异常层次结构以及checked异常和unchecked异常的区别。**

**难度：** 中级

**答案：** Java异常形成一个以Throwable为顶点的层次结构，分为Error和Exception。Error（如OutOfMemoryError）代表JVM级别的问题，应用程序不应捕获。Exception分为checked异常（编译时强制要求，必须声明或捕获）和unchecked异常（RuntimeException及其子类，编译器不强制）。Checked异常如IOException和SQLException代表调用者预期处理的 可恢复条件。Unchecked异常如NullPointerException和IllegalArgumentException通常表示编程错误或不可预测的运行时条件。存在这种区别是因为checked异常在编译时强制处理决策，鼓励健壮性，尽管许多人认为这是设计缺陷。

**追问方向：** 为什么InterruptedException是checked异常而NullPointerException是unchecked？
**WMS Reference:** 在WMS `FileImportService`中，IOException是读取导入文件时必须处理的checked异常。

---

**Q40. try-catch-finally和try-with-resources有什么区别？**

**难度：** 中级

**答案：** 传统的try-catch-finally需要在finally块中进行显式资源管理，清理代码可能抛出异常，可能掩盖原始异常。Try-with-resources（Java 7+）自动关闭实现AutoCloseable的资源，生成隐式清理代码。如果try体和close()方法都抛出，原始异常被保留，抑制的异常通过`addSuppressed()`添加。Try-with-resources更简洁，错误更少，并提供更好的异常处理。在try头中声明的资源是隐式final。这种模式是现代Java中资源管理的首选，消除了许多常见的资源泄漏错误。

**追问方向：** 如果在try-with-resources中资源声明本身发生异常会怎样？
**WMS Reference:** 在WMS `DatabaseConnectionService`中，try-with-resources确保数据库连接即使发生异常也能正确关闭。

---

**Q41. 什么是异常链？何时使用它？**

**难度：** 中级

**答案：** 异常链涉及捕获原始异常并将其包装在一个新异常中，同时保留原因关系。这通过`NewException(String, Throwable)`构造函数或initCause()方法完成。原始异常可以通过`getCause()`检索。当发生低级异常但应该转换为对调用者更有意义的高级异常时使用链，同时保留调试信息。链通过getCause()递归遍历。异常链与异常完成不同，后者只是捕获并重新抛出，因为链保留了原始异常的完整堆栈跟踪。

**追问方向：** 异常链如何与自定义WMS异常一起工作？
**WMS Reference:** 在WMS `OrderProcessingException`中，包装低级数据库异常提供上下文而不暴露实现细节。

---

**Q42. 在Java应用程序中未捕获异常的影响是什么？**

**难度：** 初级

**答案：** 当异常未被任何try-catch块捕获时，它会向上传播到调用栈。如果它在单线程应用程序中没有处理器到达栈顶，线程终止，堆栈跟踪被打印到错误流。在多线程应用程序中，线程中的未捕获异常不会影响其他线程，但会终止该特定线程。对于主线程中的未捕获异常，JVM以非零退出代码终止整个应用程序。要全局处理异常，你可以通过Thread.setDefaultUncaughtExceptionHandler()设置一个UncaughtExceptionHandler。在Web应用程序中，未捕获的异常通常由 servlet容器或框架错误处理器处理。

**追问方向：** 如何在WMS应用程序中实现全局异常处理？
**WMS Reference:** 在WMS `ApplicationErrorHandler`中，实现UncaughtExceptionHandler确保在应用程序关闭前记录关键错误。

---

**Q43. Java中throw和throws有什么区别？**

**难度：** 初级

**答案：** `throw`是一个关键字，实际触发异常创建和传播，在方法体内使用来抛出异常对象：`throw new IllegalArgumentException("invalid input")`。`throws`是方法签名修饰符，声明方法可能传播给其调用者的异常，告知它们必须处理的checked异常：`void method() throws IOException`。方法必须声明它可能抛出的所有checked异常，但可以选择不声明unchecked异常。仅throw而不捕获或声明会导致编译错误，而throws只是声明意图，实际上不会抛出任何东西。

**追问方向：** 方法可以在不声明的情况下抛出checked和unchecked异常吗？
**WMS Reference:** 在WMS `Validator`中，抛出IllegalArgumentException（unchecked）进行验证失败不需要声明。

---

**Q44. 企业应用程序中有效异常处理的指南是什么？**

**难度：** 高级

**答案：** 有效异常处理遵循几个原则：只捕获你有意义处理的，让意外异常传播；更喜欢特定异常类型而不是通用Exception；不要压制异常；在链式异常中保持原始原因；在系统边界记录以避免重复日志；使用finally或try-with-resources进行清理；避免捕获Throwable或Error；设计反映你领域的异常层次结构；提供带有上下文的有意义错误消息；考虑异常是否是控制流的正确机制。在企业应用程序中，框架的集中式错误处理提供一致性。异常处理应该与严重程度相称，绝不应该静默吞下异常。

**追问方向：** 如何设计特定于WMS的异常层次结构？
**WMS Reference:** 在WMS `ExceptionDesignService`中，具有基类WMSException、OrderNotFoundException和InventoryInsufficientException的层次结构提供有意义的错误分类。

---

**Q45. 什么是抑制异常？它与try-with-resources有什么关系？**

**难度：** 高级

**答案：** 抑制异常是在异常处理期间发生的次要异常，特别是在try-with-resources块的close()方法抛出而另一个异常已经在进行中时。当在try体中抛出异常并在资源清理期间发生另一个异常时，第一个异常被"抑制"并通过`addSuppressed()`添加到主要异常。主要异常的`getSuppressed()`返回抑制异常的数组。此机制防止在资源清理失败时丢失潜在的重要调试信息。在传统的try-finally中，finally异常会完全替换原始异常，使调试变得困难。

**追问方向：** 如何在WMS场景中检索主要和抑制的异常？
**WMS Reference:** 在WMS `FileProcessingService`中，当关闭文件流和数据库连接都失败时，抑制异常保留两个失败上下文。

---

**Q46. 解释RuntimeException类及其常见子类。**

**难度：** 初级

**答案：** RuntimeException是在运行时发生的异常的 超类，编译器不检查。常见子类包括NullPointerException（访问null引用）、IllegalArgumentException（方法无效参数）、IllegalStateException（对象对于操作无效状态）、IndexOutOfBoundsException（数组/列表索引违规）、ClassCastException（无效转换）和ArithmeticException（数学错误）。这些通常代表编程错误或不可预测的运行时条件，应该修复而不是捕获。由于它们是unchecked的，方法不需要在throws子句中声明它们。许多框架和库为调用者通常无法从中恢复的条件抛出RuntimeException。

**追问方向：** 何时应该捕获RuntimeException与让它传播？
**WMS Reference:** 在WMS `InputValidator`中，捕获IllegalArgumentException允许提供用户友好的验证消息。

---

**Q47. 异常处理和异常屏蔽有什么区别？**

**难度：** 高级

**答案：** 异常屏蔽指的是捕获异常但没有正确处理或传播它，有效地隐藏错误条件。这可能发生在异常被静默忽略时，当只解决部分错误时，或者当通用catch块捕获但不重新抛出时。这导致难以调试的静默失败。合法的异常处理涉及完全处理条件、适当传播或将转换为更有意义的异常同时保留链。正确处理确保错误为调试和适当操作显露，同时保持系统完整性。

**追问方向：** 为什么有人可能故意屏蔽异常，有什么替代方案？
**WMS Reference:** 在WMS `NonCriticalOperationExecutor`中，用日志屏蔽异常允许操作继续同时仍记录问题。

---

## Stream API（流API）

**Q48. 什么是Java Stream API？它与集合有什么区别？**

**难度：** 中级

**答案：** Java Stream API在Java 8中引入，对元素序列提供函数式操作。与集合不同，流不存储元素；它们是对数据源（如集合、数组或I/O通道）计算的视图。集合是持有元素的数据结构，而流是关于计算和数据处理的。流支持中间操作（filter、map、flatMap）返回新流和产生结果的终端操作（collect、reduce、forEach）。流可以顺序或并行处理而不改变底层数据源。一旦消耗，流不能重用；必须从源重新生成。

**追问方向：** 流能否提高WMS数据处理的性能？
**WMS Reference:** 在WMS `BatchOrderProcessor`中，并行流可以并发处理数千个订单以提高吞吐量。

---

**Q49. 解释中间操作和终端流的区别。**

**难度：** 中级

**答案：** 中间操作是惰性操作，返回新流，在调用终端操作之前不会执行。示例包括filter()、map()、flatMap()、distinct()、sorted()和limit()。它们可以链接在一起形成处理管道。终端操作触发流的实际遍历和物化，产生结果或副作用。示例包括forEach()、collect()、reduce()、toArray()和count()。这种惰性评估允许流进行优化，如短路和避免不必要的操作。没有终端操作，中间操作永远不会执行，这是一个常见错误。每个管道只有一个终端操作。

**追问方向：** 流操作中的短路是什么？
**WMS Reference:** 在WMS `OrderSearchService`中，使用limit()和anyMatch()在找到第一个匹配订单时短路流评估。

---

**Q50. 流中map()和flatMap()有什么区别？**

**难度：** 高级

**答案：** map()操作使用函数将每个元素转换为另一个元素，为每个输入元素产生一个输出元素的流。flatMap()操作将每个元素转换为一个流，然后将所有这些流平展为单个流。例如，对订单流使用`map(Order::getItems)`产生`Stream<List<Item>>`，而使用`flatMap(Order::getItems)`产生包含所有订单项目的`Stream<Item>`。flatMap用于每个输入元素产生多个输出元素的场景，如将字符串拆分为单词或提取嵌套集合。"平展"消除了嵌套流结构，创建单个元素的平面流。

**追问方向：** 如何使用flatMap从具有多个行项目的订单中提取所有SKU？
**WMS Reference:** 在WMS `SkuAggregationService`中，flatMap(Order::getLineItems)后跟map(LineItem::getSku)创建跨订单的所有SKU流。

---

**Q51. 什么是流归约？reduce()方法如何工作？**

**难度：** 高级

**答案：** 流归约使用关联累积函数将流元素组合成单个结果。reduce()方法有三个重载：identity（初始值）、accumulator（二元运算符）和combiner（用于并行执行）。例如，`stream.reduce(0, Integer::sum)`从0开始并添加每个元素。identity值必须是累加函数的 identity，意味着与任何元素组合返回该元素。累加器必须是关联的，以便并行归约正确工作。归约是产生最终输出的终端操作。并行流使用combiner来合并来自不同线程的部分结果。

**追问方向：** 如何实现并行归约来计算WMS中的总库存价值？
**WMS Reference:** 在WMS `InventoryValuationService`中，使用`reduce(BigDecimal.ZERO, BigDecimal::add, BigDecimal::add)`的并行流归约计算总库存价值。

---

**Q52. collect()操作是什么？有哪些常见的收集器？**

**难度：** 高级

**答案：** collect()操作执行可变归约，结果累积到可变容器如Collection、Map或String中。收集器是常见集合场景的预构建实现：toList()、toSet()、toMap()、groupingBy()、partitioningBy()、joining()、counting()、summingInt()、averagingDouble()等等。收集器可以使用collectingAndThen()组合进行后处理，并使用下游收集器组合进行复杂聚合。toList()收集器将元素收集到List中，而toSet()创建Set。对于Map，你指定键和值映射器。groupingBy()收集器按分类函数对元素分组，创建带有元素列表的Map。

**追问方向：** 如何使用groupingBy按仓库位置对WMS订单进行分组？
**WMS Reference:** 在WMS `OrderReportingService`中，collect(groupingBy(Order::getWarehouseId))按仓库对订单进行分组以进行区域报告。

---

**Q53. 解释流操作中短路的概念。**

**难度：** 高级

**答案：** 短路操作在确定结果后立即停止处理，无需检查所有元素。在中间操作中，limit()和skip()可以限制流大小；findFirst()、findAny()、anyMatch()、allMatch()和noneMatch()可以提前终止。例如，anyMatch()在找到匹配元素时立即返回true，而不检查其余的。这类似于布尔表达式中&&运算符的评估。短路在不需要处理完整流时提供效率，对无限流和性能优化至关重要。并非所有中间操作都是短路的；filter()必须检查所有元素，除非与短路的终端操作结合。

**追问方向：** 为什么短路对大型WMS数据集很重要？
**WMS Reference:** 在WMS `UrgencyChecker`中，对大型订单列表使用anyMatch()在找到紧急订单时立即返回。

---

**Q54. 顺序流和并行流有什么区别？**

**难度：** 中级

**答案：** 顺序流在单个线程中一个接一个地处理元素，使用主线程的CPU。并行流将数据分割成段，在通用的ForkJoinPool中并发处理，自动利用多个核心。并行流通过在现有流上调用parallel()或对集合调用parallelStream()创建。性能改进取决于数据大小、操作复杂度和操作是否为CPU密集型；对于小数据集，并行开销可能使其更慢。对于有序流，parallel()保留遇到顺序除非显式使用unordered()。操作必须是无状态的和非干扰的才能正确并行执行。

**追问方向：** 流中哪些操作会阻止有效并行化？
**WMS Reference:** 在WMS `ParallelBatchProcessor`中，理解并行流行为确保订单批次的 安全并发处理。

---

**Q55. 流排序如何与并行处理交互？**

**难度：** 高级

**答案：** 默认情况下，流维护基于源和中间操作的遇到顺序，这可能限制并行性，因为结果必须按顺序组装。使用unordered()移除排序约束，允许更好的并行化，实现工作窃取和结果组合而无需排序保证。distinct()和sorted()等操作固有地需要排序信息或施加排序。对于具有大型数据集的并行流，通过unordered()移除排序约束可以显著提高性能。然而，对于toList()或toSet()等操作，排序可能重要也可能不重要，取决于需求。forEach()操作在并行流中失去排序保证；如果需要排序，使用forEachOrdered()。

**追问方向：** 如何实现无序并行流以提高WMS订单处理性能？
**WMS Reference:** 在WMS `ParallelSkuCounter`中，对大型库存使用unordered().parallel()允许更快的计数而无需排序要求。

---

**Q56. 什么是有状态与无状态流操作？**

**难度：** 高级

**答案：** 无状态操作如filter()、map()和flatMap()独立处理每个元素，不考虑其他元素或维护状态。有状态操作需要了解所有元素或在流中维护内部状态。示例包括sorted()、distinct()和limit()，它们可能需要处理所有输入才产生输出。有状态操作在并行执行中可能出现问题，因为它们可能需要同步或从并行分支合并状态。在无限流中使用有状态操作可能导致流永不终止。理解有状态性对于编写正确的并行流代码和预测流行为很重要。

**追问方向：** 为什么sorted()在并行流中可能导致性能问题？
**WMS Reference:** 在WMS `OrderSorter`中，理解sorted()作为有状态操作有助于估计处理复杂度和内存需求。

---

**Q57. forEach和forEachOrdered在流中有什么区别？**

**难度：** 中级

**答案：** forEach()对每个元素应用操作，不保证任何执行顺序，这在并行流中更快，因为框架可以任何顺序处理元素。forEachOrdered()按遇到顺序对每个元素应用操作，遵循源或先前操作建立的顺序。对于顺序流，两者产生相同的顺序但forEachOrdered()开销略高。对于并行流，forEachOrdered()强制单线程执行以保持顺序，失去并行性能优势。当顺序重要时（如显示排序数据）使用forEachOrdered()；当顺序不重要而性能重要时使用forEach()。

**追问方向：** 在WMS报告上下文中何时会首选forEachOrdered？
**WMS Reference:** 在WMS `AuditReportGenerator`中，forEachOrdered()确保审计条目按时间顺序打印。

---

## Lambda表达式（Lambda Expressions）

**Q58. 什么是Lambda表达式？为什么在Java 8中引入它们？**

**难度：** 初级

**答案：** Lambda表达式是简洁的匿名函数，通过将行为作为一等公民来实现函数式编程。它们在Java 8中引入以支持Stream API并提供比匿名内部类更清晰、更富有表现力的代码。Lambda有三个部分：参数、箭头标记（->）和主体。编译器从上下文推断参数类型，主体可以是表达式或语句块。Lambda表达式实现函数式接口（具有单个抽象方法的接口）。它们使代码更具可读性、可维护性，并适合并行处理。Lambda对于流管道模型至关重要，其中行为作为数据传递。

**追问方向：** 编译器如何推断lambda参数类型？
**WMS Reference:** 在WMS `OrderFilter`中，lambda表达式如`order -> order.getPriority() == URGENT`简化了过滤逻辑。

---

**Q59. 解释函数式接口的概念并列出一些常见示例。**

**难度：** 中级

**答案：** 函数式接口是只有一个抽象方法的接口，尽管它可能有多个默认或静态方法。@FunctionalInterface注解在编译时强制执行此契约。java.util.function中的常见示例包括Predicate<T>（test方法返回布尔值）、Function<T,R>（apply方法将T转换为R）、Consumer<T>（accept方法消费T）、Supplier<T>（get方法返回T）和UnaryOperator<T>扩展Function<T,T>。Runnable、Callable和Comparator也是函数式接口。Lambda表达式可以分配给任何具有匹配单一抽象方法签名的函数式接口。方法引用是通过名称引用方法的lambda简写。

**追问方向：** Comparator有多个方法为什么还是函数式接口？
**WMS Reference:** 在WMS `OrderSorter`中，Comparator.comparing(Order::getDueDate)使用Comparator的函数性质进行流畅排序。

---

**Q60. 方法引用和lambda表达式有什么区别？**

**难度：** 中级

**答案：** 方法引用是调用特定方法的lambda表达式的简写记号，使代码在lambda简单转发到现有方法时更具可读性。有四种类型：静态方法引用（ClassName::staticMethod）、特定实例上的实例方法（instance::instanceMethod）、类型上的实例方法（ClassName::instanceMethod，其中第一个参数成为接收者）和构造函数引用（ClassName::new）。Lambda表达式可以表达不直接映射到方法调用的更复杂逻辑，而方法引用需要具有匹配签名的现有方法。两者都实现函数式接口，在签名匹配时可以互换。

**追问方向：** 如何将`order -> order.getOrderId()`重写为方法引用？
**WMS Reference:** 在WMS `OrderIdCollector`中，使用Order::getOrderId作为方法引用简化了映射操作。

---

**Q61. Lambda表达式中的目标类型是什么？**

**难度：** 高级

**答案：** 目标类型是编译器从使用上下文推断lambda表达式类型的能力。当lambda被分配给函数式接口变量、作为参数传递给期望函数式接口的方法或显式转换时，编译器使用目标类型检查lambda的签名是否匹配。Lambda主体不需要显式声明参数类型，因为它们是从目标函数式接口的抽象方法推断的。目标类型允许泛型代码接受具有正确推断类型的lambda，实现流畅API并减少语法噪音。具有不同函数式接口的多个重载需要显式类型以消除歧义。

**追问方向：** 为什么目标类型对接受函数式接口的泛型方法重要？
**WMS Reference:** 在WMS `FilterBuilder<T>`中，具有函数式接口参数的泛型方法支持创建可重用的过滤逻辑。

---

**Q62. 什么是有效最终变量？为什么它们对lambda重要？**

**难度：** 中级

**答案：** 如果变量的值在初始化后没有改变，无论是否显式声明为final，它都是有效最终变量。Lambda可以从其封闭作用域引用有效最终变量，这使它们能够作为闭包使用。这种限制存在是因为lambda捕获值而不是变量；运行时复制的是变量的值，而不是变量本身的引用。如果变量是可变的，lambda将使用过时值，导致令人困惑的行为。在lambda内部修改有效最终变量会导致编译错误。此设计简化了对lambda行为的推理，并支持JVM的未来潜在优化。

**追问方向：** 有效最终变量的概念如何与lambda捕获的局部变量相关？
**WMS Reference:** 在WMS `TransactionHandler`中，lambda为数据库操作捕获有效最终事务上下文。

---

**Q63. 解释捕获和非捕获lambda的区别。**

**难度：** 高级

**答案：** 非捕获lambda不引用其封闭作用域中的任何变量，本质上是无状态的；它们可以计算一次并重用。捕获lambda从封闭作用域引用一个或多个变量，每次调用需要一个新的lambda实例，因为捕获的值可能不同。捕获lambda创建成本更高，因为它们涉及分配存储捕获值的闭包对象。许多JVM优化可以应用于非捕获lambda，可能创建单个实例。对于性能关键代码，在紧密循环中创建lambda时，这种区别很重要。实现行为而不仅仅是转换数据的有状态lambda通常需要捕获。

**追问方向：** 如何在WMS性能分析中识别lambda是捕获还是非捕获？
**WMS Reference:** 在WMS `HotPathAnalyzer`中，识别在循环中创建的捕获lambda有助于优化关键订单处理路径。

---

**Q64. Java中lambda对象的生命周期是什么？**

**难度：** 高级

**答案：** Lambda对象在评估lambda表达式时创建，而不是在定义时，其生命周期跟随持有它的引用。对于非捕获lambda，JVM可能创建在调用之间重用的单个实例。捕获lambda每次评估发生时创建一个新对象，将捕获的值存储在实例中。编译器生成的lambda类实现目标函数式接口，在首次使用时加载到JVM。一旦没有引用，它们就有资格像任何其他对象一样被垃圾回收。Lambda表达式本身在源代码意义上不会被具体化为对象；只有它们的结果实例在运行时存在。

**追问方向：** invokedynamic指令与lambda创建有什么关系？
**WMS Reference:** 在WMS `ClassLoaderAnalysis`中，理解通过invokedynamic创建lambda有助于分析应用程序内存模式。

---

## Object方法（Object Methods）

**Q65. 每个Java对象从Object类继承哪些方法？**

**难度：** 初级

**答案：** 每个Java类隐式扩展java.lang.Object，继承几个基本方法：toString()返回对象的字符串表示；equals(Object)定义对象相等比较；hashCode()返回用于基于哈希的集合的整数；getClass()返回表示对象类型的Class对象；notify()、notifyAll()和wait()用于线程同步；clone()创建对象的副本；finalize()（已弃用）用于垃圾回收清理。这些方法为所有Java对象形成基础契约，提供身份、相等语义、字符串表示和并发原语。在集合和其他框架操作中，适当重写这些方法对正确行为至关重要。

**追问方向：** 为什么应该一致地重写toString()、equals()和hashCode()？
**WMS Reference:** 在WMS `Sku`类中，这些方法的一致实现确保在HashSet和HashMap中的正确行为。

---

**Q66. 解释equals()和hashCode()之间的契约。**

**难度：** 高级

**答案：** equals-hashCode契约要求当两个对象根据equals()相等时，它们必须有相同的hashCode()。反过来，具有相同hashCode()的对象不需要相等（允许哈希冲突）。如果重写equals()，必须一致地重写hashCode()以保持此契约。违反此契约会导致基于哈希的集合（如HashMap和HashSet）中的错误行为，对象可能存储在错误的桶中或变得找不到。契约由equals()检查，但必须由程序员维护。对象在哈希集合中的整个生命周期内必须具有相同的hashCode()；如果在对象在集合中时更改hashCode()会导致损坏。

**追问方向：** 当用作HashMap键的对象在插入后其hashCode被更改会发生什么？
**WMS Reference:** 在用作HashMap键的WMS `Location`类中，hashCode必须保持稳定，否则位置将变得找不到。

---

**Q67. 何时应该重写equals()？何时默认实现足够？**

**难度：** 中级

**答案：** 当对象相等性应该基于逻辑身份而不是引用身份时，重写equals()，通常对于值对象或逻辑相等重要的域对象。默认Object.equals()比较引用，适用于表示唯一实体的对象，如线程或输入流。Order、SKU或Location等域对象应重写equals()以比较相关字段。一旦重写equals()，也必须重写hashCode()以保持契约。使用@Override注解有助于捕获错误。equals()方法必须是自反的、对称的、传递的、一致的，并正确处理null。

**追问方向：** 对于WMS Order类，equals()应该包含哪些字段？
**WMS Reference:** 在WMS `Order`类中，equals()应该比较orderId，因为具有相同orderId的两个订单代表同一个订单。

---

**Q68. 用于比较对象的==和equals()有什么区别？**

**难度：** 初级

**答案：** ==运算符对对象类型比较引用（内存地址），对基本类型比较值。对于对象，==仅当两个引用指向内存中完全相同的对象时返回true。equals()是一个方法，默认行为像==但可以重写以定义自定义相等语义。对于String和包装类型，由于JVM优化（如小整数或interned字符串的缓存），==有时可能返回true，但不应依赖此行为。在比较对象的逻辑相等性时，始终使用equals()。使用==进行对象比较比较身份而非内容，这对于域对象很少是预期行为。

**追问方向：** 为什么String与==的比较在Java中有时有效？
**WMS Reference:** 在WMS `StatusChecker`中，使用equals()而不是==比较状态字符串确保正确行为，不管String interning如何。

---

**Q69. 详细解释equals()方法契约。**

**难度：** 高级

**答案：** equals()契约有四个要求：自反性（x.equals(x)必须返回true）、对称性（如果x.equals(y)则y.equals(x)）、传递性（如果x.equals(y)和y.equals(z)则x.equals(z)）和一致性（如果对象未修改，重复调用返回相同结果）。此外，对于任何非空引用x，x.equals(null)必须返回false。违反其中任何一项都可能导致集合和其他Java API中不可预测的行为。最常见的错误是在子类层次结构中，子类添加影响相等的字段，破坏对称性或传递性。常见解决方案是对于值对象优先使用组合而不是继承，或使用特定类而不是继承来实现相等键。

**追问方向：** 违反对称性如何影响HashSet行为？
**WMS Reference:** 在WMS `ItemSet`中，违反equals对称性导致contains()根据用于查找的对象表现不一致。

---

**Q70. 实现正确hashCode()的指南是什么？**

**难度：** 高级

**答案：** 正确的hashCode()应该为根据equals()相等的实例返回相同的值，在实际可行的情况下为不相等的对象产生不同的哈希码，是一致的（跨调用对相同对象返回相同值），并避免调用synchronized方法等昂贵操作。有效的实现通常使用质数乘数并使用公式如`result = 31 * result + fieldHashCode`组合重要字段的哈希码。使用的字段数量应与equals()中使用的字段匹配。使用equals()中的所有字段通常提供更好的分布。Objects.hash()（Java 7+）提供了一种组合多个字段哈希的便捷方式。糟糕的hashCode()实现会导致HashMap性能下降。

**追问方向：** 为什么在许多hashCode实现中使用31作为乘数？
**WMS Reference:** 在WMS `Location` hashCode中，使用31组合zone、aisle、rack和shelf为HashMap存储提供良好的分布。

---

**Q71. toString()方法的默认实现是什么？何时应该重写它？**

**难度：** 初级

**答案：** 默认toString()实现返回类名，后跟@和对象的哈希码的十六进制，如"Order@4d7c5a"。这对调试很少有用。重写toString()以为日志、调试和开发工具提供有意义的表示。好的toString()实现包含所有相关字段，但避免包含密码等敏感数据。对于集合，集合的toString()已经遍历元素调用它们的toString()。在SLF4J与{}-logging等框架中，toString()被自动调用。好的toString()显著减少调试时间并提高日志可读性。

**追问方向：** toString()如何与日志框架交互？
**WMS Reference:** 在WMS `LoggingAspect`中，域对象上正确的toString()确保订单处理期间有意义的日志条目。

---

**Q72. clone()方法是什么？有什么替代方案？**

**难度：** 高级

**答案：** clone()创建并返回对象的副本，但它有几个问题：它在Object中声明为protected，返回Object需要转换，对于是引用的字段执行浅拷贝，并且如果类没有实现Cloneable则抛出CloneNotSupportedException。Cloneable接口是一个没有方法的标记接口，表示对象的克隆资格。替代方案包括复制构造函数如`new Order(Order other)`，它们是类型安全的并允许深拷贝，或静态工厂方法。由于其清晰性，在现代Java中通常更喜欢复制构造函数。对于不可变对象，拷贝是不必要的。对于具有final字段的复杂对象，clone()可能不起作用。

**追问方向：** 什么是复制构造函数？它与clone()如何比较？
**WMS Reference:** 在WMS `OrderService`中，复制构造函数Order(Order)支持创建用于修改跟踪的订单快照。

---

## 其他集合框架问题

**Q73. Iterator和ListIterator有什么区别？**

**难度：** 中级

**答案：** Iterator是用于遍历集合并可选地在迭代期间移除元素的前向游标。ListIterator通过双向移动（previous和next）、添加、设置和移除元素的能力以及访问元素索引的能力扩展Iterator。ListIterator特定于List实现如ArrayList、LinkedList和Vector。Iterator接口有hasNext()、next()和remove()方法，而ListIterator添加hasPrevious()、previous()、nextIndex()、previousIndex()、add()、set()和remove()。使用ListIterator，你可以在迭代期间修改列表并以任一方向遍历，使其更强大但仅适用于有序集合。

**追问方向：** 如何使用ListIterator实现WMS订单行项目的向后遍历？
**WMS Reference:** 在WMS `ReverseOrderProcessor`中，ListIterator允许从最后到第一遍历行项目以进行反向处理。

---

**Q74. Collections工具类的目的是什么？**

**难度：** 初级

**答案：** Collections工具类提供用于操作集合的静态方法，包括不可修改包装器（unmodifiableList、unmodifiableMap）、同步包装器（synchronizedList、synchronizedMap）、用于运行时类型安全的检查包装器、搜索（binarySearch）、排序（sort、reverse、shuffle、fill）、查找极值（max、min）和旋转/交换元素。这些是Java 1.2中的遗留方法，早于Stream API。许多方法已被集合接口中的默认方法或Stream操作取代，但synchronized和不可修改包装器仍然相关。检查包装器强制对原始集合进行编译时类型检查。这些实用方法在Collection接口而不是特定实现上操作。

**追问方向：** Collections.unmodifiableList与Java 9+中的List.of()有什么区别？
**WMS Reference:** 在WMS `ReadOnlyOrderView`中，Collections.unmodifiableList()防止对底层订单列表的外部修改。

---

**Q75. List.of()和Collections.unmodifiableList()有什么区别？**

**难度：** 高级

**答案：** List.of()（Java 9+）创建一个不可变列表，不允许null元素并在任何修改尝试时抛出UnsupportedOperationException。Collections.unmodifiableList()包装现有列表，也阻止修改，但将读取传递到底层列表；底层列表仍然可以修改，影响不可修改视图。List.of()不允许null元素，而Collections.unmodifiableList()允许底层列表包含null（尽管对视图的修改会失败）。List.of()返回一个真正的不可变列表（java.util.ImmutableCollections.ListN），针对大小和内存进行了优化。对于创建新的不可变列表，首选List.of()；对于保护现有列表，可能需要Collections.unmodifiableList()。

**追问方向：** 你可以从另一个引用修改传递给Collections.unmodifiableList()的List吗？
**WMS Reference:** 在WMS `OrderSnapshotService`中，理解差异确保快照列表即使源数据更改也保持不可变。

---

**Q76. 解释HashSet、LinkedHashSet和TreeSet之间的区别。**

**难度：** 中级

**答案：** HashSet在哈希表中存储元素，提供O(1)的添加、移除和包含操作，没有顺序保证。LinkedHashSet扩展HashSet，通过链表维护插入顺序，同时保留基本操作的O(1)性能并允许可预测的迭代。TreeSet在排序的红黑树结构中存储元素，提供O(log n)的操作并根据自然顺序或Comparator维护元素排序。TreeSet要求元素实现Comparable或必须提供Comparator。LinkedHashSet由于链表开销比HashSet稍慢并使用更多内存。当需要排序迭代或范围操作时，TreeSet是合适的。

**追问方向：** 你会使用哪个实现来维护按仓库排序的WMS位置？
**WMS Reference:** 在WMS `WarehouseNavigator`中，TreeSet在物理顺序中维护Location对象用于路径规划。

---

**Q77. 不同Set实现中操作的时间复杂度是什么？**

**难度：** 中级

**答案：** HashSet为添加、移除、包含和大小操作提供O(1)的平均时间复杂度，在哈希冲突严重时最坏情况为O(n)。LinkedHashSet具有与HashSet相同的时间复杂度，外加维护插入顺序的O(1)。TreeSet由于其红黑树实现提供O(log n)的时间复杂度，外加需要排序的操作如subSet和first的O(log n)。EnumSet为枚举键使用位图，提供具有最小内存的O(1)常量时间操作。ConcurrentSkipListSet提供O(log n)的操作与线程安全。对于大型集合，选择对性能影响显著。

**追问方向：** 负载因子如何影响HashSet性能？
**WMS Reference:** 在WMS `LargeSkuCatalog`中，理解HashSet性能有助于根据是否需要排序在HashSet和TreeSet之间进行选择。

---

**Q78. HashMap和WeakHashMap有什么区别？**

**难度：** 高级

**答案：** WeakHashMap是一个Map实现，其中键存储为弱引用，允许在键不再在其他地方被引用时进行垃圾回收。与防止存储键被GC的常规映射不同，WeakHashMap条目可以在键对象变得弱可达时自动移除。这对于规范化的映射（如缓存的字符串interning或记忆化）很有用，当你希望条目在应用程序的其他部分不再引用键时消失。当WeakHashMap条目被GC时，其条目对象变得无效，将在下一个操作中移除，但时机不是确定性的。常规HashMap键被强引用并阻止GC。

**追问方向：** WeakHashMap何时在WMS上下文中合适？
**WMS Reference:** 在WMS `ObjectCacheService`中，WeakHashMap可以保存临时缓存，其中条目应在内存压力时被GC掉。

---

**Q79. 什么是IdentityHashMap？何时使用它？**

**难度：** 高级

**答案：** IdentityHashMap是一个Map实现，它使用身份（==）而不是equals()进行键比较。只有当k1 == k2（引用比较）时，两个键k1和k2才被认为是相等的。这违反了Map的一般契约，该契约假设基于equals()的相等性。IdentityHashMap使用System.identityHashCode()进行哈希，这基于对象的内存地址，与其身份比较一致。这用于场景如规范化（每个身份只保留一个实例）、维护基于对象的元数据或深度基于指针的数据结构。它不像HashMap那样使用哈希桶，而是使用不同的内部结构。

**追问方向：** 为什么IdentityHashMap对序列化规范化有用？
**WMS Reference:** 在WMS `SerializationHelper`中，IdentityHashMap跟踪已序列化的对象以处理循环引用。

---

**Q80. Queue接口及其实现的目的是什么？**

**难度：** 中级

**答案：** Queue接口通过插入、提取和检查操作扩展Collection，用于在处理之前保存元素。操作成对出现：add/eject和offer/poll用于插入和移除，具有不同的异常行为；element/peek和remove/poll用于检查和移除。FIFO语义是典型的但不是必需的；PriorityQueue按值排序。LinkedList实现Queue作为双端队列（Deque）。阻塞队列如BlockingQueue支持等待队列变为非空或非满的操作。队列对于任务调度、生产者-消费者模式和广度优先遍历算法是基础的。

**追问方向：** BlockingQueue与常规Queue在并发场景中有什么区别？
**WMS Reference:** 在WMS `TaskQueueService`中，BlockingQueue支持生产者-消费者模式，其中拣货任务排队并由工作线程执行。

---

## 其他泛型问题

**Q81. 什么是泛型类？如何定义它？**

**难度：** 初级

**答案：** 泛型类在类名后用类型参数部分（通常是<T>）定义，允许类在实例化时对类型进行操作。例如，`class Box<T>`定义了一个可以保存任何类型的盒子。类型参数可以使用extends绑定来限制可接受的类型。可以声明多个类型参数如`class Pair<K, V>`。在实例化时，你提供具体类型如`Box<String>`或`Pair<String, Integer>`。类型参数在整个类体中作为类型名可用。泛型类支持实例和类（静态）成员，尽管静态成员不能使用类的类型参数。

**追问方向：** 泛型类和原始类型有什么区别？
**WMS Reference:** 在WMS `Container<T extends Item>`中，泛型类型确保容器只保存特定项目类型。

---

**Q82. Java中泛型类型使用的限制是什么？**

**难度：** 高级

**答案：** 由于类型擦除，泛型有几个限制：不能使用原始类型作为类型实参（使用Integer而不是int）；不能声明带有类类型参数的静态字段；不能对参数化类型使用instanceof（除了无界通配符）；不能创建参数化类型数组如new List<String>[10]；不能捕获或抛出泛型类的实例；重载方法不能有擦除到相同类型的类型参数。这些限制存在是因为泛型类型信息在运行时不可用。解决方法包括使用包装类处理原始类型，使用Class对象进行类型令牌，以及使用无界通配符进行instanceof检查。

**追问方向：** 为什么你不能创建`new T()`在泛型类中？
**WMS Reference:** 在WMS `GenericFactory<T>`中，需要像Class.newInstance()或通过类型令牌的反射这样的替代方案来创建实例。

---

**Q83. 什么是泛型构造函数？它与泛型方法有什么区别？**

**难度：** 高级

**答案：** 泛型构造函数有自己的类型参数，与类的类型参数不同，在构造函数签名中声明如`public <T> MyClass(T item)`。构造函数的类型参数作用域仅限于构造函数，而不是类。这允许为构造函数推断与类不同的类型。例如，在`class Box<T> { public <U> Box(U item) { } }`中，U由传递给构造函数的参数决定，与T可能是什么不同。泛型构造函数支持类型从参数推断的流畅API。与泛型方法不同，泛型构造函数不要求类本身是泛型的。

**追问方向：** 你如何在WMS Builder模式中使用泛型构造函数？
**WMS Reference:** 在WMS `OrderBuilder`中，泛型构造函数允许类型安全的流畅构建订单对象。

---

**Q84. Get Put原则（PECS）的更多细节是什么？**

**难度：** 高级

**答案：** Get Put原则指出，在使用不知道或不关心其特定类型的集合时，你应该使用extends进行获取（读取）元素和使用super进行放入（写入）元素。对于将只输出数据的参数，使用extends；对于将只输入数据的参数，使用super；当两者都不是时，使用精确类型。这在保持类型安全的同时最大化灵活性。在同一签名中同时组合两者是不可能的，通配符用于"不知道/不关心"场景。协变（extends）允许读取为边界类型，而逆变（super）允许从边界类型写入。PECS是与泛型集合API设计的实用规则。

**追问方向：** PECS如何应用于既读取又写入集合的方法？
**WMS Reference:** 在WMS `CollectionProcessor`中，当方法既读取现有项目又添加新项目时，PECS指导有助于设计签名。

---

**Q85. Java泛型中的类型见证是什么？**

**难度：** 高级

**答案：** 类型见证是调用泛型方法时提供的显式类型实参，用于当推断不能自动确定类型时。例如，在`Collections.<String>emptyList()`中，`<String>`是类型见证。类型见证可以消除类型推断模糊的重载方法，或者确保选择正确的泛型重载。编译器使用见证检查类型兼容性并用于返回类型推断。虽然由于推断通常可选的，但显式见证在复杂泛型代码中提高可读性并解决某些边缘情况。Java 10引入了减少对显式见证需求的类型推断改进。

**追问方向：** 何时需要在WMS代码中提供显式类型见证？
**WMS Reference:** 在WMS `ReflectionHelper`中，显式类型见证帮助泛型方法返回正确类型的结果。

---

## 其他Stream API问题

**Q86. findFirst()和findAny()在流中有什么区别？**

**难度：** 中级

**答案：** findFirst()根据遇到顺序返回流的第一个元素，对于有序流总是确定性的和可预测的。findAny()返回流的任何元素，每次调用可能返回不同结果，并可以自由地为更好的性能返回任何元素。对于顺序有序流，两者返回相同的元素。对于并行流，findAny()可能从任何并行分支返回任何元素，使其更快。当顺序重要且需要确定性第一个元素时使用findFirst()。当你需要任何元素并想要最佳并行性能时使用findAny()。对于任何匹配订单都足够的并行WMS处理，findAny()可能更快。

**追问方向：** findAny()在串行执行中保证任何特定元素吗？
**WMS Reference:** 在WMS `AnyAvailablePickerFinder`中，findAny()可以找到任何可用的拣货员，而不需要第一个。

---

**Q87. count()和size()在流中有什么区别？**

**难度：** 初级

**答案：** count()是终端流操作，返回流中元素的数量作为long。它通过遍历整个流来计数元素，所以它是一个消耗流的终端操作。size()是Collection接口上的方法，返回元素数量而不进行修改。调用count()后，流被消耗，不能重用；你需要从源重新生成流。对于已知集合，直接调用size()比创建流并调用count()更高效。count()适用于任何Stream，包括无限流（在哪里不会终止）。

**追问方向：** 何时count()比size()更受青睐？
**WMS Reference:** 在WMS `FilteredOrderCounter`中，对过滤流的stream count()在源集合类型不同时很有用。

---

**Q88. 使用groupingBy的collect()方法如何工作？**

**难度：** 高级

**答案：** groupingBy收集器创建一个Map，其中键是应用于流元素分类函数的结果，值是映射到该键的元素列表。签名`Collectors.groupingBy(Function<T,K>)`返回`Map<K,List<T>>`。有重载接受下游收集器对每个组执行额外聚合，如`groupingBy(Order::getStatus, counting())`获得每个状态的计数。groupingBy在底层为并行流使用ConcurrentHashMap。你可以链接下游收集器如mapping、minBy、maxBy或collectingAndThen进行复杂的多阶段聚合。这实现了类似于SQL的GROUP BY功能。

**追问方向：** 如何实现WMS订单按仓库然后按状态的多级分组？
**WMS Reference:** 在WMS `RegionalOrderAnalyzer`中，groupingBy(Order::getWarehouseId, groupingBy(Order::getStatus))提供分层聚合。

---

**Q89. Collectors.toMap()方法是什么？重复键时会发生什么？**

**难度：** 高级

**答案：** Collectors.toMap()收集器使用键和值映射函数将流转换为Map。默认情况下，它在遇到重复键时抛出IllegalStateException。要处理重复项，请使用重载`toMap(keyMapper, valueMapper, mergeFunction)`，其中合并函数解决冲突，如`(v1, v2) -> v1`保留第一个。例如，`toMap(Order::getOrderId, order -> order, (a, b) -> a)`创建订单ID到订单的Map，保留重复项的第一个订单。使用LinkedHashMap作为集合供应商保留插入顺序。对于并发映射，使用`toConcurrentMap()`。

**追问方向：** 如何在WMS中合并具有相同订单ID的订单同时求和其数量？
**WMS Reference:** 在WMS `OrderMerger`中，toMap()与合并函数组合数量处理重复订单导入。

---

**Q90. flatMap和map在返回类型方面有什么区别？**

**难度：** 中级

**答案：** map函数接受Function<T, R>并返回Stream<R>，其中每个输入元素正好产生一个输出元素。flatMap接受Function<T, Stream<R>>并返回Stream<R>，将结果流平展为单个流。如果map与返回集合的函数一起使用，你得到的是集合流而不是单个元素的流。flatMap正是为了处理每个输入产生多个输出的函数而存在，将所有输出流合并为一个。"平展"步骤消除了嵌套流结构。理解这个区别对于正确转换流数据至关重要。

**追问方向：** 如何使用flatMap从WMS订单中提取所有属性？
**WMS Reference:** 在WMS `AttributeAggregator`中，flatMap将所有行项目属性提取到单个流中进行处理。

---

## 其他Lambda问题

**Q91. 什么是闭包？它与Java lambda有什么关系？**

**难度：** 高级

**答案：** 闭包是一个函数，它捕获并保留对其封闭作用域变量的访问。在Java中，lambda是闭包，因为它们可以引用周围作用域的有效最终变量。lambda"关闭"那些变量，保留在lambda创建时的它们的值。与某些语言中闭包通过引用捕获变量不同，Java lambda捕获有效最终变量的值（相当于按值捕获）。这种设计避免了许多并发问题，因为捕获的值不能改变。闭包支持回调和策略对象等强大模式，同时保持Java的强类型和安全。

**追问方向：** 如果在lambda创建后修改被捕获的变量会发生什么？
**WMS Reference:** 在WMS `AsyncOrderProcessor`中，理解闭包有助于预测提交任务中捕获的变量状态。

---

**Q92. lambda的早期求值和惰性求值有什么区别？**

**难度：** 高级

**答案：** 早期求值（急切求值）意味着表达式在定义点计算，而惰性求值意味着计算推迟到实际需要结果时。Java lambda在使用函数式接口时是惰性求值的；lambda主体直到函数式接口方法被调用时才执行。这允许创建无限序列、短路和在执行前组合函数。流通过中间操作不执行直到终端操作触发管道的惰性求值来实现这种优化。惰性求值支持短路和避免不必要的计算的优化，但需要理解求值何时实际发生。

**追问方向：** 惰性求值如何实现无限流的处理？
**WMS Reference:** 在WMS `OrderGenerator`中，惰性求值允许对订单序列建模，而不预先实现所有订单。

---

**Q93. 如何有效调试lambda表达式？**

**难度：** 高级

**答案：** 调试lambda具有挑战性，因为它们在传统调试中没有自己的堆栈帧；lambda主体在调用方法的上下文中执行。IDE可能在调试期间内联显示lambda表达式。lambda内部的断点起作用，但可能不明显，因为lambda不显示为命名方法。通过将lambda分配给具有描述性名称的变量来命名lambda有助于。使用流管道中的peek()观察中间流的元素。在lambda内部记录可以揭示执行流程。编译字节码使用invokedynamic，但现代调试器处理得相当好。对于复杂的lambda逻辑，提取到命名方法引用可以提高可调试性。

**追问方向：** 如何在WMS中跟踪复杂流管道的执行？
**WMS Reference:** 在WMS `StreamDiagnostics`中，临时添加peek()操作有助于跟踪流管道行为。

---

## 其他异常问题

**Q94. 什么是断言？它与异常处理有什么区别？**

**难度：** 中级

**答案：** 断言是可在编译时检查的布尔表达式，在开发期间验证程序不变量，在运行时用-ea标志启用。它们使用assert关键字并在为假时抛出AssertionError（unchecked）。断言用于在开发期间捕获程序员错误，而不是用于运行时错误处理，而异常处理可恢复或不可预测的条件。断言可以在生产中禁用，所以逻辑永远不应该依赖它们。常见用途包括检查前提条件、后置条件和应该始终持有的不变量。与向调用者发出信号预期错误条件的异常不同，断言文档化应该始终为真的假设。

**追问方向：** 何时会在WMS验证中使用断言而不是抛出异常？
**WMS Reference:** 在WMS `InternalConsistencyChecker`中，断言在开发期间验证应该始终持有的内部状态。

---

**Q95. Java异常处理中的多catch特性是什么？**

**难度：** 中级

**答案：** 多catch（Java 7+）允许在单个catch子句中捕获多个不相关的异常类型，减少处理来自不同来源的类似异常时的代码重复。语法：`catch (IOException | SQLException ex) { log(ex); }`。被捕获的异常隐式为final，因此变量不能在块内重新赋值。当不同异常需要相同处理时，多catch很有用。编译器将其视为处理多种类型的合成catch块。多catch不适用于一个 是另一个子类的相关异常。此功能在重构常见异常处理模式时减少了样板。

**追问方向：** 多catch如何帮助减少WMS中重复的错误处理代码？
**WMS Reference:** 在WMS `DataImportHandler`中，多catch使用通用错误报告处理各种导入异常。

---

**Q96. 什么是异常传播？它如何工作？**

**难度：** 初级

**答案：** 异常传播是未捕获的异常从发生的方法向上移动到其调用者，然后到调用者的调用者，直到被捕获或到达栈顶的机制。每个方法可以处理异常或在throws子句中声明它让它传播。Checked异常必须被捕获或在throws中声明；unchecked异常自动传播。传播涉及展开堆栈，这是相对昂贵的。理解传播对于在有意义操作或恢复可以发生的适当级别放置异常处理至关重要。

**追问方向：** 异常传播如何影响WMS事务边界？
**WMS Reference:** 在WMS `TransactionInterceptor`中，异常传播决定异常被捕获的位置决定事务是提交还是回滚。

---

**Q97. 什么是自定义异常？何时应该创建它们？**

**难度：** 中级

**答案：** 自定义异常是用户定义的异常类，扩展Exception（checked）或RuntimeException（unchecked），提供特定领域的错误类型。创建它们来表示现有异常不能充分描述的特定领域错误条件。自定义异常提高代码可读性，实现有针对性的捕获，并可以通过字段和方法携带领域上下文。它们应该有反映错误条件的有意义的名称，为常见错误上下文提供构造函数，并重写toString()以获得清晰的消息。在WMS中，示例包括OrderNotFoundException、InsufficientInventoryException和LocationOccupiedException。当标准异常能充分描述条件时，避免创建自定义异常。

**追问方向：** 如何为不同错误类别设计WMS异常层次结构？
**WMS Reference:** 在WMS `ExceptionArchitecture`中，具有订单、库存和位置错误的子类WMSException基类提供分类。

---

**Q98. Checked异常和函数式接口之间有什么关系？**

**难度：** 高级

**答案：** 函数式接口（lambda）不能抛出checked异常；lambda的目标类型不声明任何checked异常。当lambda主体执行可能抛出checked异常的操作时，这会造成问题。解决方法包括在lambda内部捕获和处理，将checked异常包装在RuntimeException中，或使用转换异常的实用方法。一些函数式编程库提供CheckedFunction等检查函数式接口。当在lambda中使用WMS文件或数据库操作时，checked异常必须适当处理。

**追问方向：** 如何在流map操作中处理checked IOException？
**WMS Reference:** 在WMS `OrderFileParser`中，将checked异常包装在自定义RuntimeException中允许它们出现在lambda体中。

---

## 其他Object方法问题

**Q99. hashCode契约是什么？为什么它对HashSet和HashMap重要？**

**难度：** 中级

**答案：** hashCode契约要求在同一程序执行期间，相等对象必须有相等的hashCode，不相等对象的hashCode不必不同。对于基于哈希的集合，相等的hashCode意味着对象可能进入同一个桶，需要equals()来区分它们。违反契约会导致集合失败：对象可能找不到即使存在，重复检测失败，或条目显示丢失。该契约使哈希集合能够首先通过hashCode检查桶然后equals()有效地定位元素。需要一致性：对象的hashCode在其在哈希集合中时不应改变。使用不可变字段作为hashCode源保持稳定性。

**追问方向：** 为什么hashCode()使用可变字段会有问题？
**WMS Reference:** 在WMS `OrderKey`中，对OrderId用于hashCode可以防止如果Order用于HashSet则发生损坏。

---

**Q100. 具有多个字段的类的equals/hashCode实现模式是什么？**

**难度：** 高级

**答案：** 对于具有多个字段的类，通过首先检查同一性（this == o）实现equals()，然后类型兼容性（getClass()对比 instanceof），然后使用Objects.equals()比较每个重要字段。对于hashCode()，使用equals()中使用的所有字段（通常是相同集合）调用Objects.hash()，对于手动实现使用31作为乘数。字段应以一致顺序比较。浮点字段使用Float.compare()和Double.compare()。避免比较非重要字段。IDE或lombok @EqualsAndHashCode生成的实现遵循此模式。equals和hashCode之间的一致性至关重要。

**追问方向：** 如何为WMS Location类实现equals和hashCode？
**WMS Reference:** 在WMS `Location`中，equals()比较warehouseId、zone、aisle、rack和shelf；hashCode()使用相同字段。

---

**Q101. 解释finalize()方法及其使用为何不推荐。**

**难度：** 高级

**答案：** finalize()在垃圾回收器回收对象内存之前调用，旨在清理原生资源。其使用不推荐是因为GC时机不确定，finalize()可能根本不运行，它可以复活对象，并为所有对象增加开销。资源应该通过try-with-resources或显式close()方法管理。如果重写了finalize()，对象在finalize()完成之前不能被垃圾回收。在Java 9+，Cleaner和PhantomReference为事后清理提供了更好的替代方案。在子类/超类场景中，finalize()顺序也不能保证。

**追问方向：** 现代Java中资源清理有什么替代方案？
**WMS Reference:** 在WMS `NativeResourceManager`中，try-with-resources和显式close()方法替换了用于数据库和文件句柄的finalize()。

---

## 更多集合框架问题

**Q102. Arrays.asList()和List.of()有什么区别？**

**难度：** 中级

**答案：** Arrays.asList()返回一个由原始数组支持固定大小的列表，其中修改影响原始数组，反之亦然；像add或remove这样的结构性修改抛出UnsupportedOperationException。List.of()（Java 9+）创建一个与任何数组断开连接的真正不可变列表，拒绝null元素和任何结构性修改。Arrays.asList()允许通过set()进行元素更新，变更反映在底层数组中。List.of()为不可变列表创建进行了优化，具有更好的内存特性。List.of()如果任何元素为null则抛出NullPointerException，而Arrays.asList()接受null。根据需要可变性还是不可变性进行选择。

**追问方向：** 从Arrays.asList()创建List后你能修改数组吗？
**WMS Reference:** 在WMS `SkuArrayWrapper`中，理解数组和列表之间的链接有助于避免意外修改。

---

**Q103. Java 8优化后HashMap的内部结构是什么？**

**难度：** 高级

**答案：** Java 8之后，HashMap使用数组桶进行存储的组合，当桶的链表超过8个节点的阈值时使用树节点（TreeMap条目）。该树是红黑树，提供O(log n)的最坏情况查找而不是O(n)。当移除节点且桶大小低于6时，树转换回链表。这种树化显著改善了当糟糕的哈希函数导致冲突时的最坏情况性能。哈希函数还包括一个补充哈希步骤，改善键hashCode()位的分布。阈值8是基于哈希碰撞概率分布选择的。

**追问方向：** 树化如何影响HashMap的迭代顺序？
**WMS Reference:** 在WMS `InventoryIndex`中，树化可能影响迭代顺序但保持O(log n)的最坏情况查找。

---

**Q104. HashSet和HashMap在实现方面有什么区别？**

**难度：** 中级

**答案：** HashSet内部使用HashMap，将一个虚拟对象作为所有条目的值，而集合元素成为map键。HashSet中的add()方法调用map.put(element, PRESENT)。PRESENT对象是一个共享的单例。此设计利用了HashMap经过验证的实现并提供O(1)的操作。remove()调用map.remove()。contains()调用map.containsKey()。此实现意味着HashSet不能有重复元素，因为HashMap键必须唯一。对HashSet的迭代实际上是对HashMap条目集的迭代。这是组合/委托模式的一个示例，其中HashSet重用HashMap的逻辑。

**追问方向：** 为什么HashSet的contains()具有O(1)平均复杂度？
**WMS Reference:** 在WMS `ProcessedOrderTracker`中，HashSet的O(1) contains()快速检查订单是否已被处理。

---

**Q105. List接口的subList()方法的目的是什么？**

**难度：** 中级

**答案：** subList()方法返回列表中指定fromIndex（包含）和toIndex（不包含）之间部分的视图。返回的列表由原始列表支持，意味着对子列表的修改反映在原始列表中，反之亦然。对子列表的结构性修改（add/remove）在父列表上检测到时会抛出ConcurrentModificationException。subList方法执行范围检查并为无效索引抛出IndexOutOfBoundsException。子列表可用于批量操作如clear()移除一个范围。这提供了一种无需复制即可处理列表部分的高效方式。

**追问方向：** 如何使用subList安全地处理WMS订单批次？
**WMS Reference:** 在WMS `BatchOrderProcessor`中，subList()为并行处理大型订单列表提供有效的视图。

---

**Q106. 什么是ConcurrentLinkedQueue？它与BlockingQueue有什么区别？**

**难度：** 高级

**答案：** ConcurrentLinkedQueue是一个无界的、线程安全的、基于链接节点的无锁队列，使用CAS（compare-and-swap）进行无需锁定的线程安全操作，为读写双方提供高并发性。BlockingQueue（如LinkedBlockingQueue）在空/取或满/放操作上阻塞等待线程，支持生产者-消费者模式，线程等待元素或空间。ConcurrentLinkedQueue不会阻塞；offer()立即返回。当消费者需要等待生产者时BlockingQueue合适；ConcurrentLinkedQueue适用于没有阻塞语义的高吞吐量场景。BlockingQueue可以是有界的而ConcurrentLinkedQueue始终是无界的。

**追问方向：** 何时在WMS中选择ConcurrentLinkedQueue而不是BlockingQueue？
**WMS Reference:** 在WMS `HighThroughputTaskQueue`中，ConcurrentLinkedQueue为任务分发提供更快的非阻塞访问。

---

**Q107. HashMap和TreeMap在性能和用例方面有什么区别？**

**难度：** 中级

**答案：** HashMap提供O(1)平均的get/put与无序迭代，而TreeMap提供O(log n)的操作与排序键顺序。HashMap可以处理任何键类型无需要求，而TreeMap需要Comparable键或Comparator。TreeMap通过subMap()、headMap()和tailMap()支持范围查询，并提供firstKey()、lastKey()和导航方法。TreeMap每个条目内存开销更高（红黑树节点vs哈希表条目）。当你需要快速查找且无顺序要求时使用HashMap。当你需要排序键或导航操作时使用TreeMap。HashMap通常更快且使用更少内存，除非需要排序。

**追问方向：** 如何使用TreeMap实现可导航的WMS位置目录？
**WMS Reference:** 在WMS `LocationBrowser`中，TreeMap的导航方法支持如"zone A中的所有位置"的范围查询。

---

## 更多Stream API问题

**Q108. 流中limit()和skip()有什么区别？**

**难度：** 初级

**答案：** limit(n)返回一个最多n个元素的流，在遇到前n个元素后截断。skip(n)丢弃前n个元素并返回剩余元素的流。两者都是可以组合的中间操作。在大于元素数量的流上skip()返回一个空流；limit()在较小流上返回所有元素。limit()经常与sorted()一起使用以获得前n个元素。skip()用于分页或移除标题行。两者仅在关联中间操作时才短路，而不是终端操作。

**追问方向：** 如何使用limit和skip对WMS订单结果进行分页？
**WMS Reference:** 在WMS `OrderPaginationService`中，skip((page-1)*pageSize).limit(pageSize)提供有效的服务器端分页。

---

**Q109. 流中的distinct()操作是什么？如何工作？**

**难度：** 中级

**答案：** distinct()返回一个移除重复元素的流，使用equals()确定相等性（对于没有正确equals()的对象使用身份）。它是一个有状态的中间操作，必须记住所有见过的元素以过滤重复项，需要额外内存。对于有序流，distinct()保留每个元素的第一个出现，丢弃后续重复项。对于并行流，元素跟踪更复杂，可能产生与顺序执行不同的结果。对于具有许多重复项的大型数据集，distinct()可能占用大量内存。它与正确实现的equals()和hashCode()一起正常工作。

**追问方向：** distinct()如何与并行流执行交互？
**WMS Reference:** 在WMS `UniqueSkuFinder`中，并行流上的distinct()必须首先收集结果，可能影响顺序。

---

**Q110. sorted()如何与流一起工作？有什么性能影响？**

**难度：** 中级

**答案：** 不带参数的sorted()使用自然顺序对元素排序，要求元素实现Comparable。sorted(Comparator<? super T> comparator)使用提供的比较器排序。sorted()是一个有状态的中间操作，必须在产生输出之前收集所有元素，需要O(n)的内存用于缓冲区。在顺序流中，它使用归并排序；在并行流中，它使用并行排序。排序具有O(n log n)时间复杂度。对于已知大小的流，使用sorted()与limit()获取前k个元素可能比使用专门算法效率更低。当不需要排序时，在并行sorted()之前使用unordered()可以提高性能。

**追问方向：** 如何高效地找到前10个紧急WMS订单？
**WMS Reference:** 在WMS `UrgentOrderMonitor`中，使用sorted()与limit(10)有效地检索最高优先级订单。

---

## 更多Lambda问题

**Q111. `() -> expression`和`() -> statement block`在lambda中有什么区别？**

**难度：** 初级

**答案：** 带表达式体的lambda（`() -> x + y`）不需要大括号，隐式返回表达式的值。带语句块的lambda（`() -> { int sum = x + y; return sum; }`）使用大括号，如果需要值则需要显式return语句。表达式lambda对于简单转换更简洁；语句块lambda需要多语句逻辑。没有return的语句块，如果函数式接口的方法是void的，如Consumer的accept()，则被视为void。混用大括号和隐式返回是不允许的。

**追问方向：** 何时会在WMS处理中需要语句块lambda？
**WMS Reference:** 在WMS `ComplexOrderTransformer`中，语句块lambda支持多步转换逻辑。

---

**Q112. 什么是构造函数引用？如何使用它们？**

**难度：** 中级

**答案：** 构造函数引用使用ClassName::new来引用构造函数，创建实例而无需显式lambda。数组构造函数引用使用String[]::new用于数组。它们是简单调用构造函数的lambda简写。例如，`ArrayList::new`创建对ArrayList无参构造函数的构造函数引用。对于泛型构造函数，类型从上下文推断。构造函数引用对工厂模式和需要创建对象的流操作很有用，如stream.map(Order::new)。编译器根据函数式接口的返回类型和上下文选择适当的构造函数。

**追问方向：** 如何使用构造函数引用将订单DTO列表转换为Order实体？
**WMS Reference:** 在WMS `OrderMapper`中，stream.map(OrderDTO::new)有效地将DTO转换为域对象。

---

## 更多泛型问题

**Q113. `<?>`和`<Object>`在泛型中有什么区别？**

**难度：** 高级

**答案：** `<?>`表示未知类型并通过通配符捕获提供类型安全；你只能将元素读取为Object，除了null之外不能写入任何内容。`<Object>`明确使用Object作为类型参数，意味着你可以将元素读取为Object并写入任何Object派生类型。然而，`List<Object>`与`List<?>`不同，因为`List<Object>`可以接受任何对象但`List<?>`不能接受任何内容（除了null）。关键区别是`List<Object>`可以传递给接受`List<Object>`的方法，而`List<?>`不能。当需要具有只读集合的最大灵活性时使用`List<?>`；仅在你特别需要Object语义时使用`List<Object>`。

**追问方向：** 为什么你不能将String添加到`List<?>`但可以添加到`List<Object>`？
**WMS Reference:** 在WMS `HeterogeneousCollectionProcessor`中，理解差异有助于设计灵活的处理方法。

---

**Q114. 编译器为泛型类生成的桥方法是什么？**

**难度：** 高级

**答案：** 桥方法是编译器生成的综合方法，用于在类型擦除后保持二进制兼容性。当泛型类扩展或实现参数化接口时，编译器生成桥方法以保留泛型类型信息的外观。例如，如果类Node<T>有T getValue()，实现类NodeString生成一个桥方法Object getValue()，将结果强制转换为String。桥方法在编译时具有擦除的签名但委托给实际方法。它们仅存在于字节码中，不在源代码中。此机制允许非泛型遗留代码与泛型代码无缝交互。

**追问方向：** 桥方法如何影响调试或反射？
**WMS Reference:** 在WMS `ReflectionUtilities`中，桥方法可能出现在方法列表中，需要过滤以进行准确的 方法发现。

---

## 更多异常问题

**Q115. 记录异常与抛出异常的最佳实践是什么？**

**难度：** 高级

**答案：** 最佳实践是在系统边界（入口/出口点）记录异常，而不是记录和抛出相同的异常。每个日志条目应包含有关操作的信息，但只有一个地方应该处理异常。如果捕获并重新抛出，在重新抛出之前记录或使用唯一消息以避免重复日志。使用适当的日志级别：ERROR用于真正错误，WARN用于可恢复问题。永远不要记录密码等敏感信息。抛出后记录是无效的。在分层系统中，较低层应该抛出异常；上层应该决定是否记录以及如何向用户呈现错误。

**追问方向：** 如何在WMS应用程序中实现集中式异常处理？
**WMS Reference:** 在WMS `ExceptionLoggingAspect`中，在服务边界记录确保一致的错误记录而不重复。

---

**Q116. Exception和RuntimeException在设计理念方面有什么区别？**

**难度：** 中级

**答案：** Exception（checked）代表编写良好的应用程序应该预见并处理的状况，反映调用者被期望采取纠正措施的可恢复错误。RuntimeException（unchecked）代表应该修复的编程错误，如bug、无效输入或契约违反。Checked异常在编译时强制处理，鼓励健壮性但增加样板。Unchecked异常减少仪式但可能传播意外错误。有些人认为checked异常是一个失败的实验；现代语言通常更喜欢unchecked。选择反映条件的可恢复程度。在领域驱动设计中，业务规则违反通常是RuntimeException。

**追问方向：** 何时业务规则违反在WMS中是RuntimeException？
**WMS Reference:** 在WMS `BusinessRuleValidator`中，InsufficientStockException作为RuntimeException表示前置条件检查中的编程错误。

---

## 更多Object方法问题

**Q117. 为什么应该为域对象重写toString()？**

**难度：** 初级

**答案：** 为域对象重写toString()提供超出默认ClassName@hashCode的有意义的表示。好的toString()实现包含所有相关字段，使调试更容易，日志更可读。IDE可以自动生成toString()实现。它在调试时检查对象时帮助开发。对于集合，toString()递归对元素调用toString()，所以在域对象上有适当的toString()可以改善调试输出。toString()被字符串连接和格式方法隐式调用。对于WMS域对象如Order、SKU和Location，好的toString()实现对开发和生产调试都很重要。

**追问方向：** WMS Order对象的toString()应该包含哪些字段？
**WMS Reference:** 在WMS `Order.toString()`中，包含orderId、status和lineItemCount提供有用的调试信息。

---

**Q118. String上的equalsIgnoreCase()方法是什么？它与equals()有什么区别？**

**难度：** 初级

**答案：** equalsIgnoreCase()比较两个字符串，不考虑大小写，将'A'和'a'视为相等。它为在非ASCII区域设置中大小写不同的少数字符适当处理特定于区域设置的大小写。equals()执行区分大小写的比较，其中'A'和'a'是不同的。两个方法都安全地处理null（不会对null接收者抛出NPE），但如果与null字符串比较会抛出NPE，因为它在参数上调用equals()。equalsIgnoreCase()用于不区分大小写的用户输入比较，如状态码或代码。对于一般字符串比较，equals()通常是正确的。

**追问方向：** 何时会在WMS验证中使用equalsIgnoreCase()？
**WMS Reference:** 在WMS `StatusValidator`中，使用equalsIgnoreCase()比较订单状态码可以适应输入的大小写变化。

---

## 高级问题

**Q119. 什么是双括号初始化习语？为什么它被认为是一种反模式？**

**难度：** 高级

**答案：** 双括号初始化使用`new ArrayList<>() {{ add(item); }}`，其中外层大括号创建一个匿名子类，内层大括号初始化实例字段。它为数据提供简洁的集合初始化，但每次使用创建一个新类文件，如果内部类捕获引用会导致内存泄漏，破坏序列化，并有性能开销。该习语在Java 7之前没有更好替代方案时出现。现代Java有不会产生匿名类的更好替代方案。初始化块可能无意中捕获外部类引用。

**追问方向：** 什么现代替代方案替换双括号初始化？
**WMS Reference:** 在WMS `InitialDataLoader`中，List.of()和Map.of()提供清洁的不可变集合初始化，没有反模式问题。

---

**Q120. Java类型系统如何处理泛型中的原始类型与包装类？**

**难度：** 高级

**答案：** Java泛型仅适用于引用类型，不适用于原始类型，因为类型擦除将类型参数替换为Object或其边界。原始类型在泛型上下文中使用时会自动装箱到包装类，如List<int>在运行时变为List<Integer。这种装箱对于原始类型密集型操作有性能开销。Java在java.util.concurrent中为原始类型提供专门集合，库如Trove提供原始类型集合实现。自动装箱也可能导致微妙错误，其中==比较在Integer缓存范围外的包装实例之间失败。在具有泛型算法的直接原始类型使用时需要不同方法。

**追问方向：** 在具有大型数据集的WMS库存计算中装箱的性能影响是什么？
**WMS Reference:** 在WMS `HighVolumeInventoryCounter`中，使用原始数组或专门集合避免装箱开销。

---

**Q121. 函数式接口中方法引用类型的含义是什么？**

**难度：** 高级

**答案：** 有四种方法引用：静态（Integer::parseInt）、绑定实例（order::getId）、非绑定实例（String::length）和构造函数（ArrayList::new）。绑定方法引用在创建时捕获实例，对闭包有用。非绑定引用在调用函数式接口方法之前不捕获任何东西，将实例作为第一个参数接收。静态引用不捕获实例。方法引用的签名必须与函数式接口的单一抽象方法签名匹配。方法引用不能直接表示varargs方法，需要lambda等价物。理解捕获机制有助于预测lambda/方法引用行为。

**追问方向：** 绑定方法引用如何以不同于非绑定方式捕获其实例？
**WMS Reference:** 在WMS `OrderProcessor`中，绑定方法引用到特定订单实例作为回调很有用，这些回调始终在该订单上操作。

---

**Q122. Stream的flatMap和Optional的flatMap之间有什么关系？**

**难度：** 高级

**答案：** 两种flatMap操作都处理嵌套容器，平展一级。Stream的flatMap将每个元素转换为一个流并将所有流平展为一个。Optional的flatMap将值转换为一个Optional并平展嵌套的Optional，避免Optional<Optional<T>>。两者都避免了嵌套转换导致的金字塔形代码。Optional flatMap用于转换可能不产生值的情况，你想链式操作而不需要显式null检查或Optional包装。语义是平行的：不是"对每个元素，产生多个元素"，而是"如果值存在，转换它，否则返回空"。

**追问方向：** 何时会在WMS代码中链式调用Optional flatMap操作？
**WMS Reference:** 在WMS `OrderFinder`中，optional.flatMap(Order::getCustomer).flatMap(Customer::getAddress)安全地链式可选检索。

---

**Q123. ConcurrentHashMap的forEach、search和reduce操作有什么区别？**

**难度：** 高级

**答案：** 这些是ConcurrentHashMap和ConcurrentMap特有的并发归约操作。forEach(transform)对每个键值对应用函数用于副作用。search(remappingFunction)应用函数直到找到非空结果，为效率而短路。reduce(reducer)使用提供的二元运算符组合所有元素。每个都有变体，接受并行阈值以确定何时使用顺序vs并行处理。与流操作不同，这些直接对并发映射进行操作，不创建中间对象。它们在许多情况下为以Map为中心的操作提供比流管道更好的性能。

**追问方向：** 如何使用reduce并发找到所有WMS位置的总库存？
**WMS Reference:** 在WMS `ConcurrentInventoryAggregator`中，reduce()有效地组合库存计数，没有流开销。

---

**Q124. Java中StampedLock的意义是什么？它与ReentrantReadWriteLock如何比较？**

**难度：** 高级

**答案：** StampedLock（Java 8+）除了悲观的读和写锁外，还提供乐观读锁作为第三种模式。它在锁定获取时返回一个戳，必须在解锁时验证。乐观读允许在没有写入发生时进行读访问，使用validate()检查读是否有效。这在读频繁且争用适中的情况下可能比ReadWriteLock更快。StampedLock不像ReentrantReadWriteLock那样支持条件变量。它在内部使用unsafe操作。StampedLock是不可重入的；同一线程再次获取写锁会导致死锁。它更复杂，正确使用可以 为某些读密集型工作负载提供更好的吞吐量。

**追问方向：** StampedLock的乐观读何时会使WMS并发读取受益？
**WMS Reference:** 在WMS `ReadHeavyInventoryService`中，StampedLock的乐观读可以提高频繁库存查询的吞吐量。

---

DONE_PART1
