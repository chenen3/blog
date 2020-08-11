# go 切片使用 append 的一些特殊情况
> [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
> [Arrays, slices (and strings): The mechanics of ‘append’](https://blog.golang.org/slices) 

## 切片(slice)的内部实现
可以把切片想象成结构体，内部分别有一个指针指向底层数组，一个长度字段(表示当前元素数量)，一个容量字段(表示最多能容纳多少元素)。**切片变量本身是结构体的值，虽然隐含着底层数组指针，但切片本身不是指针**，此概念很重要。
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

### append操作，切片的底层数组可能被修改
注意是**可能**被修改。切片的容量是从指向的起始元素开始到*底层数组末尾*，容量无法增长。
1. 当切片容量足够大时，直接更新底层数组
```go
	slice := []int{0, 1, 2, 3}
	newslice := slice[1:2] 
	// len(newslice) == 1
	// cap(newclice) == 3
	newslice = append(newslice, 1024)
	// newslice == []int{1, 1024}
	// slice == []int{0, 1, 1024, 3}
```
2. 当切片容量不够时，底层数组无法容纳新元素，`append`创建切片副本，用新的更大的底层数组来容纳新元素(growslice)
```go
	slice := []int{0, 1, 2, 3}
	newslice := slice[2:]
	// newslice == []int{2, 3}
	// len(newslice) == 2
	// cap(newslice) == 2
	newslice = append(newslice, 1024)
	// newslice == []int{2, 3, 1024}
	// slice == []int{0, 1, 2, 3}
```

### append函数的返回值若不被赋值给原切片变量
一般使用append，会把返回值赋值给原来的切片变量。如果不这么做，会发现原切片没有任何改变。
```go
	slice := make([]int, 0, 4)
	// slice == []int{}
	newslice := append(slice, 1) 
	// slice == []int{}
	// newslice == []int{1}
```
参数传递给函数是值(拷贝)传递机制，切片变量本身是值(不是指针)，所以被传递给append函数操作的是拷贝后的**切片副本**，尽管切片副本与原切片共享底层数组，底层数组被修改了，但原切片变量的len和cap都没改，所以最后打印出来的切片变量依然不变。

综上，**使用append时记得要把返回值赋值给原切片变量**。
