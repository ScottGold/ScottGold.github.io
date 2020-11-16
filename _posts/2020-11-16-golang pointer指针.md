读书笔记https://go101.org/article/pointer.html

尽管golang吸收了很多其它语言的特点，但是go还是算是C家庭的语言。指针就是其中的一个证据。Go的指针与C指针有相同的地方也有不同的地方。

# 什么是指针

Go中有一个叫Pointer的类型。

# Go Pointer类型与值

类型T的指针可以表示为*T。

关于什么是non-defined pointer和defined pointer

```go
*int  // A non-defined pointer type whose base type is int.
**int // A non-defined pointer type whose base type is *int.

// Ptr is a defined pointer type whose base type is int.
type Ptr *int
// PP is a defined pointer type whose base type is Ptr.
type PP *Ptr
```

显然non-defined pointer具有更好的可读性。

# 关于引用

指针的值引用了其它值。

有两种方式取到non-nil指针：

1. 关键字new，用得比较少。
2. 取地址&。

# pointer Dereference

也就是取指针的值 *p

如果p为nil，运行时会panic

```go
package main

import "fmt"

func main() {
	p0 := new(int)   // p0 points to a zero int value.
	fmt.Println(p0)  // (a hex address string)
	fmt.Println(*p0) // 0

	// x is a copy of the value at
	// the address stored in p0.
	x := *p0
	// Both take the address of x.
	// x, *p1 and *p2 represent the same value.
	p1, p2 := &x, &x
	fmt.Println(p1 == p2) // true
	fmt.Println(p0 == p1) // false
	p3 := &*p0 // <=> p3 := &(*p0) <=> p3 := p0
	// Now, p3 and p0 store the same address.
	fmt.Println(p0 == p3) // true
	*p0, *p1 = 123, 789
	fmt.Println(*p2, x, *p3) // 789 789 123

	fmt.Printf("%T, %T \n", *p0, x) // int, int
	fmt.Printf("%T, %T \n", p0, p1) // *int, *int
}
```

# 为什么需要指针

go中所有值的赋值都是值拷贝，包括函数参数传递。

```go
package main

import "fmt"

func double(x *int) {
	*x += *x
	x = nil // the line is just for explanation purpose
}

func main() {
	var a = 3
	double(&a)
	fmt.Println(a) // 6
	p := &a
	double(p)
	fmt.Println(a, p == nil) // 12 false
}
```

# 可以返回局部变量的指针，并且是安全的

```go
//这样是允许的
func newInt() *int {
	a := 3
	return &a
}
```

# 指针的限制

## Go指针值不支持算术运算

p++, p-2是非法的，主要为了安全。

```
*p++其实是(*p)++,不算指针的运算
```

## 指针值不能转换为任意指针类型

两个类型T1、T2的指针值能转换的前提是:

1. T1和T2的底层类型是相同的（忽略struct tags），特别是如果T1,T2是non-define类型并且底层类型是相同的（考虑struct tags），然后转换是隐式的。
2. 如果T1和T2都是non-defined类型，并且底层类型是相同的（忽略struct tags）。

```go
type MyInt int64
type Ta    *int64
type Tb    *MyInt
```

1. *int64 可以隐含地转换为Ta，反之也行。

2. *MyInt 可以隐含地转换为Tb，反之也行。

3. *MyInt 可以显性地转换为 *int64，反之也行。

4. Ta不能直接转换成Tb，显式也不行。需要

   ```go
   Tb((*MyInt)((*int64)(pa)))
   ```

## 指针值不能和任意指针比较

只有类型相同的指针（或可以进行隐式转换的）才能用==和!=比较，还有nil。

## 不同类型的指针不能相互赋值

# 打破上面的限制

使用包unsafe。

