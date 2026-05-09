# Java 集合框架（速记）

> 集合是面试**必考**的部分，HashMap 和 ConcurrentHashMap 几乎是每场必问。这一篇按八股 + 系统讲解的混合风格写。
>
> hash 函数 / 红黑树 / JDK 7→8→17 演进 / CHM 的 CAS+synchronized / JDK 7 死循环复盘 + 9 个事故 → **[HashMap 与 ConcurrentHashMap 深入（深文）](/article/java/hashmap)**

---

## 1. 全景图

```
                        Iterable
                           │
              ┌────────────┼────────────┐
           Collection                  Map
           │      │     │            │      │
         List   Set  Queue        HashMap  TreeMap
                                    │
                          ConcurrentHashMap
```

| 接口 | 实现 | 特征 |
| --- | --- | --- |
| List | ArrayList、LinkedList、CopyOnWriteArrayList、Vector | 有序可重复 |
| Set | HashSet、LinkedHashSet、TreeSet、CopyOnWriteArraySet | 不重复 |
| Queue | ArrayDeque、PriorityQueue、LinkedList、ArrayBlockingQueue 等 | FIFO/优先级 |
| Map | HashMap、LinkedHashMap、TreeMap、Hashtable、ConcurrentHashMap | k-v |

---

## 2. ArrayList vs LinkedList

| 对比 | ArrayList | LinkedList |
| --- | --- | --- |
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头部插入 | O(n) | O(1) |
| 内存 | 连续 + 扩容拷贝 | 节点离散 |
| 默认容量 | 10 | — |
| 扩容策略 | 1.5 倍 | — |

> 90% 场景选 ArrayList。LinkedList 因缓存命中差，**实际表现远不如理论**。

### 2.1 ArrayList 扩容
```java
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = ArraysSupport.newLength(oldCapacity,
            minCapacity - oldCapacity, oldCapacity >> 1);  // 1.5 倍
    return elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### 2.2 fail-fast 机制
内部维护 `modCount`，迭代时与期望值不符立即 `ConcurrentModificationException`。
- 单线程也会触发：迭代过程中直接 `list.remove(o)`；
- 解决：用 `Iterator.remove()`、或 `CopyOnWriteArrayList`。

---

## 3. HashMap（必背）

### 3.1 数据结构
- **JDK 7**：数组 + 链表（头插法，多线程下可能成环）；
- **JDK 8+**：数组 + 链表 + **红黑树**（链表长度 ≥ 8 且数组长度 ≥ 64 时树化；< 6 退化）；
- 默认容量 16，负载因子 0.75，扩容 2 倍。

### 3.2 put 流程
```
1. 计算 hash = (h = key.hashCode()) ^ (h >>> 16)   // 高位异或低位
2. 定位下标 = hash & (n - 1)                        // n 是数组长度
3. 桶为空 → 直接放
4. 桶不空：
   - 与首节点 key 相等 → 替换
   - 是 TreeNode → 走红黑树
   - 链表 → 遍历，最后一个节点的 next = 新节点（尾插法）
   - 链表长度 ≥ 8 → 尝试树化
