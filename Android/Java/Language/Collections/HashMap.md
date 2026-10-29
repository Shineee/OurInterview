# Java HashMap

HashMap 是 Java 集合的绝对核心。面试几乎必考：底层结构、hash 扰动、put/get 流程、JDK 7 与 JDK 8 的差异、树化与扩容。本页按 JDK 8（当前主流）为主线讲解，并对比 JDK 7 的关键行为。

## 底层结构（JDK 8）

```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, java.io.Serializable {

    transient Node<K,V>[] table;   // 桶数组
    transient int size;
    transient int modCount;
    int threshold;                 // 扩容阈值 = capacity * loadFactor
    final float loadFactor;        // 负载因子，默认 0.75
}
```

- 数组 + 链表 + 红黑树：每个桶（bucket）是一条链表；**当链表长度 ≥ 8 且数组长度 ≥ 64 时，链表树化为红黑树**（TreeNode，继承自 LinkedHashMap.Entry）。
- 若树化条件中数组长度不足 64，优先**扩容**而不是树化（避免小数组上过早树化的空间浪费）。
- 树节点在元素较少时会**退化回链表**（链表长度 ≤ 6 时拆树，中间带滞回区间防止频繁转换）。
- 默认初始容量 **16**，负载因子 **0.75**，扩容翻倍。

**为什么阈值是 8？** 源码注释解释：理想哈希下链表长度近似泊松分布，长度达到 8 的概率极低（约千万分之六），8 是"几乎不会发生"与"树化开销可接受"之间的折中。64 的最小容量约束则是因为小数组上哈希冲突本就密集，树化不解决问题、扩容才有效。

**为什么负载因子 0.75？** 空间与时间的折中：0.5 空间浪费多但冲突少；1.0 空间省但链表变长。0.75 下平均链长接近常数，同时把扩容频率控制在合理范围。

