# Semaphore

## 数据结构

```go
// Go 语言中暴露的 semaphore 实现
// 具体的用法是提供 sleep 和 wakeup 原语
// 以使其能够在其它同步原语中的竞争情况下使用
// 因此这里的 semaphore 和 Linux 中的 futex 目标是一致的
// 只不过语义上更简单一些
//
// 也就是说，不要认为这些是信号量
// 把这里的东西看作 sleep 和 wakeup 实现的一种方式
// 每一个 sleep 都会和一个 wakeup 配对
// 即使在发生 race 时，wakeup 在 sleep 之前时也是如此
//
// See Mullender and Cox, ``Semaphores in Plan 9,''
// http://swtch.com/semaphore.pdf

// 为 sync.Mutex 准备的异步信号量

// semaRoot 持有一棵 地址各不相同的 sudog(s.elem) 的平衡树
// 每一个 sudog 都反过来指向(通过 s.waitlink)一个在同一个地址上等待的其它 sudog 们
// 同一地址的 sudog 的内部列表上的操作时间复杂度都是 O(1)。顶层 semaRoot 列表的扫描
// 的时间复杂度是 O(log n)，n 是被哈希到同一个 semaRoot 的不同地址的总数，每一个地址上都会有一些 goroutine 被阻塞。
// 访问 golang.org/issue/17953 来查看一个在引入二级列表之前性能较差的程序样例，test/locklinear.go
// 中有一个复现这个样例的测试
type semaRoot struct {
    lock  mutex
    treap *sudog // root of balanced tree of unique waiters.
    nwait uint32 // Number of waiters. Read w/o the lock.
}

// Prime to not correlate with any user patterns.
const semTabSize = 251

var semtable [semTabSize]struct {
    root semaRoot
    pad  [sys.CacheLineSize - unsafe.Sizeof(semaRoot{})]byte
}

func semroot(addr *uint32) *semaRoot {
    return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}
```

```
┌─────┬─────┬─────┬─────┬─────┬────────────────────────┬─────┐                 
│  0  │  1  │  2  │  3  │  4  │         .....          │ 250 │                 
└─────┴─────┴─────┴─────┴─────┴────────────────────────┴─────┘                 
   │                                                      │                    
   │                                                      │                    
   └──┐                                                   └─┐                  
      │                                                     │                  
      │                                                     │                  
      ▼                                                     ▼                  
 ┌─────────┐                                           ┌─────────┐             
 │ struct  │                                           │ struct  │             
 ├─────────┴─────────┐                                 ├─────────┴─────────┐   
 │   root semaRoot   │──┐                              │   root semaRoot   │──┐
 ├───────────────────┤  │                              ├───────────────────┤  │
 │        pad        │  │                              │        pad        │  │
 └───────────────────┘  │                              └───────────────────┘  │
                        │                                                     │
       ┌────────────────┘                                    ┌────────────────┘
       │                                                     │                 
       │                                                     │                 
       ▼                                                     ▼                 
 ┌──────────┐                                          ┌──────────┐            
 │ semaRoot │                                          │ semaRoot │            
 ├──────────┴────────┐                                 ├──────────┴────────┐   
 │    lock mutex     │                                 │    lock mutex     │   
 ├───────────────────┤                                 ├───────────────────┤   
 │   treap *sudog    │                                 │   treap *sudog    │   
 ├───────────────────┤                                 ├───────────────────┤   
 │   nwait uint32    │                                 │   nwait uint32    │   
 └───────────────────┘                                 └───────────────────┘   
```

treap 结构:

