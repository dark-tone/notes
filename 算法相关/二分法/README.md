## 定义
对于区间[a，b]上连续不断且f（a）·f（b）<0的函数y=f（x），通过不断地把函数f（x）的零点所在的区间一分为二，使区间的两个端点逐步逼近零点，进而得到零点近似值的方法叫二分法。

## 时间复杂度
O(log(n))

## 模板
```
...
// 需要根据实际情况判断边界情况的去留问题
left, right := 0, len(data) - 1
  for left <= right {
    pointer := (left + right) / 2
    // pointer := left + (right - left) / 2
    if data[pointer] > k {
      right = pointer - 1
    } else if data[pointer] < k {
      left = pointer + 1
    } else if data[pointer] == k {
      // do something
    }
  }
```
## 问题1：数字在升序数组中出现的次数
#### 简要描述
给定一个长度为 n 的非降序数组和一个非负数整数 k ，要求统计 k 在数组中出现的次数

#### 思路
（略）

## 问题2：旋转数组的最小数字 
#### 描述
有一个长度为 n 的非降序数组，比如[1,2,3,4,5]，将它进行旋转，即把一个数组最开始的若干个元素搬到数组的末尾，变成一个旋转数组，比如变成了[3,4,5,1,2]，或者[4,5,1,2,3]这样的。请问，给定这样一个旋转数组，求数组中的最小值。

#### 思路
1. 数组可以分为两个有序的子数组。其中，左排序的数组的值大于右排序数组中的值。
2. 声明left,right 分别指向数组的左右两端；
3. mid = (left+right) / 2 为二分的中间位置。
4. mid，left,right分为三种情况：<br>
    a. rotateArray[mid] > rotateArray[right]时， 那么 最小值一定在 [mid+1,right]区间中；<br>
    b. rotateArray[mid] < rotateArray[right]时，那么最小值一定在[left,mid]区间内。（注意这里不能-1）<br>
    c. rotateArray[mid] = rotateArray[right]时，无法判断最小值在哪个区间，所以此时只能缩小right的值。


## 问题3：0～n-1中缺失的数字
（略）