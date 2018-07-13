# netpoll

socket，connect，listen，getsocketopt 都有一个全局函数变量来表示。

hook_unix.go :

```go
    // Placeholders for socket system calls.
    socketFunc        func(int, int, int) (int, error)  = syscall.Socket
    connectFunc       func(int, syscall.Sockaddr) error = syscall.Connect
    listenFunc        func(int, int) error              = syscall.Listen
    getsockoptIntFunc func(int, int, int) (int, error)  = syscall.GetsockoptInt
```

这些 hook 主要是为了能够写测试，在测试代码中，socketFunc，connectFunc ... 都会被替换成测试专用函数，main_unix_test.go:

```go
func installTestHooks() {
    socketFunc = sw.Socket
    poll.CloseFunc = sw.Close
    connectFunc = sw.Connect
    listenFunc = sw.Listen
    poll.AcceptFunc = sw.Accept
    getsockoptIntFunc = sw.GetsockoptInt

    for _, fn := range extraTestHookInstallers {
        fn()
    }
}
```

用这种全局函数 hook，或者叫注册表的方式，可以实现类似于面向对象中的 interface 功能。不过因为不同平台提供的网络编程函数差别有些大，所以这里这些全局网络函数也就只是用来方便测试。

## 数据结构

```go
// 整体的 network poller(平台无关部分)
// 具体的实现需要定义下面这些函数:
// func netpollinit()            // 初始化 poller
// func netpollopen(fd uintptr, pd *pollDesc) int32    // 用来装备边缘触发的通知
// 并且将 fd 和 pd 做好关联
// 具体实现中，必须调用 下面的函数来表示 pd 已经 ready
// func netpollready(gpp **g, pd *pollDesc, mode int32)

// pollDesc 包含两个二元信号量, rg 和 wg, 分别用来 park 读、写的 goroutine
// 信号量可以是下面几种状态:
// pdReady - io readiness notification is pending;
//           a goroutine consumes the notification by changing the state to nil.
// pdWait - a goroutine prepares to park on the semaphore, but not yet parked;
//          the goroutine commits to park by changing the state to G pointer,
//          or, alternatively, concurrent io notification changes the state to READY,
//          or, alternatively, concurrent timeout/close changes the state to nil.
// G pointer - the goroutine is blocked on the semaphore;
//             io notification or timeout/close changes the state to READY or nil respectively
//             and unparks the goroutine.
// nil - nothing of the above.

// wg 和 rg 这两个字段用法实在有点特殊
// 既会被用来当作状态值，又会被用来存储被 park 的 g 列表
// 如果没有 g 在等，也没事件，那就是 0
// 官方这里说的 nil 其实就是 0

const (
    pdReady uintptr = 1
    pdWait  uintptr = 2
)

const pollBlockSize = 4 * 1024

// Network file descriptor.
type netFD struct {
    pfd poll.FD

    // 下面这些元素在 Close 之前都是不可变的
    family      int
    sotype      int
    isConnected bool
    net         string
    laddr       Addr
    raddr       Addr
}

// FD 是对 file descriptor 的一个包装，内部的 Sysfd 就是 linux 下的
// file descriptor。net 和 os 包中使用这个类型来代表一个网络连接或者一个 OS 文件
type FD struct {
    // 对 sysfd 加锁，以使 Read 和 Write 方法串行执行
    fdmu fdMutex

    // 操作系统的 file descriptor。在关闭之前是不可变的
    Sysfd int

    // I/O poller.
    pd pollDesc

    // Writev cache.
    iovecs *[]syscall.Iovec

    // Semaphore signaled when file is closed.
    csema uint32

    // 不可变。表示当前这个 fd 是否是一个流，或者是一个基于包的 fd
    // 用来区分是 TCP 还是 UDP
    IsStream bool

    // Whether a zero byte read indicates EOF. This is false for a
    // message based socket connection.
    ZeroReadIsEOF bool

    // 这个 fd 表示的是不是一个文件，如果 false 的话就是一个 network socket
    isFile bool

    // 这个文件是否被设置为了 blocking 模式
    isBlocking bool
}

// Network poller descriptor.
//
// No heap pointers.
//
//go:notinheap
type pollDesc struct {
    link *pollDesc // in pollcache, protected by pollcache.lock

    // The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
    // This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
    // pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO readiness notification)
    // proceed w/o taking the lock. So closing, rg, rd, wg and wd are manipulated
    // in a lock-free way by all operations.
    // NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
    // that will blow up when GC starts moving objects.
    lock    mutex // protects the following fields
    fd      uintptr
    closing bool
    seq     uintptr // protects from stale timers and ready notifications
    rg      uintptr // pdReady, pdWait, G waiting for read or nil
    rt      timer   // read deadline timer (set if rt.f != nil)
    rd      int64   // read deadline
    wg      uintptr // pdReady, pdWait, G waiting for write or nil
    wt      timer   // write deadline timer
    wd      int64   // write deadline
    user    uint32  // user settable cookie
}
```

