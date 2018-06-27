# 系统调用

## 概念

TODO

## 入口

syscall 有下面几个入口，在 `syscall/asm_linux_amd64.s` 中。

```go
func Syscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err syscall.Errno)

func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err syscall.Errno)

func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err syscall.Errno)

func RawSyscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err syscall.Errno)
```

这些函数的实现都是汇编，按照 linux 的 syscall 调用规范，我们只要在汇编中把参数依次传入寄存器，并调用 SYSCALL 指令即可进入内核处理逻辑，系统调用执行完毕之后，返回值放在 RAX 中:

| RDI | RSI | RDX | R10 | R8 | R9 | RAX|
|--|--|--|--| --| --|--|
|参数一|参数二|参数三|参数四|参数五|参数六|系统调用编号/返回值|

Syscall 和 Syscall6 的区别只有传入参数不一样:

```go
// func Syscall(trap int64, a1, a2, a3 uintptr) (r1, r2, err uintptr);
TEXT ·Syscall(SB),NOSPLIT,$0-56
    CALL    runtime·entersyscall(SB)
    MOVQ    a1+8(FP), DI
    MOVQ    a2+16(FP), SI
    MOVQ    a3+24(FP), DX
    MOVQ    $0, R10
    MOVQ    $0, R8
    MOVQ    $0, R9
    MOVQ    trap+0(FP), AX    // syscall entry
    SYSCALL
    // 0xfffffffffffff001 是 linux MAX_ERRNO 取反 转无符号，http://lxr.free-electrons.com/source/include/linux/err.h#L17
    CMPQ    AX, $0xfffffffffffff001
    JLS    ok
    MOVQ    $-1, r1+32(FP)
    MOVQ    $0, r2+40(FP)
    NEGQ    AX
    MOVQ    AX, err+48(FP)
    CALL    runtime·exitsyscall(SB)
    RET
ok:
    MOVQ    AX, r1+32(FP)
    MOVQ    DX, r2+40(FP)
    MOVQ    $0, err+48(FP)
    CALL    runtime·exitsyscall(SB)
    RET

// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB),NOSPLIT,$0-80
    CALL    runtime·entersyscall(SB)
    MOVQ    a1+8(FP), DI
    MOVQ    a2+16(FP), SI
    MOVQ    a3+24(FP), DX
    MOVQ    a4+32(FP), R10
    MOVQ    a5+40(FP), R8
    MOVQ    a6+48(FP), R9
    MOVQ    trap+0(FP), AX    // syscall entry
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    ok6
    MOVQ    $-1, r1+56(FP)
    MOVQ    $0, r2+64(FP)
    NEGQ    AX
    MOVQ    AX, err+72(FP)
    CALL    runtime·exitsyscall(SB)
    RET
ok6:
    MOVQ    AX, r1+56(FP)
    MOVQ    DX, r2+64(FP)
    MOVQ    $0, err+72(FP)
    CALL    runtime·exitsyscall(SB)
    RET
```

两个函数没什么大区别，为啥不用一个呢？个人猜测，Go 的函数参数都是栈上传入，可能是为了节省一点栈空间。。在正常的 Syscall 操作之前会通知 runtime，接下来我要进行 syscall 操作了 `runtime·entersyscall`，退出时会调用 `runtime·exitsyscall`。

