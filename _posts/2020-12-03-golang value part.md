https://go101.org/article/value-part.html 

部分内容摘要

# go中两大类型分类

go可以被视为C家庭语言，go和c语言在struct和指针类型的内存结构上有很多的相似之处。

另一方面，go也被视为c语言框架的。主要反映在，go支持的类型中有几种是内存结构不透明的，而c语言的内存结构是透明的。每个c值在内存中被封装到一个内存块中。而go有几种类型会存在多个内存块中。

go类型分为两大类：

|        只存放在一个内存块中的类型        |              存放在多个内存块中的类型              |
| :--------------------------------------: | :------------------------------------------------: |
| solo Direct value part（直接指向值部分） | direct part->underlying part（还引用到底层下部分） |
|              boolean types               |                    slice types                     |
|              numeric types               |                     map types                      |
|              pointer types               |                   channel types                    |
|           unsafe pointer types           |                   function types                   |
|               struct types               |                  interface types                   |
|               array types                |                    string types                    |

注意：

- 接口和string的值是否包含底层部分（underlying parts）是由编译器决定的。标准go编译器是包含的。
- 函数值是否可能包含基础部分几乎很难甚至无法证明。go 101里视函数可能包含underlying parts的。

第二类中的类型种类通过封装许多实现细节为Go编程带来了很多便利。

# go中的两种指针类型

 type-safe pointer types和 [type-unsafe pointer types](https://go101.org/article/unsafe.html)。 包unsafe里有unsafe.Pointer类型，与c语言中void*相似。下面文中提到指针包含这两种指针。

指针指向另外一个值的地址，除非它是nil指针。我们说指针引用了这个值，或这个值被这个指针引用。值可以被隐式引用。

- 如果结构值a中有个指针字段b它引用了一个值c，我们可以说a也引用了c。
- x引用了y，y又引用了z，我们说x间接引用了z。不管x->y,y->z是直接还是间接引用。

引用关系是可以传递的。

# 第二类类型的可能内部定义

为了更好了解，可以假想第二分类类型是由第一分类类型实现。

内部关于map, channel, function类型的定义类似如下：

```go
// map types
type _map *hashtableImpl

// channel types
type _channel *channelImpl

// function types
type _function *functionImpl
```

slice的实现可能如下 ：

```go
type _slice struct {
	// referencing underlying elements
	elements unsafe.Pointer
	// number of elements and capacity
	len, cap int
}
```

string如下 ：

```go
type _string struct {
	elements *byte // referencing underlying bytes
	len      int   // number of bytes
}
```

普通的interface

```go
type _interface struct {
	dynamicType  *_type         // the dynamic type
	dynamicValue unsafe.Pointer // the dynamic value
}
```

标准go编译器的non-blank interface

```go
type _interface struct {
	dynamicTypeInfo *struct {
		dynamicType *_type       // the dynamic type
		methods     []*_function // method table
	}
	dynamicValue unsafe.Pointer // the dynamic value
}
```

# Underlying Value Parts在赋值中不会被拷贝

现在我们了解到，第二类中类型的内部定义是指针持有者（指针或指针包装器）类型。知道这一点对理解Go中的价值复制行为非常有帮助。

在Go中，在目标值和源值具有相同的类型下，**每个值的赋值（包括参数传递等）都是浅拷贝**（ shallow value copy）（如果它们的类型不同，我们可以认为在执行该赋值之前，源值将隐式转换为目标类型）。也就是说，**源值中只有直接部分被拷贝到目标值中**。如果源值中包含underlying value parts，然后目标值和源值的直接部分都引用了同样的underlying value part(s)，它们是共享同样的underlying value part(s)。



