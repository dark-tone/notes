# 冒泡排序
重复地走访过要排序的元素列，依次比较两个相邻的元素，如果顺序（如从大到小、首字母从Z到A）错误就把他们交换过来。

### 代码示例
``` GO
func BubbleSort(arr []int) {
	if arr == nil || len(arr) < 2 {
		return
	}
	for i := 0; i < len(arr); i++ {
		for j := 0; j < len(arr)-i-1; j++ {
			if arr[j] > arr[j+1] {
				temp := arr[j]
				arr[j] = arr[j+1]
				arr[j+1] = temp
			}
		}
	}
}
```

### 复杂度
平均时间复杂度：O($n^2$)<br>
最好时间复杂度：O(n)<br>
最坏时间复杂度：O($n^2$)<br>
空间复杂度：O(1)


# 选择排序
第一次从待排序的数据元素中选出最小（或最大）的一个元素，存放在序列的起始位置，然后再从剩余的未排序元素中寻找到最小（大）元素，然后放到已排序的序列的末尾。以此类推，直到全部待排序的数据元素的个数为零。

### 代码示例
``` GO
func SelectionSort(arr []int) {
	if arr == nil || len(arr) < 2 {
		return
	}
	var minIdx, temp int
	for i := 0; i < len(arr) - 1; i++ {
		minIdx = i
		for j := i + 1; j< len(arr); j++ {
			if arr[j] < arr[minIdx] {
				minIdx = j
			}
		}
		temp = arr[i]
		arr[i] = arr[minIdx]
		arr[minIdx] = temp
	}
}
```

### 复杂度
平均时间复杂度：O($n^2$)<br>
最好时间复杂度：O($n^2$)<br>
最坏时间复杂度：O($n^2$)<br>
空间复杂度：O(1)


# 插入排序
插入排序是指在待排序的元素中，假设前面n-1(其中n>=2)个数已经是排好顺序的，现将第n个数插到前面已经排好的序列中，然后找到合适自己的位置，使得插入第n个数的这个序列也是排好顺序的。

### 代码示例
``` GO
func InsertionSort(arr []int) {
    if arr == nil || len(arr) < 2 {
		return
	}
	var preIdx, current int
	for i := 1; i < len(arr); i++ {
		preIdx = i - 1
		current = arr[i]
		for preIdx >= 0 && arr[preIdx] > current {
			arr[preIdx + 1] = arr[preIdx]
			preIdx--
		}
		arr[preIdx + 1] = current
	}
}
```


### 复杂度
平均时间复杂度：O($n^2$)<br>
最好时间复杂度：O(n)<br>
最坏时间复杂度：O($n^2$)<br>
空间复杂度（不新建数组）：O(1)

# 快速排序
快速排序的基本思想：通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。

### 代码示例
``` GO
func QuickSort(arr []int, left int, right int) {
	var partitionIndex int
	if left < right {
		partitionIndex = Partition(arr, left, right)
		QuickSort(arr, left, partitionIndex-1)
		QuickSort(arr, partitionIndex+1, right)
	}
}

// 找出分界
func Partition(arr []int, left int, right int) int {
	pivot := left
	index := pivot + 1
	for i := index; i <= right; i++ {
		if arr[i] < arr[pivot] {
			Swap(arr, i, index)
			index++
		}
	}
	Swap(arr, pivot, index-1)
	return index - 1
}

func Swap(arr []int, i int, j int) {
	var temp = arr[i]
	arr[i] = arr[j]
	arr[j] = temp
}
```

### 复杂度
平均时间复杂度：O($nlog_2n$)<br>
最好时间复杂度（每次的基准都刚好平分数组）：O($nlog_2n$)<br>
最坏时间复杂度（当每次基准都是最小/最大时，退化为冒泡排序）：O($n^2$)<br>

# 归并排序
归并排序（Merge Sort）是建立在归并操作上的一种有效，稳定的排序算法，该算法是采用分治法（Divide and Conquer）的一个非常典型的应用。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为二路归并。

### 代码示例
``` GO
func MergeSort(arr []int) []int {
	if len(arr) < 2 {
		return arr
	}
	middle := len(arr) / 2
	left := arr[0:middle]
	right := arr[middle:len(arr)]
	return Merge(MergeSort(left), MergeSort(right))
}

func Merge(left, right []int) []int {
	var result []int

	for len(left) > 0 && len(right) > 0 {
		if left[0] <= right[0] {
			result = append(result, left[0])
			left = left[1:len(left)]
		} else {
			result = append(result, right[0])
			right = right[1:len(right)]
		}
	}
	for _, v := range left {
		result = append(result, v)
	}
	for _, v := range right {
		result = append(result, v)
	}
	return result
}
```

### 复杂度
平均时间复杂度：O($nlog_2n$)<br>
最好时间复杂度：O($nlog_2n$)<br>
最坏时间复杂度：O($nlog_2n$)<br>
空间复杂度：O(n)

