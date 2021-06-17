# netpoll

socket，connect，listen，getsockopt 都有一个全局函数变量来表示。

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
        // 基于流的协议
        case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
            if err := fd.listenStream(laddr, listenerBacklog); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        // 基于数据报的协议
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
        // bind()
        if err := syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
            return os.NewSyscallError("bind", err)
        }
    }

    // listenFunc 是全局函数值，在 linux 下非测试环境被绑定到 syscall.Listen
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

### netFD 初始化

在上面的 listen 流程的 socket 函数中会调用 newFD 来初始化一个 fd。

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

在 socket、bind、listen 三连发，都没有出错的情况下，会调用 fd.init():

```go
func (fd *netFD) init() error {
    return fd.pfd.Init(fd.net, true)
}
```

```go
// Init initializes the FD. The Sysfd field should already be set.
// This can be called multiple times on a single FD.
// The net argument is a network name from the net package (e.g., "tcp"),
// or "file".
// Set pollable to true if fd should be managed by runtime netpoll.
func (fd *FD) Init(net string, pollable bool) error {
    // We don't actually care about the various network types.
    if net == "file" {
        fd.isFile = true
    }
    if !pollable {
        fd.isBlocking = true
        return nil
    }
    return fd.pd.init(fd)
}
```

```go
func (pd *pollDesc) init(fd *FD) error {
    serverInit.Do(runtime_pollServerInit)
    ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
    if errno != 0 {
        if ctx != 0 {
            runtime_pollUnblock(ctx)
            runtime_pollClose(ctx)
        }
        return syscall.Errno(errno)
    }
    pd.runtimeCtx = ctx
    return nil
}
```

```go
//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
    pd := pollcache.alloc()
    lock(&pd.lock)
    if pd.wg != 0 && pd.wg != pdReady {
        throw("runtime: blocked write on free polldesc")
    }
    if pd.rg != 0 && pd.rg != pdReady {
        throw("runtime: blocked read on free polldesc")
    }
    pd.fd = fd
    pd.closing = false
    pd.seq++
    pd.rg = 0
    pd.rd = 0
    pd.wg = 0
    pd.wd = 0
    unlock(&pd.lock)

    var errno int32
    errno = netpollopen(fd, pd)
    return pd, int(errno)
}
```

每一个 fd 对会都应一个 pollDesc 结构，可以看到有 pollcache 提供一定程度的复用。

```go
func netpollopen(fd uintptr, pd *pollDesc) int32 {
    var ev epollevent
    // linux 下 epoll 采用了 ET 模式
    // epoll 下用户可以 watch 的 descriptors 是有上限的
    // 具体上限的查看及配置可以通过 /proc/sys/fs/epoll/max_user_watches 系统虚拟文件进行操作
    // https://github.com/torvalds/linux/blob/v4.9/fs/eventpoll.c#L259-L263
    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

pollDesc 初始化好之后，会当作 epoll event 的数据存储到 ev.data 中。 当有事件就续时，会取 ev.data，以判断是哪个 fd 可读/可写。

`type conn struct` 是在 accept 阶段赋值给 `type Conn interface` 的。

### accept 流程

```go
// Accept implements the Accept method in the Listener interface; it
// waits for the next call and returns a generic Conn.
func (l *TCPListener) Accept() (Conn, error) {
    if !l.ok() {
        return nil, syscall.EINVAL
    }
    c, err := l.accept()
    if err != nil {
        return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
    }
    return c, nil
}
```

```go
func (ln *TCPListener) accept() (*TCPConn, error) {
    fd, err := ln.fd.accept()
    if err != nil {
        return nil, err
    }
    return newTCPConn(fd), nil
}

