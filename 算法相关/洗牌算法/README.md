# Fisher-Yates Shuffle(抽牌)
### 思路
基本思想就是从原始数组中随机取一个之前没取过的数字到新的数组中。
- 记下从1到N的数字。
- 在1和剩余（含）的未固定数之间选择一个随机数k。
- 从低端开始计数，划掉尚未划掉的第k个数字，并将其写在单独列表的末尾。
- 从第2步开始重复，直到删除所有数字。
- 在第3步中写下的数字序列现在是原始数字的随机排列。

### 复杂度
时间复杂度为O(n*n)？,空间复杂度为O(n)

# Knuth-Durstenfeld Shuffle(换牌)
### 思路
在Fisher-Yates Shuffle基础上进行空间优化，每次从未处理的数据中随机取出一个数字，然后把该数字放在数组的尾部（即与末尾元素交换）。

### 复杂度
时间复杂度为O(n),空间复杂度为O(1)

### 缺点
需要在原数组上进行更改。

# Inside-Out Algorithm（插牌）
### 思路
Inside-Out Algorithm 算法的思想是从前往后，借助旧数组，将新数组中位置 k 和位置 i 的数字进行交互。
- 拷贝数组
- 从 i(0 - N)扫描数组，选择一个随机数 k( 0 <= k <= i)
- 新数组的[i] = 新数组的[k], 新数组的[k] = 原始数组[i]
- 重复第 2 步，直到末尾
- 最终的新数组就是随机的

### 示例代码
``` javascript
function randomArray(arr) {
  let newArr = arr.concat([])
  const length = arr.length
  for (let index = 0; index < length; index++) {
    const random = Math.floor(Math.random() * (index + 1))
    newArr[index] = newArr[random]
    newArr[random] = arr[index]
  }

  return newArr
}
```

### 复杂度
时间复杂度为O(n),空间复杂度为O(n)

# 参考资料
[三种洗牌算法shuffle](https://blog.csdn.net/qq_26399665/article/details/79831490)

[游戏常用算法-洗牌算法](https://www.cnblogs.com/millionsmultiplication/p/9570258.html)

[洗牌算法](https://zhuanlan.zhihu.com/p/60386034)

[洗牌算法](https://segmentfault.com/a/1190000019345601)

[Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#Fisher_and_Yates'_original_method)