```
                                 ┌──────────┐                                    
                            ┌─┬─▶│  sudog   │                                    
                            │ │  ├──────────┴────────────┐                       
      ┌─────────────────────┼─┼──│      prev *sudog      │                       
      │                     │ │  ├───────────────────────┤                       
      │                     │ │  │      next *sudog      │────┐                  
      │                     │ │  ├───────────────────────┤    │                  
      │                     │ │  │     parent *sudog     │    │                  
      │                     │ │  ├───────────────────────┤    │                  
      │                     │ │  │  elem unsafe.Pointer  │    │                  
      │                     │ │  ├───────────────────────┤    │                  
      │                     │ │  │     ticket uint32     │    │                  
      │                     │ │  └───────────────────────┘    │                  
      │                     │ │                               │                  
      │                     │ │                               │                  
      │                     │ │                               │                  
      │                     │ │                               │                  
      │                     │ │                               │                  
      │                     │ │                               │                  
      ▼                     │ │                               ▼                  
┌──────────┐                │ │                         ┌──────────┐             
│  sudog   │                │ │                         │  sudog   │             
├──────────┴────────────┐   │ │                         ├──────────┴────────────┐
│      prev *sudog      │   │ │                         │      prev *sudog      │
├───────────────────────┤   │ │                         ├───────────────────────┤
│      next *sudog      │   │ │                         │      next *sudog      │
├───────────────────────┤   │ │                         ├───────────────────────┤
│     parent *sudog     │───┘ └─────────────────────────│     parent *sudog     │
├───────────────────────┤                               ├───────────────────────┤
│  elem unsafe.Pointer  │                               │  elem unsafe.Pointer  │
├───────────────────────┤                               ├───────────────────────┤
│     ticket uint32     │                               │     ticket uint32     │
└───────────────────────┘                               └───────────────────────┘
```

在这个 treap 结构里，从 elem 的视角(其实就是 lock 的 addr)来看，这个结构是个二叉搜索树。从 ticket 的角度来看，整个结构就是一个小顶堆。

所以才叫树堆(treap)。

相同 addr，即对同一个 mutex 上锁的 g，会阻塞在同一个地址上。这些阻塞在同一个地址上的 goroutine 会被打包成 sudog，组成一个链表。用 sudog 的 waitlink 相连:

```
┌──────────┐                         ┌──────────┐                          ┌──────────┐             
│  sudog   │                  ┌─────▶│  sudog   │                   ┌─────▶│  sudog   │             
├──────────┴────────────┐     │      ├──────────┴────────────┐      │      ├──────────┴────────────┐
│    waitlink *sudog    │─────┘      │    waitlink *sudog    │──────┘      │    waitlink *sudog    │
├───────────────────────┤            ├───────────────────────┤             ├───────────────────────┤
│    waittail *sudog    │            │    waittail *sudog    │             │    waittail *sudog    │
└───────────────────────┘            └───────────────────────┘             └───────────────────────┘
```

中间的元素的 waittail 都会指向最后一个元素:

```
┌──────────┐                                                                                           
│  sudog   │                                                                                           
├──────────┴────────────┐                                                                              
│    waitlink *sudog    │                                                                              
├───────────────────────┤                                                                              
│    waittail *sudog    │───────────────────────────────────────────────────────────┐                  
└───────────────────────┘                                                           │                  
                                  ┌──────────┐                                      │                  
                                  │  sudog   │                                      │                  
                                  ├──────────┴────────────┐                         │                  
                                  │    waitlink *sudog    │                         │                  
                                  ├───────────────────────┤                         │                  
                                  │    waittail *sudog    │─────────────────────────┤                  
                                  └───────────────────────┘                         ▼                  
                                                                              ┌──────────┐             
                                                                              │  sudog   │             
                                                                              ├──────────┴────────────┐
                                                                              │    waitlink *sudog    │
                                                                              ├───────────────────────┤
                                                                              │    waittail *sudog    │
                                                                              └───────────────────────┘
```

## 对外封装

在 sema.go 里实现的内容，用 `go:linkname` 导出给 sync、poll 库来使用，也是在链接期做了些手脚:

```go
//go:linkname sync_runtime_Semacquire sync.runtime_Semacquire
func sync_runtime_Semacquire(addr *uint32) {
    semacquire1(addr, false, semaBlockProfile)
}

//go:linkname poll_runtime_Semacquire internal/poll.runtime_Semacquire
func poll_runtime_Semacquire(addr *uint32) {
    semacquire1(addr, false, semaBlockProfile)
}

//go:linkname sync_runtime_Semrelease sync.runtime_Semrelease
func sync_runtime_Semrelease(addr *uint32, handoff bool) {
    semrelease1(addr, handoff)
}

//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool) {
    semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile)
}

//go:linkname poll_runtime_Semrelease internal/poll.runtime_Semrelease
func poll_runtime_Semrelease(addr *uint32) {
    semrelease(addr)
}
```