```

```go
func newTCPConn(fd *netFD) *TCPConn {
    c := &TCPConn{conn{fd}}
    setNoDelay(c.fd, true)
    return c
}
```

```go
func (fd *netFD) accept() (netfd *netFD, err error) {
    d, rsa, errcall, err := fd.pfd.Accept()
    if err != nil {
        if errcall != "" {
            err = wrapSyscallError(errcall, err)
        }
        return nil, err
    }

    if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
        poll.CloseFunc(d)
        return nil, err
    }
    if err = netfd.init(); err != nil {
        fd.Close()
        return nil, err
    }
    lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
    netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
    return netfd, nil
}
```

```go
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
    if err := fd.readLock(); err != nil {
        return -1, nil, "", err
    }
    defer fd.readUnlock()

    if err := fd.pd.prepareRead(fd.isFile); err != nil {
        return -1, nil, "", err
    }
    for {
        s, rsa, errcall, err := accept(fd.Sysfd)
        if err == nil {
            return s, rsa, "", err
        }
        switch err {
        case syscall.EAGAIN:
            if fd.pd.pollable() {
                if err = fd.pd.waitRead(fd.isFile); err == nil {
                    continue
                }
            }
        case syscall.ECONNABORTED:
            // This means that a socket on the listen
            // queue was closed before we Accept()ed it;
            // it's a silly error, so try again.
            continue
        }
        return -1, nil, errcall, err
    }
}
```

```go
// Wrapper around the accept system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func accept(s int) (int, syscall.Sockaddr, string, error) {
    ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
    // On Linux the accept4 system call was introduced in 2.6.28
    // kernel and on FreeBSD it was introduced in 10 kernel. If we
    // get an ENOSYS error on both Linux and FreeBSD, or EINVAL
    // error on Linux, fall back to using accept.
    switch err {
    case nil:
        return ns, sa, "", nil
    default: // errors other than the ones listed
        return -1, sa, "accept4", err
    case syscall.ENOSYS: // syscall missing
    case syscall.EINVAL: // some Linux use this instead of ENOSYS
    case syscall.EACCES: // some Linux use this instead of ENOSYS
    case syscall.EFAULT: // some Linux use this instead of ENOSYS
    }

    // See ../syscall/exec_unix.go for description of ForkLock.
    // It is probably okay to hold the lock across syscall.Accept
    // because we have put fd.sysfd into non-blocking mode.
    // However, a call to the File method will put it back into
    // blocking mode. We can't take that risk, so no use of ForkLock here.
    ns, sa, err = AcceptFunc(s)
    if err == nil {
        syscall.CloseOnExec(ns)
    }
    if err != nil {
        return -1, nil, "accept", err
    }
    if err = syscall.SetNonblock(ns, true); err != nil {
        CloseFunc(ns)
        return -1, nil, "setnonblock", err
    }
    return ns, sa, "", nil
}

```

可以看到，最终还是用 syscall 中的 accept4 或 accept 完成了系统调用。accept4 对比 accept 的优势是，可以通过一次系统调用完成 accept 和 nonblock flag 的两个目的。而使用 accept 的话，还要手动 syscall.SetNonblock。

### Read 流程

```go
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

gopark 将当前 g 挂起，等待就绪事件到达之后再继续执行。

### Write 流程

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

```go
// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
    if err := fd.writeLock(); err != nil {
        return 0, err
    }
    defer fd.writeUnlock()
    if err := fd.pd.prepareWrite(fd.isFile); err != nil {
        return 0, err
    }
    var nn int
    for {
        max := len(p)
        if fd.IsStream && max-nn > maxRW {
            max = nn + maxRW
        }
        n, err := syscall.Write(fd.Sysfd, p[nn:max])
        if n > 0 {
            nn += n
        }
        if nn == len(p) {
            return nn, err
        }
        if err == syscall.EAGAIN && fd.pd.pollable() {
            if err = fd.pd.waitWrite(fd.isFile); err == nil {
                continue
            }
        }
        if err != nil {
            return nn, err
        }
        if n == 0 {
            return nn, io.ErrUnexpectedEOF
        }
    }
}
```

内核的写缓冲区满，这里的 syscall.Write 就会返回 EAGAIN。

```go
func (pd *pollDesc) waitWrite(isFile bool) error {
    return pd.wait('w', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
    if pd.runtimeCtx == 0 {
        return errors.New("waiting for unsupported file type")
    }
    res := runtime_pollWait(pd.runtimeCtx, mode)
    return convertErr(res, isFile)
}
```

后面的流程就和 Read 完全一致了。

### 就续通知

