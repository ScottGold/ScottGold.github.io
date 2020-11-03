参考

https://go101.org/article/channel.html

# Channcel的介绍

Rob Pike有个关于并发编程伟大的建议：不要通过共享内存来通信，而是通过通信来共享内存，也就是channels，对共享资源也一样。

当goroutines需要实现共享内存来通信，我们要用到传统的并发同步技术，例如：mutex locks，来保护共享内存，避免数据竞争。用channels 则可以实现通过通信共享内存。

channel可以被看成是一个程序内的FIFO(先进先出)队列。一些goroutines输入数据，另外一些goroutines取出数据。

伴随值在channel中转移，一些值的所有权也可能在goroutine之间发生转移。一个goroutine在将值发送到channel时释放所有权，另一个取到值时获得所有权。当然也有不发生所有权转移的情况。

所属权发生转移的值通常是引用类型，但不一定要是引用的方式去转移。请注意，当我们说所属权时，我们说的是逻辑上的。不像Rust语言，Go不从语法级别上确保所属权。channel可以帮助开发人员轻松写出解决数据竞争的代码。

尽管Go也支持传统的并发同步技术，但仅channel是第一等公民。其实每种同步机制都有它最好的特定应用场景。但channel有更广的应用范围。

# Channel的类型与值

与array,slice和map一样，每个channel有一个元素类型。一个channel只能传输该类型的值。

channel可以是全双工、双向的，也可以是单工、单向的（bi-directional or single-directional）。在编译时识别。

- **chan T** 全双工
- **chan<-T** 只能发送
- **<-chan T** 只能接收

双向的channel值可以隐式转换成单向的，但是单向的则不能变成双向的（显式也不行）。只发和只收的channel也不能互转。

每个channel是有通道容量的(capacity)。容量为零的叫unbuffered channel，非零的叫buffered channel。

channel的零值用nil表示。非零channel用make创建，和map等一样。

```go
//make a channel with buffer size 10
make(chan int, 10)
```

# channel值的比较

所有的channel是可比较的。

非空channel在语言底层是由几部分值组成的，如果一个channel赋值给另外一个，这两个channel将共享底层组成部分值。（channel的系统实现有几个部分的数据，它们共享这些数据、资源）。这两个值的比较是相等的。

# channel的操作

1. 关闭channel.

   ```go
   close(ch)// ch不能是只收的channel
   ```

2. 将值发到channel

   ```go
   ch <- v
   ```

3. 接收 v := <-ch or  <-ch or v,sentBeforeClosed = <-ch．

4. channel的容量，cap(ch)

5. channel中当前值的数量，len(ch)

Go时的大多数是不同步的，也就是它并不是并发安全的。包括赋值、传参、容器(map等)操作。

# channel操作的详细解释

将channel分为下面三类，和对应三个基本操作的响应

| 操作    | a nil channel  | a closed channel   | a not-closed not-nil channel       |
| ------- | -------------- | ------------------ | ---------------------------------- |
| close   | panic          | panic              | succeed to close**(C)**            |
| send    | block for ever | panic              | block or succeed to send**(B)**    |
| receive | block for ever | never block**(D)** | block or succeed to receive**(A)** |

为了更好理解上的情况，了解channel的底层有必要。

我们可以认为，每个channel包含３个队列（可以认为是３个FIFO的）。

1. the receiving goroutine队列，通常是FIFO。它是没有size限制的linked list。在这个队列中的goroutines全是阻塞状态，等待从这个channel接收值。
2. the sending goroutine队列，通常是FIFO。它也是没有size限制的linked list。在这个队列中的goroutines全是阻塞状态，等待往这个channel发值（数据）。直接发值，还是发值的地址，依赖于编译器的实现。每个goroutine尝试发送的值也与该goroutine一起存储在队列中。
3. the value buffer队列，完全是FIFO。是个环形队列,队列的大小是channel的capacity。有趣的是，unbuffered channel总是处于满和空两种状态。

每个channel内部都有一个mutex lock，用来避免各种操作的数据竞争态。

情况A：当一个goroutine **R**尝试从一个not-closed non-nil channel中拉取一个值时，R将先要获得这个channel的锁，然后做下步骤直到一个条件符合。

1. 如果channel的value buffer队列不为空，那么channel的receiving goroutine队列必须是空的，R将从value buffer队列获取到一个值。如果channel的sending goroutine队列也不为空，一个发送中的goroutine将被移出sending goroutine队列，同时恢复running状态。当旧值被移出时，尝试发送值的goroutine将新值压进value buffer队列。R继续running。这种情况下，channel接收操作是非阻塞状态。
2. 如果channel的value buffer队列为空，如果sending goroutine队列不为空，这种情况下，当前channel一定是个unbuffered channel，R直接从sending goroutine队列移出一个goroutine，从这个try send的goroutine接过值,这个try send goroutine将回到running状态。R继续running。这种情况下，channel接收操作是非阻塞状态。
3. 如果channel的value buffer队列和sending goroutine队列同时处于空，R将被放进channel的receiving goroutine队列，进入阻塞状态。当另外一个goroutine往channel发值时，R可能会恢复running状态。这种情况下，channel接收操作是阻塞状态。

