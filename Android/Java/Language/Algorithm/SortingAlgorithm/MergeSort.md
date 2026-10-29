# Java 归并排序

归并排序是**分治**的另一典型：先递归地把数组对半拆开，再把两个有序子数组**合并（merge）**成一个有序数组。它保证 O(n log n) 且**稳定**，是 TimSort（`Arrays.sort(Object[])`）的理论基础。

## 基本思想

1. **分解**：`mid = (low + high) / 2`，对 `[low, mid]` 与 `[mid+1, high]` 递归排序。
2. **合并**：两个有序区间用双指针按序拷入辅助数组，再写回原数组——merge 是算法的主体，O(n)。

```
[ 54 6 3 78 34 12 45 56 100 ]
        /                \
  [ 54 6 3 78 34 ]   [ 12 45 56 100 ]        ...递归到底（单元素有序）
        \                /
  [ 3 6 34 54 78 ]   [ 12 45 56 100 ]        逐层 merge
        \                /
  [ 3 6 12 34 45 54 56 78 100 ]
```

## 用 java 实现

```java
public class MergeSort {

    public static void mergeSort(int[] a) {
        if (a == null || a.length < 2) {
            return;
        }
        int[] temp = new int[a.length];       // 全程复用一个辅助数组
        sort(a, temp, 0, a.length - 1);
    }

    private static void sort(int[] a, int[] temp, int low, int high) {
        if (low < high) {
            int mid = low + ((high - low) >> 1);   // 防止 low+high 溢出
            sort(a, temp, low, mid);
            sort(a, temp, mid + 1, high);
            merge(a, temp, low, mid, high);
        }
    }

    /** 把有序区间 [low,mid] 与 [mid+1,high] 合并为整体有序 */
    private static void merge(int[] a, int[] temp, int low, int mid, int high) {
        // 1. 拷贝到辅助数组
        System.arraycopy(a, low, temp, low, high - low + 1);

        int i = low;                           // 左半起点
        int j = mid + 1;                       // 右半起点
        int k = low;                           // 写回原数组的位置

        while (i <= mid && j <= high) {
            if (temp[i] <= temp[j]) {          // <= 保证稳定性
                a[k++] = temp[i++];
            } else {
                a[k++] = temp[j++];
            }
        }
        while (i <= mid) {                     // 左半剩余
            a[k++] = temp[i++];
        }
        while (j <= high) {                    // 右半剩余
            a[k++] = temp[j++];
        }
    }

    public static void main(String[] args) {
        int[] a = {54, 6, 3, 78, 34, 12, 45, 56, 100};
        mergeSort(a);
        for (int v : a) {
            System.out.print(v + " ");         // 3 6 12 34 45 54 56 78 100
        }
    }
}
```

实现要点：

- 辅助数组**一次性分配、全程复用**（放在外层传入），避免每层 merge 都 new 造成 GC 压力。
- `mid = low + ((high - low) >> 1)` 防溢出且避免 `(low+high)/2` 的歧义。
- 合并时用 `<=` 比较左半优先，**这是稳定性的关键**（相等时先取左边的）。
- 右半已全部拷入时左半剩余必然是最大的、直接追加，反之亦然——两个收尾 while 不会同时执行元素。

## 复杂度与稳定性

| 情况 | 时间复杂度 | 说明 |
| ---- | ---- | ---- |
| 最好/平均/最坏 | **均为 O(n log n)** | 分解深度 log n，每层 merge 合计 O(n) |
| 空间 | **O(n)** | 辅助数组（递归栈 O(log n)） |
| 稳定性 | **稳定** | merge 用 `<=` 保证相等元素次序 |

**面试追问：为什么归并无论输入如何都是 O(n log n)？**

分解只取决于区间长度（与数据无关），永远对半分 → 恰好 log n 层；每层所有 merge 的工作量合计 O(n)。不像快排那样依赖数据分布——**代价是必须 O(n) 辅助空间**。

**面试追问：归并为什么比快排"稳定"而快排不稳定？**

归并的元素移动发生在"两个有序段按序取数"的过程中，相等时固定先取左段，不会产生跨位置的随机交换；快排的 partition 会把元素从区间一端交换到另一端，打乱相等元素次序。

## 应用与变体

- **TimSort**：Python 的排序与 Java `Arrays.sort(Object[])` 采用——归并 + 插入排序，利用数据中已有的"自然上升段（run）"，实际常接近 O(n)，稳定。
- **外部排序**：数据放不进内存时，归并天然适配"分块排序 → 多路归并"（大文件排序、数据库 merge）。
- **链表排序**：链表无法随机访问，快排 partition 不便，归并只需改指针（`sortList` LeetCode 经典题），空间可优化到 O(1)（递归改自底向上）。
- **逆序对计数**、**多路归并求第 K 大**等都是 merge 思想的延伸。

## 【面试追问链】

1. **空间 O(n) 能省吗？** → 原地归并理论可行（复杂旋转技巧）但常数大、实用性差；工程上宁可用 O(n) 空间换简单稳定。链表场景可用 O(1) 额外指针空间做自底向上归并。
2. **递归深度和栈溢出风险？** → 深度 O(log n)，比快排最坏 O(n) 安全得多；但每层栈帧带着参数，超大数组仍建议自底向上迭代版归并。
3. **和快排怎么选？** → 要稳定、要最坏保证、外部/链表数据 → 归并；只要内存内一般排序、追求缓存与常数 → 快排（TimSort 则是"两者优点的工程合体"）。
4. **`Arrays.sort` 对 int[] 和 Integer[] 为什么不同？** → `int[]` 用双轴快排（不需要稳定，省空间）；`Integer[]` 用 TimSort（对象排序常要求稳定，且用户可能依赖相等元素次序）。
5. **merge 中为什么 `<=` 而不是 `<`？** → `<` 会让右段的相等元素先于左段写入，破坏稳定性；稳定排序的定义要求相等元素保持输入时的相对顺序。
6. **怎么用归并求逆序对？** → merge 时若右段元素 `temp[j]` 先于左段 `temp[i]` 写入，说明 `temp[i..mid]` 全部与它构成逆序对，累加 `mid - i + 1`——分治统计的典型扩展。
