---
title: 常见排序算法
tags: [排序算法]
categories: [Go, 算法]
date: 2020-05-10 00:00:00
toc: true
---

本文介绍了`Go`语言版经典的排序算法–选择排序、冒泡排序、插入排序、快速排序、归并排序和堆排序。<!--more-->

## 排序算法

本文使用泛型时需要导入 `golang.org/x/exp/constraints` 包。

### 选择排序

```go
func selectionSort[T constraints.Ordered](arr []T) {
	if len(arr) < 2 || arr == nil {
		return
	}
	for i := 0; i < len(arr)-1; i++ { // i ~ N-1
		minIndex := i
		for j := i + 1; j < len(arr); j++ { // i ~ N-1上找最小值的下标
			if arr[j] < arr[minIndex] {
				minIndex = j
			}
		}
		swap(arr, i, minIndex)
	}
}

func swap[T constraints.Ordered](data []T, i, j int) {
	tmp := data[i]
	data[i] = data[j]
	data[j] = tmp
}
func main() {
    s := make([]float32, 0, 16)
	rand.NewSource(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Float32()*100.0)
	}
	fmt.Println(s)
	selectionSort(s)
	fmt.Println(s)
}
```

### 冒泡排序

```go
func bubbleSort[T constraints.Ordered](arr []T) {
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

func swap[T constraints.Ordered](arr []T, i, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func main() {
	s := make([]int64, 0, 16)
	rand.NewSource(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, int64(rand.Int63n(100)))
	}
	fmt.Println(s)
	bubbleSort(s)
	fmt.Println(s)
}
```

### 插入排序

```go
func insertSort[T constraints.Ordered](arr []T) {
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

func swap[T constraints.Ordered](arr []T, i, j int) {
	tmp := arr[i]
	arr[i] = arr[j]
	arr[j] = tmp
}

func main() {
	s := make([]int, 0, 16)
    rand.NewSource(time.Now().UnixNano())
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
func quickSort[T constraints.Ordered](arr []T) {
	if len(arr) <= 1 {
		return
	}
	base := arr[0]
	l, r := 0, len(arr)-1
	for i := 1; i <= r; {
		if arr[i] > base {
			arr[i], arr[r] = arr[r], arr[i]
			r--
		} else {
			arr[i], arr[l] = arr[l], arr[i]
			l++
			i++
		}
	}
	quickSort(arr[:l])
	quickSort(arr[l+1:])
}

func main() {
	s := make([]int32, 0, 16)
	for i := 0; i < 16; i++ {
		s = append(s, rand.Int31n(100))
	}
	fmt.Println(s)
	quickSort(s)
	fmt.Println(s)
}
```

### 归并排序

```go
func mergeSort[T constraints.Ordered](data []T) []T {
	length := len(data)
	if length <= 1 {
		return data
	}
	num := length / 2
	left := mergeSort(data[:num])
	right := mergeSort(data[num:])
	return merge(left, right)
}

func merge[T constraints.Ordered](left, right []T) (result []T) {
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
	s := make([]float64, 0, 16)
    rand.NewSource(time.Now().UnixNano())
	for i := 0; i < 16; i++ {
		s = append(s, rand.Float64()*100.0)
	}
	fmt.Println(s)
	s = mergeSort(s)
	fmt.Println(s)
}
```

### 堆排序

```go
func heapSort[T constraints.Ordered](arr []T) {
	m := len(arr)
	s := m / 2
	for i := s; i > -1; i-- {
		heap(arr, i, m-1)
	}
	for i := m - 1; i > 0; i-- {
		arr[i], arr[0] = arr[0], arr[i]
		heap(arr, 0, i-1)
	}
}

func heap[T constraints.Ordered](arr []T, i, end int) {
	l := 2*i + 1
	if l > end {
		return
	}
	n := l
	r := 2*i + 2
	if r <= end && arr[r] > arr[l] {
		n = r
	}
	if arr[i] > arr[n] {
		return
	}
	arr[n], arr[i] = arr[i], arr[n]
	heap(arr, n, end)
}

func main() {
	s := []string{"guangzhou", "shenzhen", "chengdu", "shanghai", "beijing"}
	fmt.Println(s)
	heapSort(s)
	fmt.Println(s)
}
```