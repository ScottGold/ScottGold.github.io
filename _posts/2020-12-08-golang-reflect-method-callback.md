参考：

[pass-method-argument-to-function](https://stackoverflow.com/questions/38897529/pass-method-argument-to-function)

# 通过反射调用method方法

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

# interface的基础1

```go
package main

import (
    "fmt"
    "reflect"
)

type base interface {}

type Foo interface {
    Cose() string
    SetV(string)
}

type Bar struct {
    cose string
}

//类型接收器是指针
func (b *Bar) Cose() string {
    return b.cose
}

func (b *Bar) SetV(v string) {
     b.cose = v
}

type Bar2 struct {
    cose string
}

//类型接收器是对象
func (b Bar2) Cose() string {
    return b.cose
}

func (b Bar2) SetV(v string) {
     b.cose = v
}

func passbyv(b Foo) {
     b.SetV("hello")
}

func passbyp(p *Foo) {
     (*p).SetV("Scott")
}

func main() {
    bar := Foo(&Bar{
        cose: "ciaone",
    })
    fmt.Println(reflect.TypeOf(bar))
    ii, ok := bar.(Foo)
    if !ok {
        panic("type assertion")
    }
    fmt.Println(" cose : " + ii.Cose())

    passbyv(bar)
    fmt.Println(" b v: " + ii.Cose())
    passbyp(&bar)
    fmt.Println(" b p: " + ii.Cose())

    bb := Foo(Bar2{
        cose: "bar22",
    })
    fmt.Println(reflect.TypeOf(bb))

    b2, ok := bb.(Foo)
    if !ok {
        panic("type assertion")
    }

    passbyv(b2)
    fmt.Println(" b v: " + b2.Cose())
    passbyp(&b2)
    fmt.Println(" b p: " + b2.Cose())
}
```

```
*main.Bar
 cose : ciaone
 b v: hello
 b p: Scott
main.Bar2
 b v: bar22
 b p: bar22
```

可以在[这里](https://play.golang.org/p/1l3jGwIJJXR)运行。

interface装struct对象或struct对象指针，是用指针还是对象，要看这个struct定义方法时类型接收器用的是对象还是指针。

给函数传interface变量时，实际传值还是指针只与interface装的值是什么有关。

## [Interface values](https://tour.golang.org/methods/11)

实际上，接口值可以看作是一个值和一个具体类型的元组：

```
(value, type)
```

An interface value holds a value of a specific underlying concrete type.

Calling a method on an interface value executes the method of the same name on its underlying type.