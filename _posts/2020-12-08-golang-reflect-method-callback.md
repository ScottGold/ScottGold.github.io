参考：

https://stackoverflow.com/questions/38897529/pass-method-argument-to-function

通过反射调用method方法。

```go
package main

import (
	"fmt"
	"reflect"
)

type Foo int

func (f Foo) A(b int) {
	fmt.Println("testA", b)
}
func (f Foo) B(b int) {
	fmt.Println("testB", b)
}
func (f Foo) C(b int) {
	fmt.Println("testC", b)
}

func main() {
	var f Foo
	bar := func(name string, b int) {
		params := make([]reflect.Value,0)
		params = append(params, reflect.ValueOf(b))
        //TODO: maybe you need to check function parameters.
        reflect.ValueOf(f).MethodByName(name).Call(params )//如果没有参数: params = nil
	}
	bar("A", 1)
	bar("B", 2)
	bar("C", 3)
}
```

可以在 [playground](https://play.golang.org/) 上运行。

```
testA 1
testB 2
testC 3
```

