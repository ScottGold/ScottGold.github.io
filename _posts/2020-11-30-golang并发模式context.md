https://blog.golang.org/context

# Go的并发模式：Context （上下文）

在Go的服务器中，每个的进来的请求都是由独立的goroutine来处理的。处理请求的goroutine中通常会启动额外的goroutines去访问数据库和RPC服务。处同一个请求的这群goroutines通常需要访问特定的请求值，例如：终端用户的身份识别，授权令牌和请求的截止日期。当一个请求被取消或超时时，这个请求所有相关的goroutines都应该快速退出，以便系统回收相关资源。

谷歌开发了一个context的包，使得在一个处理请求中的所有goroutine中跨API边界传递请求范围内的值，取消信号，截止时间很方便（makes it easy to pass request-scoped values, cancelation signals, and deadlines across API boundaries to all the goroutines involved in handling a request.）。

# Context

```go
// 一个Context包含一个 Deadline, cancelation signal, 和 request-scoped values
// across API boundaries. 它的方法是可供多个goroutines安全地同时使用
type Context interface {
    // Done返回一个channel，当Context取消时或超时时，它会被closed。
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

**Done**方法返回一个channel，它作为Context中运行的相关函数的一个取消信号，当channel关闭时，相关函数需要放弃它们的工作并返回。**Err**方法返回一个错误，代表Context为什么取消。这时有篇[文章](https://blog.golang.org/pipelines)介绍Done channel的更多细节。

Context没有一个Cancel的方法，与Done channel是只接收的原因是一样的：接收取消信号的函数通常不是发送信号函数。特别是，当父操作为子操作启动goroutines时，这些子操作应该不能取消父操作。相反，**WithCancel**函数为取消一个新的Context值提供了一个途径。

**Context**是多goroutine安全的。代码中可以将一个Context传递给多个goroutine，和取消Context以通知goroutine。

# 派生Derived contexts

context包提供了从现有context值中派生新的Context值的功能。它们形成一颗树。当一个Context取消时，所有从它派生的Contexts都会取消。

```go
// Background 返回一个空的(empty) Context. 它永远不会被取消，没有deadline,没有values。
// Background is typically used in main, init, and tests,
// and as the top-level Context for incoming requests.
func Background() Context
```

Backgroud是任何的Context树的根，永远不会被取消。

**WithCancel** 和**WithTimeout**返回的派生的context可以比父context早点取消。当请求处理返回时，与这个请求相关的那个Context通常会被取消。WithCancel对于取消冗余请求也很有用。WithTimeout对于设置对后端服务器的请求的deadline很有用：

```go
// WithCancel 返回父Context的一个拷贝，当父Context的Done被 关闭或cancel被调用时，子Context的Done也会尽可能快地被关闭。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

// A CancelFunc cancels a Context.
type CancelFunc func()

// WithTimeout returns a copy of parent whose Done channel is closed as soon as
// parent.Done is closed, cancel is called, or timeout elapses. The new
// Context's Deadline is the sooner of now+timeout and the parent's deadline, if
// any. If the timer is still running, the cancel function releases its
// resources.
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)


// WithValue returns a copy of parent whose Value method returns val for key.
func WithValue(parent Context, key interface{}, val interface{}) Context
```

# 例子：Google网页搜索

我们这个例子中是一个http服务端，处理 `/search?q=golang&timeout=1s` 这样的URLs，通过转发查询关键词"golang"给 [Google Web Search API](https://developers.google.com/web-search/docs/) ，然后渲染结果。超时参数告诉服务器在该持续时间过去之后取消请求。

代码跨三个包（The code is split across three packages:）

- [server](https://blog.golang.org/context/server/server.go) 提供main函数和/search的处理函数（handler）
- [userip](https://blog.golang.org/context/userip/userip.go) 提供从请求中提取用户IP地址并将其与上下文关联的功能。
- [google](https://blog.golang.org/context/google/google.go) 提供Search函数，将查询发给Google。

## Server程序

```go
package main

import (
	"context"
	"html/template"
	"log"
	"net/http"
	"time"

	"golang.org/x/blog/content/context/google"
	"golang.org/x/blog/content/context/userip"
)

