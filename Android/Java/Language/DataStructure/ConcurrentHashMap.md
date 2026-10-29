# Java ConcurrentHashMap

ConcurrentHashMap 是 Java 并发包的基石之一：在**不使用全局锁**的前提下提供线程安全的 Map。JDK 7 用**分段锁（Segment）**，JDK 8 改为 **CAS + synchronized 锁桶首节点**——理解这次重构是面试并发集合的核心。

## 总体定位

- 线程安全的 `Map`，支持高并发读与较高并发写；无全局锁，锁粒度极细。
- **不允许 null key，也不允许 null value**（HashMap 都允许）。
- 迭代器是**弱一致（weakly consistent）**：不会抛 `ConcurrentModificationException`，创建后其他线程的修改可能可见也可能部分可见，遍历基于开始时或遍历过程中定位到的桶。
- `size()`、`isEmpty()`、`containsValue()` 是**聚合操作**，反映的是近似/瞬时快照（遍历各桶计数），并发下不保证与任何单一时刻严格一致。

**为什么不允许 null？** 单线程 HashMap 里 `get(key) == null` 可明确解释为"值是 null"或"键不存在"，二者都可接受。并发下 `get` 返回 null 时无法区分"键不存在"与"值恰好为 null"——除非对 null 也加锁二次确认，代价高且易错。ConcurrentHashMap 干脆禁止 null，让 `get` 返回 null **只**意味着键不存在，`putIfAbsent`、`compute` 等语义也因此更干净。

## JDK 7：分段锁（Segment）

```java
// JDK 7 结构（概念示意）
static final class Segment<K,V> extends ReentrantLock {
    volatile HashEntry<K,V>[] table;
    int count;          // 段内元素数
    int modCount;
    int threshold;      // 扩容阈值
}
// ConcurrentHashMap = Segment[] + 每个 Segment 内部一个小型 HashMap
```

- 整个 Map 被划分为 **N 个 Segment**（默认并发度 16，构造时可指定），每个 Segment 继承 `ReentrantLock`——**锁的是段，不是整表**。
- 读操作：`HashEntry.value` 是 volatile，**不加锁**即可读到最新值。
- 写操作：锁住对应 Segment（可重入锁），段内结构与 HashMap 类似（JDK 7 也是头插链表）。
- 扩容：只扩容**单个 Segment** 的数组，其他段不受影响——这是分段设计的关键收益。
- `size()`：先两次各段 count 求和，若两次相等且无段被改（modCount 检查）则返回；否则**锁住所有 Segment**（升序加锁避免死锁）再求和。

局限：Segment 数量固定，锁并发度受限；跨段复合操作（如遍历中修改另一段）无原子性；实现复杂。

## JDK 8：CAS + synchronized 锁桶首节点

```java
// JDK 8 结构（概念示意）
static final class Node<K,V> {
    final int hash;
    final K key;
    volatile V val;         // value 可见性靠 volatile
    volatile Node<K,V> next; // 链表 next 可见性靠 volatile
}
transient volatile Node<K,V>[] table;
```

核心变化：

1. **数据结构回归单张表**：Node 数组 + 链表 + 红黑树（与 HashMap 8 相同，树化阈值 8 / 容量 64）。
2. **加锁单位变成"桶的首节点"**：写入/删除/树化时 `synchronized (f)`（f = 桶首 Node），只阻塞落在**同一个桶**上的写；不同桶完全并行。
3. **首次插入用 CAS**：桶为空时 `casTabAt(tab, i, null, new Node(...))` 无锁成功，失败再走 synchronized。
4. **size 用 `baseCount` + `CounterCell[]` 分散计数**（类似 `LongAdder`）：并发 `addCount` 时线程在不同 CounterCell 上自增，几乎无竞争；`size()` = baseCount + 各 cell 之和。扩容时用 `sizeCtl` 协调多线程协作迁移。
5. **扩容协作（transfer）**：一个线程触发扩容后，把原表分成多个步长区间，**其他线程 `helpTransfer` 帮忙搬**——避免"一个线程扩容、全体阻塞"。搬迁同样在桶首节点加锁，搬完用 CAS 把桶置为 `ForwardingNode`（fwd），其他线程看到 fwd 就知道去新表找或加入搬迁。

**面试追问：JDK 8 为什么放弃分段锁？**

- 分段锁并发度固定（Segment 数），且段内仍是粗粒度锁；锁整个 Segment 会让同段的读写也互斥（读虽无锁但写会挡）。
- 锁桶首节点后，并发度 = 桶数（随容量增长），比固定 Segment 更细。
- 统一结构便于复用 HashMap 的树化、resize 优化，代码路径更一致。
- 代价：`synchronized` 依赖 JDK 6+ 的锁升级，现代 JVM 上开销已接近无锁；JDK 8 把锁升级做到"轻量级偏向/轻量锁"，重量级场景才真正挂起线程。

## put 流程（JDK 8，概念步骤）

1. hash 扰动（与 HashMap 相同的 `^ (h >>> 16)`）。
2. 表空或长度 0 → CAS 初始化（`sizeCtl` 用 -1 占位防止并发初始化）。
3. 桶空 → CAS 放首节点。
4. 桶首是 fwd 节点 → 说明正在扩容，加入 `helpTransfer` 协助搬迁。
5. 否则 `synchronized (f)`：链表尾插（或覆盖同 key）、树化 `putTreeVal`（树节点插入也走锁 + 部分 CAS）。
6. `addCount(1, chk)`：分散计数 + 检查是否需要扩容。