## hash 扰动与容量为 2^n

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 定位桶
n - 1 & hash   // n 为数组长度（2^n）
```

- **高位参与低位运算**：`hashCode` 的高 16 位异或到低 16 位，让高位的变化也能影响下标分布，减少因低位相同造成的冲突（`^ (h >>> 16)` 即"扰动函数"）。
- **容量必须是 2 的幂**，这样取模可用位与 `& (n-1)` 替代 `%`，一条指令完成且等价于取模。
- JDK 8 进一步让 `TreeNode` 在树内用 `hash` 的高位比较辅助分裂，扰动效果更好。

**面试追问：HashMap 允许 null 吗？为什么？**

允许**一个 null key**（hash 为 0，落在 0 号桶），允许任意多个 null value。这是为兼容 `get(key)==null` 判断"不存在"的写法而设计的代价——也正是 ConcurrentHashMap 不允许 null 的原因（并发下无法区分"值为 null"与"键不存在"）。TreeMap（自然排序）不允许 null key，因为 `compareTo(null)` 会 NPE。

## put 流程（JDK 8）

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

1. `table` 为空或长度 0 → `resize()` 初始化（默认 16）。
2. 计算 `hash`，定位桶 `i = (n-1) & hash`。
3. 桶空 → 直接放入新节点（尾插，见下）。
4. 桶首节点 key 相同（`==` 或 `equals`）→ 覆盖 `value`，返回旧值。
5. 首节点是 TreeNode → 走红黑树的 `putTreeVal`。
6. 否则遍历链表尾插（JDK 8 用 `tail.next = newNode` 尾插法）；若插入后**长度 > 8**，调用 `treeifyBin`——**数组长度 < 64 则先扩容，否则树化**。
7. 插入成功 `modCount++`，`size++`；若 `size > threshold` → `resize()` 扩容。

**equals 与 hashCode 的契约**：`equals` 相等的两个对象 `hashCode` 必须相等；否则会出现"逻辑上相等的键存了两份、get 取不到"的经典 Bug。重写 `equals` 必须重写 `hashCode`。

## get 流程

1. `key == null` → 直接查 0 号桶。
2. 定位桶，比较首节点；命中返回。
3. 首节点是树节点 → `getTreeNode` 按 hash + key 查找。
4. 否则沿链表 `next` 遍历，`==` 或 `equals` 匹配。

## 扩容 resize 与 rehash（JDK 8 优化）

- 新容量 = 旧容量 × 2，新阈值 = 新容量 × loadFactor。
- **关键优化**：因为容量是 2^n，元素要么留在原下标 `j`，要么移到 `j + oldCap`——**只看 hash 的新增高位（第 32 位方向的那一位）即可判断，不需要重新计算 hash**。JDK 7 则需要 `indexFor(hash, len)` 重新取模，且用头插法。
- JDK 8 尾插法保证扩容后链表保持原相对顺序，**避免了 JDK 7 多线程扩容形成环形链表导致 CPU 100% 的经典死循环**（JDK 7 死链原理：头插法在并发 rehash 下两条链表交错回指，下次 get 遍历成环）。
- 注意：**HashMap 扩容不是线程安全的**——JDK 8 下并发 put 仍可能丢数据、size 不一致，只是不再出现死链。并发场景必须用 ConcurrentHashMap。

**面试追问：能否预设容量避免频繁扩容？**

可以。`new HashMap<>(100)` 会把 `threshold` 调整为**大于 100 的最小 2 的幂 × loadFactor 对应的实际容量**（内部 `tableSizeFor` 上取整到 2^n）。经验公式：`expectedSize / 0.75 + 1`，或 Guava 的 `Maps.newHashMapWithExpectedSize(n)`。批量 put 前指定容量可把扩容次数从 O(log n) 降到 0~1 次。

## remove / 遍历 / 迭代器

- `remove`：定位桶 → 树中 `removeTreeNode` 或链表中摘除节点 → `size--`。
- 遍历顺序**不保证**，且与容量、hash 分布有关；JDK 8 扩容迁移保持桶内顺序，但桶间顺序仍取决于下标。
- 迭代器是 **fail-fast**：迭代过程中发生结构性修改（`modCount != expectedModCount`）抛 `ConcurrentModificationException`。注意 `size` 变化但非结构性（如 `set` 已有 key 的 value）不改 `modCount`。
- 单线程下安全删除用 `it.remove()`；或 `forEach` + `it.remove()`；JDK 8+ 可用 `map.entrySet().removeIf(...)`。

## 线程不安全的表现

- �发并发 put：JDK 7 可能死链 + 数据覆盖丢失；JDK 8 数据丢失 + size 错乱。
- 并发扩容时 `transfer` 无同步，可能覆盖相邻桶。
- 迭代与写并发 → CME 或脏读。
- 解决：`ConcurrentHashMap`（见 [ConcurrentHashMap](../DataStructure/ConcurrentHashMap.md)）、`Collections.synchronizedMap`、或外部加锁。

## 内存占用估算（工程判断）

- 每个 `Node`：对象头 16B + hash/key 引用/value 引用/next 引用 4×4B（压缩指针）≈ 32B。
- 数组本身：`capacity × 4B`（压缩指针下）。
- 负载因子 0.75 下，n 个元素的数组约 `n/0.75 × 4B ≈ 5.3n B`，加节点 `32n B`，**每条目约 40B 左右**（不含 key/value 对象本身）；树化后每个 TreeNode 约 72B+，小规模数据别轻易树化。
- 结论：内存敏感场景（Android、大 Map）预设容量、用基本类型包装类做 key 时注意 Integer 缓存（-128~127 复用）。

## JDK 7 vs JDK 8 对比

| 维度 | JDK 7 | JDK 8 |
| ---- | ---- | ---- |
| 结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 插入 | 头插法 | 尾插法 |
| 扩容 | 重新 `indexFor` 取模 | 高位判断，原地/新桶二选一 |
| 并发扩容 | 可能形成环链 → CPU 100% | 无死链，但仍丢数据 |
| 扰动 | 4 次扰动（高位多次移位异或） | 高 16 位异或低 16 位 |
| 树化 | 无 | 长度≥8 且容量≥64 树化 |

## 与 LinkedHashMap / TreeMap / Hashtable 的关系

- **LinkedHashMap**：继承 HashMap，额外维护插入顺序（或访问顺序）的双向链表 → 用于 LRU 缓存。
- **TreeMap**：红黑树按 key 排序，`O(log n)`，支持 `subMap`/`firstKey` 等导航方法。
- **Hashtable**：JDK 1.0 遗留，方法级 synchronized，容量 11（不是 2^n）、扩容 2 倍+1，已被 ConcurrentHashMap 取代。
- **ConcurrentHashMap**：并发场景标准答案，见专页。

## 【面试追问链】

1. **为什么容量必须是 2 的幂？** → 位与代替取模、扩容只需判断一位新增 hash 位；代价是容量必须遵守 `tableSizeFor` 上取整规则。
2. **树化为什么还要求数组 ≥ 64？** → 小数组冲突是哈希分布问题，树化只掩盖症状；先扩容摊薄桶分布，树化是"链表真的太长"时的兜底。
3. **JDK 8 还会死链吗？数据还会丢吗？** → 不会死链（尾插 + 顺序迁移），但无同步的 put/扩容仍会丢数据、size 错乱——安全靠 ConcurrentHashMap，不靠 HashMap 升级。
4. **重写 equals 不重写 hashCode 会怎样？** → 两个 equals 的对象可能落在不同桶/被当作不同 key，get 取不到 put 的值，集合去重失效。
5. **put 一个已存在的 key，modCount 变吗？** → 不变。覆盖 value 是非结构性修改，不会触发迭代器 CME。
6. **为什么 ConcurrentHashMap 不允许 null key/value 而 HashMap 允许？** → 并发下 `get(key)==null` 无法区分"值为 null"与"键不存在"（需要两次查找/加锁确认），HashMap 单线程无此歧义。
7. **如何保证遍历顺序稳定？** → HashMap 本身不承诺；需要稳定顺序用 LinkedHashMap（插入序/访问序），需要排序用 TreeMap 或遍历后 `sort`。
