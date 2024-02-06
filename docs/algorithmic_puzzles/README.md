# 概览

> 《算法谜题》 Anany Levitin Maria Levitin 著，赵勇 徐章宁 高博 译，2014年3月第一版。

## 算法设计的若干策略

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

**动态规划**：最优子结构，从子问题的最优解构造出全局最优解
