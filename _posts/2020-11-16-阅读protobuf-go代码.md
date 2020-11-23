我想通过学习[protobuf-go](https://github.com/protocolbuffers/protobuf-go )更深入了解go反射的应用，就看了下它的代码。

所看版本是

```
go 1.14

require (
	github.com/golang/protobuf v1.4.3
	google.golang.org/grpc/cmd/protoc-gen-go-grpc v1.0.1 // indirect
	google.golang.org/protobuf v1.23.0
)
```

# 粗略看一下

先瞄一眼它的生成文件*.pb.go

在proto中定义的一个message

```
message TestMsg {
    string str = 1;
    int32 vint32 = 2;
    double vdouble = 3;
    repeated string arrayStr = 4;
}
```

生成的.pb.go中有下面一个struct

```go
type TestMsg struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	Str      string   `protobuf:"bytes,1,opt,name=str,proto3" json:"str,omitempty"`
	Vint32   int32    `protobuf:"varint,2,opt,name=vint32,proto3" json:"vint32,omitempty"`
	Vdouble  float64  `protobuf:"fixed64,3,opt,name=vdouble,proto3" json:"vdouble,omitempty"`
	ArrayStr []string `protobuf:"bytes,4,rep,name=arrayStr,proto3" json:"arrayStr,omitempty"`
}
```

除了proto里对应字段和struct tags外，开始处有三个字段。

1、有个变量var file_xxxx_proto_msgTypes = make([]protoimpl.MessageInfo, 1)

并且设置了里面的Exporter的值，file_msgid_proto_msgTypes[0].Exporter

2、TestMsg的ProtoReflect函数，

```go
func (x *MsgIdManager) ProtoReflect() protoreflect.Message {
	mi := &file_msgid_proto_msgTypes[0]
	if protoimpl.UnsafeEnabled && x != nil {
		ms := protoimpl.X.MessageStateOf(protoimpl.Pointer(x))
		if ms.LoadMessageInfo() == nil {
			ms.StoreMessageInfo(mi)
		}
		return ms
	}
	return mi.MessageOf(x)
}
```

先是将x指针转成MessageSate指针，这个特性和C一样。

然后分别调用LoadMessageInfo和StoreMessageInfo。如果ms中的atomicMessageInfo指针为空，则将atomicMessageInfo指向mi。也就是所有的同一类型message实例共享一个MessageInfo实例。

3、TestMsg还有个Reset函数。



首先protobuf-go内部还有几个特别的类型，在pragma.go里

```go
package pragma

import "sync"

// NoUnkeyedLiterals can be embedded in a struct to prevent unkeyed literals.
type NoUnkeyedLiterals struct{}

// DoNotImplement can be embedded in an interface to prevent trivial
// implementations of the interface.
//
// This is useful to prevent unauthorized implementations of an interface
// so that it can be extended in the future for any protobuf language changes.
type DoNotImplement interface{ ProtoInternal(DoNotImplement) }

// DoNotCompare can be embedded in a struct to prevent comparability.
type DoNotCompare [0]func()

// DoNotCopy can be embedded in a struct to help prevent shallow copies.
// This does not rely on a Go language feature, but rather a special case
// within the vet checker.
//
// See https://golang.org/issues/8005.
type DoNotCopy [0]sync.Mutex

```

MessageState在internal\impl\message_reflect.go

```go
// MessageState is a data structure that is nested as the first field in a
// concrete message. It provides a way to implement the ProtoReflect method
// in an allocation-free way without needing to have a shadow Go type generated
// for every message type. This technique only works using unsafe.
//
//
// Example generated code:
//
//	type M struct {
//		state protoimpl.MessageState
//
//		Field1 int32
//		Field2 string
//		Field3 *BarMessage
//		...
//	}
//
//	func (m *M) ProtoReflect() protoreflect.Message {
//  ...
//	}
//
// The MessageState type holds a *MessageInfo, which must be atomically set to
// the message info associated with a given message instance.
// By unsafely converting a *M into a *MessageState, the MessageState object
// has access to all the information needed to implement protobuf reflection.
// It has access to the message info as its first field, and a pointer to the
// MessageState is identical to a pointer to the concrete message value.
//
//
// Requirements:
//	• The type M must implement protoreflect.ProtoMessage.
//	• The address of m must not be nil.
//	• The address of m and the address of m.state must be equal,
//	even though they are different Go types.
type MessageState struct {
	pragma.NoUnkeyedLiterals
	pragma.DoNotCompare
	pragma.DoNotCopy

	atomicMessageInfo *MessageInfo
}
```

MessageState是作为具体消息中的第一个字段嵌套的数据结构。它提供了一种无需分配的方式来实现ProtoReflect方法，而不需要为每个消息类型生成一个shadow Go类型。这种技术只在不安全的情况下有效。

MessageState类型包含一个*MessageInfo，它必须原子性地设置为与给定消息实例关联的消息信息。
通过不安全地将*M转换为*MessageState，MessageState对象可以访问实现protobuf反射所需的所有信息。
它可以访问消息信息作为第一个字段，指向MessageState的指针与指向具体消息值的指针相同。

MessageInfo在internal\impl\message.go

```go
// MessageInfo provides protobuf related functionality for a given Go type
// that represents a message. A given instance of MessageInfo is tied to
// exactly one Go type, which must be a pointer to a struct type.
//
// The exported fields must be populated before any methods are called
// and cannot be mutated after set.
type MessageInfo struct {
	// GoReflectType is the underlying message Go type and must be populated.
	GoReflectType reflect.Type // pointer to struct

	// Desc is the underlying message descriptor type and must be populated.
	Desc pref.MessageDescriptor

	// Exporter must be provided in a purego environment in order to provide
	// access to unexported fields.
	Exporter exporter

	// OneofWrappers is list of pointers to oneof wrapper struct types.
	OneofWrappers []interface{}

	initMu   sync.Mutex // protects all unexported fields
	initDone uint32

	reflectMessageInfo // for reflection implementation
	coderMessageInfo   // for fast-path method implementations
}
```

# Unmarshal的调用堆栈

下面是一些关键点的快照

```go
    x := &TestMsg{}
    out, err := proto.Marshal(x)
	if err != nil {
		t.Error("Marshal", err)
	}
	x2 := &TestMsg{}
	proto.Unmarshal(out, x2)
```

1. Unmarshal(b []byte, m Message)
2. _, err := UnmarshalOptions{}.unmarshal(b, m.ProtoReflect())
3. methods := protoMethods(m);    out, err = methods.Unmarshal(in)
4. (mi *MessageInfo) unmarshal(in piface.UnmarshalInput);    out,err := mi.unmarshalPointer(in.Buf, p, 0, unmarshalOptions{   flags:  in.Flags,    resolver: in.Resolver})
5. (mi *MessageInfo) unmarshalPointer(b []byte, p pointer, groupTag protowire.Number, opts unmarshalOptions);   f = mi.denseCoderFields[num];      o, err = f.funcs.unmarshal(b, p.Apply(f.offset), wtyp, f, opts) 在这里会buff进行decode，解出所有字段。这里p.Apply(f.offset)对指针偏移操作，到这里，可以想明白一点，pd.go里生成的message所有字段数据都是用指针保存，这样可以统一使用指针偏移来取到对应字段的指针。不过有点需要注意的是offset不是线性计算的，TestMsg的4个字段分别是40，56 ，64，72。有个奇怪的是string字段占用了16bits?有兴趣的可以看下offset的具体计算。unmarshalPointer里每次decode数据时，先取出tag，tag是一个uint62类型，包含了field number和wire type，低3位是wire type，采用protobuf整形数据的压缩算法。
6. f.funcs.unmarshal 会根据变量的不同，会对应不同的函数。比如string的会是consumeStringValidateUTF8，在文件里internal\impl\codec_gen.go，可以搜索func consume开始的函数。如果字段类型是个message的话，对应的unmarshal函数是consumeMessageInfo。整个结构有点像棵树，所有的基本类型是叶子结点，由对应的consumeXX来decode，如果是message，则表现是个父母结点。p.SetPointer(pointerOfValue(reflect.New(f.mi.GoReflectType.Elem()))) 这里应该是new这个message的对象，并取它的指针。

.pb.go里var file_xxx_proto_rawDesc是用来做什么。是FileDescriptorProto Marshal后的数据

```go
descProto := proto.Clone(f.Proto).(*descriptorpb.FileDescriptorProto)
	descProto.SourceCodeInfo = nil // drop source code information

	b, err := proto.MarshalOptions{AllowPartial: true, Deterministic: true}.Marshal(descProto)
```

运行时用unmarshalSeed解释file_xxx_proto_rawDesc，里面包含文件名，package名，message，service等信息，也就是整个文件的描述信息。

里面大量用到指针，主要为了减少内存拷贝，提高效率。

# 反射结构的初始化

message结构反射信息初始化，Marshal和Unmarshal都要用到orderedCoderFields来遍历所有字段。

internal\impl\message.go里

```go
func (mi *MessageInfo) initOnce() {
	mi.initMu.Lock()
	defer mi.initMu.Unlock()
	if mi.initDone == 1 {
		return
	}

	t := mi.GoReflectType
	if t.Kind() != reflect.Ptr && t.Elem().Kind() != reflect.Struct {
		panic(fmt.Sprintf("got %v, want *struct kind", t))
	}
	t = t.Elem()

	si := mi.makeStructInfo(t) 
	mi.makeReflectFuncs(t, si) // 函数在internal\impl\message_reflect.go
	mi.makeCoderMethods(t, si) // orderedCoderFields的初始化，记录字段该用consumeXXX,appendXX

	atomic.StoreUint32(&mi.initDone, 1)
}
```

makeStructInfo会利用反射和struct tags（也就是.pb.go文件中，message struct字段末尾处的字符串）获得字段信息。下面是Vint32字段的struct tag。

```go
Vint32   int32    `protobuf:"varint,1,opt,name=vint32,proto3" json:"vint32,omitempty"`
```

下面是makeStructInfo函数，在internal\impl\message.go里

```go
func (mi *MessageInfo) makeStructInfo(t reflect.Type) structInfo {
	si := structInfo{
		sizecacheOffset: invalidOffset,
		weakOffset:      invalidOffset,
		unknownOffset:   invalidOffset,
		extensionOffset: invalidOffset,

		fieldsByNumber:        map[pref.FieldNumber]reflect.StructField{},
		oneofsByName:          map[pref.Name]reflect.StructField{},
		oneofWrappersByType:   map[reflect.Type]pref.FieldNumber{},
		oneofWrappersByNumber: map[pref.FieldNumber]reflect.Type{},
	}

fieldLoop:
	for i := 0; i < t.NumField(); i++ {
		switch f := t.Field(i); f.Name {
		case genname.SizeCache, genname.SizeCacheA:
			if f.Type == sizecacheType {
				si.sizecacheOffset = offsetOf(f, mi.Exporter)
			}
		case genname.WeakFields, genname.WeakFieldsA:
			if f.Type == weakFieldsType {
				si.weakOffset = offsetOf(f, mi.Exporter)
			}
		case genname.UnknownFields, genname.UnknownFieldsA:
			if f.Type == unknownFieldsType {
				si.unknownOffset = offsetOf(f, mi.Exporter)
			}
		case genname.ExtensionFields, genname.ExtensionFieldsA, genname.ExtensionFieldsB:
			if f.Type == extensionFieldsType {
				si.extensionOffset = offsetOf(f, mi.Exporter)
			}
		default:  // 这里
			for _, s := range strings.Split(f.Tag.Get("protobuf"), ",") {
				if len(s) > 0 && strings.Trim(s, "0123456789") == "" {
					n, _ := strconv.ParseUint(s, 10, 64)
					si.fieldsByNumber[pref.FieldNumber(n)] = f
					continue fieldLoop
				}
			}
			if s := f.Tag.Get("protobuf_oneof"); len(s) > 0 {
				si.oneofsByName[pref.Name(s)] = f
				continue fieldLoop
			}
		}
	}

	// Derive a mapping of oneof wrappers to fields.
	oneofWrappers := mi.OneofWrappers
	for _, method := range []string{"XXX_OneofFuncs", "XXX_OneofWrappers"} {
		if fn, ok := reflect.PtrTo(t).MethodByName(method); ok {
			for _, v := range fn.Func.Call([]reflect.Value{reflect.Zero(fn.Type.In(0))}) {
				if vs, ok := v.Interface().([]interface{}); ok {
					oneofWrappers = vs
				}
			}
		}
	}
	for _, v := range oneofWrappers {
...
	}

	return si
}
```

在internal\impl\codec_message.go里，makeCoderMethods函数中

```go
			fieldOffset = offsetOf(fs, mi.Exporter)
			childMessage, funcs = fieldCoder(fd, ft) // *
		}
		cf := &preallocFields[i]
		*cf = coderFieldInfo{
			num:        fd.Number(),
			offset:     fieldOffset,
			wiretag:    wiretag,
			ft:         ft,
			tagsize:    protowire.SizeVarint(wiretag),
			funcs:      funcs, // *
			mi:         childMessage,
			validation: newFieldValidationInfo(mi, si, fd, ft),
			isPointer:  fd.Cardinality() == pref.Repeated || fd.HasPresence(),
			isRequired: fd.Cardinality() == pref.Required,
		}
		mi.orderedCoderFields = append(mi.orderedCoderFields, cf) // *
		mi.coderFields[cf.num] = cf
```

函数fieldCoder会根据字段类型选择不同的pointerCoderFuncs，下面是uint32字段的。

```go
var coderUint32 = pointerCoderFuncs{
	size:      sizeUint32, //计算字段大小
	marshal:   appendUint32, //序列化字段
	unmarshal: consumeUint32,//反序列号字段
	merge:     mergeUint32,//合并吧
}
```



根据一个proto  message和field id得到它field的类型

```go
m := TestMsg{}
md := m.ProtoReflect().Descriptor()
fd := md.Fields().ByNumber(protoreflect.FieldNumber(fieldid))
if fd == nil {
	return nil
}
v := m.Mutable(fd)
lmi, ok := v.Interface().(interface{ LoadMessageInfo() *protoimpl.MessageInfo })
if !ok {
	return nil
}
ms := lmi.LoadMessageInfo()
fmt.Println("GoReflectType", ms.GoReflectType)
```

加菜，getMessageInfo里判断类型和是否有某个接口的写法：

```go
func getMessageInfo(mt reflect.Type) *MessageInfo {
	m, ok := reflect.Zero(mt).Interface().(pref.ProtoMessage)
	if !ok {
		return nil
	}
	mr, ok := m.ProtoReflect().(interface{ ProtoMessageInfo() *MessageInfo })
	if !ok {
		return nil
	}
	return mr.ProtoMessageInfo()
}
```