## 实现

sem 本身支持 acquire 和 release，其实就是 OS 里常说的 P 操作和 V 操作。

### 公共部分

```go
func cansemacquire(addr *uint32) bool {
    for {
        v := atomic.Load(addr)
        if v == 0 {
            return false
        }
        if atomic.Cas(addr, v, v-1) {
            return true
        }
    }
}
```

### acquire 过程

```go
type semaProfileFlags int

const (
    semaBlockProfile semaProfileFlags = 1 << iota
    semaMutexProfile
)

// Called from runtime.
func semacquire(addr *uint32) {
    semacquire1(addr, false, 0)
}

func semacquire1(addr *uint32, lifo bool, profile semaProfileFlags) {
    gp := getg()
    if gp != gp.m.curg {
        throw("semacquire not on the G stack")
    }

    // 低成本的情况
    if cansemacquire(addr) {
        return
    }

    // 高成本的情况:
    //    增加 waiter count 的值
    //    再尝试调用一次 cansemacquire，成功了就直接返回
    //    没成功就把自己作为一个 waiter 入队
    //    sleep
    //    (之后 waiter 的 descriptor 被 signaler 用 dequeue 踢出)
    s := acquireSudog()
    root := semroot(addr)
    t0 := int64(0)
    s.releasetime = 0
    s.acquiretime = 0
    s.ticket = 0

    for {
        lock(&root.lock)
        // 给 nwait 加一，这样后来的就不会在 semrelease 中进低成本的路径了
        atomic.Xadd(&root.nwait, 1)
        // 检查 cansemacquire 避免错过了唤醒
        if cansemacquire(addr) {
            atomic.Xadd(&root.nwait, -1)
            unlock(&root.lock)
            break
        }
        // 在 cansemacquire 之后的 semrelease 都可以知道我们正在等待
        // (上面设置了 nwait)，所以会直接进入 sleep
        // 注: 这里说的 sleep 其实就是 goparkunlock
        root.queue(addr, s, lifo)
        goparkunlock(&root.lock, "semacquire", traceEvGoBlockSync, 4)
        if s.ticket != 0 || cansemacquire(addr) {
            break
        }
    }
    if s.releasetime > 0 {
        blockevent(s.releasetime-t0, 3)
    }
    releaseSudog(s)
}
```

### release 过程

```go
func semrelease(addr *uint32) {
    semrelease1(addr, false)
}

func semrelease1(addr *uint32, handoff bool) {
    root := semroot(addr)
    atomic.Xadd(addr, 1)

    // 低成本情况: 没有 waiter?
    // 这个 atomic 的检查必须发生在 xadd 之前，以避免错误唤醒
    // (具体参见 semacquire 中的循环)
    if atomic.Load(&root.nwait) == 0 {
        return
    }

    // 高成本情况: 搜索 waiter 并唤醒它
    lock(&root.lock)
    if atomic.Load(&root.nwait) == 0 {
        // count 值已经被另一个 goroutine 消费了
        // 所以我们不需要唤醒其它 goroutine 了
        unlock(&root.lock)
        return
    }
    s, t0 := root.dequeue(addr)
    if s != nil {
        atomic.Xadd(&root.nwait, -1)
    }
    unlock(&root.lock)
    if s != nil { // 可能会很慢，所以先解锁
        acquiretime := s.acquiretime
        if acquiretime != 0 {
            mutexevent(t0-acquiretime, 3)
        }
        if s.ticket != 0 {
            throw("corrupted semaphore ticket")
        }
        if handoff && cansemacquire(addr) {
            s.ticket = 1
        }
        readyWithTime(s, 5)
    }
}

func readyWithTime(s *sudog, traceskip int) {
    if s.releasetime != 0 {
        s.releasetime = cputicks()
    }
    goready(s.g, traceskip)
}
```

### treap 结构

sudog 按照地址 hash 到 251 个 bucket 中的其中一个，每一个 bucket 都是一棵 treap。而相同 addr 上的 sudog 会形成一个链表。

