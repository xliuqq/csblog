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





## 极大极小算法

极小极大实际上使用了DFS来遍历当前局势以后所有可能的结果，通过『最大化』自己和『最小化』对手的方法获取下一步的动作。

- 需要一个局面评估器，评估当前步骤的得分，启发式算法；
- α-β剪枝也是类似的思想，只不过效率更高，因为它删减了一些不需要遍历的结点。


