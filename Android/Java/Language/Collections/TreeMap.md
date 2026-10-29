# Java TreeMap

TreeMap 是基于**红黑树**实现的排序 Map：key 有序存储，支持范围查询与导航。与 HashMap 的 O(1) 相比，它用 O(log n) 换来了**有序性**。

## 底层：红黑树

```java
public class TreeMap<K,V> extends AbstractMap<K,V>
        implements NavigableMap<K,V>, Cloneable, java.io.Serializable {

    private final Comparator<? super K> comparator;  // null 表示用 key 的自然顺序
    private transient Entry<K,V> root;               // 根节点
    private transient int size = 0;
}
```

- 红黑树是**自平衡二叉搜索树**：通过节点着色（红/黑）与旋转操作保持大致平衡，保证最坏情况下查找、插入、删除均为 **O(log n)**，不会退化成链表。
- 平衡的代价：每次插入/删除可能触发着色与旋转（最多 2~3 次旋转、O(1) 次重染色），常数比 HashMap 大。
- 节点 `Entry` 含 `key`、`value`、`left`、`right`、`parent`——每个节点比 HashMap.Node **多两个子引用 + 一个父引用**，内存开销显著更高。

**面试追问：为什么用红黑树而不是 AVL 树？**

AVL 树更严格平衡（左右子树高度差 ≤ 1），查找略快，但插入/删除的旋转更频繁。TreeMap 的典型负载是频繁增删 + 中等查找，红黑树"最坏 O(log n)、旋转更少"的折中更合适；HashMap 树化同样选红黑树而非 AVL，理由一致。

## 排序：自然序与 Comparator

```java
// 自然顺序：key 必须实现 Comparable
Map<String, Integer> m1 = new TreeMap<>();

// 自定义比较器
Map<Integer, String> m2 = new TreeMap<>(Comparator.reverseOrder());

// 按 key 长度排序
Map<String, String> m3 = new TreeMap<>(Comparator.comparingInt(String::length));
```

- `comparator == null` → 用 `((Comparable) key).compareTo(other)`，即**自然顺序**，此时 **key 不能为 null**（`compareTo(null)` 抛 NPE；实际上 TreeMap 即使有比较器也不允许 null key——JDK 中 `get(null)`/`put(null, ...)` 对多数实现会 NPE 或按约定禁止）。
- 插入时比较器判定 `compare == 0` 即视为**同一个 key**（覆盖 value）——判重语义来自比较器，**不走 equals/hashCode**。
- 著名坑：`BigDecimal("1.0")` 与 `BigDecimal("1.00")` 的 `compareTo == 0` 但 `equals == false`——TreeMap 会把它们当同一个 key，HashMap/HashSet 不会。**要求比较语义与 equals 一致时，比较器要自己保证。**
- 不允许 null key（无论自然序还是比较器），但**允许 null value**。

## NavigableMap：导航方法（一句话版）

TreeMap 实现了 `NavigableMap`，在 `SortedMap` 基础上提供方位导航：

```java
NavigableMap<Integer, String> nm = new TreeMap<>();
nm.put(1, "a"); nm.put(3, "c"); nm.put(5, "e");

nm.firstKey();                  // 1 —— 最小 key
nm.lastKey();                   // 5
nm.lowerKey(3);                 // 1  —— 严格小于 3 的最大 key
nm.higherKey(3);                // 5  —— 严格大于 3 的最小 key
nm.floorKey(3);                 // 3  —— 小于等于 3 的最大 key
nm.ceilingKey(4);               // 5  —— 大于等于 4 的最小 key
nm.subMap(1, true, 3, true);    // [1,3] 区间视图（含边界可选）
nm.headMap(3, false);           // < 3 的部分视图
nm.tailMap(3, true);            // >= 3 的部分视图
nm.descendingMap();             // 逆序视图
```

- `subMap/headMap/tailMap` 返回的是**视图**，写入会反映到原 TreeMap（与 `subList` 同类语义）；传入的 key 不在合法范围抛 `IllegalArgumentException`。
- 面试常考组合：用 `ceilingKey/floorKey` 做"最接近的前驱/后继"查找——这是 `TreeMap` 相对 HashMap 的独特能力。

## 复杂度与选型

| 操作 | TreeMap | HashMap |
| ---- | ---- | ---- |
| get/put/remove | O(log n) | O(1) 平均 |
| 有序遍历 | 是（中序遍历） | 否 |
| 范围查询 subMap | O(log n + k) | 不支持 |
| first/last/邻近 key | O(log n) | 需全量扫描 O(n) |
| 内存 | 高（每节点 3 个引用 + 着色位） | 低 |
| null key | 不允许 | 允许 1 个 |

**选型判断**：

- 需要按 key 有序、范围截取、找前驱后继 → TreeMap。
- 只要快速存取、不关心顺序 → HashMap。
- 需要并发有序 → `ConcurrentSkipListMap`（跳表，O(log n)，无锁实现）。

## 其他要点

- **fail-fast**：与 HashMap 一致，迭代中结构性修改抛 `ConcurrentModificationException`；`modCount` 在插入/删除时自增。
- 遍历顺序：`keySet()`/`entrySet()` 是**升序**（中序遍历）；`descendingKeySet()` 降序。
- `clone()` 是浅拷贝（比较器与节点结构共享，元素引用相同）；`putAll` 是逐个插入，不是批量 O(n)。
- 序列化按升序写出，反序列化重建树。
- Android 场景注意：TreeMap 单次操作常数与内存都高于 HashMap，大 Map 上优先 HashMap + 需要时再排序。

## 【面试追问链】

1. **TreeMap 判重靠 equals 还是 compareTo？** → 靠比较器/compareTo 返回 0；因此比较器与 equals 语义必须一致，否则与 HashSet/HashMap 的去重结果矛盾（BigDecimal 经典坑）。
2. **为什么不允许 null key？** → 自然序下 `compareTo(null)` 必然 NPE；比较器实现也通常不接受 null——与其行为不确定，不如直接禁止。允许 null value 是因为 null 不参与比较。
3. **红黑树最坏复杂度是多少？** → 查找/插入/删除最坏 O(log n)，靠"黑高平衡"保证（任意路径黑节点数相同、最长路径 ≤ 2×最短路径）。
4. **subMap 改动会影响原 TreeMap 吗？** → 会，返回的是视图；要独立副本需 `new TreeMap<>(nm.subMap(...))`。
5. **并发下要有序 Map 怎么办？** → `ConcurrentSkipListMap`：跳表结构、CAS 实现、O(log n)，支持并发有序导航；TreeMap 本身非线程安全（迭代器还是 fail-fast）。
6. **什么时候 TreeMap 反而比 HashMap 快？** → 大量 `firstKey/lastKey/floor/ceiling/subMap` 类有序查询时，HashMap 都要 O(n) 扫描；TreeMap 全部 O(log n)。
