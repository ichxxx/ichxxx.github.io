---
title: "Go中[]byte与string的转换优化"
date: 2020-11-23T22:23:09+08:00
draft: false
tags: ["Go", "Performance"]
---

先上优化代码：

```go
func bytesFromString(s string) []byte {
    tmp := (*[2]uintptr)(unsafe.Pointer(&s))
    x := [3]uintptr{tmp[0], tmp[1], tmp[1]}
    return *(*[]byte)(unsafe.Pointer(&x))
}

func stringFromBytes(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}
```



一般而言，我们在Go中面对`[]byte`和`string`相互转换时，会使用内建的操作`[]byte()`和`string()`。但当我们对程序运行性能有较高要求的时候，就不得不考虑这他们的性能消耗问题。

以下源码基于Go 1.15。



## 内部结构分析

在分析`[]byte()`和`string()`的内部实现之前，先来看一下`string`和`slice`的内部结构。

### slice的内部结构

`slice`的内部结构可以在`src/runtime/slice.go`找到：

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

其中`array`指向底层数组。



### string的内部结构

`string`的内部结构可以在`src/runtime/string.go`找到：

```go
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

`str`指向的是什么呢？

我们可以在另外一个`string`结构中找到答案：

```go
// Variant with *byte pointer type for DWARF debugging.
type stringStructDWARF struct {
	str *byte
	len int
}
```

这个结构是用于内部DWARF调试的，所以`str`指向了一个`byte`数组。



### 小结

在内部结构上，`slice`比`string`只多了一个`cap`字段。

因为`string`的底层是一个`byte`数组，所以`[]byte`和`string`唯一的差异便是`cap`字段。



## 内部实现分析

铺垫完上面的内容，我们来看看`[]byte()`和`string()`的内部实现是怎么样的。

### []byte()的内部实现

调用`[]byte()`操作时，实际调用的是`src/runtime/string.go`中的`stringtoslicebyte()`方法，该方法具体实现如下：

```go
type tmpBuf [32]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
    var b []byte
    // 如果string的长度超过32，则调用rawbyteslice()，分配一块新内存
    if buf != nil && len(s) <= len(buf) {
        *buf = tmpBuf{}
        b = buf[:len(s)]
    } else {
		b = rawbyteslice(len(s))
	}
    // 拷贝string中底层数组的内容到[]byte
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```

可以看到，`[]byte()`操作必然会产生一次拷贝。

如果要转换的`string`长度大于**32**，还会发生一次内存分配。



### string()的内部实现

调用`string()`操作时，实际调用的是`src/runtime/string.go`中的`slicebytetostring()`方法，该方法具体实现如下：

```go
// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			funcPC(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
    // 如果[]byte的长度为1
	if n == 1 {
        // 直接从预先分配好的staticuint64s数组中读取，避免重新分配内存
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if sys.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
    // 如果[]byte的长度超过32，则分配一块新内存
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
    // 拷贝[]byte中底层数组的内容到string
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}

// 获取string类型结构实例
func stringStructOf(sp *string) *stringStruct {
	return (*stringStruct)(unsafe.Pointer(sp))
}
```

`src/runtime/iface.go`：

```go
// staticuint64s is used to avoid allocating in convTx for small integer values.
var staticuint64s = [...]uint64{
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        ...
        0xf8, 0xf9, 0xfa, 0xfb, 0xfc, 0xfd, 0xfe, 0xff,

}
```

要转换的`[]byte`长度， 如果为1，则没有额外的性能消耗；

如果大于1，则必然会发生一次拷贝；

如果大于**32**，还会发生一次内存分配。



### 小结

经过一顿分析，我们可以确定，`[]byte()`和`string()`这两个内建操作，会产生一次底层数组的拷贝，如果数据量较大，还会产生一次内存分配。



## 优化

结合`[]byte`和`string`的内部结构，显然`[]byte()`和`string()`中的拷贝和内存分配是可以被绕过的，即共享它们的底层数组，手动完成结构转换。

手动转换需要使用到`unsafe`包来操作类型，具体方法如下：

```go
func bytesFromString(s string) []byte {
    // 将string类型的两个字段转化到一个临时uintptr数组
    tmp := (*[2]uintptr)(unsafe.Pointer(&s))
    // 构建一个拥有完整slice类型字段的uintptr数组
    x := [3]uintptr{tmp[0], tmp[1], tmp[1]}
    // 将手动构建出的uintptr数组还原成slice类型
    return *(*[]byte)(unsafe.Pointer(&x))
}

