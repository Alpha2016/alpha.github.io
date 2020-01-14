---
layout:     post
title:      Go slice array 的一点笔记
subtitle:   Go slice array 的一点笔记
date:       2020-01-14
author:     he xiaodong
header-img: img/default-post-bg.jpg
catalog: true
tags:
    - go
    - array
    - slice
---

##### 声明
类型 `[n]T` 表示拥有 n 个 T 类型的值的数组。声明数组 `var a [10]int` 会将变量 a 声明为拥有 10 个整数的数组。

数组的长度是其类型的一部分，因此数组不能改变大小。这看起来是个限制，不过没关系，Go 提供了更加便利的方式来使用数组。

每个数组的大小都是固定的。而切片则为数组元素提供动态大小的、灵活的视角。在实践中，切片比数组更常用。

类型 []T 表示一个元素类型为 T 的切片。切片通过两个下标来界定，即一个上界和一个下界，二者以冒号分隔：

a[low : high]它会选择一个半开区间，包括第一个元素，但排除最后一个元素。
以下表达式创建了一个切片，它包含 a 中下标从 1 到 3 的元素：`a[1:4]`

**切片并不存储任何数据，它只是描述了底层数组中的一段。更改切片的元素会修改其底层数组中对应的元素。与它共享底层数组的切片都会观测到这些修改。**

易错点：
```go
package main

import "fmt"

func main(){
	arr := [2]int{1,2}
	s := arr     // 直接这样相当于是数组的复制，go 数组是值复制
	//s := arr[:]   // 这样才是声明切片，才会根据数组值的改变而改变
	arr[1] = 0
	fmt.Print(s)
	fmt.Println(arr)
}
```

切片文法类似于没有长度的数组文法。
这是一个数组文法：`[3]bool{true, true, false}`
下面这样则会创建一个和上面相同的数组，然后构建一个引用了它的切片：
`[]bool{true, true, false}`

##### slice 特性
在进行切片时，你可以利用它的默认行为来忽略上下界。切片下界的默认值为 0，上界则是该切片的长度。
对于数组 `var a [10]int` 来说，以下切片是等价的：
```go
a[0:10]
a[:10]
a[0:]
a[:]
```

切片拥有 长度 和 容量。切片的长度就是它所包含的元素个数。切片的容量是从它的第一个元素开始数，到其底层数组元素末尾的个数。切片 s 的长度和容量可通过表达式 len(s) 和 cap(s) 来获取。你可以通过重新切片来扩展一个切片，给它提供足够的容量。试着修改示例程序中的切片操作，向外扩展它的容量，看看会发生什么。

slice 的 0 值是 `nil`

```go
package main

import "fmt"

func main() {
	var s []int
	fmt.Println(s, len(s), cap(s))
	if s == nil {
		fmt.Println("nil!")
	}
}
// output: [] 0 0
// output: nil!
```

**用 make 创建切片**
切片可以用内建函数 make 来创建，这也是你创建动态数组的方式。
make 函数会分配一个元素为零值的数组并返回一个引用了它的切片：
`a := make([]int, 5)  // len(a)=5`

要指定它的容量，需向 make 传入第三个参数：

```go
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

**切片可包含任何类型，甚至包括其它的切片。**

**向切片追加元素**
为切片追加新的元素是种常用的操作，为此 Go 提供了内建的 append 函数。内建函数的文档对此函数有详细的介绍。
`func append(s []T, vs ...T) []T`
append 的第一个参数 s 是一个元素类型为 T 的切片，其余类型为 T 的值将会追加到该切片的末尾。
append 的结果是一个包含原切片所有元素加上新添加元素的切片。
当 s 的底层数组太小，不足以容纳所有给定的值时，它就会分配一个更大的数组。返回的切片会指向这个新分配的数组。
扩容规则参见 [golangSlice的扩容规则](https://jodezer.github.io/2017/05/golangSlice%E7%9A%84%E6%89%A9%E5%AE%B9%E8%A7%84%E5%88%99)

##### 使用
for 循环的 range 形式可遍历切片或映射。
当使用 for 循环遍历切片时，每次迭代都会返回两个值。第一个值为当前元素的下标，第二个值为该下标所对应元素的一份副本。

```go
package main

import "fmt"

var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
	for i, v := range pow {
		fmt.Printf("2**%d = %d\n", i, v)
	}
}
```

可以将下标或值赋予 _ 来忽略它。
```go
for i, _ := range pow
for _, value := range pow
```
若你只需要索引，忽略第二个变量即可。
```go
for i := range pow
```

还有搜到的一段有趣的代码
```go
package main

import "golang.org/x/tour/pic"

func Pic(dx, dy int) [][]uint8 {
	// 外层slice
	a := make([][]uint8, dy)
	for x := range a {
		// 里层slice
		b := make([]uint8, dx)
		for y := range b {
			// 给里层slice每个元素赋值
			b[y] = uint8(x +y)
		}
		// 给外层slice每个元素赋值
		a[x] = b
	}
	return a
}

func main() {
	pic.Show(Pic)
}
```

内容来自 [go 官方指南](https://tour.go-zh.org/moretypes/18)