为啥同一个地址的 sudog 不需要展开放在 treap 中呢？显然，sudog 唤醒的时候，block 在同一个 addr 上的 goroutine，说明都是加的同一把锁，这些 goroutine 被唤醒肯定是一起被唤醒的，相同地址的 g 并不需要查找才能找到，只要决定是先进队列的被唤醒(fifo)还是后进队列的被唤醒(lifo)就可以了。

```go
// queue 函数会把 s 添加到 semaRoot 上阻塞的 goroutine 们中
// 实际上就是把 s 添加到其地址对应的 treap 上
func (root *semaRoot) queue(addr *uint32, s *sudog, lifo bool) {
    s.g = getg()
    s.elem = unsafe.Pointer(addr)
    s.next = nil
    s.prev = nil

    var last *sudog
    pt := &root.treap
    for t := *pt; t != nil; t = *pt {
        if t.elem == unsafe.Pointer(addr) {
            // Already have addr in list.
            if lifo {
                // treap 中在 t 的位置用 s 覆盖掉 t
                *pt = s
                s.ticket = t.ticket
                s.acquiretime = t.acquiretime
                s.parent = t.parent
                s.prev = t.prev
                s.next = t.next
                if s.prev != nil {
                    s.prev.parent = s
                }
                if s.next != nil {
                    s.next.parent = s
                }
                // 把 t 放在 s 的 wait list 的第一个位置
                s.waitlink = t
                s.waittail = t.waittail
                if s.waittail == nil {
                    s.waittail = t
                }
                t.parent = nil
                t.prev = nil
                t.next = nil
                t.waittail = nil
            } else {
                // 把 s 添加到 t 的等待列表的末尾
                if t.waittail == nil {
                    t.waitlink = s
                } else {
                    t.waittail.waitlink = s
                }
                t.waittail = s
                s.waitlink = nil
            }
            return
        }
        last = t
        if uintptr(unsafe.Pointer(addr)) < uintptr(t.elem) {
            pt = &t.prev
        } else {
            pt = &t.next
        }
    }

    // 把 s 作为树的新的叶子插入进去
    // 平衡树使用 ticket 作为堆的权重值，这个 ticket 是随机生成的
    // 也就是说，这个结构以元素地址来看的话，是一个二叉搜索树
    // 同时用 ticket 值使其同时又是一个小顶堆，满足
    // s.ticket <= both s.prev.ticket and s.next.ticket.
    // https://en.wikipedia.org/wiki/Treap
    // http://faculty.washington.edu/aragon/pubs/rst89.pdf
    //
    // s.ticket 会在一些地方和 0 相比，因此只设置最低位的 bit
    // 这样不会明显地影响 treap 的质量?
    s.ticket = fastrand() | 1
    s.parent = last
    *pt = s

    // 按照 ticket 来进行旋转，以满足 treap 的性质
    for s.parent != nil && s.parent.ticket > s.ticket {
        if s.parent.prev == s {
            root.rotateRight(s.parent)
        } else {
            if s.parent.next != s {
                panic("semaRoot queue")
            }
            root.rotateLeft(s.parent)
        }
    }
}

// dequeue 会搜索到阻塞在 addr 地址的 semaRoot 中的第一个 goroutine
// 如果这个 sudog 需要进行 profile，dequeue 会返回它被唤醒的时间(now)，否则的话 now 为 0
func (root *semaRoot) dequeue(addr *uint32) (found *sudog, now int64) {
    ps := &root.treap
    s := *ps
    for ; s != nil; s = *ps {
        if s.elem == unsafe.Pointer(addr) {
            goto Found
        }
        if uintptr(unsafe.Pointer(addr)) < uintptr(s.elem) {
            ps = &s.prev
        } else {
            ps = &s.next
        }
    }
    return nil, 0

Found:
    now = int64(0)
    if s.acquiretime != 0 {
        now = cputicks()
    }
    if t := s.waitlink; t != nil {
        // 替换掉同样在 addr 上等待的 t。
        *ps = t
        t.ticket = s.ticket
        t.parent = s.parent
        t.prev = s.prev
        if t.prev != nil {
            t.prev.parent = t
        }
        t.next = s.next
        if t.next != nil {
            t.next.parent = t
        }
        if t.waitlink != nil {
            t.waittail = s.waittail
        } else {
            t.waittail = nil
        }
        t.acquiretime = now
        s.waitlink = nil
        s.waittail = nil
    } else {
        // 向下旋转 s 到叶节点，以进行删除，同时要考虑优先级
        for s.next != nil || s.prev != nil {
            if s.next == nil || s.prev != nil && s.prev.ticket < s.next.ticket {
                root.rotateRight(s)
            } else {
                root.rotateLeft(s)
            }
        }
        // Remove s, now a leaf.
        // 删除 s，现在是叶子节点了
        if s.parent != nil {
            if s.parent.prev == s {
                s.parent.prev = nil
            } else {
                s.parent.next = nil
            }
        } else {
            root.treap = nil
        }
    }
    s.parent = nil
    s.elem = nil
    s.next = nil
    s.prev = nil
    s.ticket = 0
    return s, now
}
```

