# Java LinkedHashMap

LinkedHashMap 继承自 HashMap，在其之上**额外维护了一条贯穿所有节点的双向链表**，从而让迭代顺序可预测：默认按**插入顺序**，也可配置为按**访问顺序**（access-order）。它是 **LRU 缓存的标准实现基础**。

## 底层结构

```java
public class LinkedHashMap<K,V> extends HashMap<K,V>
        implements Map<K,V> {

    transient LinkedHashMap.Entry<K,V> head;   // 链表头（最老）
    transient LinkedHashMap.Entry<K,V> tail;   // 链表尾（最新）
    final boolean accessOrder;                 // false=插入序，true=访问序
}
```

- 节点 `LinkedHashMap.Entry extends HashMap.Node`，**多两个指针 `before`/`after`**——每个 entry 比 HashMap.Node 多约 8~16B（压缩指针下 2 个引用）。
- HashMap 的数组桶结构完全保留（查找仍是 O(1)），链表只是**额外的迭代顺序索引**，不参与定位。
- 插入时：`newNode` 会同时接进桶（hash 结构）与链表尾部（顺序结构）。
- 迭代 `keySet/entrySet/values` 走链表 → 顺序稳定、可预期。

```
桶数组（HashMap 部分，定位 O(1)）：
  [0] -> null
  [1] -> A -> C
  [2] -> B
  ...

顺序链表（LinkedHashMap 部分）：
  head <-> A <-> B <-> C <-> tail      （插入序：A, B, C）
```

## 两种顺序模式

```java
// 1. 插入顺序（默认）：迭代顺序 = key 首次插入的顺序
Map<String, Integer> m1 = new LinkedHashMap<>();

// 2. 访问顺序：每次 get/put 会把该 entry 移到链表尾部
Map<String, Integer> m2 = new LinkedHashMap<>(16, 0.75f, true);
```

- 插入序：`afterNodeAccessed` 不重排，仅尾插新节点；删除中间节点时链表前后衔接。
- 访问序（`accessOrder = true`）：`get`、`getOrDefault`、`put`、`putIfAbsent` 等命中已有 key 时，把节点移到尾部——**链表从 head 到 tail 是"最久未访问 → 最近访问"**。
- 注意：`get` 对访问序模式而言是**结构性语义上的重排**（但 JDK 实现中 `afterNodeAccessed` 不增加 `modCount`，因此单纯 get 重排**不会**触发迭代器 CME——迭代器在 `modCount` 层面与 HashMap 相同）。

## LRU 缓存实现（面试高频手写题）

利用 `removeEldestEntry` 钩子，超过容量时自动淘汰最久未访问的 entry：

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);   // accessOrder = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;       // 超容量即淘汰链表头（最久未访问）
    }
}

// 用法
LRUCache<String, Integer> cache = new LRUCache<>(100);
cache.put("a", 1);
cache.get("a");          // "a" 移到尾部（最近使用）
cache.put("b", 2);       // ... 超过 100 时自动移除最久未访问的 head
```

要点：

- `removeEldestEntry` 在**每次 `put/putAll` 之后**被调用，默认返回 `false`；重写为 `size() > capacity` 即可。
- 淘汰的是链表**头部**（最久未访问），因为访问序把新访问的挪到了尾部。
- 该类非线程安全；并发 LRU 需 `Collections.synchronizedMap` 包装或自己用锁/`ReentrantReadWriteLock`，或改用 `LinkedHashMap` + 显式同步、Caffeine 等现代缓存库。

**面试追问：LinkedHashMap 与 HashMap 内存/性能差多少？**

每个节点多 2 个引用（before/after）+ 2 个链表端点对象；链表维护是常数时间。查找/插入复杂度与 HashMap 相同（O(1) 平均），**常数略大、内存略高**。当需要可预测顺序或 LRU 时，这点开销是值得的；否则直接用 HashMap。

## 与 HashMap 的差异汇总

| 维度 | HashMap | LinkedHashMap |
| ---- | ---- | ---- |
| 迭代顺序 | 不保证（随 hash/容量变化） | 插入序或访问序，稳定 |
| 额外结构 | 无 | 双向链表 |
| 每节点开销 | Node（hash/key/value/next） | + before/after |
| get 是否重排 | 否 | accessOrder=true 时是 |
| LRU | 需自己实现 | `removeEldestEntry` 天然支持 |
| fail-fast | 是 | 是（modCount 语义相同） |

## 使用注意

- 需要**稳定顺序的去重 Map/Set**：`LinkedHashMap` / `LinkedHashSet`。
- 遍历中删除同样用迭代器 `remove()`，否则 CME。
- 线程不安全；`Collections.synchronizedMap` 包装后迭代仍需手动加锁（或用 `ConcurrentHashMap` 放弃顺序）。
- 序列化：链表结构不单独序列化，反序列化按写入顺序重建（JDK 会调 `afterNodeInsertion` 等回调恢复语义——读源码时注意 `eldest` 判断在反序列化后不会再次触发淘汰逻辑）。

## 【面试追问链】

1. **LRU 为什么用访问序而不是插入序？** → LRU 定义是"最久未被访问的先淘汰"；只有访问序能在每次 get 后把热点挪到尾部，让头部始终是冷数据。插入序只能实现 FIFO。
2. **访问序下反复 get 会把 map 变成"有序"吗？** → 会把链表重排，但**不改 modCount**，迭代器不抛 CME——这是与"结构性修改"的重要区别，面试可用来区分 fail-fast 的触发条件。
3. **LinkedHashMap 的查找为什么还是 O(1)？** → 链表只用于维护顺序，定位仍走 HashMap 的桶数组；链表不参与 get 的查找路径。
4. **如何实现线程安全的 LRU？** → 外层加 `synchronized`（锁整个缓存）、读写锁（读多写少但 LRU 的 get 会重排，写锁粒度难细）、或直接用 Caffeine/Guava Cache（内部用 W-TinyLFU + 分段锁，性能远超简单 LRU）。
5. **LinkedHashMap 的 removeEldestEntry 会在 get 时触发吗？** → 不会，只在 `put/putAll` 后调用；纯读不会淘汰。
6. **Guava/Caffeine 为什么不用纯 LRU？** → 纯 LRU 会被一次性扫描（全表扫一遍）污染缓存，把热点全部挤出；Caffeine 用 TinyLFU（频率近似）做准入与淘汰，命中率显著更高——这是"学通"层面的延伸。