```go
// func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
    MOVQ    a1+8(FP), DI
    MOVQ    a2+16(FP), SI
    MOVQ    a3+24(FP), DX
    MOVQ    $0, R10
    MOVQ    $0, R8
    MOVQ    $0, R9
    MOVQ    trap+0(FP), AX    // syscall entry
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    ok1
    MOVQ    $-1, r1+32(FP)
    MOVQ    $0, r2+40(FP)
    NEGQ    AX
    MOVQ    AX, err+48(FP)
    RET
ok1:
    MOVQ    AX, r1+32(FP)
    MOVQ    DX, r2+40(FP)
    MOVQ    $0, err+48(FP)
    RET

// func RawSyscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall6(SB),NOSPLIT,$0-80
    MOVQ    a1+8(FP), DI
    MOVQ    a2+16(FP), SI
    MOVQ    a3+24(FP), DX
    MOVQ    a4+32(FP), R10
    MOVQ    a5+40(FP), R8
    MOVQ    a6+48(FP), R9
    MOVQ    trap+0(FP), AX    // syscall entry
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    ok2
    MOVQ    $-1, r1+56(FP)
    MOVQ    $0, r2+64(FP)
    NEGQ    AX
    MOVQ    AX, err+72(FP)
    RET
ok2:
    MOVQ    AX, r1+56(FP)
    MOVQ    DX, r2+64(FP)
    MOVQ    $0, err+72(FP)
    RET
```

RawSyscall 和 Syscall 的区别也非常微小，就只是在进入 Syscall 和退出的时候没有通知 runtime，这样 runtime 理论上是没有办法通过调度把这个 g 的 m 的 p 调度走的，所以如果用户代码使用了 RawSyscall 来做一些阻塞的系统调用，是有可能阻塞其它的 g 的，下面是官方开发的原话:

> Yes, if you call RawSyscall you may block other goroutines from running.  The system monitor may start them up after a while, but I think there are cases where it won't.  I would say that Go programs should always call Syscall.  RawSyscall exists to make it slightly more efficient to call system calls that never block, such as getpid. But it's really an internal mechanism.

RawSyscall 只是为了在执行那些一定不会阻塞的系统调用时，能节省两次对 runtime 的函数调用消耗。

vdso:

```go
// func gettimeofday(tv *Timeval) (err uintptr)
TEXT ·gettimeofday(SB),NOSPLIT,$0-16
    MOVQ    tv+0(FP), DI
    MOVQ    $0, SI
    MOVQ    runtime·__vdso_gettimeofday_sym(SB), AX
    CALL    AX

    CMPQ    AX, $0xfffffffffffff001
    JLS    ok7
    NEGQ    AX
    MOVQ    AX, err+8(FP)
    RET
ok7:
    MOVQ    $0, err+8(FP)
    RET
```

## 系统调用管理

先是系统调用的定义文件:

```shell
/syscall/syscall_linux.go
```

可以把系统调用分为三类:

1. 阻塞系统调用
2. 非阻塞系统调用
3. wrapped 系统调用

阻塞系统调用会定义成下面这样的形式:

```go
//sys   Madvise(b []byte, advice int) (err error)
```

非阻塞系统调用:

```go
//sysnb    EpollCreate(size int) (fd int, err error)
```

然后，根据这些注释，mksyscall.pl 脚本会生成对应的平台的具体实现。mksyscall.pl 是一段 perl 脚本，感兴趣的同学可以自行查看，这里就不再赘述了。

看看阻塞和非阻塞的系统调用的生成结果:

```go
func Madvise(b []byte, advice int) (err error) {
    var _p0 unsafe.Pointer
    if len(b) > 0 {
        _p0 = unsafe.Pointer(&b[0])
    } else {
        _p0 = unsafe.Pointer(&_zero)
    }
    _, _, e1 := Syscall(SYS_MADVISE, uintptr(_p0), uintptr(len(b)), uintptr(advice))
    if e1 != 0 {
        err = errnoErr(e1)
    }
    return
}

func EpollCreate(size int) (fd int, err error) {
    r0, _, e1 := RawSyscall(SYS_EPOLL_CREATE, uintptr(size), 0, 0)
    fd = int(r0)
    if e1 != 0 {
        err = errnoErr(e1)
    }
    return
}
```

显然，标记为 sys 的系统调用使用的是 Syscall 或者 Syscall6，标记为 sysnb 的系统调用使用的是 RawSyscall 或 RawSyscall6。

wrapped 的系统调用是怎么一回事呢？

```go
func Rename(oldpath string, newpath string) (err error) {
    return Renameat(_AT_FDCWD, oldpath, _AT_FDCWD, newpath)
}
```

