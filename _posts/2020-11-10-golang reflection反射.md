看https://go101.org/article/reflection.html

example 1:(channel map array)

```go
package main

import "fmt"
import "reflect" // use this package

func main() {
	type A = [16]int16
	var c <-chan map[A][]byte
	tc := reflect.TypeOf(c)
	fmt.Println(tc.Kind())    // chan
	fmt.Println(tc.ChanDir()) // <-chan
	tm := tc.Elem()
	ta, tb := tm.Key(), tm.Elem()
	// The next line prints: map array slice
	fmt.Println(tm.Kind(), ta.Kind(), tb.Kind())
	tx, ty := ta.Elem(), tb.Elem()

	// byte is an alias of uint8
	fmt.Println(tx.Kind(), ty.Kind()) // int16 uint8
	fmt.Println(tx.Bits(), ty.Bits()) // 16 8
	fmt.Println(tx.ConvertibleTo(ty)) // true
	fmt.Println(tb.ConvertibleTo(ta)) // false

	// Slice and map types are incomparable.
	fmt.Println(tb.Comparable()) // false
	fmt.Println(tm.Comparable()) // false
	fmt.Println(ta.Comparable()) // true
	fmt.Println(tc.Comparable()) // true
}
```

example 2:(pointer)

```
package main

import "fmt"
import "reflect"

type T []interface{m()}
func (T) m() {}

func main() {
	tp := reflect.TypeOf(new(interface{}))
	tt := reflect.TypeOf(T{})
	fmt.Println(tp.Kind(), tt.Kind()) // ptr slice

	// Get two interface Types indirectly.
	ti, tim := tp.Elem(), tt.Elem()
	// The next line prints: interface interface
	fmt.Println(ti.Kind(), tim.Kind())

	fmt.Println(tt.Implements(tim))  // true
	fmt.Println(tp.Implements(tim))  // false
	fmt.Println(tim.Implements(tim)) // true

	// All types implement any blank interface type.
	fmt.Println(tp.Implements(ti))  // true
	fmt.Println(tt.Implements(ti))  // true
	fmt.Println(tim.Implements(ti)) // true
	fmt.Println(ti.Implements(ti))  // true
}
```

example 3:(struct and fields ,  function)

```go
package main

import "fmt"
import "reflect"

type F func(string, int) bool //non-interface types
func (f F) m(s string) bool {
	return f(s, 32)
}
func (f F) M() {}

type I interface{m(s string) bool; M()}

func main() {
	var x struct {
		F F
		i I
	}
	tx := reflect.TypeOf(x)
	fmt.Println(tx.Kind())        // struct
	fmt.Println(tx.NumField())    // 2
	fmt.Println(tx.Field(1).Name) // i
	// Package path is an intrinsic property of
	// non-exported selectors (fields or methods).
	fmt.Println(tx.Field(0).PkgPath) // 
	fmt.Println(tx.Field(1).PkgPath) // main

	tf, ti := tx.Field(0).Type, tx.Field(1).Type
	fmt.Println(tf.Kind())               // func
	fmt.Println(tf.IsVariadic())         // false
	fmt.Println(tf.NumIn(), tf.NumOut()) // 2 1
	t0, t1, t2 := tf.In(0), tf.In(1), tf.Out(0)
	// The next line prints: string int bool
	fmt.Println(t0.Kind(), t1.Kind(), t2.Kind())

	fmt.Println(tf.NumMethod(), ti.NumMethod()) // 1 2
	fmt.Println(tf.Method(0).Name)              // M
	fmt.Println(ti.Method(1).Name)              // m
	_, ok1 := tf.MethodByName("m")
	_, ok2 := ti.MethodByName("m")
	fmt.Println(ok1, ok2) // false true
}
```

总结：

non-interface types： 是除了struct这外的吧，例如func.

1. 对于non-interface types，**reflect.Type.NumMethod** 仅能返回导出的方法的数量。也不能通过**reflect.Type.MethodByName** 获取非导出的方法。对于接口类型(interface types)，则不受导出，非导出限制。这个规则对应下面reflect.Value也是一样。
2. 尽管reflect.Type.NumField能返回所有字段的数量（包括非导出部分），但是通过reflect.Type.FieldByName获取非导出字段是不好的。

example 4:(struct tag)

```go
package main

import "fmt"
import "reflect"

type T struct {
	X int  `max:"99" min:"0"`
	Y bool `optional:"yes"`
}

func main() {
	t := reflect.TypeOf(T{})
	x, y := t.Field(0).Tag, t.Field(1).Tag
	fmt.Println(reflect.TypeOf(x)) // reflect.StructTag

	tag, present := y.Lookup("default")
	fmt.Println(len(tag), present)    // 0 false
	fmt.Println(y.Lookup("optional")) // yes true

	fmt.Println(x.Get("max"), x.Get("min")) // 99 0
}
```