情况B：当一个goroutine **S** 尝试发一个值到not-closed non-nil channel时，S需要先获得channel的锁，然后做下列步骤直到一个步骤条件满足。

1. 如果channel的receiving goroutine队列不为空，那么value buffer一定是空的，S将从channel的receiving goroutine队列移出一个receiving goroutine，并将要发的值发给这个receiving goroutine。receiving goroutine恢复到running状态。S继续running。这种情况下，channel发送操作是非阻塞的。
2. 如果channel的receiving goroutine队列是空的，如果value buffer队列不满，那么sending goroutine队列是空的，S要发送的值被放进value buffer队列，S继续运行。这种情况下，channel发送操作是非阻塞的。
3. 如果receiving goroutine是空的，value buffer队列是满的，S将被放进sending goroutine队列，进入阻塞状态。当有其它goroutine从channel取数据时，S才有可能恢复running状态。这种情况下，channel发送操作是阻塞的。

情况C：当一个goroutine尝试close一个not-closed non-nil channel时，一旦获得锁后，将按顺序执行下面两个步骤。

1. 如果channel的receiving goroutine队列不为空，这种情况下它的value buffer一定是空的，所有的receiving goroutine队列中的goroutine被一个接一个地移出，被移出的receiving goroutine将收到一个默认值，同时恢复到running状态。
2. 如果channel的sending goroutine队列不为空，所有在sending goroutine队列的goroutine被一个接一个地移出，同时产生一个因为给closed channel发数据的panic。这也就是我们应该防止在同一个channel中并发send和close操作。事实上，当数据竞态设置(**-race**)是enabled的话，go的标准编译器可能报告send和close操作的竞争态状况。

情况D：在一个non-nil channel关闭后，在这个channel上的接收操作将永不阻塞。在value buffer队列中的值还可以取。随附的第二个可选的布尔返回值仍然为true。一旦value buffer都被取出并接收，任何接收操作将接收该通道的元素类型的默认值，并且channel接收返回的第二个bool将是false。

了解什么是阻塞和非阻塞通道的发送或接收操作对于理解select控制流块的机制很重要，这将在后面的部分中介绍。

根据上面列出的说明，我们可以获得有关channel内部队列的一些事实。

- 如果channel关闭了，它的sending和receiving队列一定是空，但是value buffer队列不一定是空。
- 在任何时候，如果value buffer不为空，它的receiving goroutine队列一定是空
- 在任何时候，如果value buffer不为满，它的sending goroutine队列一定是空。
- 如果channel的buffer size不为零，在任何时候，它的sending, receiving goroutine队列必然有一个是空的。
- 如果channel的buffer size为零，unbuffered channel，在任何时候，通常下它的sending, receiving goroutine队列必然有一个是空的。但是，有一个例外，一个goroutine可能在select控制块里被放进两个队列。

# Channel元素值通过copy传输

当一个值从一个goroutine传到另一个goroutine，它至少会被copy一次。如果这个值有在value buffer中待过，那个会有两次的copy。当将值从发送程序goroutine复制到值缓冲区时，将发生一个副本，当将值从值缓冲区复制到接收程序goroutine时，将发生另一个副本。就像赋值和函数参数传递一样，在传递值时，仅复制其直接部分。

对于标准的Go编译器，通道元素类型的大小必须小于65536。

但是，通常，我们不应该创建具有大元素类型的通道，以避免在goroutine之间传递值的过程中复制成本过高。因此，如果传递的值大小太大，则最好改用指针元素类型，以避免大量的值复制成本。

# 关于channel和goroutine的垃圾回收

注意，该通道的发送或接收goroutine队列中的所有goroutine都引用了该通道，因此，如果该通道的两个队列都不为空，则该通道肯定不会被垃圾回收。另一方面，如果goroutine被阻止并停留在通道的发送或接收goroutine队列中，那么即使该goroutine仅引用了该通道，也不能肯定地对其进行垃圾回收。实际上，goroutine只能在退出时才被垃圾回收。

# channel的send / receive操作是simple statements

channel的receive操作可以始终用作单值表达式。

# Channel的`for-range`

```go
for v := range aChannel {
	// use v
}
```

和下面等同

```go
for {
	v, ok = <-aChannel
	if !ok {
		break
	}
	// use v
}
```

当然，此处的aChannel值一定不能是仅发送通道。如果它是零通道，则循环将永远在那里阻塞。

# select-case

select-case是专为channel设计。与switch-case相似，但有下面不同。

- 在select {之间不能有表达式和语句
- case之间允许有fallthrough语句
- case后必须是channel receive 或 send操作。
- 如果有多个非阻塞的case操作，go将随机选择一个运行。
- 当所有的case操作处于阻塞态时，default会运行。如果没有default，goroutine将进入阻塞状态。

select {}将使当前goroutine永远保持阻塞状态。

可能用select实现try-send和try-receive

# select的实现机制

具体看原文章。下次继。

https://go101.org/article/channel.html#select-implementation



