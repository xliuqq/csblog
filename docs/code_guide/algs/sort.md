# 排序算法

## 基于比较的排序算法

时间复杂度**基于比较次数**，基于比较的排序算法最好时间复杂度为`O(nlog(n))`：

- `stable`：保留等值元素在输入中的相对顺序
- `in-place`：不额外消耗内存（空间）；

| 典型实现 | 时间复杂度 <br />best, worst, average | 空间复杂度 | 稳定性     | 特点                          |
| -------- | ------------------------------------- | ---------- | ---------- | ----------------------------- |
| 选择排序 | $O(n2), O(n^2), O(n^2)$               | O(1)       | Not Stable | in-place sort，最多n-1次交换  |
| 插入排序 | $O(n), O(n^2), O(n^2)$                | O(1)       | Stable     | in-place sort，适用small data |
| 快速排序 | $O(nlog(n)), O(n^2),  O(nlog(n))$     | O(1)       | Not Stable | in-place sort                 |
| 归并排序 | $O(nlog(n))$                          | O(n)       | Stable     | Not in-place sort             |
| 归并排序 | $O(nlog(n))$                          | O(1)       | Stable     | in-place sort                 |
| 堆排序   | $O(nlog(n))$                          | O(1)       | Not Stable | in-place sort                 |
| 冒泡排序 | $O(n),O(n^2),O(n^2)$                  | O(1)       | Stable     | in-place sort                 |

快速排序A[a, b]

- 随机选择初始的比较值；

- 单个指针，[a, low]和[low, b]；（无法处理元素不随机情况，如元素全部相等O(n2）

- 双向指针，[a, low]和[high, b]；（当遇到相同的元素时停止扫描解决元素全部相等O(nlog(n))）

  ```c
  void qsort(x, a, b) {
      if (a >= b) return
      t = x[a];i = a; j = b;
      while (true) {   
          do i++ whi1e(i <= b && x[il < t);    
          do j-- whi1e(x[j] > t);
          if (i > j) break;
          swap(x[i], x[j]);
      }
      swap(x[a], x[j]);
      qsort(x, a, j-l);
      qsort(x, j+1, b);  
  }     
  ```

## 非比较排序算法

### 桶排序

稳定，分成m个桶

### 计数排序

稳定，保存[max-min]的数组个数，累加，倒序； 

- n 个 0 到 k 之间的整数时，运行时间是 Θ(n + k)；空间复杂度是Θ(n+k)；

### 基数排序

稳定，将待排数据中的每组关键字（如位上的数字）依次进行桶分配；

- MSD：最高位优先，分成不同的桶(计数排序)，桶内再排序；**O(d(n+r))**，r为基数，d为位数；

- LSD：最低位优先，采用计数排序，每次根据当前位对数组重排；**O(d(n+r))**，r为基数，d为位数；