example 5:创建非定义的值（反射类型的值）

```go
package main

import "fmt"
import "reflect"

func main() {
	ta := reflect.ArrayOf(5, reflect.TypeOf(123))
	fmt.Println(ta) // [5]int
	tc := reflect.ChanOf(reflect.SendDir, ta)
	fmt.Println(tc) // chan<- [5]int
	tp := reflect.PtrTo(ta)
	fmt.Println(tp) // *[5]int
	ts := reflect.SliceOf(tp)
	fmt.Println(ts) // []*[5]int
	tm := reflect.MapOf(ta, tc)
	fmt.Println(tm) // map[[5]int]chan<- [5]int
	tf := reflect.FuncOf([]reflect.Type{ta},
				[]reflect.Type{tp, tc}, false)
	fmt.Println(tf) // func([5]int) (*[5]int, chan<- [5]int)
	tt := reflect.StructOf([]reflect.StructField{
		{Name: "Age", Type: reflect.TypeOf("abc")},
	})
	fmt.Println(tt)            // struct { Age string }
	fmt.Println(tt.NumField()) // 1
}
```

限制：1、从Go 1.15起，不能通过反射创建interface types.

2、尽管我们可以通过反射创建一个将其他类型嵌入为匿名字段的结构类型，结构类型可能会或可能不会获得嵌入类型的方法，并使用匿名字段创建结构类型，甚至在运行时可能会崩溃。换句话说，使用匿名字段创建结构类型的行为部分取决于编译器。

3、不能通过反射定义新的类型。



ValueOf 1:

```go
package main

import "fmt"
import "reflect"

func main() {
	n := 123
	p := &n
	vp := reflect.ValueOf(p)
	fmt.Println(vp.CanSet(), vp.CanAddr()) // false false
	vn := vp.Elem() // get the value referenced by vp
	fmt.Println(vn.CanSet(), vn.CanAddr()) // true true
	vn.Set(reflect.ValueOf(789)) // <=> vn.SetInt(789)
	fmt.Println(n)               // 789
}
```

ValueOf 2:

非导出字段不能通过反射修改

```go
package main

import "fmt"
import "reflect"

func main() {
	var s struct {
		X interface{} // an exported field
		y interface{} // a non-exported field
	}
	vp := reflect.ValueOf(&s)
	// If vp represents a pointer. the following
	// line is equivalent to "vs := vp.Elem()".
	vs := reflect.Indirect(vp)
	// vx and vy both represent interface values.
	vx, vy := vs.Field(0), vs.Field(1)
	fmt.Println(vx.CanSet(), vx.CanAddr()) // true true
	// vy is addressable but not modifiable.
	fmt.Println(vy.CanSet(), vy.CanAddr()) // false true
	vb := reflect.ValueOf(123)
	vx.Set(vb)     // okay, for vx is modifiable
	// vy.Set(vb)  // will panic, for vy is unmodifiable
	fmt.Println(s) // {123 <nil>}
	fmt.Println(vx.IsNil(), vy.IsNil()) // false true
}
```

有两种获取指针原来的类型；

1. 一种方法是调用代表指针值的reflect.Value值的Elem方法。
2. 另一种方法是将reflect.Value值传递给代表reflect.Indirect函数调用的指针值。

```go
package main

import "fmt"
import "reflect"

func main() {
	var z = 123
	var y = &z
	var x interface{} = y
	v := reflect.ValueOf(&x)
	vx := v.Elem()
	vy := vx.Elem()
	vz := vy.Elem()
	vz.Set(reflect.ValueOf(789))
	fmt.Println(z) // 789
}
```

```go
package main

import "fmt"
import "reflect"

func InvertSlice(args []reflect.Value) (result []reflect.Value) {
	inSlice, n := args[0], args[0].Len()
	outSlice := reflect.MakeSlice(inSlice.Type(), 0, n)
	for i := n-1; i >= 0; i-- {
		element := inSlice.Index(i)
		outSlice = reflect.Append(outSlice, element)
	}
	return []reflect.Value{outSlice}
}

func Bind(p interface{}, f func ([]reflect.Value) []reflect.Value) {
	// invert represents a function value.
	invert := reflect.ValueOf(p).Elem()
	invert.Set(reflect.MakeFunc(invert.Type(), f))
}

func main() {
	var invertInts func([]int) []int
	Bind(&invertInts, InvertSlice)
	fmt.Println(invertInts([]int{2, 3, 5})) // [5 3 2]

	var invertStrs func([]string) []string
	Bind(&invertStrs, InvertSlice)
	fmt.Println(invertStrs([]string{"Go", "C"})) // [C Go]
}

```

