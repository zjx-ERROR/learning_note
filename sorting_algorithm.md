<!--
 * @Author: zhangjiaxi
 * @Date: 2021-03-05 15:18:44
 * @LastEditors: zhangjiaxi
 * @LastEditTime: 2021-03-08 15:48:26
 * @FilePath: /learning_note/sorting_algorithm.md
 * @Description: 
-->
# 算法概述
## 算法分类

### 比较类排序：通过比较来决定元素间的相对次序，由于其时间复杂度不能突破O(nlogn)，因此称为非线性时间比较类排序。
### 非比较类排序：不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此也称为线性时间非比较类排序。

![1](img/sorting_algorithm/1.png)

### 算法复杂度

![2](img/sorting_algorithm/2.png)

### 相关概念
- 稳定：如果a原本在b前面，而a=b，排序之后a仍然在b的前面
- 不稳定：如果a原本在b前面，而a=b，排序之后a可能出现在b后面
- 时间复杂度：对排序数据的总的操作次数。反映当n变化时，操作次数呈现什么规律。
- 空间复杂度：是指算法在计算机内执行时所需存储空间的度量，它也是数据规模n的函数

# 冒泡排序(Bubble Sort)

冒泡排序是一种简单的排序算法。它重复地走访要排序的数列，一次比较两个元素，如果它们的顺序错误就把它们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。

## 算法描述

- 比较相邻的元素。如果第一个比第二个大，就交换它们两个
- 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样再最后的元素就会是最值
- 针对所有元素重复以上步骤，除了最后一个，直到排序完成

![1.gif](img/sorting_algorithm/1.gif)

## 代码实现
```go
func BubbleSort(arr []int) []int {
	var j = len(arr) - 1
	for j > 0 {
		for i := 0; i < j; i++ {
			if arr[i] > arr[i+1] {
				arr[i], arr[i+1] = arr[i+1], arr[i]
			}
		}
		j--
	}
	return arr
}
```

# 选择排序(Selection Sort)

选择排序(Selection Sort)是一种简单直观的排序算法。它的工作原理是：首先在未排序序列中找到最值元素，存放到排序序列的起始位置，然后再从剩余未排序元素中继续寻找最值元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。

## 算法描述

n个记录的直接选择排序可经过n-1趟直接选择排序得到有序结果。具体算法描述如下：

- 初始状态：无序区为R[1..n]，有序区为空
- 第i趟排序(i=1,2,3...n-1)开始时，当前有序区和无序区分别为R(1..i-1)和R(i..n)。该趟排序从当前无序区中选出关键字最小的记录R[k]，将它与无序区的第1个记录R交换，使R[1..i]和R[i+1..n)分别变为记录个数增加1个的新有序区和记录个数减少1个的新无序区；
- n-1趟结束，数组有序化了。

![2.gif](img/sorting_algorithm/2.gif)

## 代码实现

```go
func SelectionSort(arr []int) []int {

	for i := 0; i < len(arr)-1; i++ {
		for j := i; j < len(arr); j++ {
			if arr[j] < arr[i] {
				arr[i], arr[j] = arr[j], arr[i]
			}
		}
	}
	return arr
}
```

# 插入排序(Insertion Sort)

插入排序(Insertion Sort)的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，再已排序序列中从后向前扫描，找到相应位置并插入。

## 算法描述

- 从第一个元素开始，该元素可以认为已经被排序
- 取出下一个元素，在已排序的元素序列中从后向前扫描
- 如果该元素（已排序）大于新元素，将该元素移到下一位置
- 重复步骤3,直到找到已排序元素大于或等于新元素的位置
- 将新元素插入到该位置后，重复步骤2～5

![3.gif](img/sorting_algorithm/3.gif)

## 代码实现

```go
func InsertionSort(arr []int) []int {
	for i := 1; i < len(arr); i++ {
		k := i
		for j := i - 1; j >= 0; j-- {
			if arr[k] < arr[j] {
				arr[k], arr[j] = arr[j], arr[k]
				k -= 1
			} else {
				break
			}
		}
	}
	return arr
}
```

# 希尔排序(Shell Sort)

第一个突破O(n<sup>2</sup>)的排序算法，是简单插入排序的改进版。它与插入排序的不同之处在于，它会优先比较距离较远的元素。希尔排序又叫缩小增量排序。

## 算法描述

- 将整个待排序的序列分割成若干子序列
- 选择一个增量序列t1,t2,...,tk，其中ti>tj,tk=1
- 按增量序列个数k，对序列进行k趟排序
- 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m的子序列，分别对各子表进行直接插入排序。仅增量因子为1时，整个序列作为一个表来处理，表长度即为整个序列长度。

![4.gif](img/sorting_algorithm/4.gif)

## 代码实现

```go
func ShellSort(num []int) []int {
	for i := len(num) / 2; i > 0; i /= 2 {
		for j := i; j < len(num); j++ {
			tmp := num[j]
			for t := j - i; t >= 0; t -= i {
				if tmp < num[t] {
					num[t+i] = num[t]
					num[t] = tmp
				} else {
					break
				}
			}
		}
	}
	return num
}
```

# 归并排序(Merge Sort)

归并排序时建立再归并操作上的一种有效的排序算法。该算法时采用分治法的一个典型应用。将已有序的子序列合并，得到完全有序的序列；即先使每个子序列有序，再使子序列段间有序。若将两个有序表合并成一个有序表，称为2路归并。

## 算法描述

- 把长度为n的输入序列分成两个长度为n/2的子序列
- 对这两个子序列分别采用归并排序
- 将两个排序好的子序列合并成一个最终的排序序列

![5.gif](img/sorting_algorithm/5.gif)

## 代码实现

```go
func MergeSort(num []int) []int {
	if len(num) < 2 {
		return num
	}
	left := MergeSort(num[:len(num)/2])
	right := MergeSort(num[len(num)/2:])
	res := merge(left, right)
	return res
}

func merge(left, right []int) []int {
	res := make([]int, 0)
	m, n := 0, 0
	l, r := len(left), len(right)
	for m < l && n < r {
		if left[m] > right[n] {
			res = append(res, right[n])
			n++
			continue
		}
		res = append(res, left[m])
		m++
	}
	res = append(res, right[n:]...)
	res = append(res, left[m:]...)
	return res
}
```

# 快速排序(Quick Sort)

通过一趟排序将待排序分割成独立的两部分，其中一部分记录的关键字比另一部分的关键字都小，则可分别对这两部分记录进行排序，以达到整个序列有序。

## 算法描述

- 从数列中挑出一个元素，称为“基准”
- 重新排序数列，所有元素比基准值小的放在基准前面，所有元素比基准值大的放在基准后面，在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区操作
- 递归地把小于基准值元素的子序列和大于基准值元素的子序列排序

![6.gif](img/sorting_algorithm/6.gif)

## 代码实现

```go

```