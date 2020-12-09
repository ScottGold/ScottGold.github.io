参考：

[pass-method-argument-to-function](https://stackoverflow.com/questions/38897529/pass-method-argument-to-function)

通过反射调用method方法。

```go
package main

import (
	"fmt"
	"reflect"
)

type Foo struct {
	i int
}

func (f Foo) A(b int) {
	fmt.Println("testA", b, f.i)
}
func (f Foo) B(b int) {
	fmt.Println("testB", b, f.i)
}
func (f Foo) C(b int) {
	fmt.Println("testC", b, f.i)
}

func main() {
	var f Foo
	f.i = 10
	bar := func(name string, b int) {
		params := make([]reflect.Value, 0)
		params = append(params, reflect.ValueOf(b))
		//TODO: maybe you need to check function parameters.
		reflect.ValueOf(f).MethodByName(name).Call(params) //如果没有参数: params = nil
	}
	bar("A", 1)
	bar("B", 2)
	bar("C", 3)

	f.i = 999
	t := func(m func(int), b int) {
		m(b)
	}

	t(f.A, 888)
}
```

可以在 [playground](https://play.golang.org/) 上运行。

```
testA 1 10
testB 2 10
testC 3 10
testA 888 999
```

上面方法中如果对象接收器是指针，那么MethodByName前的对象也要是指针

```go
func (f *Foo) A(b int) {
	fmt.Println("testA", b, f.i)
}
//相应
reflect.ValueOf(&f).MethodByName(name).Call(params) 
```

