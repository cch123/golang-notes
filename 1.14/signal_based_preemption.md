# 1.14 scheduler

## 信号概念

信号是一种在有事件发生时，发送给进程的通知。也可以认为信号是进程之间的一种通信机制。

信号也被称为软件中断。从中断用户控制流来说，信号和硬件中断是类似的；在大多场景下，信号何时到达进程是无法预测的。

信号可以由内核发给用户进程，可以用户进程发给自己，也可以用户进程发给其它的用户进程。

在进程内部，信号还可以发给某个具体的线程。在 Go 语言中，抢占的信号就是发给指定的线程的。

## 信号是易失的么

有一种传言认为信号是易失的，但实际上在早期的 unix 实现中才是易失的。这里的易失说的是信号在发送给进程的过程中完全丢失。由 POSIX.1-1990 标准定义之后实现的信号基本都是可靠信号了。

但不易失不代表你向同一个进程重复发送相同的信号 100 次，进程就能收到 100 次这个信号，这是怎么回事？

## 不丢失不代表每次收到的信号都会被触发

当进程不能处理某个具体的信号时，内核会自动将该信号加入到 signal mask 中，以 block 住后面来的相同信号。

比如进程正在执行 SIGINT 的 sighandler，还没执行完。这时候 SIGINT 是在 sigmask 中的，如果又来了一个 SIGINT，内核会发现 sigmask 里有 SIGINT，便会把这个 SIGINT 放进 pending signals 的集合里。注意这里是个集合，所以同一个信号在 block 期间如果多次触发，那在 pending signals 的集合里会被去重，unblock 了以后，只会触发一次 sighandler。

TODO，画个图

实时信号比较特殊，不过不在我们讨论范围内，感兴趣的可以去看 TLPI。

## 信号处理

![](../images/signal.png)

信号处理涉及几个 syscall:

### tgkill

向某个进程的某个线程发信号。

> tgkill() sends the signal sig to the thread with the thread ID tid in the thread group tgid. (By contrast, kill(2) can be used to send a signal only to a process (i.e., thread group) as a whole, and the signal will be delivered to an arbitrary thread within that process.)

> tkill() is an obsolete predecessor to tgkill(). It allows only the target thread ID to be specified, which may result in the wrong thread being signaled if a thread terminates and its thread ID is recycled. Avoid using this system call.

tgkill 是 thread group kill 的缩写，可以给某个进程的某个线程发信号，在 tgkill 之前有一个不带进程 id 的 tkill，不过这个 syscall 可能会杀掉错误的进程，比如正好在发送之前，老进程被销毁，而新进程使用了同样的线程 id。也就是说现在使用的话，都是 tgkill 这个 syscall 了。

与 tgkill 相比，kill 则只能给进程整体发信号，信号可能被进程内的任意线程处理。平常使用比较多的 kill -9 就是使用的 kill 这个 syscall，可以用 strace 来确认:

sudo strace kill -9 pid

```
...
kill(444444, SIGKILL)                   = -1 ESRCH (No such process)
...
```

### signal

最简单的修改信号行为的 syscall 就是 signal，因为在 Go 里写裸的 signal handler 不太容易，这里用 c 来演示:

```cpp
// https://www.geeksforgeeks.org/signals-c-language/
#include<stdio.h> 
#include<signal.h> 

void handle_sigint(int sig) 
{ 
	printf("Caught signal %d\n", sig); 
} 

int main() 
{ 
	signal(SIGINT, handle_sigint); 
	while (1) ; 
	return 0; 
} 
```

在命令行中执行该程序，每次按下 ctrl+c 都会看到有 `^CCaught signal 2` 的输出。但是 signal 这个 syscall 实在是太简单了，信号到来的时候如果有更多的可配置项和程序上下文，就可以让我们做更复杂的操作。所以才会有 sigaction 这样的稍微复杂一些的信号处理 syscall。

### sigaction

用 sigaction 可以写出和用 signal 完全一致的效果:

```cpp
#include<stdio.h> 
#include<signal.h> 

void handle_sigint(int sig) 
{ 
	printf("Caught signal %d\n", sig); 
} 

int main() 
{ 
	struct sigaction act;
	act.sa_handler = handle_sigint;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	sigaction(SIGINT, &act, NULL); 
	while (1) ; 
	return 0; 
} 
```

这段代码和上面的效果是一样的，每次按 ctrl+c 都会输出 `^CCaught signal 2`。

sigaction 的 sa_handler 和 sa_sigaction 共同组成一个 C 语言的 union 结构(在 Mac OS 上是这样，暂时没 linux 环境，凑和看吧):

```cpp
/* union for signal handlers */
union __sigaction_u {
	void    (*__sa_handler)(int);
	void    (*__sa_sigaction)(int, struct __siginfo *,
	    void *);
};
```

也就是说我们想简单处理，就用 sa_handler，想做复杂些的功能就用 sa_sigaction。