### treap 左旋，右旋

```go
// rotateLeft rotates the tree rooted at node x.
// turning (x a (y b c)) into (y (x a b) c).
func (root *semaRoot) rotateLeft(x *sudog) {
    // p -> (x a (y b c))
    p := x.parent
    a, y := x.prev, x.next
    b, c := y.prev, y.next

    y.prev = x
    x.parent = y
    y.next = c
    if c != nil {
        c.parent = y
    }
    x.prev = a
    if a != nil {
        a.parent = x
    }
    x.next = b
    if b != nil {
        b.parent = x
    }

    y.parent = p
    if p == nil {
        root.treap = y
    } else if p.prev == x {
        p.prev = y
    } else {
        if p.next != x {
            throw("semaRoot rotateLeft")
        }
        p.next = y
    }
}

// rotateRight rotates the tree rooted at node y.
// turning (y (x a b) c) into (x a (y b c)).
func (root *semaRoot) rotateRight(y *sudog) {
    // p -> (y (x a b) c)
    p := y.parent
    x, c := y.prev, y.next
    a, b := x.prev, x.next

    x.prev = a
    if a != nil {
        a.parent = x
    }
    x.next = y
    y.parent = x
    y.prev = b
    if b != nil {
        b.parent = y
    }
    y.next = c
    if c != nil {
        c.parent = y
    }

    x.parent = p
    if p == nil {
        root.treap = x
    } else if p.prev == y {
        p.prev = x
    } else {
        if p.next != y {
            throw("semaRoot rotateRight")
        }
        p.next = x
    }
}

```

### notifyList 结构

notifyList 结构提供给 sync.Cond 使用，用来做条件变量进行通知和唤醒，比较简单。