```go
// poll 已经就绪的网络连接
// 返回那些已经可以跑的 goroutine 列表
func netpoll(block bool) *g {
    if epfd == -1 {
        return nil
    }
    waitms := int32(-1)
    if !block {
        waitms = 0
    }
    var events [128]epollevent
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    if n < 0 {
        if n != -_EINTR {
            println("runtime: epollwait on fd", epfd, "failed with", -n)
            throw("runtime: netpoll failed")
        }
        goto retry
    }
    var gp guintptr
    for i := int32(0); i < n; i++ {
        ev := &events[i]
        if ev.events == 0 {
            continue
        }
        var mode int32
        if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'r'
        }
        if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'w'
        }
        if mode != 0 {
            pd := *(**pollDesc)(unsafe.Pointer(&ev.data))

            netpollready(&gp, pd, mode)
        }
    }
    if block && gp == 0 {
        goto retry
    }
    return gp.ptr()
}

// 让 pd 就续，新的可以运行的 goroutine 会 set 到 wg/rg
// May run during STW, so write barriers are not allowed.
//go:nowritebarrier
func netpollready(gpp *guintptr, pd *pollDesc, mode int32) {
    var rg, wg guintptr
    if mode == 'r' || mode == 'r'+'w' {
        rg.set(netpollunblock(pd, 'r', true))
    }
    if mode == 'w' || mode == 'r'+'w' {
        wg.set(netpollunblock(pd, 'w', true))
    }
    if rg != 0 {
        rg.ptr().schedlink = *gpp
        *gpp = rg
    }
    if wg != 0 {
        wg.ptr().schedlink = *gpp
        *gpp = wg
    }
}

// 按照 mode 把 pollDesc 的 wg 或者 rg 捞出来，返回
func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
    gpp := &pd.rg
    if mode == 'w' {
        gpp = &pd.wg
    }

    for {
        old := *gpp
        if old == pdReady {
            return nil
        }
        if old == 0 && !ioready {
            // Only set READY for ioready. runtime_pollWait
            // will check for timeout/cancel before waiting.
            return nil
        }
        var new uintptr
        if ioready {
            new = pdReady
        }
        if atomic.Casuintptr(gpp, old, new) {
            if old == pdReady || old == pdWait {
                old = 0
            }
            return (*g)(unsafe.Pointer(old))
        }
    }
}
```

三个函数配合完成就续后唤醒对应的 g 的工作，netpollunblock 从 pollDesc 中捞出 rg/wg，netpollready 然后再把所有的 rg/wg 通过 schedlink 串成一个链表。findrunnable 之类需要 g 的场景下，调度器会主动调用 netpoll 函数来寻找是否有已经就绪的网络事件对应的 g。

netpoll 这个函数是平台相关的，实现在对应的 netpoll_epoll、netpoll_kqueue 文件中。

### 读写 g 的挂起和恢复

在上面读写流程，syscall.Read 或者 syscall.Write 返回 EAGAIN 时，会挂起当前正在进行这个读/写操作的 g，具体是调用 gopark，并执行 netpollblockcommit，并将 gpp 挂起，netpollblockcommit 比较简单:

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

EAGAIN 的时候:

```go
gopark(netpollblockcommit, unsafe.Pointer(gpp), "IO wait", traceEvGoBlockNet, 5)
```

```go
// Puts the current goroutine into a waiting state and calls unlockf.
// If unlockf returns false, the goroutine is resumed.
// unlockf must not access this G's stack, as it may be moved between
// the call to gopark and the call to unlockf.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason string, traceEv byte, traceskip int) {
    mp := acquirem()
    gp := mp.curg
    status := readgstatus(gp)
    if status != _Grunning && status != _Gscanrunning {
        throw("gopark: bad g status")
    }
    mp.waitlock = lock
    mp.waitunlockf = *(*unsafe.Pointer)(unsafe.Pointer(&unlockf)) // unlockf = netpollblockcommit
    gp.waitreason = reason
    mp.waittraceev = traceEv
    mp.waittraceskip = traceskip
    releasem(mp)
    // can't do anything that might move the G between Ms here.
    mcall(park_m)
}
```