sigaction 的三个参数:

* int 就是 signal number
* siginfo 中包含一些信号的上下文信息，比如发送信号的进程 id，出现信号的指令地址等等。
* void * 是一个很特殊的参数

### sigaltstack

修改信号执行时所用的函数栈。

## 简单的信号处理函数

简单起见，这里我们使用 C 来做演示:

## 执行 non-local goto

### goto 的限制

goto 虽然看起来无所不能，但实际上高级语言的 goto 是跳不出函数作用域的，比如下面这样的代码就没法通过编译:

```go
TODO
```

在其它语言里也一样:

```c
TODO
```

### non-local goto

```
#include<stdio.h>
#define __USE_GNU
#include<signal.h>
#include<ucontext.h>

void myhandle(int mysignal, siginfo_t *si, void* arg)
{    
  ucontext_t *context = (ucontext_t *)arg;
  printf("Address from where crash happen is %x \n",context->uc_mcontext.gregs[REG_RIP]);
  context->uc_mcontext.gregs[REG_RIP] = context->uc_mcontext.gregs[REG_RIP] + 0x04 ;

}

int main(int argc, char *argv[])
{
  struct sigaction action;
  action.sa_sigaction = &myhandle;
  action.sa_flags = SA_SIGINFO;
  sigaction(11,&action,NULL);

  printf("Before segfault\n");

  int *a=NULL;
  int b;
  b =*a; // Here crash will hapen

  printf("I am still alive\n");

  return 0;
}
```

## Go 语言中的信号式抢占

有了上述的储备知识，我们在看 Go 的信号式抢占前，至少知道了以下几个知识点:

* 信号可以在任意位置打断用户代码
* 信号的处理函数可以自定义
* 信号的执行栈可以自定义
* 信号执行完毕之后默认返回用户代码的下一条汇编指令继续执行
* 可以通过修改 pc 寄存器的值，使信号处理完之后执行非本地的 goto，其实就是跳到任意的代码位置去

### SIGURG

为什么选用了 SIGURG。

## gsignal

gsignal 是一个特殊的 goroutine，类似 g0，每一个线程都有一个，创建的 m 的时候，就会创建好这个 gsignal，在 linux 中为 gsignal 分配 32KB 的内存:

newm -> allocm -> mcommoninit -> mpreinit -> malg(32 * 1024)

在线程处理信号时，会短暂地将栈从用户栈切换到 gsignal 的栈，执行 sighandler，执行完成之后，会重新切换回用户的栈继续执行用户逻辑。

## 流程概述

抢占流程主要有两个入口，GC 和 sysmon。

TODO，control flow pic show

### GC 抢占流程

markroot -> fetch g from allgs -> suspendG -> scan g stack -> resumeG

除了 running 以外，任何状态(如 dead，runnable，waiting)的 g 实际上都不是正在运行的 g，对于这些 g 来说，只要将其相应的字段打包返回就可以了。

running 状态的 g 正在系统线程上执行代码，是需要真正发送信号来抢占的。

### sysmon 抢占流程

preemptone -> asyncPreempt -> globalrunqput

### sighandler

```go
func sighandler(...)
	if sig == sigPreempt {
		doSigPreempt(gp, c)
    }
}

func doSigPreempt(gp *g, ctxt *sigctxt) {
	if wantAsyncPreempt(gp) && isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()) {
		ctxt.pushCall(funcPC(asyncPreempt))
    }
}
```

```go
func (c *sigctxt) pushCall(targetPC uintptr) {
	// Make it look like the signaled instruction called target.
	pc := uintptr(c.rip())
	sp := uintptr(c.rsp())
	sp -= sys.PtrSize
	*(*uintptr)(unsafe.Pointer(sp)) = pc
	c.set_rsp(uint64(sp))
	c.set_rip(uint64(targetPC))
}
```

pushCall 其实就是把用户代码中即将执行的下一条指令的地址(即 pc 寄存器的值)，保存在栈顶，然后 sp 寄存器下移(就是扩栈，栈从高地址向低地址增长)。

TODO，这里需要图

### 现场保存和恢复

在 sighandler 修改了信号返回时的 pc 寄存器，所以从 sighandler 返回之后，之前在 running 的 g 不是执行下一条指令，而是执行 pc 寄存器中保存的函数去了，这里就是 asyncPreempt。

asyncPreempt 是汇编实现的，分为三个部分:

1. 将当前 g 的所有寄存器均保存在 g 的栈上
2. 执行 asyncPreempt2
3. 恢复 g 的所有寄存器，继续执行用户代码

TODO，图

## 总结

TODO，总览图

## 参考资料

1. The Linux Programming Interface
2. [non-cooperative goroutine preemption](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)
3. https://stackoverflow.com/questions/34989829/linux-signal-handling-how-to-get-address-of-interrupted-instruction