可能是觉得系统调用的名字不太好，或者参数太多，我们就简单包装一下。没啥特别的。

## runtime 中的 SYSCALL

除了上面提到的阻塞非阻塞和 wrapped syscall，runtime 中还定义了一些 low-level 的 syscall，这些是不暴露给用户的。

提供给用户的 syscall 库，在使用时，会使 goroutine 和 p 分别进入 Gsyscall 和 Psyscall 状态。但 runtime 自己封装的这些 syscall 无论是否阻塞，都不会调用 entersyscall 和 exitsyscall。 虽说是 “low-level” 的 syscall，
不过和暴露给用户的 syscall 本质是一样的。这些代码在 `runtime/sys_linux_amd64.s` 中，举个具体的例子:

```go
TEXT runtime·write(SB),NOSPLIT,$0-28
    MOVQ    fd+0(FP), DI
    MOVQ    p+8(FP), SI
    MOVL    n+16(FP), DX
    MOVL    $SYS_write, AX
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    2(PC)
    MOVL    $-1, AX
    MOVL    AX, ret+24(FP)
    RET

TEXT runtime·read(SB),NOSPLIT,$0-28
    MOVL    fd+0(FP), DI
    MOVQ    p+8(FP), SI
    MOVL    n+16(FP), DX
    MOVL    $SYS_read, AX
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    2(PC)
    MOVL    $-1, AX
    MOVL    AX, ret+24(FP)
    RET
```

下面是所有 runtime 另外定义的 syscall 列表:

```go
#define SYS_read        0
#define SYS_write        1
#define SYS_open        2
#define SYS_close        3
#define SYS_mmap        9
#define SYS_munmap        11
#define SYS_brk         12
#define SYS_rt_sigaction    13
#define SYS_rt_sigprocmask    14
#define SYS_rt_sigreturn    15
#define SYS_access        21
#define SYS_sched_yield     24
#define SYS_mincore        27
#define SYS_madvise        28
#define SYS_setittimer        38
#define SYS_getpid        39
#define SYS_socket        41
#define SYS_connect        42
#define SYS_clone        56
#define SYS_exit        60
#define SYS_kill        62
#define SYS_fcntl        72
#define SYS_getrlimit        97
#define SYS_sigaltstack     131
#define SYS_arch_prctl        158
#define SYS_gettid        186
#define SYS_tkill        200
#define SYS_futex        202
#define SYS_sched_getaffinity    204
#define SYS_epoll_create    213
#define SYS_exit_group        231
#define SYS_epoll_wait        232
#define SYS_epoll_ctl        233
#define SYS_pselect6        270
#define SYS_epoll_create1    291
```

这些 syscall 理论上都是不会在执行期间被调度器剥离掉 p 的，所以执行成功之后 goroutine 会继续执行，而不像用户的 goroutine 一样，若被剥离 p 会进入等待队列。

## 和调度的交互

### entersyscall

```go
// syscall 库和 cgo 调用的标准入口
//go:nosplit
func entersyscall() {
    reentersyscall(getcallerpc(), getcallersp())
}

//go:nosplit
func reentersyscall(pc, sp uintptr) {
    _g_ := getg()

    // 需要禁止 g 的抢占
    _g_.m.locks++

    // entersyscall 中不能调用任何会导致栈增长/分裂的函数
    _g_.stackguard0 = stackPreempt
    // 设置 throwsplit，在 newstack 中，如果发现 throwsplit 是 true
    // 会直接 crash
    // 下面的代码是 newstack 里的
    // if thisg.m.curg.throwsplit {
    //     throw("runtime: stack split at bad time")
    // }
    _g_.throwsplit = true

    // Leave SP around for GC and traceback.
    // 保存现场，在 syscall 之后会依据这些数据恢复现场
    save(pc, sp)
    _g_.syscallsp = sp
    _g_.syscallpc = pc
    casgstatus(_g_, _Grunning, _Gsyscall)
    if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
        systemstack(func() {
            print("entersyscall inconsistent ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
            throw("entersyscall")
        })
    }

    if atomic.Load(&sched.sysmonwait) != 0 {
        systemstack(entersyscall_sysmon)
        save(pc, sp)
    }

    if _g_.m.p.ptr().runSafePointFn != 0 {
        // runSafePointFn may stack split if run on this stack
        systemstack(runSafePointFn)
        save(pc, sp)
    }

    _g_.m.syscalltick = _g_.m.p.ptr().syscalltick
    _g_.sysblocktraced = true
    _g_.m.mcache = nil
    _g_.m.p.ptr().m = 0
    atomic.Store(&_g_.m.p.ptr().status, _Psyscall)
    if sched.gcwaiting != 0 {
        systemstack(entersyscall_gcwait)
        save(pc, sp)
    }

    _g_.m.locks--
}
```