## 流程

### netFD 初始化

```go
func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
    ret := &netFD{
        pfd: poll.FD{
            Sysfd:         sysfd,
            IsStream:      sotype == syscall.SOCK_STREAM,
            ZeroReadIsEOF: sotype != syscall.SOCK_DGRAM && sotype != syscall.SOCK_RAW,
        },
        family: family,
        sotype: sotype,
        net:    net,
    }
    return ret, nil
}
```

```go
func (fd *netFD) init() error {
    return fd.pfd.Init(fd.net, true)
}
```

### listen

```mermaid
graph LR

ListenTCP --> listenTCP
listenTCP --> internetSocket
internetSocket --> socket
socket --> listenStream
```

```go
func ListenTCP(network string, laddr *TCPAddr) (*TCPListener, error) {
    switch network {
    case "tcp", "tcp4", "tcp6":
    default:
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: UnknownNetworkError(network)}
    }
    if laddr == nil {
        laddr = &TCPAddr{}
    }
    ln, err := listenTCP(context.Background(), network, laddr)
    if err != nil {
        return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: err}
    }
    return ln, nil
}
```

```go
func listenTCP(ctx context.Context, network string, laddr *TCPAddr) (*TCPListener, error) {
    fd, err := internetSocket(ctx, network, laddr, nil, syscall.SOCK_STREAM, 0, "listen")
    if err != nil {
        return nil, err
    }
    return &TCPListener{fd}, nil
}
```

```go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
    if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
        raddr = raddr.toLocal(net)
    }
    family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
    return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}
```

```go
// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr) (fd *netFD, err error) {
    s, err := sysSocket(family, sotype, proto)
    if err != nil {
        return nil, err
    }
    if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }
    if fd, err = newFD(s, family, sotype, net); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }

    if laddr != nil && raddr == nil {
        switch sotype {
        case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
            if err := fd.listenStream(laddr, listenerBacklog); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        case syscall.SOCK_DGRAM:
            if err := fd.listenDatagram(laddr); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        }
    }
    if err := fd.dial(ctx, laddr, raddr); err != nil {
        fd.Close()
        return nil, err
    }
    return fd, nil
}
```

```go
func (fd *netFD) listenStream(laddr sockaddr, backlog int) error {
    if err := setDefaultListenerSockopts(fd.pfd.Sysfd); err != nil {
        return err
    }
    if lsa, err := laddr.sockaddr(fd.family); err != nil {
        return err
    } else if lsa != nil {
        if err := syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
            return os.NewSyscallError("bind", err)
        }
    }
    if err := listenFunc(fd.pfd.Sysfd, backlog); err != nil {
        return os.NewSyscallError("listen", err)
    }
    if err := fd.init(); err != nil {
        return err
    }
    lsa, _ := syscall.Getsockname(fd.pfd.Sysfd)
    fd.setAddr(fd.addrFunc()(lsa), nil)
    return nil
}
```

Go 的 listenTCP 一个函数就把 c 网络编程中 `socket()`，`bind()`，`listen()` 三步都完成了。大大减小了用户的心智负担。

这里有一点需要注意，listenStream 虽然提供了 backlog 的参数，但用户层是没有办法通过 Go 的代码来修改 listen 的 backlog 的。

```go
func maxListenerBacklog() int {
    fd, err := open("/proc/sys/net/core/somaxconn")
    if err != nil {
        return syscall.SOMAXCONN
    }
    defer fd.close()
    l, ok := fd.readLine()
    if !ok {
        return syscall.SOMAXCONN
    }
    f := getFields(l)
    n, _, ok := dtoi(f[0])
    if n == 0 || !ok {
        return syscall.SOMAXCONN
    }
    // Linux stores the backlog in a uint16.
    // Truncate number to avoid wrapping.
    // See issue 5030.
    if n > 1<<16-1 {
        n = 1<<16 - 1
    }
    return n
}
```

如上，在 linux 中，如果配置了 /proc/sys/net/core/somaxconn，那么就用这个值，如果没有配置，那么就使用 syscall 中的 SOMAXCONN:

```go
const (
    SOMAXCONN = 0x80 // 128
)
```

社区里有很多人吐槽，希望能有手段能修改这个值，不过看起来官方并不打算支持。所以现阶段只能通过修改 /proc/sys/net/core/somaxconn 来修改 listen 的 backlog。

