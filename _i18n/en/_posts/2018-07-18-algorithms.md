---
layout: post
title: 《数据结构与算法JavaScript描述》算法部分学习心得
date: 2018-07-18 22:49:20
tags: [算法]
categories: algorithm
---
## 导言
最近读了《数据结构与算法JavaScript描述》，除去书中一些代码错误之外，是一本让我有所收获的书。这里会说一些书中提及的算法及一点自己的理解，希望以后能熟练运用这些算法解决更复杂的问题。
<!-- more -->
## 排序算法
### 快速排序
快速排序是处理大数据集最快的排序算法之一。很多编程语言的排序函数都用到了快速排序，比如Chrome 的 JavaScript 引擎V8的`Array.sort()`就使用快速排序和插入排序的结合。
快速排序的算法简单描述：
 1. 先确定一个“支点”（pivot）
 2. 将所有小于“支点”的值都放在该点的左侧，大于“支点”的值都放在该点的右侧
 3. 然后对左右两侧不断重复这个过程，直到所有排序完成。

如下图的例子：
![快速排序](/assets/img/2018/07/quickSort-1.jpeg)
用递归的思想可以这样实现，时间复杂度O(nlog2n)
``` js
function quickSort(arr) {
  if (arr.length <= 1) {
    return arr
  }
  const pivot = Math.floor(arr.length / 2)
  const left = []
  const right = []
  for (let i = 0, len = arr.length; i < len; i++) {
    if (arr[i] < arr[pivot]) {
      left.push(arr[i])
    } else if (arr[i] >= arr[pivot] && i !== pivot) {
      right.push(arr[i])
    }
  }
  return quickSort(left).concat(arr[pivot], quickSort(right))
}
// 调用
const arr = [2,1,4,5,8,3]
quickSort(arr, 0, arr.length - 1)
```
但是，这个实现的方法声明了一些数组，空间复杂度较大，内存占用较多。
下面是更好的原地排序的实现方法。
先看动图
![快速排序](/assets/img/2018/07/quickSort-2.gif)
具体实现：
``` js
function partition(arr, left, right) {
  const pivot = arr[left] // 支点
  let p = left
  for (let i = left + 1; i <= right; i++) {
    if (arr[i] < pivot) {
      swap(arr[++p], arr[i])
    }
  }
  swap(pivot, arr[p])
  return p // 返回支点所在位置
}
function quickSort(arr, left = 0, right = arr.length - 1) {
  if (arr.length == 1) {
    return arr
  }
  if (left < right) {
    const p = partition(arr, left, right)
    quickSort(arr, left, p - 1)
    quickSort(arr, p + 1, right)
  }
}
// 交换函数
function swap (a, b) {
  let temp = b
  b = a
  a = temp
}
```
## 检索算法
### 顺序查找
对于查找数据来说，顺序查找是最容易理解的方法，也属于暴力查找技巧的一种，适合于元素随机排列的数组。
具体实现如下，时间复杂度O(n)
``` js
function seqSearch(arr, data) {
  for (let i = 0, len = arr.length; i < len; i++) {
    if (arr[i] === data) {
      return i
    }
  }
  return -1
}
```
因为在执行查找时可能会访问到数据结构里的所有元素，这种查找方法效率较低，尤其是数据量较大的情况下。
有一种优化的方法是使用自组织数据。这种策略具体是：通过将频繁查找到的元素置于数据集的起始位置来最小化查找次数。我们可以在程序运行过程中由程序自动组织数据，如下面的例子：
``` js
// 对于未排序数组，使用自组织数据，并通过简单的顺序查找快速找到元素
function seqSearch(arr, data) {
  for (let i = 0, len = arr.length; i < len; i++) {
    if (arr[i] === data) {
      if (i > 0) {
        swap(arr[i], arr[i - 1]) // 查找到的元素向前移动一位，逐渐将经常查找的元素移到最前
        return i - 1
      }
    }
  }
  return -1
}
```
### 二分查找算法
而对于有序的数据，二分查找算法比顺序查找算法更高效。
二分查找算法简单描述：
 1. 将数组的第一个位置设置为下边界，最后一个元素设置为上边界
 2. 若下边界小于等于上边界则将中点(mid)设置为(上边界 + 下边界) / 2
 3. 如果mid小于查询值，则将下边界设置为 mid + 1，重复步骤2、3；如果mid大于查询值，则将上边界设置为mid - 1，重复步骤2、3；否则返回mid

