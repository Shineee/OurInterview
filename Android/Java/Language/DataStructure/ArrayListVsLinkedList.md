# ArrayList 与 LinkedList 选型对比

两者都实现了 `List` 接口，但存储方式完全不同：ArrayList 是连续数组，LinkedList 是双向链表。选型不是背表格，而是从**缓存、内存、操作分布**三个维度做工程判断。

## 结构差异回顾

| 维度 | ArrayList | LinkedList |
| ---- | ---- | ---- |
| 存储 | `Object[]` 连续内存 | 分散的 `Node` 对象（item/prev/next） |
| 随机访问 `get(i)` | O(1)，直接下标 | O(n)，从头/尾就近遍历 |
| 尾部 `add` | 摊还 O(1)（1.5 倍扩容） | O(1)（改 tail 指针） |
| 中间 `add(i,e)` | O(n) 搬移（`arraycopy`） | O(n) 定位 + O(1) 指针操作 |
| 删除 | O(n) 搬移 | O(n) 定位 + O(1) 摘除 |
| 每元素内存 | 仅对象本身（引用 4B） | + Node 对象（约 32B：头 16 + 3 引用 12 + 对齐） |
| 缓存局部性 | 优秀（连续访问） | 差（节点随机分布，缓存行利用率低） |
| 扩容成本 | O(n) 拷贝 | 无（链表天然增长） |

## 三个判断维度

### 1. 操作分布

- **读多、随机访问多、遍历多** → ArrayList。`get(i)`、`contains`、排序（`Collections.sort` 对随机访问 List 走 TimSort）都依赖下标。
- **频繁头尾插入删除、几乎不按下标访问** → LinkedList（或更好的 `ArrayDeque`/`Deque`）。
- **中间插入删除很频繁**：理论 LinkedList 占优，但见下条缓存因素——常常反而不占优。

### 2. 缓存与实测

- ArrayList 的遍历/搬移是**连续内存 + `System.arraycopy`（接近 memcpy 的速度）**，CPU 预取友好。
- LinkedList 每步 `next` 都可能跳到不相邻的缓存行，且要创建/读取 Node 对象头——指针追踪的"随机内存访问"代价高。
- 工程经验：**中小规模（几十万以下）下，ArrayList 的中间插入经常不比 LinkedList 慢，甚至更快**；LinkedList 的优势主要体现在"已知节点位置的头尾操作"。结论：**测量优先于理论复杂度**。

### 3. 内存

- 同样 N 个 Integer：ArrayList = 引用数组 4N + 元素对象；LinkedList = 4N（引用分散）+ 每元素一个 Node ≈ **每元素多 32B 左右**。
- 链表还有 GC 压力：大量小对象增加 GC 扫描成本；ArrayList 一块大数组，GC 更友好。
- Android/内存敏感场景：**默认 ArrayList，预设容量**；除非确有头尾队列需求。

## 决策速查

```
默认 → ArrayList（new ArrayList<>(expectedSize) 预分配）
要队列/栈语义 → ArrayDeque（不要用 LinkedList 当队列，除非需要按索引混合操作）
频繁头尾增删、不随机访问 → LinkedList 或 ArrayDeque
必须按下标随机插删 + 数据量小 → 两者皆可，实测为准
```

## 面试常问的"陷阱题"

1. **`list.get(i)` 循环哪个快？** → ArrayList O(1)；LinkedList O(n²)，绝对避免。
2. **`add(0, e)` 谁快？** → 理论 LinkedList；实测中 ArrayList 的 `arraycopy` 常赢。回答时点出缓存因素即加分。
3. **为什么 JDK 不把 LinkedList 优化掉？** → 它是 `Deque`/`AbstractSequentialList` 生态的一部分，`ListIterator` 双向、作 Deque 实现之一；且删除已知节点引用时 O(1) 是数组做不到的。
4. **`Arrays.asList` 返回的是 ArrayList 吗？** → 是 `java.util.Arrays$ArrayList`（固定长度的 List 包装），不是 `java.util.ArrayList`，`add/remove` 会 UOE。

## 【面试追问链】

1. **为什么 ArrayList 扩容 1.5 倍、LinkedList 不需要扩容？** → 数组要整块拷贝，倍数太小则扩容太频繁、太大则浪费；链表按需挂节点，无容量概念，代价转移到每元素 Node 开销。
2. **ArrayList 的 O(1) get 依赖什么前提？** → 下标有效 + 随机访问存储；这也是 `RandomAccess` 标记接口的意义——算法库据此选择二分/顺序查找（`indexOf` 对实现 RandomAccess 的 List 走优化路径）。
3. **两者线程安全性如何？** → 都非线程安全、都 fail-fast；并发读多写少用 `CopyOnWriteArrayList`（数组拷贝，写贵读无锁）。
4. **LinkedList 作队列为什么不如 ArrayDeque？** → ArrayDeque 是环形数组：无 Node 对象、无指针、缓存连续，O(1) 两端操作；LinkedList 多了对象与 GC 开销，作队列纯属浪费其"按索引访问"能力。
5. **什么时候"理论复杂度更差"的实现反而更快？** → 当常数因子（缓存、分支预测、库函数优化）差异大于渐进复杂度差异时——ArrayList vs LinkedList 是教科书案例，回答时给出 arraycopy 与缓存行两个机制即可。
