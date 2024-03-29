tree.go

这个文件中定义了 node 结构体。

# node 结构体

在 gin 中，engine 会将每个注册的路由进行解构，然后拆成一个一个的节点，形成一棵树，专业的说法是 Radix Tree。

这种树的特点是，它是一种前缀树，有公共前缀的节点，会共享同一个父节点，如下图：



![](https://www.liwenzhou.com/images/Go/gin/radix_tree.png)

如上图，可以看到每个单词都是怎么存储的；从根节点开始，顺着每个节点一直到叶子，即可得到一个完整的单词。

当然，gin 中这个树要更复杂，因为毕竟不是单纯的字符串匹配，还要考虑模糊匹配和参数匹配两种情况。

## node 定义

node 的定义如下：

```go
type node struct {
	path      string
	indices   string
	wildChild bool
	nType     nodeType
	priority  uint32
	children  []*node
	handlers  HandlersChain
	fullPath  string
}
```

以上各字段解释如下：

- path，每个节点都有 path 属性，它会记录当前节点相对父节点不同以及所有子节点共享的部分；
- indices，一个字符串，每个节点的子节点的第一个字符都会存储在这个字符串里边，方便快速搜索；此外这个字符串中各个字符顺序是有讲究的，它会按照子节点的 priority 排序，即一条路径的优先级越高，则它的首字符在这个字段存储的位置越靠近0；
- wildChild，节点是否是模糊匹配；
- nType，节点的类型，节点共有四种类型，static 是普通节点，root  是根节点 ，param 是参数节点，catchAll 是模糊匹配；从 0 开始 自增，nodeType 就是一个整形；
- priority，优先级，记录的是一个节点中，所有有处理函数的子节点及自身的节点数总和，有的节点是中间节点，只是表示多个子节点共享前缀，没有处理函数；
- children, 一个数组，跟踪一个节点的所有子节点，它和 indices 字段是一一相对应的，虽然初始化时会有不一致，但一条路由注册成功后，就是强一致的，即一个子节点在 childre 中的顺序，和它的第一个字符在 indices 中的顺序是一致的；另外，如果一个节点的 wildChild 为 true，则负责模糊匹配的那个子节点一定在最后，而且一个节点只能有一个子节点做这件事；
- handlers，没啥好说的，路由匹配后，负责按序处理请求；
- fullapath,表示从根节点到当前节点的完整路径；说是这么说，但通过查看和运行 node 的源码，可知面对 catchAll 类型时，并不完全这样。

当向 gin 的 Engine 注册一条路由时，会先找到相应类型的 root node，然后以这个 node 作为起始，递归遍历下去，直到找到合适的插入节点，而调用的方法即是 addRoute 方法。



## node 的方法

node 定义的方法不多，且跟 node 本身定义一样，都是仅限于内部使用。

### addRoute 方法

这个方法会被 engine 调用，新增节点注册路由。

方法原型如下：

```go
func (n *node) addRoute(path string, handlers HandlersChain)
```

- path，是要注册的完整路由；
- handlers，处理这条路由使用的函数数组；

HandlersChain 定义如下，本身就是个数组：

```go
type HandlerFunc func(*Context)
type HandlersChain []HandlerFunc
```

要注意的是，这里的 Context 类不是 go 标准库 context 中的那个 Context，完全是 gin 自己使用的 Context 接口，但它也确实实现了这个接口，这是后话。

addRoute 方法本身比较复杂，而 且巨长，需要一步一步分析。

#### 判断当前节点是否为空

```go
if len(n.path) == 0 && len(n.children) == 0 { // todo gold 什么情况下，两者会有一个不满足？ 应该是这样的场景，比如刚加入根节点，还没有子节点时，会前 false 后 true；但是应该不会出现前 true 后 false 的情况
		n.insertChild(path, fullPath, handlers)
		n.nType = root
		return
	}
```

因为 addRoute 是新建一条路由时， node 类型中调用的第一个方法，那么最开始时 node 一定是一个根节点。如果它没有 path ，也没有任何 children，则说明相应类型的路由树还是一个空树，因此会直接调用 insertChild 方法，添加好路径后，将当前的 node 直接设置为 root 就退出了。



#### 循环对 path 进行处理

这部分是一个大循环，而且基本就是整个 addRoute 的主体了。它内部对 path 一段段切香肠，直到切割完毕。

##### 查找与当前 path 的公共部分

首先是查找与当前 path 的公共部分。

找到以后，会先比较这个公共部分是否和当前节点的 path 完全一样，如果一样，说明这段要处理的 path，将会是当前节点的一个新节点，否则就要拆分当前节点。



##### 拆分当前节点

上一步中，会找到 path 和当前节点的最长公共部分，这一部分判断并执行相关的拆分 当前节点工作 。

假如当前节点是 abcd，而 path 是 abcdef，则会基于 ef 新生成一个节点，而这个新节点将是 abcd 节点的一个新子节点；则这一步会直接跳过。

但是如果公共部分比较短，无法覆盖当前节点的 path，则说明要拆分当前节点。

假如当前节点是 abcdef，而新节点是 abcgh，公共部分只有 abc，需要拆分 abcdef 节点为 abc 和 def 节点，abc 节点占用原来节点的位置，而 def 节点将作为 abc 节点到目前为止的唯一个子节点。此外，原来处理 abcdef 的函数链会被 def 节点完整继承，而 abc 节点将没有函数处理，这是为下一步考虑的。



##### 基于 path 构建新节点

依然是基于上面找到的公共部分，判断 path 是不是还有剩余。如果没有剩余，只有两种可能性。

- 相对原来的当前节点信息，该新路由更短，是一条要插入的路由；这个情况会导致上一步“拆分当前节点”操作一定执行，原来的节点分裂，其原节点的函数链为空，此时，就可以使用传入进来的函数链来处理这条路由。而当前 path 的注册过程也就此结束，函数退出；
- 这条路由已经注册过，那么 path 其实和当前节点的 path 是相同的，那么该循环会直接 break，跳到方法的最后。由于已经注册过，那么上面一部“拆分当前节点”就不会执行，导致当前节点的 handlers 非空，初始化直接 panic。

上面是对于没有剩余的情况分析，而有剩余，说明需要添加一个新的节点出来，并且这个新节点会作为当前节点的一个新的子节点。



###### path 剩余部分可以被直接忽略

如果当前节点是个参数类型节点且只有一个子节点（<font color=red>难道参数类型节点会出现两个子节点的情况吗？</font>），而 path 剩余部分又是一个完全新的部分，会对这个子节点进行操作，因为参数类型节点压根没什么东西。



###### 查看可以插到哪个子节点下

需要对当前节点的子节点进行遍历，查看新节点插入哪个子节点中。走到这里，只有三种情况，要么当前节点不是参数类型节点，要么它是但它没有子节点或者有多个子节点，要不 path 剩余部分不是完整的 path。如果能找到符合条件的子节点，就用这个子节点作为新的下次循环的节点，进入下次循环。



###### 以上分支都没有触发

情况比较多，如果 path 是个普通字符串而当前节点不是 catchall 类型，会直接用 path 生成一个新节点，并且作为当前节点的一个新节点。

否则如果当前节点是模糊匹配的，太难 了！！！！



#### insertChild 方法

这个方法本身也是一个大号循环，接受传入的 path、全路径和句柄，对当前节点自身进行操作。

这个方法有三个入参：

1. path，string，表示要处理的字符串；
2. fullpath，string，表示到当前节点处，完整的路径；
3. handlers，HandlersChain，表示处理节点对应的所有函数链；

我们假设有以下 path 要处理，方便理解。

- /abc/def
- /abc/:para/def
- /abc/:para/
- /abc/:para
- /abc/:para:def/xyz
- /abc/:/xyz
- 

##### 查找是否有模糊匹配部分

首先是调用 findWildcard 函数，查找是否有模糊匹配部分，这个函数返回三个值：

1. wildcard 表示模糊部分；
2. i 表示模糊部分在 path 中开始的索引；
3. valid 表示 path 是不是有效的；

查找规则是从头开始遍历，直到遇到 "*"或者 ":"，然后从"\*"或者":" 的位置继续一次内循环，因为这时候会被判定为包含模糊匹配部分，需要找到完整的参数名，标记是下一个"/" ，此时表示本次查找结束。同时这个过程也是一个校验 path 有效性的过程，即参数中不能包含多于一个"*\*"或者":"。

如：

/abc/:para/def

外循环会在遍历到 : 后，开始内循环，直到获取到完整的 :para/，然后结束函数调用，返回到上层；

/abc/:para:def/xyz

由于:para后面跟着是 :def，会被识别为一个变量，但是却包含了两个 :，被认为是无效的。

/abc/:/xyz

只有 :，后面不包含作为变量名字的字符串，无效。



##### 没有模糊匹配部分

对于没有模糊匹配部分的情况，判断条件是 i 小于0，直接对当前节点设置 path、fullpath 和 handlers 即可，并退出 insertChild。

如上面的 /abc/def，它就不包含任何模糊匹配部分，可以直接跳出循环。

这里有个地方要注意，即 insertChild 这个方法是先判断 i 是否小于 0，然后才判断 valid 是否有效的；因为 findWildcard 这个函数中，当没有找到有模糊部分时，返回 i=-1 的同时，返回的 valid 是 false，所以不能先判断 valid 的有效性。



##### 处理 path 包含非法模糊的情况

主要是看 valid 是否有效或者是 wildcard 是否有效；前者如果无效的话，表示 path 的模糊匹配部分，包含多个 "\*" 或者 ":"，而后者则表示模糊匹配部分无法提取有效的参数。这两种情况都是无法处理的异常情况。



##### 判断是否以 : 开头

进入这里，说明 path 中包含以 “:” 开头的模糊匹配部分。

这时候，会提取出变量部分，基于该部分构建一个新节点 child，类型是 param，然后将其加入到当前节点的子节点中（<font color=red>什么地方对当前节点的 indices 做操作呢？</font>）。

要注意的是这里的新节点 child，它的 path 是":"开头的一个字符串。

比如上面的 /abc/:param，会直接新建一个 param 类型节点，而 path=":param"；

/abc/:param/，也会直接新建一个 param 类型节点，path=":param"；

/abc/:param/def，也会直接新建一个 param 类型节点，path=":param"；



###### 判断 path 是否还有剩余部分

在上面一步的基础上，这一步会判断是不是还有没处理完毕的部分。如果没有，说明指定的 handlers 就是给新建立的 child 节点用的，对其设置 handlers 后，退出 insertChild 方法即可；否则，会新建一个不包含有效信息（path和handlers）的普通类型节点 child，并把这个节点插入到原来 child 的最后，继续下一轮循环。

如 /abc/:param/def，在处理完 :param 后，会继续进入循环处理 def 部分。

而 /abc/:param/，在 处理完 :param 后，同样还要再处理最后的 / ；

而 /abc/:param，就不会进入这一步，在上面就处理完了。



##### 判断在*参数的情况下，是否还有其它末尾

走到这里，只有一种情况，就是 path 中包含的模糊部分，是"\*"开头的，gin 中这种情况表示节点为 catchAll 类型，而这种类型下，参数必须在整个 path 的最后，而不能再有其它部分了，如：/abc/def/\*ghi 是有效的，而/abc/def/\*ghi/jkl 是无效的。一旦出现后面这种情况，直接就是异常了。



##### 判断父节点是否以 "/" 结尾

如果是的话，也会异常；如父节点的 path 是"/post/" ，则是无效的，即 gin 中要求这种情况下，"/" 只能出现在 子节点中。

<font color=red>具体原因尚在研究</font>，可能是因为无法区分。



##### 判断 wildcard 是否以"/" 开头

这是和上面一致的逻辑，如果 wildcard 不以 "/" 开头，则同样会异常。



##### 新增 catchAll 节点

走到这里，会先将 path 中 wildcard 前面的部分设置为当前 node 的 path，因为当前 node 不管是在第一次执行循环中直接过来的，还是某次循环后过来的，它都还没有设置 path。

然后新增加一个 catchAll 类型节点，这个节点的 wildChild 同样为 true，但也同样没什么有效消息，也是匹配链条上的一个标记性节点；它被创建后，同样添加到当前节点的 child 中，将当前节点的 indices 设置为 "/"，表示它的子节点部分。然后当前节点指向这个子节点。

接着再创建一个 catchAll 类型节点，它真正包含了 wildcard 部分，而处理函数也全部交给该新节点。

如 /abc/def/:all 这样一个 path，它会创建两个节点：



------------------------------------

鉴于 node 添加路由这块太复杂，然后浪费时间又多，不容易出活儿，暂时告一段落。

下面是一些路由注册的层级结构，先暂时记录下来：

r.GET("/get",getting2)
r.GET("/get/:abc",getting2)
r.GET("/get/def/ghi",getting2)

  type= 1
  wildChild= false
  priority= 3
  indices= /
  fullPath= /get
  path= /get
  handler= 3
	  type= 0
	  wildChild= true
	  priority= 2
	  indices= d
	  fullPath= /get/:abc
	  path= /
	  handler= 0
		  type= 0
		  wildChild= false
		  priority= 1
		  indices= 
		  fullPath= /get/def/ghi
		  path= def/ghi
		  handler= 3
		  type= 2
		  wildChild= false
		  priority= 1
		  indices= 
		  fullPath= /get/:abc
		  path= :abc
		  handler= 3

---------------------------------------------

r.GET("/get/",getting2)
r.GET("/get/:abc",getting2)
r.GET("/get/def/ghi",getting2)

  type= 1
  wildChild= true
  priority= 3
  indices= d
  fullPath= /get/
  path= /get/
  handler= 3
	  type= 0
	  wildChild= false
	  priority= 1
	  indices= 
	  fullPath= /get/def/ghi
	  path= def/ghi
	  handler= 3
	  type= 2
	  wildChild= false
	  priority= 1
	  indices= 
	  fullPath= /get/:abc
	  path= :abc
	  handler= 3


-------------------------------------------
r.GET("/get",getting)
r.GET("/get/*abc",getting)

  type= 1
  wildChild= false
  priority= 2
  indices= /
  fullPath= /get
  path= /get
  handler= 3
	  type= 0
	  wildChild= false
	  priority= 1
	  indices= /
	  fullPath= /get/*abc
	  path= 
	  handler= 0
		  type= 3
		  wildChild= true
		  priority= 1
		  indices= 
		  fullPath= /get/*abc
		  path= 
		  handler= 0
			  type= 3
			  wildChild= false
			  priority= 1
			  indices= 
			  fullPath= /get/*abc
			  path= /*abc
			  handler= 3

----------------------------------------
r.GET("/get/\*abc",getting)
 type= 1
  wildChild= false
  priority= 1
  indices= /
  fullPath= /
  path= /get
  handler= 0
	  type= 3
	  wildChild= true
	  priority= 1
	  indices= 
	  fullPath= /get/*abc
	  path= 
	  handler= 0
		  type= 3
		  wildChild= false
		  priority= 1
		  indices= 
		  fullPath= /get/\*abc
		  path= /*abc
		  handler= 3



一些参考的帖子

https://juejin.cn/post/6844904070784745486

https://studygolang.com/articles/26914

https://www.infoq.cn/article/8tg1alapeyfcakf6zwgh

https://www.liwenzhou.com/posts/Go/gin-sourcecode/
