# http.Handler wrapper 中文

原文链接:  [The http.Handler wrapper technique in #golang UPDATED](https://medium.com/@matryer/the-http-handler-wrapper-technique-in-golang-updated-bc7fbcffa702)

## http.Handler Wrapper 是只有一个输入和输出，且两者都为 `http.Handler`类型的函数。

Wrapper's signature `func(http.Handler) http.Handler`

它的概念是: 函数接受 `http.Handler` 作为参数并且返回一个新的`http.Handler`，可以在 `http.Handler` 被调用前后做想做的事情，甚至可以决定最终是否调用传入的`http.Handler`。

```go
func log(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
		log.Println("Before")
		h.ServeHTTP(w, r) // call original
		log.Println("After")
	})
}
```

在这里， `log` 函数返回一个新的 `http.Handler`.(`http.HandlerFunc` 也是一个合法的 `http.Handler`). 它将先打印 "Before". 然后在打印"after"之前调用传入的 handler.

然后这样使用 `log`

`http.Handle("/path", handleThing)`

变成

`http.Handle("/path", log(handleThing))`

## 什么时候使用 wrappers?
这个方法可以被用于很多地方，例如

* 记录和追踪日志和用户行为
* 校验请求，例如检测登陆信息
* 写入通用的 header 信息。

## 是否调用原有 handler
Wrappers 需要决定是否调用传入的 handler。如果有需要，wrapper 甚至可以自己拦截请求。假设在 API 中`key`URL参数是必须的:

```go
func checkAPIKey(h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if len(r.URL.Query().Get("key") == 0) {
      http.Error(w, "missing key", http.StatusUnauthorized)
      return // don't call original handler
    }
    h.ServeHTTP(w, r)
  })
}
```

`checkAPIKey` wrapper 会确认 url 里是否包含 key。如果没有，直接返回 `Unauthorized`错误。这个 wrapper 可以被扩展，例如：在数据仓库中校验 key。或者确认请求发送频率在可接受的范围之内.

## Deferring (延迟)
通过 Go 的 `defer` 方法，我们可以确保无论传入的 handler 内发生了什么，都会被调用的代码:

```go
func log(h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    log.Println("Before")
    defer log.Println("After")
    h.ServeHTTP(w, r)
  })
}
```

这样，就算 `h.ServeHTTP(w, r)` 出现了 panic，我们依然可以看到 "After" 被打印出来.

## 向 wrapper 传递参数

我们可以通过向 wrapper 传递参数来实现更多的功能。例如之前的 `checkAPIKey` wrapper 可以被改为支持检测任意参数。而不仅仅是只能检测 `key` 这一个。

```go
func MustParams(h http.Handler, params ...string) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){

    q := r.URL.Query()
    for _, param := range params {
      if len(q.Get(param)) == 0 {
        http.Error(w, "missing "+param, http.StatusBadRequest)
        return // exit early
      }
    }
    h.ServeHTTP(w, r) // all params present, proceed
  })
}
```

我们可以通过向 `MustParams` 传入不同的参数来检测不同的 parameter。`params`参数在返回的闭包函数中是可见的。所以不需要创建 struct 来存储状态。(原文: so there’s no need to add a struct to store state here. 个人理解是不需要额外显式传递 params 到内部函数).

可以这样调用 `MustParams`

```go
http.Handler("/user", MustParams(handleUser, "key", "auth"))
http.Handler("/group", MustParams(handleGroup, "key"))
http.Handler("/items", MustParams(handleSearch, "key", "q"))
```

## 在 wrappers 中的 wrappers
考虑到这种模式的自相似性, 且实际上我们并没有更改 `http.Handler`的签名。我们可以很容易地把 wrappers 串联起来调用，如同把它们有趣地嵌套在一起.

假设我们有一个名为  `MustAuth` 的 wrapper。它的作用是校验请求中的 token。它需要 auth 参数必须存在。那么我们可以在 `MustAuth`中使用 `MustParams`:

```go
func MustAuth(h http.Handler) http.Handler {
  checkauth := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
    err := validateAuth(r.URL.Query().Get("auth"))
    if err != nil {
      http.Error(w, "bad auth param", http.StatusUnauthorized)
    }
    h.ServeHTTP(w, r)
  })
  return MustParams(checkauth, "auth")
}
```

## 拦截 ResponseWriter

`http.ResponseWriter`是一个接口， 这就意味着我们可以拦截一个请求并把它转向另一个对象，只要我们提供了接口所需的方法。

你可以决定是否要捕获 response body，或者只是记录它

```go
func log(h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w := NewResponseLogger(w)
    h.ServeHTTP(w, r)
  })
}
```

`NewResponseLogger`将会提供它自己的 `Write` 函数并记录数据。

```go
func (r *ResponseLogger) Write(b []byte) (int, error) {
  log.Print(string(b)) // log it out
  return r.w.Write(b) // pass it to the original ResponseWriter
}
```

这个函数记录了 response body 并且把读取出来的值又写入了初始的 `http.ResponseWritter`. 如此 API 的调用者可以依然得到 response body。（Go 中 response body 一次性读取的，读取之后就清空了）

## Handlers, all the way down
除了像我们刚刚所做的包装单个handler之外，既然所有都是 `http.Handler`.我们可以把整个 server 包在一起，像这样:

`http.ListenAndServe(addr, log(server))`


关于更多 middleware 的信息，请查看   [https://medium.com/@matryer/writing-middleware-in-golang-and-how-go-makes-it-so-much-fun-4375c1246e81](https://medium.com/@matryer/writing-middleware-in-golang-and-how-go-makes-it-so-much-fun-4375c1246e81)






















