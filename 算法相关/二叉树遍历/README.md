# 1 层次遍历
## 1.1 概念
从上往下逐层遍历元素。

## 1.2 思路
使用队列，先进先出。

## 1.3 代码示例
``` golang
// 结点
type Node struct {
	Val int
	Left *Node
	Right *Node
}
// 层次遍历
func LevelTraversal(root *Node) []int  {
	if root == nil {
		return []int{}
	}
	var result []int
	queue := []*Node{root}
	var item *Node
	for {
		if len(queue) == 0 {
			break
		}
		item = queue[0]
		queue = queue[1:len(queue)]
		result = append(result, item.Val)
		if item.Left != nil {
			queue = append(queue, item.Left)
		}
		if item.Right != nil {
			queue = append(queue, item.Right)
		}
	}
	return result
}
```

# 2 深度遍历
## 2.1 分类
按中间结点的遍历顺序，可分为：
- 先序遍历<br>
中 → 左节点 → 右节点
- 中序遍历<br>
左节点 → 中 → 右节点
- 后序遍历<br>
左节点 → 右节点 → 中

## 2.2 代码实现
- 递归<br>
``` golang
// 中序遍历
func DepthTraversal(root *Node) []int {
    if root == nil {
        return []int{}
    }
    leftResult := DepthTraversal(root.Left)
    rightResult := DepthTraversal(root.Right)
    return append(append(leftResult, root.Val), rightResult...)
}
```
- 使用栈