```go
// park continuation on g0.
func park_m(gp *g) {
    _g_ := getg()

    casgstatus(gp, _Grunning, _Gwaiting)
    dropg()

    if _g_.m.waitunlockf != nil {
        // fn = netpollblockcommit
        fn := *(*func(*g, unsafe.Pointer) bool)(unsafe.Pointer(&_g_.m.waitunlockf))
        ok := fn(gp, _g_.m.waitlock)
        _g_.m.waitunlockf = nil
        _g_.m.waitlock = nil
        // 原子操作没成功，那还得先继续执行当前的逻辑
        // 回到最外面的读/写循环
        if !ok {
            casgstatus(gp, _Gwaiting, _Grunnable)
            execute(gp, true) // Schedule it back, never returns.
        }
    }
    // g 成功挂起，m 成功解绑，当前的 m 进入调度循环，去找其它可以执行的 g
    schedule()
}

// 切换到 m->g0 的栈，然后调用 fn(g)
// fn 必须不能返回
// 这个 g 之后应该用 gogo(&g->sched) 来恢复执行
TEXT runtime·mcall(SB), NOSPLIT, $0-8
    MOVQ    fn+0(FP), DI

    get_tls(CX)
    MOVQ    g(CX), AX    // save state in g->sched
    MOVQ    0(SP), BX    // caller's PC
    // 把当前 g 的运行现场保存到 g.gobuf 中
    MOVQ    BX, (g_sched+gobuf_pc)(AX)
    LEAQ    fn+0(FP), BX    // caller's SP
    MOVQ    BX, (g_sched+gobuf_sp)(AX)
    MOVQ    AX, (g_sched+gobuf_g)(AX)
    MOVQ    BP, (g_sched+gobuf_bp)(AX)

    // switch to m->g0 & its stack, call fn
    MOVQ    g(CX), BX
    MOVQ    g_m(BX), BX
    MOVQ    m_g0(BX), SI
    CMPQ    SI, AX    // if g == m->g0 call badmcall
    JNE    3(PC)
    MOVQ    $runtime·badmcall(SB), AX
    JMP    AX
    MOVQ    SI, g(CX)    // g = m->g0
    MOVQ    (g_sched+gobuf_sp)(SI), SP    // sp = m->g0->sched.sp
    PUSHQ    AX
    MOVQ    DI, DX
    MOVQ    0(DI), DI
    CALL    DI
    POPQ    AX
    MOVQ    $runtime·badmcall2(SB), AX
    JMP    AX
    RET

// 把 g 当前所在的 m 的指针都清 0
// 将 m 腾出来干别的事去
func dropg() {
    _g_ := getg()
    setMNoWB(&_g_.m.curg.m, nil)
    setGNoWB(&_g_.m.curg, nil)
}
```

上面就是挂起的所有流程了，可见所谓的挂起，就是把 g 当前的运行状态: bp、sp、pc 寄存器，以及 g 的地址保存起来。

至于唤醒流程，当调度器在 findrunnable、startTheWorldWithSema 或者 sysmon 中调用 netpoll 函数时，会获取到上面说的就绪的 g 列表。把这些 g 的 bp/sp/pc 都从 g.gobuf 中恢复出来，就可以继续执行它们的 Read/Write 操作了。

因为调度中有讲，这里就不赘述了。

### 读写超时

```go
// 设置底层连接的读超时
// 超时时间是 0 值的话永远都不会超时
func (c *Conn) SetReadDeadline(t time.Time) error {
    return c.conn.SetReadDeadline(t)
}

// 设置底层连接的读超时
// 超时时间是 0 值的话永远都不会超时
// 写超时发生之后， TLS 状态会被破坏，未来的所有写都会返回相同的错误
func (c *Conn) SetWriteDeadline(t time.Time) error {
    return c.conn.SetWriteDeadline(t)
}
```

```go
// 实现 Conn 接口中的方法
func (c *conn) SetReadDeadline(t time.Time) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := c.fd.pfd.SetReadDeadline(t); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}

// 实现 Conn 接口中的方法
func (c *conn) SetWriteDeadline(t time.Time) error {
    if !c.ok() {
        return syscall.EINVAL
    }
    if err := c.fd.pfd.SetWriteDeadline(t); err != nil {
        return &OpError{Op: "set", Net: c.fd.net, Source: nil, Addr: c.fd.laddr, Err: err}
    }
    return nil
}
```

```go
// 设置关联 fd 的读取 deadline
func (fd *FD) SetReadDeadline(t time.Time) error {
    return setDeadlineImpl(fd, t, 'r')
}

// 设置关联 fd 的写入 deadline
func (fd *FD) SetWriteDeadline(t time.Time) error {
    return setDeadlineImpl(fd, t, 'w')
}
```

