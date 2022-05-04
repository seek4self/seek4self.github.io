---
title: 打家劫舍系列
top: false
cover: false
toc: true
mathjax: false
date: 2022-05-04 22:43:33
password:
summary: 动态规划解决打家劫舍问题
categories: 算法
tags:
- leetcode
- dynamic programming
---

## 打家劫舍Ⅰ

你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，**如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警**。

给定一个代表每个房屋存放金额的非负整数数组，计算你**不触动警报装置的情况下**，一夜之内能够偷窃到的最高金额。

示例1：

```txt
输入：[1,2,3,1]
输出：4
解释：偷窃 1 号房屋 (金额 = 1) ，然后偷窃 3 号房屋 (金额 = 3)。
     偷窃到的最高金额 = 1 + 3 = 4 。
```

示例2：

```txt
输入：[2,7,9,3,1]
输出：12
解释：偷窃 1 号房屋 (金额 = 2), 偷窃 3 号房屋 (金额 = 9)，接着偷窃 5 号房屋 (金额 = 1)。
     偷窃到的最高金额 = 2 + 9 + 1 = 12 。
```

提示：

- `1 <= nums.length <= 100`
- `0 <= nums[i] <= 400`

来源：力扣（LeetCode）
链接：<https://leetcode-cn.com/problems/house-robber>
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

### 题解

小偷进入每个房屋后，有两种选择，偷或者不偷，设定 `dp[i]` 是进入第 `i` 间房子能偷到最多的钱，如果偷第 `i` 间房子，而邻居不能偷，则有 `dp[i]= dp[i-2]+nums[i]`, 若不偷第 `i` 间房子，则有 `dp[i] = dp[i-1]`, 因此有状态转移方程：

```go
dp[i] = max(dp[i-1], dp[i-2]+nums[i])
```

如果只有 1 间房，则只能偷该房间的钱，即 `dp[1] = nums[0]`

```go
package main

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 *
 * 
 * @param nums int整型一维数组 
 * @return int整型
*/
func rob(nums []int) int {
    // write code here
    if len(nums) <2 {
        return nums[0]
    }
    dp := make([]int, len(nums)+1)
    dp[1] = nums[0]
    for i:= 2; i<=len(nums); i++ {
        // 每家可选择偷或者不偷
        dp[i] = max(dp[i-1] ,dp[i-2] + nums[i-1])
    }
    return dp[len(nums)]
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

可以看到 dp 的值只和 `dp[i]` 、`dp[i-1]` 、`dp[i-2]`三者有关，因此可以优化空间复杂度 O(N) 为 O(1)

```go
package main

func rob(nums []int) int {
    n := len(nums) 
    if n == 0 {
        return 0
    }
    if n == 1 {
        return nums[0]
    }
    prev, cur := nums[0], max(nums[0], nums[1])
    for i := 2; i< n; i++ {
       prev, cur = cur, max(cur, prev+nums[i])
    }
    return cur
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

## 打家劫舍Ⅱ

你是一个专业的小偷，计划偷窃沿街的房屋，每间房内都藏有一定的现金。这个地方所有的房屋都**围成一圈** ，这意味着第一个房屋和最后一个房屋是紧挨着的。同时，相邻的房屋装有相互连通的防盗系统，**如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警** 。

给定一个代表每个房屋存放金额的非负整数数组，计算你**在不触动警报装置的情况下** ，今晚能够偷窃到的最高金额。

示例 1：

```txt
输入：nums = [2,3,2]
输出：3
解释：你不能先偷窃 1 号房屋（金额 = 2），然后偷窃 3 号房屋（金额 = 2）, 因为他们是相邻的。
```

示例 2：

```txt
输入：nums = [1,2,3,1]
输出：4
解释：你可以先偷窃 1 号房屋（金额 = 1），然后偷窃 3 号房屋（金额 = 3）。
     偷窃到的最高金额 = 1 + 3 = 4 。
```

示例 3：

```txt
输入：nums = [1,2,3]
输出：3
```

提示：

- `1 <= nums.length <= 100`
- `0 <= nums[i] <= 1000`

来源：力扣（LeetCode）
链接：<https://leetcode-cn.com/problems/house-robber-ii>
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

### 题解

由于房屋首尾相连，所以偷了第一家，就不能偷最后一家，不偷第一家，就能偷最后一家，其他条件和打家劫舍Ⅰ相同，所以可以分两种情况，去掉头/尾两间房，分别计算出各自的最大值，再两者选最大

```go
package main

func rob(nums []int) int {
    n := len(nums)
    if n == 1 {
        return nums[0]
    }
    // 0 ~ n-2 间房能偷的最大值
    pre, cur := nums[0], max(nums[0], nums[1])
    for i:= 2; i< n-1; i++ {
        pre, cur = cur, max(cur, pre + nums[i])
    }
    m := cur
    // 1 ~ n-1 间房能偷的最大值
    pre, cur = 0, max(0, nums[1])
    for i := 2; i < n; i++ {
        pre, cur = cur, max(cur, pre + nums[i])
    }
    return max(m, cur)
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```
