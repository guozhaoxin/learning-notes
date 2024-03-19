tunny 是一个 golang 的协程池库，它的逻辑大概是这样的：程序启动时，用户指定一个数量，先创建一些协程放在进程中。当用户需要处理业务时，就通过 channel 同步给这些协程发送请求；这些协程中，如果有空闲的，则有一个可以获取到请求进行处理；如果都在忙碌，则调用方只能同步阻塞等待。



tunny 代码很短，大概不到一千行，而且还有一个有点就是没有依赖任何非标库。



Pool 结构体

tunny 中对外暴露的是 Pool 结构体，就如它的字面意思，它负责追踪所有负责消费的协程。

它的结构体如下：

```golang
type Pool struct {
	queuedJobs int64 // 记录现在有多少协程在实际处理请求
	ctor    func() Worker // 一个函数，在构建 Pool 时指定好，负责生成 Worker 对象
	workers []*workerWrapper // 追踪所有开辟好的协程
	reqChan chan workRequest // 内部使用的一个通道，调用方通过这个channel获取一个空闲协程
	workerMut sync.Mutex // 内部锁，主要是当用户调整 Pool 的大小时使用
}
```

相关的参数说明如注释所示，没有一个参数是多余的。

这个结构体有很多方法：

同步阻塞处理请求

```golang
func (p *Pool) Process(payload interface{}) interface{}
```

这是一个同步方法，当被其它方调用时，会阻塞其它协程，直到其中一个 woker 将结果处理完毕。

带超时的同步阻塞处理请求

```golang
func (p *Pool) ProcessTimed(payload interface{},timeout time.Duration) (interface{}, error)
```

带 ctx 的同步阻塞处理请求

```golang
func (p *Pool) ProcessCtx(ctx context.Context, payload interface{}) (interface{}, error)
```

上面两个方法与 Process 很像，唯一的区别就在于它俩可以被取消，取消后，返回给调用方的可能会有 error。

获取正在实际请求的 worker 数量

```golang
func (p *Pool) QueueLength() int64
```

池中虽然会提前创建好协程，但是可能并不是所有协程都在处理请求，因此 Pool 提供一个方法，用来表示当前有多少个协程在实际处理请求。

调整池中协程数量

```golang
func (p *Pool) SetSize(n int)
```

tunny 的 Pool 支持动态调整协程池大小，当它变大时，会新增一些协程；如果变小时，会停掉多余的协程。

获取协程池中 worker 数量

```golang
func (p *Pool) GetSize() int
```

关闭 Pool

```golang
func (p *Pool) Close()
```

这个方法会停掉池中所有 worker 的运行，并关闭相应的 channel。



Worker 接口

这个接口代表一个 working agent，它需要实现的方法如下：

```golang
type Worker interface {
	// 这是一个同步方法，当需要处理一个请求时，就是通过该方法进行处理
  Process(interface{}) interface{}
	// 一个 worker 处理业务时，可能在每次处理前，需要做一些预制工作，这个方法就是这个作用，
  // 这个方法执行完毕后，表示协程已经做好准备，可以处理请求了
	BlockUntilReady()

	// 当任务被取消时，本 worker 不需要再继续执行下去了，则调用该方法
	Interrupt()

	// 当一个 worker 从池中移走时，这个方法负责做一些清理工作
	Terminate()
}
```

tunny 中，有两个类型实现了 Worker 接口，一个是 callbackWorker，一个是 closureWorker，这两个结构体没有大的区别，后面三个方法都是空，只有 Process 不同；对于 callbackWorker 来说，它 Process 方法接受的参数要求是一个无参 function；对于 closureWorker，它的 Process 方法接受的 payload 则没有这个限制，因为它有一个属性 processor，本身就是一个 func(interface{}) interface{}，在执行 Process 时，payload 参数会传给这个方法进行处理。

tunny 中提供了几个构造 Pool 的函数：

```golang
func New(n int, ctor func() Worker) *Pool
```

设置 Pool 的初始大小和扩容时如何生成一个 Worker。

```golang
func NewFunc(n int, f func(interface{}) interface{}) *Pool
```

这个函数没有扩容时如何生成 Worker，但它传了一个 function 进来，因为它扩容时，是放 closureWorker 进去的。

```golang
func NewCallback(n int) *Pool
```

这个更简单，它只能指定初始化大小，然后内部使用 callbackWorker 进行封装。所以使用这个类型的话，payload 就只能是一个 function。



workRequest 结构体

先看看这个结构体的定义：

```golang
type workRequest struct {
	// 其它协程通过这个给 work 传入要处理的数据
	jobChan chan<- interface{}
	// 其它协程从这里读取处理结果
	retChan <-chan interface{}
	// 如果其它协程想终止这个处理，则调用这个接口，关闭 retChan 
	interruptFunc func()
}
```

Pool 中每个协程，当它空闲时，就会自己生成一个这样的对象，并告诉外界如何给它传参，以及如何获取处理结果。

它纯粹就是一个掮客，没有任何自己的方法。



workerWrapper 结构体

Pool 生成协程时，并不是直接操作 Worker ，而是通过 wrapper 进行操作的，参见 Pool 中的 workers 属性。每个 workerWrapper 会包一个 Worker，负责管理 Worker 和协程的生命周期。

```golang
type workerWrapper struct {
	worker        Worker
	interruptChan chan struct{} // 消费被取消
	// reqChan is NOT owned by this type, it is used to send requests for work.
	reqChan chan<- workRequest
	// closeChan can be closed in order to cleanly shutdown this worker. 关闭 worker 这个和下面的通道由什么区别？
	closeChan chan struct{}
	// closedChan is closed by the run() goroutine when it exits. 协程退出时要调用这个 channel，表示协程确实退出了
	closedChan chan struct{}
}
```

workerWrapper 有几个自己的方法：

```golang
func (w *workerWrapper) interrupt()
```

当协程正在处理数据，然后其它协程想把它取消时，就调用这个，提前终止处理。

```golang
func (w *workerWrapper) run()
```

当新建一个 workerWrapper 时使用，需要在 new 的同时，调用 run 启动一个协程，这个就是协程池中的成员。它内部本质上就是一个大型循环，不停的通过 reqChan 从外界获取要处理的数据，处理结束后进入下一次循环。这个方法是 tunny 的核心逻辑之一。

```golang
func (w *workerWrapper) stop()
```

异步结束协程，它会给协程发信号，告诉它别跑了，终止吧。

```golang
func (w *workerWrapper) join()
```

与上面有一些类似，但它是同步的，等待本协程确实终止。

Tunny 中提供了一个  newWorkerWrapper 函数，用来生成一个 workerWrapper 对象，在生成这个对象的同时，启动一个新协程负责消费。



当用户构建好一个 Pool 后，会启动指定的消费协程，这些协程每个对应一个 workerWrapper，构建 workerWrapper 时，Pool 会将 reqChan 传给各个对象；这些协程在空闲时会尝试往这个 chan 中发送一个 request 对象，而这个对象包含了三个属性，jobChan 用来给该协程传递要消费的参数，retChan 用来给用户传递消费结果，interruptFunc 可以用来终止本次消费；当用户想要消费时，就会尝试从 reqChan 中获取一个该对象，因为如果有空闲协程，至少有一个会发送request成功，用户这时候就从中获取到一个request对象，然后通过其中的 jobChan 将要消费的数据传给协程，并且监听 retChan，从中获取消费结果。

正是通过这样的机制，tunny 实现了协程池消费的需求。