可以看到，进入 syscall 的 G 是铁定不会被抢占的。

### exitsyscall

```go
// g 已经退出了 syscall
// 需要准备让 g 在 cpu 上重新运行
// 这个函数只会在 syscall 库中被调用，在 runtime 里用的 low-level syscall
// 不会用到
// 不能有 write barrier，因为 P 可能已经被偷走了
//go:nosplit
//go:nowritebarrierrec
func exitsyscall(dummy int32) {
    _g_ := getg()

    _g_.m.locks++ // see comment in entersyscall
    if getcallersp(unsafe.Pointer(&dummy)) > _g_.syscallsp {
        // throw calls print which may try to grow the stack,
        // but throwsplit == true so the stack can not be grown;
        // use systemstack to avoid that possible problem.
        systemstack(func() {
            throw("exitsyscall: syscall frame is no longer valid")
        })
    }

    _g_.waitsince = 0
    oldp := _g_.m.p.ptr()
    if exitsyscallfast() {
        if _g_.m.mcache == nil {
            systemstack(func() {
                throw("lost mcache")
            })
        }
        // There's a cpu for us, so we can run.
        _g_.m.p.ptr().syscalltick++
        // We need to cas the status and scan before resuming...
        casgstatus(_g_, _Gsyscall, _Grunning)

        // Garbage collector isn't running (since we are),
        // so okay to clear syscallsp.
        _g_.syscallsp = 0
        _g_.m.locks--
        if _g_.preempt {
            // restore the preemption request in case we've cleared it in newstack
            _g_.stackguard0 = stackPreempt
        } else {
            // otherwise restore the real _StackGuard, we've spoiled it in entersyscall/entersyscallblock
            _g_.stackguard0 = _g_.stack.lo + _StackGuard
        }
        _g_.throwsplit = false
        return
    }

    _g_.sysexitticks = 0
    _g_.m.locks--

    // Call the scheduler.
    mcall(exitsyscall0)

    if _g_.m.mcache == nil {
        systemstack(func() {
            throw("lost mcache")
        })
    }

    // Scheduler returned, so we're allowed to run now.
    // Delete the syscallsp information that we left for
    // the garbage collector during the system call.
    // Must wait until now because until gosched returns
    // we don't know for sure that the garbage collector
    // is not running.
    _g_.syscallsp = 0
    _g_.m.p.ptr().syscalltick++
    _g_.throwsplit = false
}
```

这里还调用了 exitsyscallfast 和 exitsyscall0。

#### exitsyscallfast

```go
//go:nosplit
func exitsyscallfast() bool {
    _g_ := getg()

    // Freezetheworld sets stopwait but does not retake P's.
    if sched.stopwait == freezeStopWait {
        _g_.m.mcache = nil
        _g_.m.p = 0
        return false
    }

    // Try to re-acquire the last P.
    if _g_.m.p != 0 && _g_.m.p.ptr().status == _Psyscall && atomic.Cas(&_g_.m.p.ptr().status, _Psyscall, _Prunning) {
        // There's a cpu for us, so we can run.
        exitsyscallfast_reacquired()
        return true
    }

    // Try to get any other idle P.
    oldp := _g_.m.p.ptr()
    _g_.m.mcache = nil
    _g_.m.p = 0
    if sched.pidle != 0 {
        var ok bool
        systemstack(func() {
            ok = exitsyscallfast_pidle()
        })
        if ok {
            return true
        }
    }
    return false
}
```

