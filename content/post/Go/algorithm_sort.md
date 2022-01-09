---
title: 常见排序算法
tags: [排序算法]
categories: [Go, 算法]
date: 2020-05-10 00:00:00
toc: true
---

本文介绍了Go语言版经典的排序算法–选择排序、冒泡排序、插入排序、快速排序、归并排序和堆排序。<!--more-->

## 排序算法

### 选择排序

```go
func selectionSort(data []int) {
    if len(data) < 2 || data == nil {
        return
    }
    for i := 0; i < len(data)-1; i++ { // i ~ N-1
        minIndex := i
        for j := i + 1; j < len(data); j++ { // i ~ N-1上找最小值的下标
            if data[j] < data[minIndex] {
                minIndex = j
            }
        }
        swap(data, i, minIndex)
    }
}

func swap(data []int, i, j int) {
    tmp := data[i]
    data[i] = data[j]
    data[j] = tmp
}

func main() {
    s := make([]int, 0, 16)
    rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Intn(100))
	}
	fmt.Println(s)
    selectionSort(s)
	fmt.Println(s)
}
```

### 冒泡排序

```go
func bubbleSort(arr []int) {
	if arr == nil || len(arr) < 2 {
		return
	}
	for e := len(arr) - 1; e > 0; e-- { // 0 ~ e
		for i := 0; i < e; i++ {
			if arr[i] > arr[i+1] {
				swap(arr, i, i+1)
			}
		}
	}
}

// 交换arr的i和j位置上的值
func swap(arr []int, i, j int) {
	// arr[i] = arr[i] ^ arr[j]
	// arr[j] = arr[i] ^ arr[j]
	// arr[i] = arr[i] ^ arr[j]
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func main() {
	s := make([]int, 0, 16)
    rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Intn(100))
	}
	fmt.Println(s)
	bubbleSort(s)
	fmt.Println(s)
}
```

### 插入排序

```go
func insertSort(arr []int) {
	if arr == nil || len(arr) < 2 {
		return
	}
	// 0~0有序的
	// 0~i想有序
	for i := 1; i < len(arr); i++ { // 0 ~ i 做到有序
		for j := i - 1; j >= 0; j-- {
			if arr[j] > arr[j+1] {
				swap(arr, j, j+1)
			}
		}
	}
}

func swap(arr []int, i, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func main() {
	s := make([]int, 0, 16)
    rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Intn(100))
	}
	fmt.Println(s)
	insertSort(s)
	fmt.Println(s)
}
```

### 快速排序

```go
func quickSort(arr []int) {
	if arr == nil || len(arr) < 2 {
		return
	}
	sort(arr, 0, len(arr)-1)
}

// arr[l..r]排好序
func sort(arr []int, L, R int) {
	if L < R {
		rand.Seed(time.Now().UnixNano())
		swap(arr, L+rand.Intn(R-L+1), R)
		p := partition(arr, L, R)
		sort(arr, L, p[0]-1) // < 区
		sort(arr, p[1]+1, R) // > 区
	}
}

// 这是一个处理arr[l..r]的函数
// 默认以arr[r]做划分，arr[r] -> p    <p   ==p   >p
// 返回等于区域(左边界, 右边界)，所以返回一个长度为2的数组res，res[0] res[1]
func partition(arr []int, L, R int) []int {
	less := L - 1  // <区右边界
	more := R      // >区左边界
	for L < more { // L表示当前数的位置 arr[R] -> 划分值
		if arr[L] < arr[R] { // 当前数 < 划分值
			less++
			swap(arr, less, L)
			L++
		} else if arr[L] > arr[R] { // 当前数 > 划分值
			more--
			swap(arr, more, L)
		} else {
			L++
		}
	}
	swap(arr, more, R)
	return []int{less + 1, more}
}

func swap(arr []int, i, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func main() {
	arr := make([]int, 0, 16)
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		arr = append(arr, rand.Intn(100))
	}
	fmt.Println(arr)
	quickSort(arr)
	fmt.Println(arr)
}
```

### 归并排序

```go
func mergeSort(data []int) []int {
	length := len(data)
	if length <= 1 {
		return data
	}
	num := length / 2
	left := mergeSort(data[:num])
	right := mergeSort(data[num:])
	return merge(left, right)
}

func merge(left, right []int) (result []int) {
	l, r := 0, 0
	for l < len(left) && r < len(right) {
		if left[l] < right[r] {
			result = append(result, left[l])
			l++
		} else {
			result = append(result, right[r])
			r++
		}
	}
	result = append(result, left[l:]...)
	result = append(result, right[r:]...)
	return
}

func main() {
	s := make([]int, 0, 16)
    rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Intn(100))
	}
	fmt.Println(s)
	s = mergeSort(s)
	fmt.Println(s)
}
```

### 堆排序

```go
func heapSort(array []int) {
	m := len(array)
	s := m / 2
	for i := s; i > -1; i-- {
		heap(array, i, m-1)
	}
	for i := m - 1; i > 0; i-- {
		array[i], array[0] = array[0], array[i]
		heap(array, 0, i-1)
	}
}

func heap(array []int, i, end int) {
	l := 2*i + 1
	if l > end {
		return
	}
	n := l
	r := 2*i + 2
	if r <= end && array[r] > array[l] {
		n = r
	}
	if array[i] > array[n] {
		return
	}
	array[n], array[i] = array[i], array[n]
	heap(array, n, end)
}

func main() {
	s := make([]int, 0, 16)
    rand.Seed(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Intn(100))
	}
	fmt.Println(s)
	heapSort(s)
	fmt.Println(s)
}
```