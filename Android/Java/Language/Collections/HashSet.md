# Java HashSet

HashSet 是 Set 接口最常用的实现：**不允许重复元素、不保证顺序、允许一个 null 元素**。理解它的关键只有一句话——**它就是把元素作为 key 包在一个 HashMap 里**。

## 基于 HashMap 的实现要点

```java
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable {

    private transient HashMap<E,Object> map;
    private static final Object PRESENT = new Object();

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;   // 值永远是共享的 PRESENT
    }
    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
    public int size() {
        return map.size();
    }
}
```

- 所有元素作为 HashMap 的 **key**，value 统一是同一个静态哨兵对象 `PRESENT`（省内存，只占一个引用槽）。
- 因此 HashSet 的全部行为都由 HashMap 决定：
  - `add` 返回 `map.put` 是否返回 null（null 表示原先不存在 → 添加成功）。
  - **去重语义 = HashMap 的 key 去重语义**：依赖 `hashCode` + `equals`，重写其一必须重写另一个。
  - `O(1)` 平均 add/contains/remove（最坏链表/树化前 O(n)，树化后 O(log n)）。
  - 迭代顺序不可预测，且同样是 **fail-fast**（结构性修改抛 CME）。
  - 允许一个 `null` 元素（HashMap 允许 null key）。
- 迭代时 `map.keySet()` 的视图——`iterator()` 直接来自 keySet。

## 构造与初始容量

```java
Set<String> set = new HashSet<>(100);   // 传入的是期望元素数对应的 capacity 需按 0.75 折算
```

`HashSet(int initialCapacity)` 把参数作为内部 HashMap 的初始容量；`HashSet(int initialCapacity, float loadFactor)` 可调负载因子。批量导入前按 `expectedSize / 0.75` 预留容量可避免反复扩容。

## LinkedHashSet 与 TreeSet

| 实现 | 底层 | 顺序 | 复杂度 | 重复判定 |
| ---- | ---- | ---- | ---- | ---- |
| HashSet | HashMap | 无序 | O(1) | hashCode+equals |
| LinkedHashSet | HashMap + 双向链表 | 插入序（或访问序） | O(1) | 同上 |
| TreeSet | 红黑树（TreeMap） | 自然序/Comparator | O(log n) | compareTo/compare 返回 0 |

- 需要**稳定遍历顺序** → LinkedHashSet（内存略高，多维护一条链表）。
- 需要**范围查询/排序**（`first`/`last`/`subSet`）→ TreeSet。
- TreeSet 用比较器判重而非 equals：**两个 equals 的对象若 compareTo 返回 0 会被当作重复**，这是 `BigDecimal` 类对象的著名坑（`new BigDecimal("1.0")` 与 `new BigDecimal("1.00")` equals 为 false 但 compareTo 为 0——TreeSet 会去重，HashSet 不会）。

## 常用操作速记

```java
Set<String> a = new HashSet<>(Arrays.asList("a", "b", "c"));
Set<String> b = new HashSet<>(Arrays.asList("b", "c", "d"));

a.addAll(b);        // 并集
a.retainAll(b);     // 交集
a.removeAll(b);     // 差集（a - b）
```

JDK 8+ 还可用 Stream：`Stream.concat(a.stream(), b.stream()).collect(Collectors.toSet())`。注意 `Collectors.toSet()` 返回的也是 HashSet，**不保证顺序**；要保序用 `toCollection(LinkedHashSet::new)`。

## 使用注意

- 自定义对象作元素时必须重写 `hashCode`/`equals`，否则去重失效（同一逻辑对象存两份）。
- 遍历中删除：用 `Iterator.remove()` 或 `set.removeIf(...)`，直接 `for-each` 中 `set.remove` 会 CME。
- 线程不安全；并发用 `ConcurrentHashMap.newKeySet()`（JDK 8+）或 `Collections.synchronizedSet`。
- 内存：比 ArrayList 多了每个 entry 的 Node 开销与 hash 表空桶（负载因子 0.75 下约 25% 空桶），**同样元素数 HashSet 比 ArrayList 更吃内存**，换来了 O(1) 判在。

## 【面试追问链】

1. **HashSet 为什么允许一个 null 却"无序"？** → 无序是因为它就是 HashMap：顺序取决于 hash 与桶下标；允许 null 是因为 HashMap 允许 null key（hash 为 0）。
2. **`set.add` 返回 false 说明什么？** → 元素已存在（`map.put` 返回了旧的 PRESENT），这正是"幂等添加"的判重语义。
3. **HashSet 与 HashMap 的内存差异？** → HashSet 只是 HashMap 的薄封装，value 全是共享 PRESENT，**不额外翻倍**；真正多出来的是相对 List 的桶数组与 Node 开销。
4. **TreeSet 判重用 equals 还是 compareTo？** → compareTo/compare。因此要保证二者语义一致，否则出现"equals 不等却被 TreeSet 去重"或反之。
5. **要一个"去重且保序"的集合怎么办？** → `LinkedHashSet`；并发场景 `ConcurrentHashMap.newKeySet()`（无序）或在外部同步下维护 LinkedHashSet。
6. **为什么重写对象的 hashCode 还要注意分布均匀？** → HashSet 性能依赖 hash 均匀；全落在一个桶就退化成链表遍历。重写 hashCode 时不要只返回常量（如 `return 1;`），否则最坏 O(n)。