func main() {
	http.HandleFunc("/search", handleSearch)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

// handleSearch handles URLs like /search?q=golang&timeout=1s by forwarding the
// query to google.Search. If the query param includes timeout, the search is
// canceled after that duration elapses.
func handleSearch(w http.ResponseWriter, req *http.Request) {
	// ctx is the Context for this handler. Calling cancel closes the
	// ctx.Done channel, which is the cancellation signal for requests
	// started by this handler.
	var (
		ctx    context.Context
		cancel context.CancelFunc
	)
	timeout, err := time.ParseDuration(req.FormValue("timeout"))
	if err == nil {
		// The request has a timeout, so create a context that is
		// canceled automatically when the timeout expires.
		ctx, cancel = context.WithTimeout(context.Background(), timeout)
	} else {
		ctx, cancel = context.WithCancel(context.Background())
	}
	defer cancel() // Cancel ctx as soon as handleSearch returns.

	// Check the search query.
	query := req.FormValue("q")
	if query == "" {
		http.Error(w, "no query", http.StatusBadRequest)
		return
	}

	// Store the user IP in ctx for use by code in other packages.
	userIP, err := userip.FromRequest(req)
	if err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	ctx = userip.NewContext(ctx, userIP)

	// Run the Google search and print the results.
	start := time.Now()
	results, err := google.Search(ctx, query)
	elapsed := time.Since(start)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	if err := resultsTemplate.Execute(w, struct {
		Results          google.Results
		Timeout, Elapsed time.Duration
	}{
		Results: results,
		Timeout: timeout,
		Elapsed: elapsed,
	}); err != nil {
		log.Print(err)
		return
	}
}

var resultsTemplate = template.Must(template.New("results").Parse(`
<html>
<head/>
<body>
  <ol>
  {{range .Results}}
    <li>{{.Title}} - <a href="{{.URL}}">{{.URL}}</a></li>
  {{end}}
  </ol>
  <p>{{len .Results}} results in {{.Elapsed}}; timeout {{.Timeout}}</p>
</body>
</html>
`))
```

## userip包

```go
package userip

import (
	"context"
	"fmt"
	"net"
	"net/http"
)

// FromRequest 从req中提取用户IP地址, 如果有的话
func FromRequest(req *http.Request) (net.IP, error) {
	ip, _, err := net.SplitHostPort(req.RemoteAddr)
	if err != nil {
		return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
	}

	userIP := net.ParseIP(ip)
	if userIP == nil {
		return nil, fmt.Errorf("userip: %q is not IP:port", req.RemoteAddr)
	}
	return userIP, nil
}

// key的类型不导出，通过这样防止其它包中使用同样的key而引起冲突。got it
type key int

// userIPkey 是用户IP地址的context key, 它的零值是任意的。
// 如果当前包内定义其它context keys, 它们会有不同的整形值。
const userIPKey key = 0

// NewContext returns a new Context carrying userIP.
func NewContext(ctx context.Context, userIP net.IP) context.Context {
	return context.WithValue(ctx, userIPKey, userIP)
}

// FromContext extracts the user IP address from ctx, if present.
func FromContext(ctx context.Context) (net.IP, bool) {
	// ctx.Value returns nil if ctx has no value for the key;
	// the net.IP type assertion returns ok=false for nil.
	userIP, ok := ctx.Value(userIPKey).(net.IP)
	return userIP, ok
}
```

## google包

```go
package google

import (
	"context"
	"encoding/json"
	"net/http"

	"golang.org/x/blog/content/context/userip"
)

// Results is an ordered list of search results.
type Results []Result

// A Result contains the title and URL of a search result.
type Result struct {
	Title, URL string
}

// Search sends query to Google search and returns the results.
func Search(ctx context.Context, query string) (Results, error) {
	// Prepare the Google Search API request.
	req, err := http.NewRequest("GET", "https://ajax.googleapis.com/ajax/services/search/web?v=1.0", nil)
	if err != nil {
		return nil, err
	}
	q := req.URL.Query()
	q.Set("q", query)

	// If ctx is carrying the user IP address, forward it to the server.
	// Google APIs use the user IP to distinguish server-initiated requests
	// from end-user requests.
	if userIP, ok := userip.FromContext(ctx); ok {
		q.Set("userip", userIP.String())
	}
	req.URL.RawQuery = q.Encode()

	// Issue the HTTP request and handle the response. The httpDo function
	// cancels the request if ctx.Done is closed.
	var results Results
	err = httpDo(ctx, req, func(resp *http.Response, err error) error {
		if err != nil {
			return err
		}
		defer resp.Body.Close()

		// Parse the JSON search result.
		// https://developers.google.com/web-search/docs/#fonje
		var data struct {
			ResponseData struct {
				Results []struct {
					TitleNoFormatting string
					URL               string
				}
			}
		}
		if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
			return err
		}
		for _, res := range data.ResponseData.Results {
			results = append(results, Result{Title: res.TitleNoFormatting, URL: res.URL})
		}
		return nil
	})
	// httpDo waits for the closure we provided to return, so it's safe to
	// read results here.
	return results, err
}

// httpDo issues the HTTP request and calls f with the response. If ctx.Done is
// closed while the request or f is running, httpDo cancels the request, waits
// for f to exit, and returns ctx.Err. Otherwise, httpDo returns f's error.
func httpDo(ctx context.Context, req *http.Request, f func(*http.Response, error) error) error {
	// Run the HTTP request in a goroutine and pass the response to f.
	c := make(chan error, 1)
	req = req.WithContext(ctx)
	go func() { c <- f(http.DefaultClient.Do(req)) }()
	select {
	case <-ctx.Done():
		<-c // Wait for f to return.
		return ctx.Err()
	case err := <-c:
		return err
	}
}
```

# 总结

在Google，我们要求Go程序员，在传入和传出请求之间的调用路径上，将Context作为每个函数中的第一个参数传递。这使许多不同团队开发的Go代码可以很好地进行互操作。它提供对超时和取消的简单控制，并确保诸如安全性凭证之类的关键值正确地传递Go程序。