```go
// notifyList 基于 ticket 进行通知，以实现 sync.Cond 的功能
type notifyList struct {
    // wait 是下一个 waiter 应该持有的 ticket 号。该字段会在 notifyList.lock
    // 之外被原子地递增。
    wait uint32

    // notify 字段表示下一个应该唤醒的 waiter 的 ticket 号。该值可以在不持有 notifyList.lock
    // 时读取，但写入时，必须持有 lock
    //
    // Both wait & notify can wrap around, and such cases will be correctly
    // handled as long as their "unwrapped" difference is bounded by 2^31.
    // For this not to be the case, we'd need to have 2^31+ goroutines
    // blocked on the same condvar, which is currently not possible.
    notify uint32

    // parked 的 waiter，串在 sudog 链表上
    lock mutex
    head *sudog
    tail *sudog
}

// less checks if a < b, considering a & b running counts that may overflow the
// 32-bit range, and that their "unwrapped" difference is always less than 2^31.
func less(a, b uint32) bool {
    return int32(a-b) < 0
}

// 这个破玩艺儿其实就只是原子 add l.wait
// 并且要求用户必须把传回去的值，作为 ticket，再传给 notifyListWait，作为 t 参数
// 这样才能把 caller 挂在 l 列表上等待 notification。
//go:linkname notifyListAdd sync.runtime_notifyListAdd
func notifyListAdd(l *notifyList) uint32 {
    // This may be called concurrently, for example, when called from
    // sync.Cond.Wait while holding a RWMutex in read mode.
    return atomic.Xadd(&l.wait, 1) - 1
}

// notifyListWait 等待通知。如果之前已经有人发送过了通知，那么会立刻返回。
// 否则，则会阻塞等待下一个通知
//go:linkname notifyListWait sync.runtime_notifyListWait
func notifyListWait(l *notifyList, t uint32) {
    lock(&l.lock)

    // 如果这个 ticket 号已经被唤醒过了，马上 return
    if less(t, l.notify) {
        unlock(&l.lock)
        return
    }

    // 将自己入队
    s := acquireSudog()
    s.g = getg()
    s.ticket = t
    s.releasetime = 0
    t0 := int64(0)

    if l.tail == nil {
        l.head = s
    } else {
        l.tail.next = s
    }
    l.tail = s
    goparkunlock(&l.lock, "semacquire", traceEvGoBlockCond, 3)
    if t0 != 0 {
        blockevent(s.releasetime-t0, 2)
    }
    releaseSudog(s)
}

// 这个函数会唤醒在 list 上等待的所有 sudog
//go:linkname notifyListNotifyAll sync.runtime_notifyListNotifyAll
func notifyListNotifyAll(l *notifyList) {
    // Fast-path: 如果从上一次唤醒之后，没有新的 waiter 的话
    // 就不需要获取锁了
    if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
        return
    }

    // Pull the list out into a local variable, waiters will be readied
    // outside the lock.
    lock(&l.lock)
    s := l.head
    l.head = nil
    l.tail = nil

    // 直接把 l.wait 的值赋给 l.notify，因为 l 里的 sudog 全都已经被唤醒了。
    // 即使没被唤醒，之后他们也会在将自己添加到 list 之前，注意到自己已经被通知过了(从而立刻返回)。
    atomic.Store(&l.notify, atomic.Load(&l.wait))
    unlock(&l.lock)

    // 遍历列表，ready 所有的 waiter
    for s != nil {
        next := s.next
        s.next = nil
        readyWithTime(s, 4)
        s = next
    }
}

// 这个函数唤醒 list 上的一个 sudog
//go:linkname notifyListNotifyOne sync.runtime_notifyListNotifyOne
func notifyListNotifyOne(l *notifyList) {
    // Fast-path: 如果上次通知发生后没有任何新的 waiter 的话
    // 完全不需要获取锁
    if atomic.Load(&l.wait) == atomic.Load(&l.notify) {
        return
    }

    lock(&l.lock)

    // Re-check under the lock if we need to do anything.
    t := l.notify
    if t == atomic.Load(&l.wait) {
        unlock(&l.lock)
        return
    }

    // 更新下一个需要唤醒的 ticket 号
    atomic.Store(&l.notify, t+1)

    // 尝试找到需要唤醒的 g
    // 如果 g 还没有来得及入队，那我们是找不到它的
    // 不过这个 g 在 park 之前发现 notify 号变了，会放弃 park 自己
    //
    // 这个 scan 过程看起来是线性时间的，但基本上都停止得很快。
    // Because g's queue separately from taking numbers,
    // 可能列表中会有少数重排的情况，但我们期望的是我们寻找的 g 在较前面的位置就行。
    // 只有在竞争失败的情况下，g 的前面才会有其它的 g，所以迭代过程不会太持久。
    // 这种逻辑甚至适用于 g missing 的情况:
    // 没有来得久 sleep，并且已经与我们在 list 上找到的其它 g 的竞争中失败了?
    for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
        if s.ticket == t {
            n := s.next
            if p != nil {
                p.next = n
            } else {
                l.head = n
            }
            if n == nil {
                l.tail = p
            }
            unlock(&l.lock)
            s.next = nil
            readyWithTime(s, 4)
            return
        }
    }
    unlock(&l.lock)
}

//go:linkname notifyListCheck sync.runtime_notifyListCheck
func notifyListCheck(sz uintptr) {
    if sz != unsafe.Sizeof(notifyList{}) {
        print("runtime: bad notifyList size - sync=", sz, " runtime=", unsafe.Sizeof(notifyList{}), "\n")
        throw("bad notifyList size")
    }
}

//go:linkname sync_nanotime sync.runtime_nanotime
func sync_nanotime() int64 {
    return nanotime()
}
```

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
