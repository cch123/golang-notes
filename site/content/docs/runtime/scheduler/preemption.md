---
title: 抢占
weight: 2
---

# 抢占

从 Go 1.14 开始，通过使用信号，Go 语言实现了调度和 GC 过程中的真“抢占“。

抢占流程由抢占的发起方向被抢占线程发送 SIGURG 信号。

当被抢占线程收到信号后，进入 SIGURG 的处理流程，将 asyncPreempt 的调用强制插入到用户当前执行的代码位置。

本节会对该过程进行详尽分析。

## 抢占发起的时机

抢占会在下列时机发生：

* STW 期间
* 在 P 上执行 safe point 函数期间
* sysmon 后台监控期间
* gc pacer 分配新的 dedicated worker 期间
* panic 崩溃期间

{{< rawhtml >}}
<img src='https://g.gravizo.com/svg?
 digraph G {
   rankdir="LR";
   node [shape="record"];
   edge [style="dashed"];
   startpanic_m -> freezetheworld -> preemptall;
   "debug.WriteHeapDump" -> stopTheWorld;
   "runtime.Stack" -> stopTheWorld;
   "runtime.ReadMemstats" -> stopTheWorld;
   "runtime.GoroutineProfile" -> stopTheWorld;
   "runtime.StartTrace" -> stopTheWorldGC;
   "runtime.StopTrace" -> stopTheWorldGC;
   "debug.GOMAXPROCS" -> stopTheWorldGC;
   "syscall.AllThreadsSyscall" -> stopTheWorldGC;
   stopTheWorldGC -> stopTheWorld;
   stopTheWorld -> stopTheWorldWithSema;
   gcStart -> stopTheWorldWithSema;
   gcMarkDone -> stopTheWorldWithSema;
   stopTheWorldWithSema -> preemptall;
   gcMarkDone -> forEachP;
   gcMarkTermination -> forEachP;
   forEachP -> preemptall;
   preemptall -> preemptone;
   sysmon -> retake;
   retake -> preemptone;
   enlistWorker -> preemptone;
 }
'/>
{{< /rawhtml >}}

除了栈扫描，所有触发抢占最终都会去执行 preemptone 函数。栈扫描流程比较特殊：

{{< rawhtml >}}
<img src='https://g.gravizo.com/svg?
 digraph G {
   rankdir="LR";
   ranksep = 1;
   node [shape="record", width=0.4];
   edge [style="dashed"];
   markroot -> suspendG -> "scan stack" -> resumeG;
 }
'/>
{{< /rawhtml >}}

从这些流程里，我们挑出三个来一探究竟。

### STW 抢占

![GC 的多个阶段](/images/runtime/memory/gcphases.jpg)

上图是现在 Go 语言的 GC 流程图，在两个 STW 阶段都需要将正在执行的线程上的 running 状态的 goroutine 停下来。

```go
func stopTheWorldWithSema() {
	.....
	preemptall()
	.....
	// 等待剩余的 P 主动停下
	if wait {
		for {
			// wait for 100us, then try to re-preempt in case of any races
			// 等待 100us，然后重新尝试抢占
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}

```


### GC 栈扫描

goroutine 的栈是 GC 扫描期间的根，所有 markroot 中需要将用户的 goroutine 停下来，主要是 running 状态：

```go
func markroot(gcw *gcWork, i uint32) {
	// Note: if you add a case here, please also update heapdump.go:dumproots.
	switch {
	......
	default:
		// the rest is scanning goroutine stacks
		var gp *g
		......

		// scanstack must be done on the system stack in case
		// we're trying to scan our own stack.
		systemstack(func() {
			stopped := suspendG(gp)
			scanstack(gp, gcw)
			resumeG(stopped)
		})
	}
}
```

suspendG 中会调用 preemptM -> signalM 对正在执行的 goroutine 所在的线程发送抢占信号。

### sysmon 后台监控

```go
func sysmon() {
	idle := 0 // how many cycles in succession we had not wokeup somebody
	for {
		......
		// retake P's blocked in syscalls
		// and preempt long running G's
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
	}
}
```

执行 syscall 太久的，需要将 P 从 M 上剥离；运行用户代码太久的，需要抢占停止该 goroutine 执行。这里我们只看抢占 goroutine 的部分：

```go
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
	......
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		s := _p_.status
		if s == _Prunning || s == _Psyscall {
			// Preempt G if it's running for too long.
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
				preemptone(_p_)
			}
		}
		......
	}
	......
}
```

## 协作式抢占原理

cooperative preemption 关键在于 cooperative，抢占的时机在各个版本实现差异不大，我们重点来看看这个协作过程。

### 函数头、函数尾插入的栈扩容检查

