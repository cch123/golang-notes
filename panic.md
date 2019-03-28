# panic

## panic 的本质

panic 的本质其实就是 gopanic，用户代码和 runtime 中都会有对 panic() 的调用，最终会转为这个 runtime.gopanic 函数。

```go
func gopanic(e interface{}) {
    // 获取当前 g
    gp := getg()

    // 生成一个新的 panic 结构体
    var p _panic
    p.arg = e
    // 链上之前的 panic 结构
    p.link = gp._panic
    // 新节点做整个链表的表头
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    // 统计
    atomic.Xadd(&runningPanicDefers, 1)

    for {
        d := gp._defer
        if d == nil {
            break
        }

        // If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
        // take defer off list. The earlier panic or Goexit will not continue running.
        if d.started {
            if d._panic != nil {
                d._panic.aborted = true
            }
            d._panic = nil
            d.fn = nil
            gp._defer = d.link
            freedefer(d)
            continue
        }

        // 标记 defer 为已启动，暂时保持在链表上
        // 这样 traceback 在栈增长或者 GC 的时候，能够找到并更新 defer 的参数栈帧
        // 并用 reflectcall 执行 d.fn
        d.started = true

        // 记录在 defer 中发生的 panic
        // 如果在 defer 的函数调用过程中又发生了新的 panic，那个 panic 会在链表中找到 d
        // 然后标记 d._panic(指向当前的 panic) 为 aborted 状态。
        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        p.argp = unsafe.Pointer(getargp(0))
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
        p.argp = nil

        // reflectcall 没有 panic，移除 d
        if gp._defer != d {
            throw("bad defer entry in panic")
        }
        d._panic = nil
        d.fn = nil
        gp._defer = d.link

        pc := d.pc
        sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
        freedefer(d)
        if p.recovered {
            atomic.Xadd(&runningPanicDefers, -1)

            gp._panic = p.link
            // Aborted panics are marked but remain on the g.panic list.
            // Remove them from the list.
            for gp._panic != nil && gp._panic.aborted {
                gp._panic = gp._panic.link
            }
            if gp._panic == nil { // 必须已处理完 signal
                gp.sig = 0
            }
            // 把需要恢复的栈帧信息传给 recovery 函数
            gp.sigcode0 = uintptr(sp)
            gp.sigcode1 = pc
            mcall(recovery)
            throw("recovery failed") // mcall 永远不会返回
        }
    }

    // ran out of deferred calls - old-school panic now
    // Because it is unsafe to call arbitrary user code after freezing
    // the world, we call preprintpanics to invoke all necessary Error
    // and String methods to prepare the panic strings before startpanic.
    preprintpanics(gp._panic)
    startpanic()

    // startpanic set panicking, which will block main from exiting,
    // so now OK to decrement runningPanicDefers.
    atomic.Xadd(&runningPanicDefers, -1)

    printpanics(gp._panic)
    dopanic(0)       // 永远不会返回
}

```

可见这里面的 panic 也是一个链表，用 link 指针相连接:

```go
// panics
type _panic struct {
    argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
    arg       interface{}    // argument to panic
    link      *_panic        // link to earlier panic
    recovered bool           // whether this panic is over
    aborted   bool           // the panic was aborted
}
```

下面这种写法应该会形成一个 _panic 的链表:

```go
package main

func fxfx() {
    defer func() {
        if err := recover(); err != nil {
            println("recover here")
        }
    }()

    defer func() {
        panic(1)
    }()

    defer func() {
        panic(2)
    }()
}

func main() {
    fxfx()
}
```

非 defer 中的 panic 的话，函数的后续代码就没办法运行了，写代码的时候要注意这个特性。

## recovery

上面的 panic 恢复过程，最终走到了 recovery:

```go
// 把需要恢复的栈帧信息传给 recovery 函数
gp.sigcode0 = uintptr(sp)
gp.sigcode1 = pc
mcall(recovery)
```

```go
// 在被 defer 的函数中调用的 recover 从 panic 中已恢复，展开栈继续运行。
// 现场恢复为发生 panic 的函数正常返回时的情况
func recovery(gp *g) {
    // 之前之前传入的 sp 和 pc 恢复
    sp := gp.sigcode0
    pc := gp.sigcode1

    // d 的参数需要放在栈上
    if sp != 0 && (sp < gp.stack.lo || gp.stack.hi < sp) {
        print("recover: ", hex(sp), " not in [", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
        throw("bad recovery")
    }

    // 让这个 defer 结构体的 deferproc 位置的调用重新返回
    // 这次将返回值修改为 1
    gp.sched.sp = sp
    gp.sched.pc = pc
    gp.sched.lr = 0
    gp.sched.ret = 1 // ------> 没有调用 deferproc，但直接修改了返回值，所以跳转到 deferproc 的下一条指令位置，且设置了 1，假装作为 deferproc 的返回值
    gogo(&gp.sched)
}
```

这里比较 trick，实际上 recovery 中传入的 pc 寄存器的值是 call deferproc 的下一条汇编指令的地址:

```go
	0x0034 00052 (x.go:4)	CALL	runtime.deferproc(SB)
	0x0039 00057 (x.go:4)	TESTL	AX, AX -----> 传入到 deferproc 中的 pc 寄存器指向的位置
	0x003b 00059 (x.go:4)	JNE	135
```

和刚注册 defer 结构体链表时的情况不同，panic 时，我们没有调用 deferproc，而是直接跳到了 deferproc 的下一条指令的地址上，并且检查 AX 的值，这里已经被改成 1 了。

所以会直接跳转到函数最后对应该 deferproc 的 deferreturn 位置去。

最后确认一次，runtime.gogo 会把 sched.ret 搬到 AX 寄存器:

```go
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX
	MOVQ	0(DX), CX		// make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)
	MOVQ	gobuf_sp(BX), SP	// restore SP
	MOVQ	gobuf_ret(BX), AX  // ----------> 重点在这里
	MOVQ	gobuf_ctxt(BX), DX
```

这样所有流程就都打通了。