func stringFromBytes(b []byte) string {
    // 直接进行类型转换，多余的字段会被舍弃
    return *(*string)(unsafe.Pointer(&b))
}
```



## 性能测试

```go
var _s = strings.Repeat("foo", 1024)
var _b = []byte(_s)
var result interface{}

func BenchmarkString1(b *testing.B) {
	var bytes string
	for i := 0; i < b.N; i++ {
		bytes = string(_b)
	}
	result = bytes
}

func BenchmarkString2(b *testing.B) {
	var bytes string
	for i := 0; i < b.N; i++ {
		bytes = stringFromBytes(_b)
	}
	result = bytes
}

func BenchmarkBytes1(b *testing.B) {
	var bytes []byte
	for i := 0; i < b.N; i++ {
		bytes = []byte(_s)
	}
	result = bytes
}

func BenchmarkBytes2(b *testing.B) {
	var bytes []byte
	for i := 0; i < b.N; i++ {
		bytes = bytesFromString(_s)
	}
	result = bytes
}
```

编写如上测试，转换一个3KB的数据，得到如下结果：

```bash
BenchmarkString1       2912431      416 ns/op    3072 B/op    1 allocs/op
BenchmarkString2    1000000000    0.407 ns/op       0 B/op    0 allocs/op
BenchmarkBytes1        2850360      409 ns/op    3072 B/op    1 allocs/op
BenchmarkBytes2      502088688     2.08 ns/op       0 B/op    0 allocs/op
```

采用直接转换，在无需分配额外内存的同时，其转换耗时也几乎可以忽略。

当然，采用内建操作的这400ns差异在大部分场景下也是几乎可以忽略的。

那如果需要转换的数据不是3KB，而是3MB呢？

```bash
BenchmarkString1          5917    199307 ns/op    3145730 B/op    1 allocs/op
BenchmarkString2    1000000000     0.399 ns/op          0 B/op    0 allocs/op
BenchmarkBytes1        2850360    190067 ns/op    3145734 B/op    1 allocs/op
BenchmarkBytes2      558141092      2.10 ns/op          0 B/op    0 allocs/op
```

可以看到，直接转换不受数据量增大的影响，而内建操作的耗时则随着数据量的增大而增加。

在3MB的数据量下，其一次操作的耗时已经接近0.2ms。

假设我们需要在1秒内进行1000次运算，即1ms运算一次，而一次内建操作的转换就已经占用了五分之一的耗时，同时需要分配跟转换对象同等大小的内存，如此的性能消耗使得我们不得不考虑进去。



## 局限

直接转换的性能提升是以牺牲安全性为代价的，因为我们在两个不同类型结构中，共享了同一个底层数组。

在Go中，`string`类型是不可变的，而`[]byte`类型则是可以被修改的。

假设我们需要修改通过直接转换而来的`[]byte`，那么同时也会对`string`的底层数组做出修改，这样会造成严重的错误！

**如果你的程序不能保证通过直接转换而来的`[]byte`不被修改，则不推荐使用该方法。**



## 总结

在对程序运行性能有较高要求且对数据只有读操作的场景，通过使用`unsafe`包，我们可以在高性能且无需分配额外内存的情况下，完成`[]byte`和`string`的相互转换。


