# go struct{} 空结构体

发现一处特别的代码：`chan struct{}`，《Go并发编程实战》和《Go语言并发之道》这两本书的示例代码都出现了这个写法，里面提到“空结构体不占用内存”，但没细说原因。查资料了解了情况，在此做个笔记。

### 空结构体是什么

空结构体是没有字段的结构体。参考《The Go Programming Language》:

> The struct type with no fields is called the *empty struct*, written struct{}. It has size zero and carries no information but may be useful nonetheless

```go
type s struct {}
```

### 空结构体不占用内存

**width**是存储一个类型实例所占用的字节数。结构体的width是其所有字段width的总和再加上填充字节数，可以借助函数`unsafe.Sizeof()`查看。

```go
var s struct {
	i int64
	j int64
}
fmt.Println(unsafe.Sizeof(ss)) // print 16
```

空结构体没有字段，也不需要填充字节，所以宽度是零，意味着不占用存储空间。

```go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // print 0
```

### 空结构体的用途

#### 作为无意义的值

当map作为集合的时候，map的value可以用空结构体填充（常见用布尔值填充）。**不过这时空结构体节省的空间很少，而且语法显得更加笨重，所以我们一般会避免使用**。

```go
	seen := make(map[string]struct{}) // set of strings
	// ...
	if _, ok := seen[s]; !ok {
	seen[s] = struct{}{}
		// ...first time seeing s...
	}
```

当chan不在乎元素值的时候，可以用空结构体做元素，声明为`chan struct{}`。

```go
func foo() {
	done := make(chan struct{})
	go func() {
		time.Sleep(time.Second)
		close(done)
		// 或者
		// done <- struct{}{}
	}()

	for {
		select {
		case <-done:
			fmt.Println("done")
			return
		}
	}
}
```

#### 作为方法接收者

例如想实现一个inerface，不需要具体字段的时候：

```go
type Greeter interface {
	Greet()
}

type Cat struct{}

func (c Cat) Greet() {
	fmt.Println("miu...")
}
```

### 总结

当我们需要一个数据容器，却不在乎它的元素值的时候，可以考虑用空结构体来表示无意义的值。与此同时请小心避免把事情搞复杂。



参考资料：

>  [the-empty-struct](https://dave.cheney.net/2014/03/25/the-empty-struct) 
> The Go Programming Language