在 Go 语言中发生函数调用时，如果函数的 framesize > 0，说明在调用该函数时可能会发生 goroutine 的栈扩张，这时会在函数头、函数尾分别插入一段汇编码：

```go
package main

func main() {
	add(1, 2)
}

//go:noinline
func add(x, y int) (int, bool) {
	var z = x + y
	println(z)
	return x + y, true
}
```

add 函数在使用 go tool compile -S 后会生成下面的结果：

```go
// 这里的汇编代码使用 go1.14 生成
// 在 go1.17 之后，函数的调用规约发生变化
// 1.17 与以往版本头部的汇编代码也会有所不同，但逻辑保持一致
"".add STEXT size=103 args=0x20 locals=0x18
	0x0000 00000 (add.go:8)	TEXT	"".add(SB), ABIInternal, $24-32
	0x0000 00000 (add.go:8)	MOVQ	(TLS), CX
	0x0009 00009 (add.go:8)	CMPQ	SP, 16(CX)
	0x000d 00013 (add.go:8)	JLS	96
	...... func body
	0x005f 00095 (add.go:11)	RET
	0x0060 00096 (add.go:11)	NOP
	0x0060 00096 (add.go:8)	CALL	runtime.morestack_noctxt(SB)
	0x0065 00101 (add.go:8)	JMP	0
```

TLS 中存储的是 G 的指针，偏移 16 字节即是 G 的结构体中的 stackguard0。由于 goroutine 的栈也是从高地址向低地址增长，因此这里检查当前 SP < stackguard0 的话，说明需要对栈进行扩容了。

### morestack 中的调度逻辑

```go
// morestack_noctxt 是个简单的汇编方法
// 直接跳转到 morestack
TEXT runtime·morestack_noctxt(SB),NOSPLIT|NOFRAME,$0-0
	MOV	ZERO, CTXT
	JMP	runtime·morestack(SB)

TEXT runtime·morestack(SB),NOSPLIT,$0-0
	......
	// 前面会切换将执行现场保存到 goroutine 的 gobuf 中
	// 并将执行栈切换到 g0
	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	CALL	runtime·newstack(SB)
	......
	RET
```

morestack 会将 goroutine 的现场保存在当前 goroutine 的 gobuf 中，并将执行栈切换到 g0，然后在 g0 上执行 runtime.newstack。