5. size > threshold（容量 × 0.75） → resize 2 倍
```

### 3.3 为什么容量是 2 的幂？
- `index = hash & (n-1)` 等价于 `hash % n`，但前者更快；
- 仅当 `n` 是 2 的幂，这两者才等价。

### 3.4 为什么扰动函数 `h ^ (h >>> 16)`？
- 高 16 位与低 16 位异或，让高位也参与索引计算；
- 显著降低哈希冲突。

### 3.5 为什么链表转树的阈值是 8？
泊松分布：负载因子 0.75 下，桶里 8 个元素的概率约 6×10⁻⁸。**几乎不可能** —— 真碰到说明哈希极差，需要树化。

### 3.6 HashMap 不是线程安全的
- JDK7 多线程并发 `resize` 会**形成环**，CPU 100%；
- JDK8 不会成环，但仍可能丢数据、出现 `ConcurrentModificationException`。

---

## 4. ConcurrentHashMap

### 4.1 JDK 7：分段锁（Segment）
- 数组里每个 Segment 是一把可重入锁；
- 默认 16 段，并发度最大 16；
- 已被淘汰。

### 4.2 JDK 8+：CAS + synchronized + 链表/红黑树
- 数据结构同 HashMap；
- **桶为空**：CAS 直接放；
- **桶不空**：`synchronized` 锁首节点，操作链表/红黑树；
- **扩容**：多线程协助迁移（`transfer` + `ForwardingNode`）；
- **size**：通过 `baseCount` + `counterCells` 累计（类似 LongAdder）。

> 锁粒度 = 单个桶，并发度 ≈ 桶数量，远超 JDK7 的 16。

### 4.3 不允许 null
HashMap 允许 key=null（放第 0 桶）、value=null。
**ConcurrentHashMap 都不允许** —— `get(k) == null` 在并发下无法区分"不存在"还是"存了 null"。

---

## 5. LinkedHashMap

- HashMap 的子类，额外维护一个**双向链表**串联所有 Entry；
- 支持两种顺序：
  - **插入顺序**（默认）；
  - **访问顺序**（`accessOrder=true`）→ 实现 LRU。

```java
// 简易 LRU
Map<K, V> lru = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> e) {
        return size() > 100;
    }
};
```

> Spring Boot 配置加载、JSON 序列化保证顺序时大量使用。

---

## 6. TreeMap

- 红黑树（自平衡 BST）；
- key 必须可比较（`Comparable` 或 `Comparator`）；
- 提供 `floorKey` / `ceilingKey` / `subMap` 等区间查询；
- 增删查 O(log n)。

应用场景：按时间戳查范围、按价格查区间、跳表的替代品。

---

## 7. CopyOnWriteArrayList

- **写时复制**：每次写都新建数组，写完替换引用（`volatile` 保证可见性）；
- 读无锁；
- 适合**读多写少且数据量小**（事件监听器列表、白名单）；
- 内存消耗大、写性能差，不要用于"频繁更新"。

---

## 8. 阻塞队列（重要）

| 队列 | 特征 |
| --- | --- |
| **ArrayBlockingQueue** | 数组实现，有界，单锁 |
| **LinkedBlockingQueue** | 链表实现，默认无界（坑：会 OOM），双锁分离 |
| **SynchronousQueue** | 容量为 0，put/take 必须配对 |
| **PriorityBlockingQueue** | 优先级堆，无界 |
| **DelayQueue** | 延迟队列，元素到期才能取 |
| **LinkedTransferQueue** | TransferQueue 实现，更高效的传递 |

> 线程池里 newFixedThreadPool 用 LinkedBlockingQueue **默认无界** —— 阿里规约禁用，要自己 new。

---

## 9. 性能对比速记

| 操作 | ArrayList | LinkedList | HashMap | TreeMap |
| --- | --- | --- | --- | --- |
| 随机访问 | O(1) | O(n) | O(1) avg | O(log n) |
| 末尾插入 | 摊还 O(1) | O(1) | O(1) | O(log n) |
| 头部插入 | O(n) | O(1) | — | — |
| 删除 | O(n) | O(1)（已知节点） | O(1) | O(log n) |

---

## 10. Stream API 速通

```java
List<Integer> result = users.stream()
    .filter(u -> u.getAge() > 18)
    .map(User::getId)
    .distinct()
    .sorted()
    .limit(100)
    .collect(Collectors.toList());

// 分组
Map<String, Long> byCity = users.stream()
    .collect(Collectors.groupingBy(User::getCity, Collectors.counting()));

// 并行流（CPU 密集 + 数据量大才有意义）
list.parallelStream().filter(...).count();
```

> 注意：`parallelStream` 默认用 `ForkJoinPool.commonPool`，与全局共享，避免阻塞它。

---

## 11. 高频面试题速记

1. **HashMap 1.7 / 1.8 区别？** 头插→尾插、链表→链表+红黑树、resize 不再成环。
2. **HashMap 为什么是 2 的幂？** 让 `hash & (n-1)` 等价于取模，且分布更均匀。
3. **HashMap 扰动函数？** `h ^ (h >>> 16)`，让高位参与下标计算。
4. **链表转红黑树的阈值？** 8（同时数组长度 ≥ 64）；退化 6。
5. **ConcurrentHashMap 怎么保证并发安全？** CAS + synchronized 锁桶头；扩容支持多线程协助；size 用 LongAdder 思路。
6. **CHM 为什么不允许 null？** `get` 返回 null 在并发下无法区分"无 key"还是"value=null"。
7. **怎么实现 LRU？** LinkedHashMap + accessOrder + 重写 removeEldestEntry。
8. **CopyOnWrite 适用场景？** 读多写少、数据量小。
9. **fail-fast 与 fail-safe 区别？** 前者立刻抛 CME（HashMap、ArrayList）；后者基于副本，不抛但可能读到旧数据（CopyOnWriteArrayList、ConcurrentHashMap）。

---

> 下一篇：[并发编程总览](/article/java/concurrent)
