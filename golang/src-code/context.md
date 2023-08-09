context 接口

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

emptyCtx

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

实现 context 接口，但都是空实现；而且永远取消不了。

这个结构体只有两个实例化的对像：

```go
func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```

这两个实例化对像，可以通过 TODO 和 Background 函数获取到。



CancelFunc 

```go
type CancelFunc func()
```



cancelCtx

```go
type cancelCtx struct {
	Context
	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

这个结构体中，可以挂载很多子类 canceler。

**<font color=red>cancelCtx 中的一个疑问：</font>**

```go
func (c *cancelCtx) Err() error {
	c.mu.Lock() // todo gzx 这里为什么也要上锁？？？？
	err := c.err
	c.mu.Unlock()
	return err
}
```

它最复杂的方法就是 cancel 了：

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan) // 为什么这里是赋值 而下面却是关闭？因为这个 closedChan 早关闭了
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err) // 子类要不要移走由子类自己决定？
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```



```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```



canceler 接口

```go
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

这种类型可以直接取消。

context 中有两个结构体实现了它，一个是上面的 cancelCtx，一个是 timerCtx。

