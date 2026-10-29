# Java LinkedList

LinkedList 实现了 `List` 和 `Deque` 双重接口，底层是**双向链表**（JDK 8 起为双向链表，早期版本是单向链表 + 前向指针优化）。它既当列表用，也当队列/双端队列用。

## 底层结构

```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient int size = 0;
    transient Node<E> first;
    transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.prev = prev;
            this.next = next;
        }
    }
}
```

要点：

- 每个节点 `Node` 持有前驱 `prev`、后继 `next` 与数据 `item`——**每个元素都有额外的对象头开销**（约 32 字节以上，取决于 JVM），存储同样的数据比 ArrayList 更费内存。
- 维护 `first`/`last` 指针，因此**头尾操作是 O(1)**。
- 没有 `modCount` 之外的随机访问能力：`get(index)` 需要从头或尾按 `index < size/2` 就近遍历，**O(n)**。

## 头尾操作与作队列/栈

```java
Deque<String> dq = new LinkedList<>();

// 作队列（FIFO）
dq.offerLast("tail");    // 入队
dq.pollFirst();          // 出队（空返回 null）

// 作栈（LIFO）
dq.push("top");          // 等价于 addFirst
dq.pop();                // 等价于 removeFirst（空抛 NoSuchElementException）

// 作双端队列
dq.offerFirst("head");
dq.peekLast();           // 查看队尾（空返回 null）
```

- `add`/`addLast`/`offer`/`offerLast` 追加到尾部；`addFirst`/`push` 头部插入。
- `remove`/`removeFirst`/`poll`/`pollFirst` 头部移除；尾部对应 `removeLast`/`pollLast`。
- `poll*` 空集合返回 `null`，`remove*`/`pop` 空集合抛异常——作队列时优先用 `poll`，可避免 try-catch。

**面试追问：为什么 LinkedList 用双向链表而不是单向？**

1. 头尾操作都 O(1)（单向链表尾插需要维护尾指针，删除尾节点仍需前驱）。
2. `ListIterator` 支持双向遍历与高效 `previous()`。
3. `remove(node)` 在已持有节点引用时可 O(1) 完成（需要前驱），双向链表天然具备。

## 复杂度一览

| 操作 | 复杂度 | 说明 |
| ---- | ---- | ---- |
| `get(index)` / `set(index)` | O(n) | 需遍历到下标位置 |
| `addFirst` / `addLast` / `poll*` | O(1) | 只动指针 |
| `add(index, e)` 中间插入 | O(n) 定位 + O(1) 插入 | 定位占主导 |
| `remove(Object)` | O(n) | 先查找 |
| `contains` / `indexOf` | O(n) | 线性查找 |
| `size()` | O(1) | 维护了 `size` 字段 |

注意：虽然插入动作本身是指针操作 O(1)，但**先要 O(n) 定位**，所以整体仍是 O(n)。只有"已知节点引用"或"在头尾操作"时才真正 O(1)。

## 与 ArrayList 选型

| 维度 | ArrayList | LinkedList |
| ---- | ---- | ---- |
| 存储 | 连续数组，元素紧凑 | 分散节点，指针开销 |
| 随机访问 | O(1)，缓存友好 | O(n)，缓存不友好 |
| 头尾插入 | O(n)（需搬移）或扩容 | O(1) |
| 内存 | 紧凑，无额外对象 | 每元素一个 Node 对象 |
| 实际表现 | 绝大多数场景更快 | 仅特定头尾队列场景占优 |

结论（面试金句）：

- 默认选 **ArrayList**：CPU 缓存局部性让它在遍历、随机访问上碾压链表；JDK 源码里大量场景甚至用数组模拟队列。
- 只有当**频繁在已知位置做头尾插入删除、几乎不做随机访问**时，LinkedList（或 `ArrayDeque`）才更合适。
- 需要队列语义时，现代首选其实是 **`ArrayDeque`**（环形数组，无指针开销，O(1) 队列操作），LinkedList 作队列已逐渐被替代。

**面试追问：`ArrayList` 和 `LinkedList` 谁的 `add(0, e)` 更快？**

理论 LinkedList O(1) 定位（头节点已知），ArrayList 需 O(n) 搬移。但 ArrayList 搬移是 `System.arraycopy` 的内存拷贝（极快），LinkedList 要创建 Node 对象并写指针（缓存不友好），中小规模下两者常打平甚至 ArrayList 更快；大规模数据时 LinkedList 的内存与 GC 压力会显现。测量胜于臆断。

## 常见使用注意

- 不要用 `for (int i...) list.get(i)` 遍历 LinkedList——每次都从头找，O(n²)。用增强 for 或迭代器。
- `LinkedList` 同样非线程安全，且 fail-fast 行为与 ArrayList 一致。
- 序列化：`first`/`last`/`Node` 都是 `transient`，通过 `writeObject` 按序写入元素，反序列化重建链表。

## 【面试追问链】

1. **`LinkedList` 的 `get(index)` 为什么还要 O(n)？** → 链表没有下标映射，必须从 `first` 或 `last` 沿指针走 `min(index, size-1-index)` 步；`size/2` 就近选择只是把常数减半，不改变 O(n)。
2. **为什么 `Node` 和 `first/last` 都是 transient？** → 链表指针是运行时结构，序列化按逻辑顺序写入元素即可；反序列化按写入顺序重建 `prev/next`，避免把指针对象本身写进流（也防止序列化格式被内部结构锁死）。
3. **`removeFirst` 与 `pollFirst` 怎么选？** → 空集合时前者抛 `NoSuchElementException`，后者返回 null。作队列循环取数用 `poll`，需要对"不能为空"快速失败用 `remove`。
4. **LinkedList 实现了 `RandomAccess` 吗？** → 没有（ArrayList 实现了）。`Collections.binarySearch` 据此对 ArrayList 走二分、对 LinkedList 走顺序遍历——这是标记接口影响算法选择的标准案例。
5. **单向链表行不行？** → 尾删需要前驱（或维护 tail 但删除尾仍要前驱）、`ListIterator.previous` 需要前驱、删除已知节点 O(1) 需要前驱——双向链表用每节点 2 个引用换来了这些 O(1) 与双向迭代能力。
6. **什么场景下 LinkedList 真的优于 ArrayList？** → 已持有节点引用的频繁删除、两端高频操作、且数据量大到数组搬移不可接受——典型如"维护一个按时间排序的活动窗口"。日常业务列表 99% 选 ArrayList。