具体实现如下，时间复杂度O(log2n)。
``` js
function binSearch(arr, data) {
  const upperBound = arr.length - 1
  const lowerBound = 0
  while(lowerBound <= upperBound) {
    const mid = Math.floor((lowerCound + upperBound) / 2)
    if (data > arr[mid]) {
      lowerBound = mid + 1
    } else if (data < arr[mid]) {
      upperBound = mid - 1
    } else {
      return mid
    }
  }
  return -1
}
```
对于查找频繁的无序数据集，可以先进行一次快速排序然后实现二分查找算法。
### 数组去重
我们经常会遇到要对数组去重的情况。最容易理解的实现是遍历数组，进行顺序查找。具体实现如下，时间复杂度O(n^2)。
``` js
function unique(arr) {
  let uqArr = []
  for (let i = 0, len = arr.length; i < len; i++) {
    if (uqArr.indexOf(arr[i]) === -1) {
      uqArr.push(arr[i])
    }
  }
  return uqArr
}
```
时间复杂度更低的实现方式是利用散列表结构，但空间复杂度较高，用空间换时间。具体实现如下，时间复杂度O(n)。
``` js
function uniqueHash(arr) {
  let hash = {}
  let uqArr = []
  let item
  for (let i = 0, len = arr.length; i < len; i++) {
    item = arr[i]
    if(!hash[item]) {
      hash[item] = true
      uqArr.push(item)
    }
  }
  return uqArr
}
```
在上面的基础上，用开链法区分数据类型，解决如果数组的某元素是`__proto__`和碰撞的问题。
``` js
function uniqueHash2(arr) {
  let hash = Object.create(null)
  let uqArr = []
  let item
  for (let i = 0, len = arr.length; i < len; i++) {
    item = arr[i]
    if(!hash[item][typeof item]) {
      hash[item][typeof item] = true
      uqArr.push(item)
    }
  }
  return uqArr
}
```
## 高级算法
### 动态规划
我们可以通过与递归比较来理解动态规划：
 - 递归：从顶部开始将问题分解，通过解决掉所有分解出小问题的方式，来解决整个问题
 - 动态规划：从底部开始解决问题，将所有小问题解决掉，然后合并成一个整体解决方案，从而解决掉整个大问题

下面看一个简单的例子：
要实现斐波那契数列(0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55…)的计算，用递归的思想可以这样实现：
``` js
function recurFib(n) {
  if (n < 2) {
    return n
  } else {
    return recurFib(n - 1) + recurFib(n - 2)
  }
}
```
这是较容易理解的一种实现方式。但是，我们会发现，太多值在递归调用中被重新计算，当要计算的n较大时，执行效率很低。
这时，我们可以发现这个“大问题”是可以通过解决若干个“小问题”来解决的。而解决“小问题”有一个固定的公式，就是：`Fib(n) = Fib(n - 1) + Fib(n - 2)`。从而想到，用动态规划来实现：
``` js
function dynFib(n) {
  if (n < 3) {
    const fib = n < 2 ? n : 1
    return fib
  } else {
    const val = new Array(n + 1)
    val[1] = 1
    val[2] = 1
    for (let i = 3; i <= n; i++) {
      val[i] = val[i - 1] + val[i - 2]
    }
    return val[n]
  }
}
```
这种实现方式用数组保存了中间结果，效率更高。
以后遇到能用递归解决的问题，如果有子问题结构，可以考虑尝试用动态规划解决。

### 贪心算法
贪心算法的策略是：总是会选择当下的最优解，而不去考虑这一次的选择会不会对未来的选择造成影响。通过做出一系列的局部“最优”选择，有可能带来最终的整体“最优”选择，也有可能是“次优”选择。
下面是商店找零的例子，最优解是令使用的硬币数最少。
``` js
function makeChange(money) {
  let coins = []
  if (money % .25 < money) {
    coins[3] = parseInt(money / .25)
    money = money % .25
  }
  if (money % .1 < money) {
    coins[2] = parseInt(money / .1)
    money = money % .1
  }
  if (money % .05 < money) {
    coins[1] = parseInt(money / .05)
    money = money % .05
  }
  coins[0] = parseInt(money / .01)

  console.log(coins[0] + '个1美分')
  console.log(coins[1] + '个5美分')
  console.log(coins[2] + '个10美分')
  console.log(coins[3] + '个25美分')
}
```
在这种情况下，这种方案总是能找到最优解。但如果硬币的面额改为：.25, .6, .5, .3, .1。当遇到1时，按照贪心算法将分解为`.6 + .3 + .1`，而最优解应该是`.5 * 2`。
这里只举出几个简单的例子，关于动态规划和贪心算法还有更多的应用场景，这里不一一叙述了。


## 参考
 - 《数据结构与算法JavaScript描述》
 - [JavaScript专题之解读 v8 排序源码](https://github.com/mqyqingfeng/Blog/issues/52)
 - v8 array源码 (https://github.com/v8/v8/blob/master/src/js/array.js)