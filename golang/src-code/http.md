### Server 结构体

该结构体实现从 path 到 handler 的注册和映射能力。它里边的属性实在太多，不方便一一举出来。

可以先关注几个：

```go
type Server struct{
	Addr string
	Handler Handler
}
```



### Handler 接口

```go
type Handler interface {
   ServeHTTP(ResponseWriter, *Request)
}
```



### ResponseWriter 接口

```go
type ResponseWriter interface{
    Header() Header
    Write([]byte)(int,error)
    Write(statusCode int)
}
```



