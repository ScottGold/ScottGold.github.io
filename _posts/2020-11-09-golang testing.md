# 概述

```go
func TestXxx(*testing.T)
```

测试函数的名字，Test要以大写开始。

测试的文件名是 **xxx_test.go**, "_test.go"结尾，文件内是TestXxx的函数。文件放在被测试模块同一目录。

具体错误可以用testing.T的Errorf来输出。

# Benchmarks

```go
func BenchmarkXxx(*testing.B)
```

go test  -bench 将会**依序**运行benchmarks的用例。example:

```go
func BenchmarkRandInt(b *testing.B) {
    for i := 0; i < b.N; i++ {
        rand.Int()
    }
}
```

b.N是动态改变的，直到运行次数足够多，能可靠计算每次耗时为止。会有如下输出：

```
BenchmarkRandInt-8   	68453040	        17.8 ns/op
```

如果benchmark需要平行测试性能，可以使用**RunParallel**函数，一般会go test -cpu flag一起用。

```go
func BenchmarkTemplateParallel(b *testing.B) {
    templ := template.Must(template.New("test").Parse("Hello, {{.}}!"))
    b.RunParallel(func(pb *testing.PB) {
        var buf bytes.Buffer
        for pb.Next() {
            buf.Reset()
            templ.Execute(&buf, "World")
        }
    })
}
```

#  Examples

```go
func ExampleHello() {
    fmt.Println("hello")
    // Output: hello
}

func ExampleSalutations() {
    fmt.Println("hello, and")
    fmt.Println("goodbye")
    // Output:
    // hello, and
    // goodbye
}
```

Output那个注释是期待输出，对比结果时，会忽然前导和尾随空白。不按顺序时用Unordered output。

```go
func ExamplePerm() {
    for _, value := range Perm(5) {
        fmt.Println(value)
    }
    // Unordered output: 4
    // 2
    // 1
    // 3
    // 0
}
```

Example中如果没有output的注释则只编译，不运行。

下面的suffix需要小写开头

```go
func Example_suffix() { ... }
func ExampleF_suffix() { ... }
func ExampleT_suffix() { ... }
func ExampleT_M_suffix() { ... }
```

# Skipping

路过。。

```go
func TestTimeConsuming(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping test in short mode.")
    }
    ...
}
```

# Subtests and Sub-benchmarks

T和B类型中的Run可以定义子测试和子benchmarks，不需要为sub们定义函数。This enables uses like table-driven benchmarks and creating hierarchical tests. It also provides a way to share common setup and tear-down code:（这个可以使用表驱动benchmark和层级式测试，也能提供公共setup和tear-down代码）

```go
func TestFoo(t *testing.T) {
    // <setup code>
    t.Run("A=1", func(t *testing.T) { ... })
    t.Run("A=2", func(t *testing.T) { ... })
    t.Run("B=1", func(t *testing.T) { ... })
    // <tear-down code>
}
```

运行测试时，可以-run和-bench flags设置运行那些测试：

```go
go test -run ''      # 运行所有测试
go test -run Foo     # 运行一级测试，匹配 "Foo", 例如 "TestFooBar".
go test -run Foo/A=  # 运行一级测试，匹配 "Foo", 运行二级测试，匹配 "A=".
go test -run /A=1    # 运行所有一级测试, 运行二级测试，匹配 "A=1".
```

子用例可以并行运行，父级仅在所有子用例运行完才完成。

```go
func TestGroupedParallel(t *testing.T) {
    for _, tc := range tests {
        tc := tc // capture range variable
        t.Run(tc.Name, func(t *testing.T) {
            t.Parallel()
            ...
        })
    }
}
```

子测试间是并行的运行。

# Main

有时需要在测试前或测试后有一些设置或tear-down的控制，或者控制那些代码需要在主线程运程，可以在一个文件内添加一个

```
func TestMain(m *testing.M)
```