**覆盖已有 key**：只改 `node.val`（volatile 写），不改结构，不加锁也保证可见性——这是把 `value` 设为 volatile 的意义。

## get / remove 要点

- `get`：完全无锁。定位桶后沿链表/树遍历，靠 `val`、`next` 的 volatile 读保证可见性；读到 fwd 节点则去新表查。
- `remove`：锁桶首节点，按链表/树摘除；树缩容在节点少时退化回链表。
- `computeIfAbsent` / `merge` / `replace`：整段逻辑在**桶锁内**完成，保证复合操作原子性（JDK 7 要跨段时需额外手段）。

## size 与扩容协作（面试深水区）

| 机制 | 作用 |
| ---- | ---- |
| `baseCount` | 无竞争时的直接计数 |
| `CounterCell[]` | 类 LongAdder 分散竞争，`size = base + Σcell` |
| `sizeCtl` | 多义标志：`-1` 初始化中、`-N` 正在扩容（线程数 N）、正数 = 扩容阈值/新容量 |
| `transfer` | 按步长分段搬迁，CAS 抢占步长，`helpTransfer` 协助 |
| `ForwardingNode` | 桶迁移完成标记，携带 `nextTable` 引用，转发读写到新表 |

`size()` 在扩容期间会先尝试无锁求和（基于 `addCount` 的校验），必要时自旋等待扩容完成再返回稳定值——因此**高并发写时调用 `size()` 可能短暂变慢**，这是准确性与吞吐的权衡。

## 弱一致迭代器

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
Iterator<String> it = map.keySet().iterator();   // 不会抛 CME
while (it.hasNext()) {
    String k = it.next();
    map.put(k + "x", "v");    // 迭代期间的修改：安全，但可能/不可能被迭代看到
}
```

- **不 fail-fast**：无 modCount 检查，专为并发设计。
- 弱一致性：能看到迭代开始前已完成的修改；对迭代开始后新插入的桶，**可能**看到也可能看不到；不会抛 CME，也不会无限循环（搬迁用 fwd 衔接）。
- `size()` 同理是近似值——需要严格一致的计数要么外部加锁，要么接受近似。

## 与其他并发 Map 对比

| 实现 | 并发策略 | 适用 |
| ---- | ---- | ---- |
| `ConcurrentHashMap` | CAS + 桶锁 + 分散计数 | 通用并发 Map，最常用 |
| `Collections.synchronizedMap` | 全局单锁 | 低并发，简单包装；迭代要手动同步 |
| `Hashtable` | 方法级 synchronized | 遗留，全表锁，别用 |
| `ConcurrentSkipListMap` | 跳表 + CAS，无锁读 | 需要有序/并发导航 |
| `CopyOnWriteArrayList`（List） | 写时整表拷贝 | 极读多写少 |

## 工程使用注意

- 批量导入前 `new ConcurrentHashMap<>(expectedSize)` 预设容量，减少扩容搬迁。
- 复合逻辑（先查再改）用 `computeIfAbsent`、`merge`、`replace(k, old, new)`，**不要**"get 判断 + put"两步走（存在 check-then-act 竞态）。
- `putIfAbsent(k, v)`：key 不存在**或映射值为 null** 时放入给定的 v，返回原值（或 null）。`computeIfAbsent(k, fn)`：仅当 key 不存在（或值为 null）时才调用 `fn` 计算并放入，`fn` 返回 null 则不放入。二者都保证"查 + 放"在一个桶锁内原子完成；`computeIfAbsent` 的函数应尽量无副作用、且不要在里面反过来修改同一个 map（可能递归/死锁）。
- `forEach`、`search`、`reduce` 等并行风格 API 基于分段遍历，适合大 Map 上的聚合。

## 【面试追问链】

1. **JDK 7 分段锁的 size() 怎么算的？** → 两次无锁求和 + modCount 校验，不一致才锁全部 Segment 升序求和；JDK 8 换成 baseCount + CounterCell（LongAdder 思想），无锁为主。
2. **为什么 value 和 next 要 volatile？** → get 无锁，靠 volatile 读保证可见性；否则可能读到扩容/写入过程中的陈旧引用（甚至 JDK 7 的死链问题在 JDK 8 依靠 volatile + 顺序迁移避免）。
3. **一个线程 put 时另一个线程 get 同一个 key 会怎样？** → get 不阻塞；写者锁桶首节点改 val（volatile 写），读者立刻或在写完成后读到新值——不会读到撕裂值（引用写入原子）。
4. **不允许 null key/value 换来了什么？** → `get` 语义无歧义、`computeIfAbsent`/`merge` 不需要处理 null 传播、免去为 null 单独加锁判断；代价是 API 与 HashMap 不完全兼容。
5. **扩容时其他线程在干嘛？** → 落在已迁移桶上的操作走 fwd 转发到新表；未迁移桶上的操作照常加桶锁；空闲线程可 `helpTransfer` 参与搬迁——扩容从"停机"变成"边搬边用"。
6. **迭代为什么不用 fail-fast？** → fail-fast 依赖 modCount 独占假设，并发下要么误报要么漏检；弱一致迭代让遍历始终有界终止且不抛异常，把一致性级别明确暴露给调用方。
7. **和 Hashtable 比锁粒度差多少？** → Hashtable 锁整个对象（任何读写都互斥，读也常被挡）；CHM 写只锁单桶，读基本无锁——高并发下吞吐差数量级。
