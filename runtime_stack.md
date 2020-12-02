# Stack Dump

> 引言：在工程中经常需要在异常发生的时候打印相应的日志，或利用成熟的第三方日志库，或自己进行简单的实现，在相应日志输出的时候，常常会有不同的级别，当遇到 error 或者 panic 之类的情况的时候，通常需要输出相应堆栈信息来辅助我们进行问题的定位以及解决，那么，关于堆栈这块的信息我们该怎么获取，获取的时候需要注意什么，以及堆栈的获取会对程序造成什么影响，会带来多大的性能损耗？日志库的使用或者进行相关配置我们该怎么选择？本章的目的就是深入一下 golang 中 stack dump 的细节

## STACK APIs
- debug.Stack
  - debug.PrintStack
- runtime.Stack
- runtime.Callers
  - runtime.Caller
  
## debug.Stack & debug.PrintStack
```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	fmt.Println(debug.Stack())
}
```
runtime/debug/stack.go
```go
func Stack() []byte {
	buf := make([]byte, 1024)
	for {
       /* 
        * 可以看到实质上 debug.Stack 调用的就是 runtime.Stack 方法
        * 并且会不断扩大 buf 大小直到读取完所有 stack 信息
        */
		n := runtime.Stack(buf, false)
		if n < len(buf) {
			return buf[:n]
		}
		buf = make([]byte, 2*len(buf))
	}
}
```
而 runtime.Stack 方法入参为 `buf []byte` 以及 `all bool` 两个参数
- `buf []byte` 是用来接收 stack dump 信息的
- `all bool` 参数则是用来决定是否获取所有协程的堆栈

`buf` 参数比较好理解，而 `all` 参数如果设置为获取所有协程信息，则会触发 `STW` 操作，这是一个代价十分巨大的操作，整个 runtime 会被暂停以获取所有协程的堆栈信息而后再进行恢复，

runtime/mprof.go
```go
func Stack(buf []byte, all bool) int {
	if all {
		stopTheWorld("stack trace")
	}

	n := 0
	if len(buf) > 0 {
		gp := getg()
		sp := getcallersp()
		pc := getcallerpc()
		systemstack(func() {
			g0 := getg()
			// Force traceback=1 to override GOTRACEBACK setting,
			// so that Stack's results are consistent.
			// GOTRACEBACK is only about crash dumps.
			g0.m.traceback = 1
			g0.writebuf = buf[0:0:len(buf)]
			goroutineheader(gp)
			traceback(pc, sp, 0, gp)
			if all {
				tracebackothers(gp)
			}
			g0.m.traceback = 0
			n = len(g0.writebuf)
			g0.writebuf = nil
		})
	}

	if all {
		startTheWorld()
	}
	return n
}
```
分析下具体调用过程，首先判断是否进行 `STW`，而后通过`getg()` 获取当前 goroutine
- `getcallersp()`: 汇编实现，获取当前调用 stack pointer (SP)
- `getcallerpc()`: 汇编实现，获取当前调用方的程序计数器 program counter (PC)
- `systemstack`: 切换到系统栈执行传入函数（如果当前就在系统栈则直接执行），可以看到在传入的函数中再次调用 getg() 获取到了 g0 （主协程）。并且将传入的 buf 放置在了 g0 的 writebuf 上（slice 底层 data 地址不变）
- `goroutineheader` 获取 gp 协程头部信息
  
runtime/traceback.go
```go
func goroutineheader(gp *g) {
	gpstatus := readgstatus(gp)

	isScan := gpstatus&_Gscan != 0
	gpstatus &^= _Gscan // drop the scan bit

	// Basic string status
	var status string
	if 0 <= gpstatus && gpstatus < uint32(len(gStatusStrings)) {
		status = gStatusStrings[gpstatus]
	} else {
		status = "???"
	}

	// Override.
	if gpstatus == _Gwaiting && gp.waitreason != waitReasonZero {
		status = gp.waitreason.String()
	}

	// approx time the G is blocked, in minutes
	var waitfor int64
	if (gpstatus == _Gwaiting || gpstatus == _Gsyscall) && gp.waitsince != 0 {
		waitfor = (nanotime() - gp.waitsince) / 60e9
	}
	print("goroutine ", gp.goid, " [", status)
	if isScan {
		print(" (scan)")
	}
	if waitfor >= 1 {
		print(", ", waitfor, " minutes")
	}
	if gp.lockedm != 0 {
		print(", locked to thread")
	}
	print("]:\n")
}
```
函数内部直接调用内置函数 `print` 将信息输入到刚才设置的 `writebuf` 中，`print` 内置函数具体实现可以查看 `runtime/print.go`，内部是将 `print` 的值通过 copy 复制到当前协程的 `writebuf` 上

runtime/print.go
```go
func gwrite(b []byte) {
	if len(b) == 0 {
		return
	}
	recordForPanic(b)
	gp := getg()
	// Don't use the writebuf if gp.m is dying. We want anything
	// written through gwrite to appear in the terminal rather
	// than be written to in some buffer, if we're in a panicking state.
	// Note that we can't just clear writebuf in the gp.m.dying case
	// because a panic isn't allowed to have any write barriers.
	if gp == nil || gp.writebuf == nil || gp.m.dying > 0 {
		writeErr(b)
		return
	}

	n := copy(gp.writebuf[len(gp.writebuf):cap(gp.writebuf)], b)
	gp.writebuf = gp.writebuf[:len(gp.writebuf)+n]
}
```
继续回到之前的分析
- `traceback`

