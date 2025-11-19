# 算法

## Master 定理

> $O$：表示上界（tightness unknown），即小于等于
>
> $\Theta$：该时间复杂度既是上界也是下界（tight），即等于的意思
>
> $\Omega$：表示下界（tightness unknown），即大于等于的意思

递推公式的复杂度计算，*n* 是问题规模，*b* 是递归子问题的数量，*n/c* 为每个子问题的规模（假设每个子问题的规模基本一样），*f(n)* 为递归以外进行的计算工作。

$T(n)=b*T(\frac{n}{c})+f(n)$

如果对任意正数 $\epsilon $，$E = lg(b) / lg(c) = lg_c{b}$

- $f(n) = O(n^{(E−\epsilon)})，则T(n) = \Theta(n^E)$；
- $f(n)= \Theta(n^E)，则T(n)=Θ(f(n)log⁡(n))$；
- $f(n)=Ω(n^{(E−ϵ)}), 且对某个δ≥ϵ有f(n)=Ο(n^{(E+δ))}，则T(n)=Θ(f(n))$；

示例：

- 二叉树遍历：$T(n)=2*T(\frac{n}{2}) + \Omega(1)=\Theta(n)$
- 归并排序：$T(n)=2T(\frac{n}{2}) + \Omega(n)=\Theta(nlog n)$



## 递归转循环

> 代码实现见 [递归变循环通用方法](https://gitee.com/oscsc/data-structure-and-algorithm/blob/master/src/main/java/org/xliu/cs/algs_ds/algs/recursive/RecursiveToFor.java)。

递归转循环的通用方法：通过**手动模拟栈帧**的执行。

```java
// 树的后序遍历：递归版
public static <E> void postOrderRecursive(BinaryTreeNode<E> root, List<E> store) {
    if (root == null) {
        return;
    }
    if (root.getLeft() != null) postOrderRecursive(root.getLeft(), store);
    if (root.getRight() != null) postOrderRecursive(root.getRight(), store);
    store.add(root.getVal());
}

// 树的后序遍历：非递归版
public static <E> void postOrder(BinaryTreeNode<E> root, List<E> store) {
    if (root == null) {
        return;
    }
    Deque<Frame<E>> stack = new LinkedList<>();
    stack.add(new Frame<>(root, store));
    while (!stack.isEmpty()) {
        Frame<E> current = stack.getLast();
        switch (current.pc) {
            case 0:
                if (current.node == null) stack.removeLast();
                break;
            case 1:
                if (current.node.getLeft() != null) stack.add(new Frame<>(current.node.getLeft(), store));
                break;
            case 2:
                if (current.node.getRight() != null) stack.add(new Frame<>(current.node.getRight(), store));
                break;
            case 3:
                store.add(current.node.getVal());
                break;
            case 4:
                stack.removeLast();
                break;
        }
        current.pc += 1;
    }
}
// 栈帧保存的内容
private static class Frame<E> {
    int pc;
    BinaryTreeNode<E> node;
    List<E> store;
    public Frame(BinaryTreeNode<E> node, List<E> store) {
        this.pc = 0;
        this.node = node;
        this.store = store;
    }
}
```

## 算法设计的若干策略

> 《算法谜题》 Anany Levitin Maria Levitin 著，赵勇 徐章宁 高博 译，2014年3月第一版。

**穷举搜索**：尝试问题的所有的可能解，直到找到问题的解为止。

**回溯法**：对穷举搜索所采取的蛮力做法的一种改进。

- 涉及一定数量的错误选项的撤销动作（**剪枝**），减少后续的无用搜索；

**减而治之**：将问题减少为一系列、规模越来越小的子问题

- 如 1-100 猜数字；

**分而治之**：将问题分解为若干个较小的子问题（最小规模的子问题可以解决），最后将子问题的解组合起来；

- 如 归并排序；
- **分而治之，每次需要解决多个子问题，而减而治之每次仅需解决一个**；

**变而治之**：将问题修改或变换成另一个问题（该问题更容易求解）；

- 谜面简化、表示变更、问题规约

**贪心法**：每一步都选择最好的一个选择，**局部最优解产生全局最优解**；

- 难点在于证明真的能产生最优解

**迭代改进**：从易得的估计解出发，并重复地应用同样的简单步骤不断改进估计解；

- **单变性（monovariant）**，保证局部最优也是全局最优

**动态规划**：最优子结构，从子问题的最优解构造出全局最优解。



## 极大极小算法

极小极大实际上使用了DFS来遍历当前局势以后所有可能的结果，通过『最大化』自己和『最小化』对手的方法获取下一步的动作。

- 需要一个局面评估器，评估当前步骤的得分，启发式算法；
- α-β剪枝也是类似的思想，只不过效率更高，因为它删减了一些不需要遍历的结点。