在未实现信号抢占之前，用户的 g 到底啥时候能停下来，负责 GC 栈扫描的 goroutine 也不知道，所以 scanstack 也就只能设置一下 preemptscan 的标志位，最终栈扫描要 [newstack](https://github.com/golang/go/blob/2bc8d90fa21e9547aeb0f0ae775107dc8e05dc0a/src/runtime/stack.go#L917) 来配合，下面的 newstack 是 Go 1.13 版本的实现：

```go
func newstack() {
	thisg := getg()
	gp := thisg.m.curg
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt
	if preempt {
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if preempt {
		// 要和 scang 过程配合
		// 老版本的 newstack 和 gc scan 过程是有较重的耦合的
		casgstatus(gp, _Grunning, _Gwaiting)
		if gp.preemptscan {
			for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
			}
			if !gp.gcscandone {
				gcw := &gp.m.p.ptr().gcw
				// 注意这里，偶合了 GC 的 scanstack 逻辑代码
				scanstack(gp, gcw)
				gp.gcscandone = true
			}
			gp.preemptscan = false
			gp.preempt = false
			casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
			casgstatus(gp, _Gwaiting, _Grunning)
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}

		casgstatus(gp, _Gwaiting, _Grunning)
		gopreempt_m(gp) // never return
	}
	......
}

```

抢占成功后，当前的 goroutine 会被放在全局队列中：

```go
func gopreempt_m(gp *g) {
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	status := readgstatus(gp)
	......

	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
	globrunqput(gp) // 将当前 goroutine 放进全局队列
	unlock(&sched.lock)

	schedule() // 当前线程重新进入调度循环
}
```

### 信号式抢占实现后的 newstack

在实现了信号式抢占之后，对于用户的 goroutine 何时中止有了一些预期，所以 newstack 就不需要耦合 scanstack 的逻辑了，新版的 [newstack](https://github.com/golang/go/blob/c3b47cb598e1ecdbbec110325d9d1979553351fc/src/runtime/stack.go#L948) 实现如下:

```go
func newstack() {
	thisg := getg()
	gp := thisg.m.curg
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	if preempt {
		if !canPreemptM(thisg.m) {
			// 让 goroutine 继续执行
			// 下次再抢占它
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if preempt {
		// 当 GC 需要发起 goroutine 的栈扫描时
		// 会设置这个 preemptStop 为 true
		// 这时候需要 goroutine 自己去 gopark
		if gp.preemptStop {
			preemptPark(gp) // never returns
		}

		// 除了 GC 栈扫描以外的其它抢占场景走这个分支
		// 看起来就像 goroutine 自己调用了 runtime.Gosched 一样
		gopreempt_m(gp) // never return
	}
	...... 后面就是正常的栈扩展逻辑了
}
```

newstack 中会使用 [canPreemptM](https://github.com/golang/go/blob/287c5e8066396e953254d7980a80ec082edf11bd/src/runtime/preempt.go#L287)判断哪些场景适合抢占，哪些不适合。如果当前 goroutine 正在执行(即 status == running)，并且满足下列任意其一：

* 持有锁(主要是写锁，读锁其实判断不出来)；
* 正在进行内存分配
* preemptoff 非空

便不应该进行抢占，会在下一次进入到 newstack 时再进行判断。

## 非协作式抢占

非协作式抢占，就是通过信号处理来实现的。所以我们只要关注 SIGURG 的处理流程即可。

### 信号处理初始化

当 m0(即程序启动时的第一个线程)初始化时，会进行信号处理的初始化工作：

```go
// mstartm0 implements part of mstart1 that only runs on the m0.
func mstartm0() {
	initsig(false)
}

// Initialize signals.
func initsig(preinit bool) {
	for i := uint32(0); i < _NSIG; i++ {
		setsig(i, funcPC(sighandler))
	}
}

var sigtable = [...]sigTabT{
	......
	/* 23 */ {_SigNotify + _SigIgn, "SIGURG: urgent condition on socket"},
	......
}

```

最后都是执行 sigaction：

```go
TEXT runtime·rt_sigaction(SB),NOSPLIT,$0-36
	MOVQ	sig+0(FP), DI
	MOVQ	new+8(FP), SI
	MOVQ	old+16(FP), DX
	MOVQ	size+24(FP), R10
	MOVL	$SYS_rt_sigaction, AX
	SYSCALL
	MOVL	AX, ret+32(FP)
	RET
```

与一般的 syscall 区别不大。

信号处理初始化的流程比较简单，就是给所有已知的需要处理的信号绑上 sighandler。

### 发送信号

```go
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	gp := mp.curg
	gp.preempt = true
	gp.stackguard0 = stackPreempt

	// 向该线程发送 SIGURG 信号
	if preemptMSupported && debug.asyncpreemptoff == 0 {
		_p_.preempt = true
		preemptM(mp)
	}

	return true
}
```

preemptM 的流程较为线性：

```go
func preemptM(mp *m) {
	if atomic.Cas(&mp.signalPending, 0, 1) {
		signalM(mp, sigPreempt)
	}
}

func signalM(mp *m, sig int) {
	tgkill(getpid(), int(mp.procid), sig)
}
```

最后使用 tgkill 这个 syscall 将信号发送给指定 id 的线程：

```go
TEXT ·tgkill(SB),NOSPLIT,$0
	MOVQ	tgid+0(FP), DI
	MOVQ	tid+8(FP), SI
	MOVQ	sig+16(FP), DX
	MOVL	$SYS_tgkill, AX
	SYSCALL
	RET
```

### 接收信号后的处理

当线程 m 接收到信号后，会从用户栈 g 切换到 gsignal 执行信号处理逻辑，即 sighandler 流程：

```go
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
	_g_ := getg()
	c := &sigctxt{info, ctxt}

	......
	if sig == sigPreempt && debug.asyncpreemptoff == 0 {
		doSigPreempt(gp, c)
	}
	......
}
```

如果收到的是抢占信号，那么执行 doSigPreempt 逻辑：

```go
func doSigPreempt(gp *g, ctxt *sigctxt) {
	// 检查当前 G 被抢占是否安全
	if wantAsyncPreempt(gp) {
		if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
			// Adjust the PC and inject a call to asyncPreempt.
			ctxt.pushCall(funcPC(asyncPreempt), newpc)
		}
	}
	......
}
```

isAsyncSafePoint 中会把一些不应该抢占的场景过滤掉，具体包括：

* 当前代码在汇编编写的函数中执行
* 代码在 runtime，runtime/internal 或者 reflect 包中执行

doSigPreempt 代码中的 pushCall 是关键步骤：

```go
func (c *sigctxt) pushCall(targetPC, resumePC uintptr) {
	// Make it look like we called target at resumePC.
	sp := uintptr(c.rsp())
	sp -= sys.PtrSize
	*(*uintptr)(unsafe.Pointer(sp)) = resumePC
	c.set_rsp(uint64(sp))
	c.set_rip(uint64(targetPC))
}
```

pushCall 相当于将用户将要执行的下一条代码的地址直接 push 到栈上，并 jmp 到指定的 target 地址去执行代码：

{{< columns >}}

before

```shell
-----                     PC = 0x123
local var 1
-----
local var 2
----- <---- SP
```

<--->

after

```shell
-----                  PC = targetPC
local var 1
-----
local var 2
-----
prev PC = 0x123
----- <---- SP
```

{{</ columns >}}

{{< columns>}}
{{< /columns >}}

这里的 target 就是 asyncPreempt。

### asyncPreempt 执行流程分析

asyncPreempt 分为上半部分和下半部分，中间被 asyncPreempt2 隔开。上半部分负责将 goroutine 当前执行现场的所有寄存器都保存到当前的运行栈上。

下半部分负责在 asyncPreempt2 返回后将这些现场恢复出来。

```go
TEXT ·asyncPreempt<ABIInternal>(SB),NOSPLIT|NOFRAME,$0-0
	PUSHQ BP
	MOVQ SP, BP
	...... 保存现场 1
	MOVQ AX, 0(SP)
	MOVQ CX, 8(SP)
	MOVQ DX, 16(SP)
	MOVQ BX, 24(SP)
	MOVQ SI, 32(SP)
	...... 保存现场 2
	MOVQ R15, 104(SP)
	MOVUPS X0, 112(SP)
	MOVUPS X1, 128(SP)
	......
	MOVUPS X15, 352(SP)

	CALL ·asyncPreempt2(SB)

	MOVUPS 352(SP), X15
	...... 恢复现场 2
	MOVUPS 112(SP), X0
	MOVQ 104(SP), R15
	...... 恢复现场 1
	MOVQ 8(SP), CX
	MOVQ 0(SP), AX
	......
	RET

```

asyncPreempt2 中有两个分支：

```go
func asyncPreempt2() {
	gp := getg()
	gp.asyncSafePoint = true
	if gp.preemptStop { // 这个 preemptStop 是在 GC 的栈扫描中才会设置为 true
		mcall(preemptPark)
	} else { // 除了栈扫描，其它抢占全部走这条分支
		mcall(gopreempt_m)
	}
	gp.asyncSafePoint = false
}
```

GC 栈扫描走 if 分支，除栈扫描以外所有情况均走 else 分支。

**栈扫描抢占流程**

suspendG -> preemptM -> signalM 发信号。

sighandler -> asyncPreempt -> 保存执行现场 -> asyncPreempt2 -> **preemptPark**

preemptPark 和 gopark 类似，挂起当前正在执行的 goroutine，该 goroutine 之前绑定的线程就可以继续执行调度循环了。

scanstack 执行完之后：

resumeG -> ready -> runqput 会让之前被停下来的 goroutine 进当前 P 的队列或全局队列。

**其它流程**

preemptone -> preemptM - signalM 发信号。

sighandler -> asyncPreempt -> 保存执行现场 -> asyncPreempt2 -> **gopreempt_m**

gopreempt_m 会直接将被抢占的 goroutine 放进全局队列。

无论是栈扫描流程还是其它流程，当 goroutine 程序被调度到时，都是从汇编中的 `CALL ·asyncPreempt2(SB)` 的下一条指令开始执行的，即 asyncPreempt 汇编函数的下半部分。

这部分会将之前 goroutine 的现场完全恢复，就和抢占从来没有发生过一样。

## 动画演示

下面的动画是可以点的哦~

**sighandler 收到抢占信号，保存 PC，并将 PC 指向 asyncPreempt 的过程动画：**

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="100%" height="464" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FJYM6TcdzBx7WtanhcJX0rP%2Fbootstrap-Copy%3Fpage-id%3D5185%253A2%26node-id%3D5185%253A3%26viewport%3D446%252C382%252C0.037524838000535965%26scaling%3Dscale-down-width" allowfullscreen></iframe>
{{</rawhtml>}}


**GC 栈扫描时的抢占和恢复过程：**

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="100%" height="464" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FJYM6TcdzBx7WtanhcJX0rP%2Fbootstrap-Copy%3Fpage-id%3D5187%253A0%26node-id%3D5187%253A1%26viewport%3D446%252C382%252C0.028840554878115654%26scaling%3Dscale-down-width" allowfullscreen></iframe>
{{</rawhtml>}}


**除了栈扫描之外的抢占和恢复过程：**

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="100%" height="464" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FJYM6TcdzBx7WtanhcJX0rP%2Fbootstrap-Copy%3Fpage-id%3D5187%253A387%26node-id%3D5187%253A388%26viewport%3D446%252C382%252C0.037981197237968445%26scaling%3Dscale-down-width" allowfullscreen></iframe>
{{</rawhtml>}}
