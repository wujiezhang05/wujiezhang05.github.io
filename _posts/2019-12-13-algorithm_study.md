---
title: 回溯算法 和 动态规划 的应用
category: Algorithm
---

> 回溯算法 和 动态规划 的应用

## 回溯算法

  适用于：很多节点,每个节点有若干个搜索分支的问题。把解问题分解为若干个子问题求解的算法

  当搜索到达某个点时，发现无法继续搜索下去，就让搜索过程回到该节点的前一个节点。继续搜索
  这个节点的其他尚未搜索过的分支。如果仍然无法继续，就继续回到前一个节点。直到搜索完所有节点

- 应用
  ```
  给定一组不含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。
  说明：解集不能包含重复的子集。
  示例:
  输入: nums = [1,2,3]
  输出:
  [
    [3],
    [1],
    [2],
    [1,2,3],
    [1,3],
    [2,3],
    [1,2],
    []
  ]
  ```

  需要遍历所有可能分支的情况。就适合回溯法。如图
  ![]({{ site.url }}/images/algorithm/backtracking.svg)
  结果集就是所有能达到的路径。路径[1,2,3]和路径[1,3,2] 是相同结果集。
  为了达到去重的目的: 下面的节点要比上面的节点大。也即图中红色部分是需要去除的。

  代码实现为：
  ```java
  List<List<Integer>> list = new ArrayList<List<Integer>>();

  private void findChildPath(int start, int[] nums, LinkedList<Integer> pre) {
      for (int i = start; i < nums.length; i++) {
          pre.addLast(nums[i]);
          list.add(new LinkedList<Integer>(pre));//加入结果集
          findChildPath(i + 1, nums, pre);//继续向下寻找可能的结果集
          pre.removeLast();//回溯到上个节点继续寻找
        }
      }

      public List<List<Integer>> subsets(int[] nums) {
        Arrays.sort(nums);
        list.add(new LinkedList<Integer>());
        findChildPath(0, nums, new LinkedList<Integer>());
        return list;
      }
  ```
## 动态规划
  动态规划常用来优化递归算法，倾向于以指数方式进行扩展。
  将复杂问题（带有许多递归调用）分解为更小的子问题，然后把他们保存到内存中，这样每次不必
  在每次使用是重新计算ta。

  递归中子问题 被记录下来，这样下次使用是不需要重复计算。

  应用
  ```
  给定不同面额的硬币 coins 和一个总金额 amount。编写一个函数来计算可以凑成总金额所需的最少的硬币个数。
  如果没有任何一种硬币组合能组成总金额，返回 -1。

  示例 1:

  输入: coins = [1, 2, 5], amount = 11
  输出: 3
  解释: 11 = 5 + 5 + 1
  示例 2:

  输入: coins = [2], amount = 3
  输出: -1
  ```

  我们假设 amout 为 i 时，最小硬币个数为 memo[i]。 硬币种类为{n1, n2, n3}
  那么
  ```
  memo[i] = Math.min(memo[i - n1]+1, memo[i-n2]+1, memo[i-n3]+1)
  ```

  也即 假设前面几次都是最佳选择，最后要选择的一枚硬币的面值为 n，
  ```
  memo[i] = memo[i-n] + 1
  ```
  代码如下:
  ```java
  public int coinChange(int[] coins, int amount) {
      int[] memo = new int[amount + 1];
      Arrays.fill(memo, amount + 1);
      memo[0] = 0;
      for (int i = 1; i <= amount; i++) {
          for (int coin : coins) { //遍历硬币种类，选择硬币个数最少的。
              if (i - coin >= 0) {
                  memo[i] = Math.min(memo[i], memo[i - coin] + 1);
              }
          }
      }
      return memo[amount] == amount + 1 ? -1 : memo[amount];
  }
  ```
