# package Context里的说明

golang有个context的包，它定义了一个Context 类型，该类型在API边界之间以及进程之间传递截止日期，取消信号和其他请求范围的值。

传入服务器的请求应创建一个Context，传出调用到服务器的就接受一个Context（Incoming requests to a server should create a Context, and outgoing calls to servers should accept a Context.）。在函数调用链上，函数必须传递这个Context，或使用 **WithCancel**, **WithDeadline**, **WithTimeout**, or **WithValue**创建一个派生的Context替代它。当一个Context取消，所有派生自它的Contexts也会被取消。

WithCancel, WithDeadline, and WithTimeout 函数以一个父Context作为参数，返回一个派生的Context（子Context）和CancelFunc取消函数。调用CancelFunc取消这个子Context和它的 孩子Context，删除父Context的引用，和停止相关时钟。未能调用CancelFunc会使子代及其子代泄漏，直到父代被取消或计时器触发。go vet tool 检查所有控制流路径上是否都使用了CancelFuncs。

使用Context的程序应遵循以下规则，以使各个包之间的接口保持一致，并使静态分析工具可以检查上下文传播：

不要将Context存储在结构类型中；而是将Context明确传递给需要它的每个函数。Context应该是第一个参数，通常命名为ctx：

```go
func DoSomething(ctx context.Context, arg Arg) error {
	// ... use ctx ...
}
```

不要传递一个nil Context，即便函数允许。如果不知道传递哪个Context就用context.TODO。

context Values仅用作传递进程和API的请求范围数据，而不用于将可选参数传递给函数。（Use context Values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions.）

同样的一个Context可以传递给运行在不同的goroutines的函数，Context对于由多个goroutine同时使用是安全的。

# Context应该有哪里用？

按我目前的理解是，在有请求进入服务器时，这时可以创建个Context，然后在其它需要异步执行的操作中，例如读写数据库、Redis、IO等，在发读写请求时传入Context。在超时或客户端已断开链接时，可以让还没完成任务的goroutine提前退出，节省服务器资源。

# 创建Context

package context里的函数和方法