```go
func setDeadlineImpl(fd *FD, t time.Time, mode int) error {
    diff := int64(time.Until(t))
    d := runtimeNano() + diff
    if d <= 0 && diff > 0 {
        // 如果用户提供了未来的 deadline，但是 delay 计算溢出了，那么设置 dealine 到最大的可能的值
        d = 1<<63 - 1
    }
    if t.IsZero() {
        // IsZero reports whether t represents the zero time instant,
        // January 1, year 1, 00:00:00 UTC.
        // func (t Time) IsZero() bool {
        //     return t.sec() == 0 && t.nsec() == 0
        // }
        d = 0
    }
    if err := fd.incref(); err != nil {
        return err
    }
    defer fd.decref()
    if fd.pd.runtimeCtx == 0 {
        return ErrNoDeadline
    }
    runtime_pollSetDeadline(fd.pd.runtimeCtx, d, mode)
    return nil
}
```

```go
//go:linkname poll_runtime_pollSetDeadline internal/poll.runtime_pollSetDeadline
func poll_runtime_pollSetDeadline(pd *pollDesc, d int64, mode int) {
    lock(&pd.lock)
    if pd.closing {
        unlock(&pd.lock)
        return
    }
    pd.seq++ // invalidate current timers
    // 重置当前的 timer
    if pd.rt.f != nil {
        // 删除 pollDesc 相关的 read timer
        deltimer(&pd.rt)
        pd.rt.f = nil
    }
    if pd.wt.f != nil {
        // 删除 pollDesc 相关的 write timer
        deltimer(&pd.wt)
        pd.wt.f = nil
    }
    // 设置新 timer
    if d != 0 && d <= nanotime() {
        d = -1
    }
    if mode == 'r' || mode == 'r'+'w' {
        // 记录 read deadline
        pd.rd = d
    }
    if mode == 'w' || mode == 'r'+'w' {
        // 记录 write deadline
        pd.wd = d
    }
    if pd.rd > 0 && pd.rd == pd.wd {
        pd.rt.f = netpollDeadline
        pd.rt.when = pd.rd
        // Copy current seq into the timer arg.
        // Timer func will check the seq against current descriptor seq,
        // if they differ the descriptor was reused or timers were reset.
        pd.rt.arg = pd
        pd.rt.seq = pd.seq
        // 插入 timer 到时间堆
        addtimer(&pd.rt)
    } else {
        if pd.rd > 0 {
            pd.rt.f = netpollReadDeadline
            pd.rt.when = pd.rd
            pd.rt.arg = pd
            pd.rt.seq = pd.seq
            addtimer(&pd.rt)
        }
        if pd.wd > 0 {
            pd.wt.f = netpollWriteDeadline
            pd.wt.when = pd.wd
            pd.wt.arg = pd
            pd.wt.seq = pd.seq
            addtimer(&pd.wt)
        }
    }
    // If we set the new deadline in the past, unblock currently pending IO if any.
    var rg, wg *g
    atomicstorep(unsafe.Pointer(&wg), nil) // full memory barrier between stores to rd/wd and load of rg/wg in netpollunblock
    if pd.rd < 0 {
        rg = netpollunblock(pd, 'r', false)
    }
    if pd.wd < 0 {
        wg = netpollunblock(pd, 'w', false)
    }
    unlock(&pd.lock)
    if rg != nil {
        netpollgoready(rg, 3)
    }
    if wg != nil {
        netpollgoready(wg, 3)
    }
}
```

根据 read deadline 和 write deadline 给要插入时间堆的 timer 设置不同的回调函数。

```go
func netpollDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, true, true)
}

func netpollReadDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, true, false)
}

func netpollWriteDeadline(arg interface{}, seq uintptr) {
    netpolldeadlineimpl(arg.(*pollDesc), seq, false, true)
}
```

调用最终的实现函数:

```go
func netpolldeadlineimpl(pd *pollDesc, seq uintptr, read, write bool) {
    lock(&pd.lock)
    // Seq arg is seq when the timer was set.
    // If it's stale, ignore the timer event.
    if seq != pd.seq {
        // The descriptor was reused or timers were reset.
        unlock(&pd.lock)
        return
    }
    var rg *g
    if read {
        if pd.rd <= 0 || pd.rt.f == nil {
            throw("runtime: inconsistent read deadline")
        }
        pd.rd = -1
        atomicstorep(unsafe.Pointer(&pd.rt.f), nil) // full memory barrier between store to rd and load of rg in netpollunblock
        rg = netpollunblock(pd, 'r', false)
    }
    var wg *g
    if write {
        if pd.wd <= 0 || pd.wt.f == nil && !read {
            throw("runtime: inconsistent write deadline")
        }
        pd.wd = -1
        atomicstorep(unsafe.Pointer(&pd.wt.f), nil) // full memory barrier between store to wd and load of wg in netpollunblock
        wg = netpollunblock(pd, 'w', false)
    }
    unlock(&pd.lock)
    // rg 和 wg 是通过 netpollunblock 从 pollDesc 结构中捞出来的
    if rg != nil {
        // 恢复 goroutine 执行现场
        // 继续执行
        netpollgoready(rg, 0)
    }
    if wg != nil {
        // 恢复 goroutine 执行现场
        // 继续执行
        netpollgoready(wg, 0)
    }
}
```

```go
func netpollgoready(gp *g, traceskip int) {
    atomic.Xadd(&netpollWaiters, -1)
    goready(gp, traceskip+1)
}

func goready(gp *g, traceskip int) {
    // switch -> g0
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}

// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {

    status := readgstatus(gp)

    // 标记为 runnable
    _g_ := getg()
    _g_.m.locks++ // disable preemption because it can be holding p in a local var
    if status&^_Gscan != _Gwaiting {
        dumpgstatus(gp)
        throw("bad g->status in ready")
    }

    // status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
    casgstatus(gp, _Gwaiting, _Grunnable)
    // 放到执行队列里
    runqput(_g_.m.p.ptr(), gp, next)
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
        // 如果有空闲的 p，那么就叫起床干活
        wakep()
    }
    _g_.m.locks--
    if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in Case we've cleared it in newstack
        _g_.stackguard0 = stackPreempt
    }
}
```

### 连接关闭

```go
// Close closes the connection.
func (c *conn) Close() error {
    if !c.ok() {
        return syscall.EINVAL
    }
    err := c.fd.Close()
    if err != nil {
        err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return err
}
```

```go
func (fd *netFD) Close() error {
    runtime.SetFinalizer(fd, nil)
    return fd.pfd.Close()
}
```

```go
// Close closes the FD. The underlying file descriptor is closed by the
// destroy method when there are no remaining references.
func (fd *FD) Close() error {
    if !fd.fdmu.increfAndClose() {
        return errClosing(fd.isFile)
    }

    // Unblock any I/O.  Once it all unblocks and returns,
    // so that it cannot be referring to fd.sysfd anymore,
    // the final decref will close fd.sysfd. This should happen
    // fairly quickly, since all the I/O is non-blocking, and any
    // attempts to block in the pollDesc will return errClosing(fd.isFile).
    fd.pd.evict()

    // The call to decref will call destroy if there are no other
    // references.
    err := fd.decref()

    // Wait until the descriptor is closed. If this was the only
    // reference, it is already closed. Only wait if the file has
    // not been set to blocking mode, as otherwise any current I/O
    // may be blocking, and that would block the Close.
    if !fd.isBlocking {
        runtime_Semacquire(&fd.csema)
    }

    return err
}
```

```go
// Evict evicts fd from the pending list, unblocking any I/O running on fd.
func (pd *pollDesc) evict() {
    if pd.runtimeCtx == 0 {
        return
    }
    runtime_pollUnblock(pd.runtimeCtx)
}
```

```go
//go:linkname poll_runtime_pollUnblock internal/poll.runtime_pollUnblock
func poll_runtime_pollUnblock(pd *pollDesc) {
    lock(&pd.lock)
    if pd.closing {
        throw("runtime: unblock on closing polldesc")
    }
    pd.closing = true
    pd.seq++
    var rg, wg *g
    atomicstorep(unsafe.Pointer(&rg), nil) // full memory barrier between store to closing and read of rg/wg in netpollunblock
    rg = netpollunblock(pd, 'r', false)
    wg = netpollunblock(pd, 'w', false)
    if pd.rt.f != nil {
        deltimer(&pd.rt)
        pd.rt.f = nil
    }
    if pd.wt.f != nil {
        deltimer(&pd.wt)
        pd.wt.f = nil
    }
    unlock(&pd.lock)
    if rg != nil {
        netpollgoready(rg, 3)
    }
    if wg != nil {
        netpollgoready(wg, 3)
    }
}
```

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