```go
func traceback1(pc, sp, lr uintptr, gp *g, flags uint) {
	// If the goroutine is in cgo, and we have a cgo traceback, print that.
	if iscgo && gp.m != nil && gp.m.ncgo > 0 && gp.syscallsp != 0 && gp.m.cgoCallers != nil && gp.m.cgoCallers[0] != 0 {
		// Lock cgoCallers so that a signal handler won't
		// change it, copy the array, reset it, unlock it.
		// We are locked to the thread and are not running
		// concurrently with a signal handler.
		// We just have to stop a signal handler from interrupting
		// in the middle of our copy.
		atomic.Store(&gp.m.cgoCallersUse, 1)
		cgoCallers := *gp.m.cgoCallers
		gp.m.cgoCallers[0] = 0
		atomic.Store(&gp.m.cgoCallersUse, 0)

		printCgoTraceback(&cgoCallers)
	}

	var n int
	if readgstatus(gp)&^_Gscan == _Gsyscall {
		// Override registers if blocked in system call.
		pc = gp.syscallpc
		sp = gp.syscallsp
		flags &^= _TraceTrap
	}
	// Print traceback. By default, omits runtime frames.
	// If that means we print nothing at all, repeat forcing all frames printed.
	n = gentraceback(pc, sp, lr, gp, 0, nil, _TracebackMaxFrames, nil, nil, flags)
	if n == 0 && (flags&_TraceRuntimeFrames) == 0 {
		n = gentraceback(pc, sp, lr, gp, 0, nil, _TracebackMaxFrames, nil, nil, flags|_TraceRuntimeFrames)
	}
	if n == _TracebackMaxFrames {
		print("...additional frames elided...\n")
	}
	printcreatedby(gp)

	if gp.ancestors == nil {
		return
	}
	for _, ancestor := range *gp.ancestors {
		printAncestorTraceback(ancestor)
	}
}
```
函数首先判断了 `cgo` 相关信息，如果是处于 `cgo` 的模式中，那么输出一下 `cgo` 的堆栈信息（如果有的话）

而后调用关键的 `gentraceback` 函数
```go
func gentraceback(pc0, sp0, lr0 uintptr, gp *g, skip int, pcbuf *uintptr, max int, callback func(*stkframe, unsafe.Pointer) bool, v unsafe.Pointer, flags uint) int
```
可以看到，通过 `debug.Stack` 函数调用时，max 参数传入的值为 `_TracebackMaxFrames`
```go
// The maximum number of frames we print for a traceback
const _TracebackMaxFrames = 100
```
最大帧数为固定值 100，意味着在 stack dump 时需要追溯至多 100 个栈帧的信息，O(N) 复杂度。

## runtime.Caller & runtime.Callers
runtime/extern.go
```go
// Caller reports file and line number information about function invocations on
// the calling goroutine's stack. The argument skip is the number of stack frames
// to ascend, with 0 identifying the caller of Caller.  (For historical reasons the
// meaning of skip differs between Caller and Callers.) The return values report the
// program counter, file name, and line number within the file of the corresponding
// call. The boolean ok is false if it was not possible to recover the information.
func Caller(skip int) (pc uintptr, file string, line int, ok bool) {
	rpc := make([]uintptr, 1)
	n := callers(skip+1, rpc[:])
	if n < 1 {
		return
	}
	frame, _ := CallersFrames(rpc).Next()
	return frame.PC, frame.File, frame.Line, frame.PC != 0
}

// Callers fills the slice pc with the return program counters of function invocations
// on the calling goroutine's stack. The argument skip is the number of stack frames
// to skip before recording in pc, with 0 identifying the frame for Callers itself and
// 1 identifying the caller of Callers.
// It returns the number of entries written to pc.
//
// To translate these PCs into symbolic information such as function
// names and line numbers, use CallersFrames. CallersFrames accounts
// for inlined functions and adjusts the return program counters into
// call program counters. Iterating over the returned slice of PCs
// directly is discouraged, as is using FuncForPC on any of the
// returned PCs, since these cannot account for inlining or return
// program counter adjustment.
func Callers(skip int, pc []uintptr) int {
	// runtime.callers uses pc.array==nil as a signal
	// to print a stack trace. Pick off 0-length pc here
	// so that we don't let a nil pc slice get to it.
	if len(pc) == 0 {
		return 0
	}
	return callers(skip, pc)
}
```
以上两个函数最终调用的都是 `callers` 函数
```go
func callers(skip int, pcbuf []uintptr) int {
	sp := getcallersp()
	pc := getcallerpc()
	gp := getg()
	var n int
	systemstack(func() {
		n = gentraceback(pc, sp, 0, gp, skip, &pcbuf[0], len(pcbuf), nil, nil, 0)
	})
	return n
}
```
`callers` 函数大部分逻辑和之前的 `runtime.Stack`函数中 `traceback` 大同小异，唯一不同的就是
`gentraceback` 调用的入参 `max` 参数是人为设置的，并且仅针对当前 `goroutine` 进行
## 总结
- stack dump 操作是否会有性能损耗（是），损耗在哪儿
  - 与调用方式有关，如果是通过类似 `runtime.Stack` 方法打印所有堆栈信息的， 会触发 `STW` 操作，是一个代价比较大的操作
  - 与需要追溯的栈帧数量有关
  - 会触发协程的切换
  - 更细致化一些例如 `stack dump` 出的信息会进行 `copy` 操作等等

- 如何更优雅地获取 `stack dump` 信息？
  - 以 `runtime.Callers` 与 `runtime.CallersFrames` 相结合代替 `runtime.Stack` 、`debug.Stack`，避免使用默认的 `stack dump` 配置，根据自己的业务情况选择合适的栈帧数量（如不同级别的日志打印不同数量的栈帧信息），知名的开源日志库 `github.com/uber-go/zap` 正是使用这种思路
  
----
### 参考：
[zap stacktrace 实现](https://github.com/uber-go/zap/blob/v1.16.0/stacktrace.go)