## 定义
定义两个指针，以不同速度移动两个指针，在处理循环链表或数组时非常有用。

## 问题1：判断一个链表中是否含有环
#### 思路
同时将fast和slow置为开头节点，一快一慢进行移动，如果有环则必相遇。

## 问题2：若一个链表中含有环，找到环开始的位置
#### 思路
1. 初始化：快指针fast指向头结点， 慢指针slow指向头结点
2. 让fast一次走两步， slow一次走一步，第一次相遇在C处，停止
3. 然后让fast指向头结点，slow原地不动，然后fast，slow每次走一步，当再次相遇，就是入口结点。

#### 原理
可列方程进行求证，设slow走过路程为x，环长为y，则有2x = x + ny，即x = ny，那么fast第二次从头开始则会在环的起点处相遇。

## 问题3：求有序链表的值域的中位数
#### 思路
fast每次走两步，slow走一步，则fast到头时slow刚好位于中间（需要注意链表的长度的奇偶数问题）。

## 问题4：求链表最后k个节点
#### 思路
fast比slow先走k步，注意边界情况即可。

## 问题5：判断链表是否是回文链表
#### 思路
用快慢指针找到链表的中点， 把后半部分链表逆序，再和前半部分链表做比较是否每个元素都相等。



## 参考文章
[leetcode算法汇总 （三）快慢指针](https://zhuanlan.zhihu.com/p/72886883)