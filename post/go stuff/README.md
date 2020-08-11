# go stuff

## 如何修改url参数

直接对URL.Query()的返回值做Add操作，并不会修改到原始URL，正确做法是更新`URL.RawQuery`字段。

```go
s := "http://foo.com"
u, _ := url.Parse(s)
u.Query().Add("bar", "1")
fmt.Println(u.String()) // http://foo.com

query := u.Query()
query.Add("bar", "1")
u.RawQuery = query.Encode()
fmt.Println(u.String()) // http://foo.com?bar=1
```

## 为什么有些结构体方法的 receiver 是指针

**结构体方法调用时，receiver作为参数被传给方法本身，遵循值传递**。想要在方法内部修改receiver，则要传递它的指针。如果receiver是值类型，则方法内部拿到的是receiver的拷贝副本，修改副本并不会影响原来的receiver。

```go
package main

import "fmt"

type foo struct {
	s string
}

func (f foo) Update() {
	f.s = "updated"
}

func main() {
	f := foo{}
	oldF := f
	f.Update()
	newF := f
	fmt.Printf("f updated: %t", oldF != newF)
	// f updated: false
}
```

## unsafe.Pointer

Pointer 代表一个指向任意类型的指针
Pointer专有的转换操作： **普通指针 <—> Pointer <—>  uintptr**

## 拷贝结构体遇到有字段是map类型

map的具体实现是hmap，hmap有一个指针字段指向底层bucket数组，键值对就存储在里面。结构体复制时，小心指向同一个map

```go
type foo struct {
	m map[int]int
}

f := foo{}
f.m = make(map[int]int)
newf := f
newf.m[2] = 2
fmt.Print(f.m)
// map[2:2]
```

## encoding/gob 包不支持文件的seek

gob包是处理数据流的，如果编码数据写入文件，不支持seek来读取。
参考:
[Use Gob to write logs to a file in an append style](https://stackoverflow.com/a/43199852/2883173) 
[Efficient Go serialization of struct to disk](https://stackoverflow.com/questions/37618399/efficient-go-serialization-of-struct-to-disk) 

## 安全拷贝http.DefaultTransport

有时发起http请求需要自行新建 client ，例如关闭证书验证：

```go
client = new(http.Client)
tr := *(http.DefaultTransport.(*http.Transport))
tr.TLSClientConfig = &tls.Config{
	InsecureSkipVerify: true,
}
client.Transport = &tr
```

这样值拷贝获取transport的方式，在后续使用中有并发读写map引起panic的隐患。因为`tr.idleConnCh`和 `DefaultTransport.idleConnCh`指向同一个map，但用来确保此map线程安全的互斥锁`DefaultTransport.idleMu`并不会对 tr 起作用。当使用 client 发起请求时，如果其他goroutine正好在用DefaultTransport(例如调用http.Post用到DefaultClient，DefaultClient的Transport是DefaultTransport)，可能panic。

解决办法，复制http包DefaultTransport的赋值语句：

```go
tr := http.Transport{
	Proxy: http.ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```