总之就是努力获取一个 P 来执行 syscall 之后的逻辑。如果哪都没有 P 可以给我们用，那就进入 exitsyscall0 了。

```go
mcall(exitsyscall0)
```

调用 exitsyscall0 时，会切换到 g0 栈。

#### exitsyscall0

```go
// exitsyscall slow path on g0.
// Failed to acquire P, enqueue gp as runnable.
//
//go:nowritebarrierrec
func exitsyscall0(gp *g) {
    _g_ := getg()

    casgstatus(gp, _Gsyscall, _Grunnable)
    dropg()
    lock(&sched.lock)
    _p_ := pidleget()
    if _p_ == nil {
        globrunqput(gp)
    } else if atomic.Load(&sched.sysmonwait) != 0 {
        atomic.Store(&sched.sysmonwait, 0)
        notewakeup(&sched.sysmonnote)
    }
    unlock(&sched.lock)
    if _p_ != nil {
        acquirep(_p_)
        execute(gp, false) // Never returns.
    }
    if _g_.m.lockedg != 0 {
        // Wait until another thread schedules gp and so m again.
        stoplockedm()
        execute(gp, false) // Never returns.
    }
    stopm()
    schedule() // Never returns.
}
```

### entersyscallblock

```go
// The same as entersyscall(), but with a hint that the syscall is blocking.
//go:nosplit
func entersyscallblock(dummy int32) {
    _g_ := getg()

    _g_.m.locks++ // see comment in entersyscall
    _g_.throwsplit = true
    _g_.stackguard0 = stackPreempt // see comment in entersyscall
    _g_.m.syscalltick = _g_.m.p.ptr().syscalltick
    _g_.sysblocktraced = true
    _g_.m.p.ptr().syscalltick++

    // Leave SP around for GC and traceback.
    pc := getcallerpc()
    sp := getcallersp(unsafe.Pointer(&dummy))
    save(pc, sp)
    _g_.syscallsp = _g_.sched.sp
    _g_.syscallpc = _g_.sched.pc
    if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
        sp1 := sp
        sp2 := _g_.sched.sp
        sp3 := _g_.syscallsp
        systemstack(func() {
            print("entersyscallblock inconsistent ", hex(sp1), " ", hex(sp2), " ", hex(sp3), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
            throw("entersyscallblock")
        })
    }
    casgstatus(_g_, _Grunning, _Gsyscall)
    if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
        systemstack(func() {
            print("entersyscallblock inconsistent ", hex(sp), " ", hex(_g_.sched.sp), " ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
            throw("entersyscallblock")
        })
    }

    systemstack(entersyscallblock_handoff)

    // Resave for traceback during blocked call.
    save(getcallerpc(), getcallersp(unsafe.Pointer(&dummy)))

    _g_.m.locks--
}
```

### entersyscallblock_handoff

```go
func entersyscallblock_handoff() {
    handoffp(releasep())
}
```

### entersyscall_sysmon

```go
func entersyscall_sysmon() {
    lock(&sched.lock)
    if atomic.Load(&sched.sysmonwait) != 0 {
        atomic.Store(&sched.sysmonwait, 0)
        notewakeup(&sched.sysmonnote)
    }
    unlock(&sched.lock)
}
```

### entersyscall_gcwait

```go
func entersyscall_gcwait() {
    _g_ := getg()
    _p_ := _g_.m.p.ptr()

    lock(&sched.lock)
    if sched.stopwait > 0 && atomic.Cas(&_p_.status, _Psyscall, _Pgcstop) {
        _p_.syscalltick++
        if sched.stopwait--; sched.stopwait == 0 {
            notewakeup(&sched.stopnote)
        }
    }
    unlock(&sched.lock)
}
```
