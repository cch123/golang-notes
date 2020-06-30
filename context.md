# context

先简单的了解一下几种 ctx:

- emptyCtx，所有 ctx 类型的根，用 context.TODO()，或 context.Background() 来生成。
- valueCtx，主要就是为了在 ctx 中嵌入上下文数据，一个简单的 k 和 v 结构，同一个 ctx 内只支持一对 kv，需要更多的 kv 的话，会形成一棵树形结构。
- cancelCtx，用来取消程序的执行树，一般用 WithCancel，WithTimeout，WithDeadline 返回的取消函数本质上都是对应了 cancelCtx。
- timerCtx，在 cancelCtx 上包了一层，支持基于时间的 cancel。

# basic usage

## 使用 emptyCtx 初始化 context

用来实现 context.TODO() 和 context.Background()，一般是所有 context 树的根。

```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)
```

todo 和 background 两者本质上只有名字区别，在按 string 输出的时候会有区别。

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
```

## 使用 valueCtx 嵌入数据

### valueCtx 使用

```go
package main

import (
	"context"
	"fmt"
)

type orderID int

func main() {
	var x = context.TODO()
	x = context.WithValue(x, orderID(1), "1234")
	x = context.WithValue(x, orderID(2), "2345")
	x = context.WithValue(x, orderID(3), "3456")
	fmt.Println(x.Value(orderID(2)))
}
```

这样的代码会生成下面这样的树：

```
           ┌────────────┐            
           │  emptyCtx  │            
           └────────────┘            
                  ▲                  
                  │                  
                  │                  
                  │    parent        
                  │                  
                  │                  
┌───────────────────────────────────┐
│      valueCtx{k: 1, v: 1234}      │
└───────────────────────────────────┘
                  ▲                  
                  │                  
                  │                  
                  │    parent        
                  │                  
                  │                  
                  │                  
┌───────────────────────────────────┐
│      valueCtx{k: 2, v: 2345}      │
└───────────────────────────────────┘
                  ▲                  
                  │                  
                  │    parent        
                  │                  
                  │                  
┌───────────────────────────────────┐
│      valueCtx{k: 3, v: 3456}      │
└───────────────────────────────────┘
```

简单改一下代码，让结果更像一棵树：

```go
package main

import (
	"context"
	"fmt"
)

type orderID int

func main() {
	var x = context.TODO()
	x = context.WithValue(x, orderID(1), "1234")
	x = context.WithValue(x, orderID(2), "2345")

	y := context.WithValue(x, orderID(3), "4567")
	x = context.WithValue(x, orderID(3), "3456")

	fmt.Println(x.Value(orderID(3)))
	fmt.Println(y.Value(orderID(3)))
}
```

就是像下面这样的图了：

```
                          ┌────────────┐                                       
                          │  emptyCtx  │                                       
                          └────────────┘                                       
                                 ▲                                             
                                 │                                             
                                 │                                             
                                 │    parent                                   
                                 │                                             
                                 │                                             
               ┌───────────────────────────────────┐                           
               │      valueCtx{k: 1, v: 1234}      │                           
               └───────────────────────────────────┘                           
                                 ▲                                             
                                 │                                             
                                 │                                             
                                 │    parent                                   
                                 │                                             
                                 │                                             
                                 │                                             
               ┌───────────────────────────────────┐                           
               │      valueCtx{k: 2, v: 2345}      │                           
               └───────────────────────────────────┘                           
                                 ▲                                             
                                 │                                             
                  ┌──────────────┴──────────────────────────┐                  
                  │                                         │                  
                  │                                         │                  
┌───────────────────────────────────┐     ┌───────────────────────────────────┐
│      valueCtx{k: 3, v: 3456}      │     │      valueCtx{k: 3, v: 4567}      │
└───────────────────────────────────┘     └───────────────────────────────────┘
             ┌───────┐                                   ┌───────┐             
             │   x   │                                   │   y   │             
             └───────┘                                   └───────┘
```

### valueCtx 分析

valueCtx 主要就是用来携带贯穿整个逻辑流程的数据的，在分布式系统中最常见的就是 trace_id，在一些业务系统中，一些业务数据项也需要贯穿整个请求的生命周期，如 order_id，payment_id 等。

WithValue 时即会生成 valueCtx：

```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

key 必须为非空，且可比较。

在查找值，即执行 Value 操作时，会先判断当前节点的 k 是不是等于用户的输入 k，如果相等，返回结果，如果不等，会依次向上从子节点向父节点，一直查找到整个 ctx 的根。没有找到返回 nil。是一个递归流程：

```go
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key) // 这里发生了递归，c.Context 就是 c.parent
}
```

通过分析，ctx 这么设计是为了能让代码每执行到一个点都可以根据当前情况嵌入新的上下文信息，但我们也可以看到，如果我们每次加一个新值都执行 WithValue 会导致 ctx 的树的层数过高，查找成本比较高 O(H)。

