---
layout:       post
title:        "通过一道 leetcode 算法题更加深刻理解 golang struct 底层结构"
description:  "通过实践深入理解 golang struct 结构体的底层实现，并避免在实际应用中踩坑"
author:       "Quinlan"
header-style: text
catalog:      true
tags:
    - ['leetcode', 'golang原理', '数据结构']
---

# 通过一道 leetcode 算法题更加深刻理解 golang struct 底层结构

今天用 golang 写 leetcode 算法第 427 题，处理二维数组构成的矩阵时，暴露了自己对 golang struct 底层结构的认识仍然不够深刻的问题，特此总结。

题目背景不再赘述，可以直接去 leetcode 网址看原始题目，[题目链接](https://leetcode.com/problems/construct-quad-tree/description/)。这里将其中重要部分摘取出来
> 假定 `grid` 是一个二维数组，`n*n`，其中 `n=2^x`，即是一个正方形矩阵，现将 `grid` 切分为 `topLeft`, `topRight`, `bottomLeft`, `bottomRight` 四个子矩阵，其同样是正方形的。
![leetcode 427题示意图](https://assets.leetcode.com/uploads/2020/02/11/new_top.png)

这里的题意相对还是比较简单，就是将一个正方形横向和纵向平分为4个小正方形。

```golang
func split(grid [][]int) (tl, tr, bl, br [][]int) {
	n := getDimension(grid)
	if n < 2 {
		return nil, nil, nil, nil
	}

	// 4 grids must has same shape
	n = n >> 1
	topLeft := grid[0:n][0:n]
	topRight := grid[0:n][n : n<<1]
	bottomLeft := grid[n : n<<1][0:n]
	bottomRight := grid[n : n<<1][n : n<<1]

	return topLeft, topRight, bottomLeft, bottomRight
}
```

非常快速地写了一个 split 函数，还为自己使用了位移运算进行2的幂次的乘除而有点飘飘然，却不知道这里完全是一种想当然的写法。毫无疑问，这里存在数组越界的问题。

我们来重点分析下 `grid[0:n][0:n]` 这个表达式是什么含义，是把第一维的前n位和第二位的前n位的元素取出来吗？那 `grid[0:n][0:n][0:n]` 会不会报编译错误呢？

这里引用 draveness 的解释，golang 里的 slice 就是一个如下的结构体 
```golang
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
对这个结构体核心理解要点是：Data 是指向一片连续内存空间的头指针，连续空间已经使用的容量以及总共的容量，分别有 Len 和 Cap 表示。
所以对切片的操作只是对结构体的操作，并不会真正发生内存拷贝。 

那么回到这里的问题，对 `grid[0:n]` 具体是什么含义，用如下的伪代码表示
```golang
target := SliceHeader{
    Data: grid.Data,
    Len: n,
    Cap: n
}
```
同样 `grid[0:n][0:n]` 并没有所谓的二维操作，只是将 `grid[0:n]`的操作重复了两次，但是 `grid[0:n][n:n<<1]` 就会存在数组越界的问题了，这里 grid[0:n] 是一个 len = n  的切片, 但尝试获取其 `[n:n<<1]`的数据，这显然是越界访问。

所以应该修改其代码实现。快速地就写出了第二版代码
```golang
func split(grid [][]int) (tl, tr, bl, br [][]int) {
	n := getDimension(grid)
	if n < 2 {
		return nil, nil, nil, nil
	}

	// 4 grids must has same shape
	n = n >> 1
	topLeft := grid[0:n]
	for i := range n {
		topLeft[i] = topLeft[i][0:n]
	}

	topRight := grid[0:n]
	fmt.Println(topRight)
	for i := range n {
		topRight[i] = topRight[i][n : n<<1]
	}

	bottomLeft := grid[n : n<<1]
	fmt.Println(bottomLeft)
	for i := range n {
		bottomLeft[i] = bottomLeft[i][0:n]
	}

	bottomRight := grid[n : n<<1]
	fmt.Println(bottomRight)
	for i := range n {
		bottomRight[i] = bottomRight[i][n : n<<1]
	}

	return topLeft, topRight, bottomLeft, bottomRight
}
```

这里的逻辑也非常简单，既然无法通过连续 `[][]` 来完成二维数组变量的取值，那我分两次，第一次先取第一维的前n个或者后n的元素，然后通过 for 循环对第二维数组再次进行切分。

这样应该就没问题了把，测试用例运行！

叮！再次数组越界。这次错误发生在 `topRight[i] = topRight[i][n : n<<1]`，仔细看看这行代码，没什么问题啊，第一维和第二维变量都不可能越界啊。通过单步调试，进去后发现，topRight 第二维竟然只有 n 个元素，这是为什么呢？

再次进行单步调试，发现topRight 在从 grid 取值后，变量就只有 n 个元素了。通过单步调试可以发现，在 `topLeft[i] = topLeft[i][0:n] ` 执行完成后，grid 的变量竟然也发生了变化！！！

这让人非常惊讶，这是为什么，我只是修改了 SliceHeader，并没有对 grid 做任何操作。

我们认真思考下，
```golang
	topLeft := grid[0:n]
	for i := range n {
		topLeft[i] = topLeft[i][0:n]
	}
```
topLeft 和 grid 究竟发生了什么？

![](https://img.draveness.me/2019-02-20-golang-slice-struct.png)

这里如果读者有 C 语言的背景，可以认为 grid 里的元素是数组指针，即其元素里每个变更都是一个数组的头指针。所以这里 topLeft 同样也是指向这片内存区域的 SliceHeader.

![](https://cdn.z.wiki/autoupload/20240918/9oTl/555X261/golang-slice-header.png)

虽然 topLeft 是一个新的 SliceHeader 变量，但仍然和grid 共享底层的内存空间。

所以 ptr1 指向空间的头地址没有发生变化，但其 Len 和 Cap 变量发生了变化，导致后续 topRight 变量继续处理时，发生了数组越界。

所以这里的核心要点就是，避免 topLeft 变量和 grid 变量共享同一片内存空间。所以这里需要使用内建函数 copy 进行深拷贝。

![深拷贝](https://cdn.z.wiki/autoupload/20240918/Kqo6/555X261/golang-slice-header-deep-copy.png)

这里可以看到 tl 并没有和 grid 共享内存空间，其分别控制着自己的变量地址，这样可以保证修改元素后，不会影响其他元素获取这里的元素。

完整代码如下
```golang
func split(grid [][]int) (tl, tr, bl, br [][]int) {
	n := getDimension(grid)
	if n < 2 {
		return nil, nil, nil, nil
	}

	// 4 grids must has same shape
	n = n >> 1
	topLeft := make([][]int, n)
	copy(topLeft, grid[0:n])
	for i := range n {
		topLeft[i] = topLeft[i][0:n]
	}

	topRight := make([][]int, n)
	copy(topRight, grid[0:n])
	for i := range n {
		topRight[i] = topRight[i][n : n<<1]
	}

	bottomLeft := make([][]int, n)
	copy(bottomLeft, grid[n:n<<1])
	for i := range n {
		bottomLeft[i] = bottomLeft[i][0:n]
	}

	bottomRight := make([][]int, n)
	copy(bottomRight, grid[n:n<<1])
	for i := range n {
		bottomRight[i] = bottomRight[i][n : n<<1]
	}

	return topLeft, topRight, bottomLeft, bottomRight
}
```