### poll 流程

```go
type conn struct {
    fd *netFD
}

TODO，conn 是什么时候赋值给 Conn 类型的？


#### Read 流程


func (c *conn) ok() bool { return c != nil && c.fd != nil }

// Implementation of the Conn interface.

// Read implements the Conn Read method.
func (c *conn) Read(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Read(b)
    if err != nil && err != io.EOF {
        err = &OpError{Op: "read", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}

```

```go
func (fd *netFD) Read(buf []byte) (int, error) {
    n, err := fd.pfd.Read(buf)
    runtime.KeepAlive(fd)
    return n, wrapSyscallError("wsarecv", err)
}
```

```go
// Read implements io.Reader.
func (fd *FD) Read(p []byte) (int, error) {
    if err := fd.readLock(); err != nil {
        return 0, err
    }
    defer fd.readUnlock()
    if len(p) == 0 {
        return 0, nil
    }

    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return 0, err
    }
    if fd.IsStream && len(p) > maxRW {
        p = p[:maxRW]
    }
    for {
        // 第一次调用 syscall.Read 之后，如果读到了数据
        // 那么直接就返回了
        n, err := syscall.Read(fd.Sysfd, p)
        if err != nil {
            n = 0
            // 如果 os 返回 EAGAIN，说明可能暂时没数据
            // 判断 fd 是 pollable 的话，说明可以走 poll 流程
            if err == syscall.EAGAIN && fd.pd.pollable() {
                if err = fd.pd.waitRead(fd.isFile); err == nil {
                    continue
                }
            }

        }
        err = fd.eofError(n, err)
        return n, err
    }
}

```

```go
func (pd *pollDesc) waitRead(isFile bool) error {
    return pd.wait('r', isFile)
}


func (pd *pollDesc) wait(mode int, isFile bool) error {
    if pd.runtimeCtx == 0 {
        return errors.New("waiting for unsupported file type")
    }
    res := runtime_pollWait(pd.runtimeCtx, mode)
    return convertErr(res, isFile)
}
```

runtime_pollWait 是用 `go:linkname` 来链接期链接到的函数，实现在 `runtime/netpoll.go` 中:

```go
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
    err := netpollcheckerr(pd, int32(mode))
    if err != 0 {
        return err
    }

    for !netpollblock(pd, int32(mode), false) {
        err = netpollcheckerr(pd, int32(mode))
        if err != 0 {
            return err
        }
        // Can happen if timeout has fired and unblocked us,
        // but before we had a chance to run, timeout has been reset.
        // Pretend it has not happened and retry.
    }
    return 0
}
```

```go
// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
    gpp := &pd.rg
    if mode == 'w' {
        gpp = &pd.wg
    }

    // set the gpp semaphore to WAIT
    for {
        old := *gpp
        if old == pdReady {
            *gpp = 0
            return true
        }
        if old != 0 {
            throw("runtime: double wait")
        }
        if atomic.Casuintptr(gpp, 0, pdWait) {
            break
        }
    }

    // need to recheck error states after setting gpp to WAIT
    // this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
    // do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
    if waitio || netpollcheckerr(pd, mode) == 0 {
        gopark(netpollblockcommit, unsafe.Pointer(gpp), "IO wait", traceEvGoBlockNet, 5)
    }
    // be careful to not lose concurrent READY notification
    old := atomic.Xchguintptr(gpp, 0)
    if old > pdWait {
        throw("runtime: corrupted polldesc")
    }
    return old == pdReady
}
```

gopark 会执行 netpollblockcommit，并将 gpp 挂起，netpollblockcommit 比较简单:

```go
func netpollblockcommit(gp *g, gpp unsafe.Pointer) bool {
    r := atomic.Casuintptr((*uintptr)(gpp), pdWait, uintptr(unsafe.Pointer(gp)))
    if r {
        // Bump the count of goroutines waiting for the poller.
        // The scheduler uses this to decide whether to block
        // waiting for the poller if there is nothing else to do.
        atomic.Xadd(&netpollWaiters, 1)
    }
    return r
}
```

#### Write 流程

```go
// Write implements the Conn Write method.
func (c *conn) Write(b []byte) (int, error) {
    if !c.ok() {
        return 0, syscall.EINVAL
    }
    n, err := c.fd.Write(b)
    if err != nil {
        err = &OpError{Op: "write", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return n, err
}

```

```go
func (fd *netFD) Write(buf []byte) (int, error) {
    n, err := fd.pfd.Write(buf)
    runtime.KeepAlive(fd)
    return n, wrapSyscallError("wsasend", err)
}
```