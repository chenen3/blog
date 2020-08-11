# go example 与行尾空格
在一次非常普通的测试中，我发现一个诡异现象。例如测试代码：
```go
func Foo(n int) {
	for i := 0; i < n; i++ {
		fmt.Printf("%d ", i)
	}
	fmt.Print("\n")
}

func ExampleFoo() {
	Foo(3)
	Foo(5)
	// Output:
	// 0 1 2
	// 0 1 2 3 4
}
```

执行测试，出来下面的出错信息，got 和 want 看起来似乎一样，为什么还报错？
```
$ go test -v -run=^ExampleFoo$
=== RUN   ExampleFoo
--- FAIL: ExampleFoo (0.00s)
got:
0 1 2
0 1 2 3 4
want:
0 1 2
0 1 2 3 4
```

问题在于 Example 测试代码的output注释，它的行尾空格会被清除(如果执行go fmt，行尾空格也会被清除)。当目标函数在行尾输出空格，而测试程序期望没有行尾空格，测试结果则是失败。解决办法是目标函数不要打印行尾的空格。
```go
func Foo(n int) {
	for i := 0; i < n; i++ {
		if i == n-1 {
			fmt.Printf("%d", i)
		} else {
			// debug 把空格改成逗号，就能看出问题
			fmt.Printf("%d ", i)
		}
	}
	fmt.Print("\n")
}
```

### 总结
用`Example`测试代码时，注意目标程序是否有打印行尾空格，可能会因为行尾空格不一致而导致测试失败。如果总是测试失败，报错信息看起来又不像是有问题的样子，不妨把“打印空白符”改为“打印逗号”试试，看有没有“意想不到”。