[func WithCancel(parent Context) (ctx Context, cancel CancelFunc)](https://golang.org/pkg/context/#WithCancel)

[func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)](https://golang.org/pkg/context/#WithDeadline)

[func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)](https://golang.org/pkg/context/#WithTimeout)

[type CancelFunc](https://golang.org/pkg/context/#CancelFunc)

[type Context](https://golang.org/pkg/context/#Context)

  [func Background() Context](https://golang.org/pkg/context/#Background)

  [func TODO() Context](https://golang.org/pkg/context/#TODO)

  [func WithValue(parent Context, key, val interface{}) Context](https://golang.org/pkg/context/#WithValue)

## context.Background() ctx Context

返回一个empty context，它用于最顶级的地方（main或者请求handler里的开始处）。它可以生成子Context。

## context.TODO() ctx Context

也返回一个empty context，在最顶级或你不知道用哪个Context时，或在实现某功能时还没有具体的Context时用。

在[代码](https://golang.org/src/context/context.go)里，这个TODO的实现和上面的Background是一样的，区别在于，静态分析工具可以使用它来验证Context是否正确传递，这是一个重要的细节，因为静态分析工具可以帮助及早发现潜在的错误，并且可以将其连接到CI / CD管道中。

下面四个通过派生的方式创建context:

## func WithValue(parent Context, key, val interface{}) Context

从parent Context派生出一个子Context，里面包含key val的键值对，键值对是随着context树传递下去。这意味着一旦获得带有值的Context，从此派生的任何Context都将获得该值。不建议使用Context值来传递关键参数，相反是在函数中明确使用对应参数。

## context.WithCancel(parent Context) (ctx Context, cancel CancelFunc)

这里有个CancelFunc函数，正常是那个函数调用WithCancel创建的，由它来调用CancelFunc。不要将这个CancelFunc作为参数来传递。

## context.WithDeadline(parent Context, d time.Time) (ctx Context, cancel CancelFunc)

超时和调用Cancel时可取消。

```go
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2 * time.Second))
```

## context.WithTimeout(parent Context, timeout time.Duration) (ctx Context, cancel CancelFunc)

与WithDeadline相似，只是第二个参数不一样。

# 在函数中接受和使用Context

下面的例子中，在一个函数里接收一个Context并启动一个goroutine，同时等待goroutine返回或context取消。select语句帮助我们选择最先出现的情况。

**总结**：如下例子所示，接收Context的函数开始一个新goroutine后，通过select  case <-ctx.Done(): 和case goroutine是否完成（通常是一个数据channel，看数据是否回来）来判断怎么退出函数。

例子：

```go
package main

import (
  "context"
  "fmt"
  "math/rand"
  "time"
)

//Slow function
func sleepRandom(fromFunction string, ch chan int) {
  //defer cleanup
  defer func() { fmt.Println(fromFunction, "sleepRandom complete") }()

  //Perform a slow task
  //For illustration purpose,
  //Sleep here for random ms
  seed := time.Now().UnixNano()
  r := rand.New(rand.NewSource(seed))
  randomNumber := r.Intn(100)
  sleeptime := randomNumber + 100
  fmt.Println(fromFunction, "Starting sleep for", sleeptime, "ms")
  time.Sleep(time.Duration(sleeptime) * time.Millisecond)
  fmt.Println(fromFunction, "Waking up, slept for ", sleeptime, "ms")

  //write on the channel if it was passed in
  if ch != nil {
    ch <- sleeptime
  }
}

//Function that does slow processing with a context
//Note that context is the first argument
func sleepRandomContext(ctx context.Context, ch chan bool) {

  //Cleanup tasks
  //There are no contexts being created here
  //Hence, no canceling needed
  defer func() {
    fmt.Println("sleepRandomContext complete")
    ch <- true
  }()

  //Make a channel
  sleeptimeChan := make(chan int)

  //Start slow processing in a goroutine
  //Send a channel for communication
  go sleepRandom("sleepRandomContext", sleeptimeChan)

  //Use a select statement to exit out if context expires
  select {
  case <-ctx.Done():
    //If context is cancelled, this case is selected
    //This can happen if the timeout doWorkContext expires or
    //doWorkContext calls cancelFunction or main calls cancelFunction
    //Free up resources that may no longer be needed because of aborting the work
    //Signal all the goroutines that should stop work (use channels)
    //Usually, you would send something on channel, 
    //wait for goroutines to exit and then return
    //Or, use wait groups instead of channels for synchronization
    fmt.Println("sleepRandomContext: Time to return")
  case sleeptime := <-sleeptimeChan:
    //This case is selected when processing finishes before the context is cancelled
    fmt.Println("Slept for ", sleeptime, "ms")
  }
}

//A helper function, this can, in the real world do various things.
//In this example, it is just calling one function.
//Here, this could have just lived in main
func doWorkContext(ctx context.Context) {

  //Derive a timeout context from context with cancel
  //Timeout in 150 ms
  //All the contexts derived from this will returns in 150 ms
  ctxWithTimeout, cancelFunction := context.WithTimeout(ctx, time.Duration(150)*time.Millisecond)

  //Cancel to release resources once the function is complete
  defer func() {
    fmt.Println("doWorkContext complete")
    cancelFunction()
  }()

  //Make channel and call context function
  //Can use wait groups as well for this particular case
  //As we do not use the return value sent on channel
  ch := make(chan bool)
  go sleepRandomContext(ctxWithTimeout, ch)

  //Use a select statement to exit out if context expires
  select {
  case <-ctx.Done():
    //This case is selected when the passed in context notifies to stop work
    //In this example, it will be notified when main calls cancelFunction
    fmt.Println("doWorkContext: Time to return")
  case <-ch:
    //This case is selected when processing finishes before the context is cancelled
    fmt.Println("sleepRandomContext returned")
  }
}

func main() {
  //Make a background context
  ctx := context.Background()
  //Derive a context with cancel
  ctxWithCancel, cancelFunction := context.WithCancel(ctx)

  //defer canceling so that all the resources are freed up 
  //For this and the derived contexts
  defer func() {
    fmt.Println("Main Defer: canceling context")
    cancelFunction()
  }()

  //Cancel context after a random time
  //This cancels the request after a random timeout
  //If this happens, all the contexts derived from this should return
  go func() {
    sleepRandom("Main", nil)
    cancelFunction()
    fmt.Println("Main Sleep complete. canceling context")
  }()
  //Do work
  doWorkContext(ctxWithCancel)
}

```









参考：

https://golang.org/pkg/context/

http://p.agnihotry.com/post/understanding_the_context_package_in_golang/