---
author: Alex Shi
pubDatetime: 2024-3-31T14:21:00+00:00
# 2024-03-29-T14:15:00+08:00
title: Golang-逃逸分析
postSlug: "escape-analysis-in-golang"
featured: false
draft: false
tags:
  - Golang
  - 内存管理
---

看一段示例

```go
package main

//go:noinline
func foo() *int {

	var val_3 int = 3

	println(&val_1, &val_2, &val_3, &val_4)
	return &val_3
}

func main() {
	ptr := foo()
	println(*ptr, ptr)
}

```

使用 C/C++ 语言的开发者看到这段代码会很奇怪，一个函数怎么可以返回一个局部变量，理论上来讲，随着 foo 函数的生命周期结束，局部变量都应该销毁才对。

但是编译这段 go 程序却不会报错，程序正常运行。这是因为 Golang 的开发者希望使用这门语言的程序员专注于程序逻辑本身，这种繁琐的事情交给 Golang 编译器就好。

## Golang 编译器的逃逸分析

Golang 编译器会自动决定把一个变量放在栈还是放在堆，编译器在编译期间会做 **逃逸分析** (escape analysis)，**当发现变量的作用域没有跳脱出函数范围，就可以在分配在栈上，反之则必须分配在堆**。

看这样一段示例：

```go
package main

//go:noinline
func foo() *int {
	var val_1 int = 1
	var val_2 int = 2
	var val_3 int = 3
	var val_4 int = 4

	println(&val_1, &val_2, &val_3, &val_4)
	return &val_3
}

func main() {
	ptr := foo()
	println(*ptr, ptr)
}
```

`//go:noinline` 的作用是防止编译器对 foo 函数进行内联优化，内联优化之后 foo 会在 main函数中进行内联展开，效果和没有函数调用一样，相当于直接拷贝 foo 函数体的代码到 main 执行。

运行得到的结果：

```shell
$ go run main.go
0xc000048708 0xc000048700 0xc000096000 0xc0000486f8
3 0xc000096000
```

我们能看到 `val_3` 是返回给 main 的局部变量, 其中他的地址应该是 `0xc000096000` ,很明显与其他的val_1,2,4是不同的。

用go tool compile测试一下：

```shell
 go tool compile -m main.go
/home/alexshi/code/experiment/go-ex/escape/pro_5/main.go:14:6: can inline main
/home/alexshi/code/experiment/go-ex/escape/pro_5/main.go:7:6: moved to heap: val_3
```

`main.go:7:6: moved to heap: val_3` ：果然，在编译的时候， foo_val3 被编译器判定为逃逸变量, 将foo_val3放在堆中开辟。

## new 的变量一定在堆上吗

看这样一段示例：

```go
package main

//go:noinline
func foo() *int {

	var val_1 *int = new(int)
	var val_2 *int = new(int)
	var val_3 *int = new(int) // escapes to heap
	var val_4 *int = new(int)

	println(val_1, val_2, val_3, val_4)

	//返回foo_val3给main函数
	return val_3
}

func main() {
	ptr := foo()

	println(*ptr, ptr)
}
```

我们将val_1~4全部用new的方式来分配内存, 编译运行看结果

```shell
$ go run main.go
0xc000048708 0xc000048700 0xc000012070 0xc0000486f8
0 0xc000012070
```

显然, `val_3` 的地址 `0xc000012070` 依然与其他的不是连续的. 依然具备逃逸行为。

用go tool compile测试一下没有问题：`main.go:8:22: new(int) escapes to heap`

```shell
$ go tool compile -m main.go
/home/alexshi/code/experiment/go-ex/escape/pro_6/main.go:17:6: can inline main
/home/alexshi/code/experiment/go-ex/escape/pro_6/main.go:6:22: new(int) does not escape
/home/alexshi/code/experiment/go-ex/escape/pro_6/main.go:7:22: new(int) does not escape
/home/alexshi/code/experiment/go-ex/escape/pro_6/main.go:8:22: new(int) escapes to heap
/home/alexshi/code/experiment/go-ex/escape/pro_6/main.go:9:22: new(int) does not escape
```

结论：Golang中一个函数内局部变量，不管是不是动态new出来的，它会被分配在堆还是栈，是由编译器做逃逸分析之后做出的决定。逃逸分析后，如果编译器发现这个变量在该函数结束后不会再调用了，就会把这个变量分配到栈上，毕竟使用栈速度快、不会产生内存碎片、函数执行结束后自动销毁。如果编译器发现某个变量在函数之外还有其他地方要引用，那么就会把这个变量分配到堆上。

## Refence

- [Golang中逃逸现象, 变量“何时栈?何时堆?](https://www.yuque.com/aceld/golang/yyrlis)
- [Go中new到底在堆还是栈中分配](https://learnku.com/articles/71370)
- [Stackoverflow: forbid inlining in golang](https://stackoverflow.com/questions/68280753/forbid-inlining-in-golang)