很多业务场景中，我们希望在请求入口存入值，在请求过程中随时取用。这时候我们可以将 value 作为一个 map 整体存入。

```go
context.WithValue(context.Background(), info,
	 map[string]string{"order_id" : "111", "payment_id" : "222"}
)
```

## 使用 cancelCtx 取消流程

### cancelCtx 使用

在没有 ctx 的时代，我们想要取消一个 go 出去的协程是很难的，因为 go func() 这个操作没有任何返回，所以我们也没有办法去跟踪这么一个新创建的 goroutine。

有了 cancelCtx 之后，我们想要取消就比较简单了：

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	jobChan := make(chan struct{})
	ctx, cancelFn := context.WithCancel(context.TODO())
	worker := func() {
		jobLoop:
		for {
			select {
			case <-jobChan:
				fmt.Println("do my job")
			case <-ctx.Done():
				fmt.Println("parent call me to quit")
				break jobLoop
			}
		}
	}

	// start worker
	go worker()

	// stop all worker
	cancelFn()
	time.Sleep(time.Second)
}
```

不过取消操作一定是需要 fork 出的 goroutine 本身要做一些配合动作的：

```go
select {
	case <-jobChan:
		// do my job
		fmt.Println("do my job")
	case <-ctx.Done():
		// parent want me to quit
		fmt.Println("parent call me to quit")
		break jobLoop
}
```

这里我们一边消费自己的 job channel，一边还需要监听 ctx.Done()，如果不监听 ctx.Done()，那显然也就不知道什么时候需要退出了。

### cancelCtx 分析

```go
// A cancelCtx can be canceled. When canceled, it also cancels any children
// that implement canceler.
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

使用 WithCancel 可以得到一个 cancelCtx:

```go
// WithCancel returns a copy of parent with a new Done channel. The returned
// context's Done channel is closed when the returned cancel function is called
// or when the parent context's Done channel is closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

```go
// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	if parent.Done() == nil {
		return // 说明父节点一定是 emptyCtx，或者用户自己实现的 context.Context
	}
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// cancel 发生的时候，err 字段一定会被赋值，这里说明父节点已经被赋值了
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{} // 把当前 cancelCtx 追加到父节点去
		}
		p.mu.Unlock()
	} else { // 如果用户把 context 包在了自己的 struct 内就会到这个分支。
		go func() {
			select {
			case <-parent.Done(): // 父节点取消，需要将这个取消指令同步给子节点
				child.cancel(false, parent.Err())
			case <-child.Done(): // 子节点取消的话，就不用等父节点了
			}
		}()
	}
}
```

parentCancelCtx 只识别 context 包内的三种类型，如果用户自己的类实现了 context.Context 接口，或者把  ctx 包在了自己的类型内，那这里始终返回的是 nil，false。

```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	for {
		switch c := parent.(type) {
		case *cancelCtx:
			return c, true
		case *timerCtx:
			return &c.cancelCtx, true
		case *valueCtx:
			parent = c.Context
		default:
			return nil, false
		}
	}
}
```

## 使用 timerCtx 超时取消

### timerCtx 使用

### timerCtx 分析

```go
// A timerCtx carries a timer and a deadline. It embeds a cancelCtx to
// implement Done and Err. It implements cancel by stopping its timer then
// delegating to cancelCtx.cancel.
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}
```

用 WithTimeout 和 WithDeadline 本质都是生成一个 timerCtx。

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// 如果父节点的 dealine 更靠前，那当然以父节点的为准，当前节点的 deadline 可以抛弃
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}

  // 向上冒泡，把当前节点的 cancel 函数关联到父 cancelCtx 节点上
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // 已经超时了，退出吧
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil { // 说明父节点到现在还没有取消呢
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded) // 这个方法到时间了之后会自动执行，当前的 goroutine 不会被阻塞
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

每次执行都会创建新的 timer。

## 树形结构

为什么设计成树形结构呢。因为对于 fork-join 的模型(Go 的原地 go func 就是这种模型)来说，程序代码的执行本来就是树形的。在进入、退出某个子节点的时候，既要加新的数据，又不能影响父节点的数据，所以这种链式结构可以完美地匹配。

TODO，这里应该有一张大图

取消某个父节点，又能够用树的遍历算法将该取消指令传导到整棵树去。

TODO，这里应该有一张大图

## traps

### cancelCtx 的性能问题

TODO

### ctx 的打印 panic

TODO

# 总结

ctx 的结构显然是根据代码的执行模型来设计的，虽然设计得比较巧妙，但因为将取消和上下文携带功能混合在一起，在一些情况下还是会给我们埋些比较隐蔽的坑。使用时需要多多注意。
