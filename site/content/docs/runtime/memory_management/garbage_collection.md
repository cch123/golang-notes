---
title: 垃圾回收
weight: 10
---

# 垃圾回收(WIP)

基于 Go 1.17。

![GC 的多个阶段](/images/runtime/memory/gcphases.jpg)

## 三色抽象

在 Go 的代码中并无直接提示对象颜色的代码，对象的颜色主要由：

* 对象对应的 gcmarkbit 位是否为 1
* 对象的子对象是否已入队完成，若已完成，对象本身应该已经在队列外了

这两个状态来决定，三种颜色分别为：

* 黑色：对象的 gcmarkbit 为 1，且对象已从队列中弹出
* 灰色：对象的 gcmarkbit 为 1，其子对象未被处理完成，对象本身还在队列中
* 白色：对象的 gcmarkbit 为 0，还未被标记流程所处理

## GC 触发

当前 GC 有三个触发点：

* runtime.GC
* forcegchelper
* heap trigger

## 并发标记流程


{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="540" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D278%253A33%26node-id%3D4442%253A24%26viewport%3D241%252C48%252C0.21%26scaling%3Dcontain%26starting-point-node-id%3D4442%253A24" allowfullscreen></iframe>
{{</rawhtml>}}

### 关键组件及启动流程

worker 的三种模式

* 全职模式：gcMarkWorkerDedicatedMode
* 比例模式：gcMarkWorkerFractionalMode
* 兼职模式：gcMarkWorkerIdleMode


### gc roots

垃圾回收的标记流程是将存活对象对应的 bit 位置为 1，堆上存活对象在内存中会形成森林结构，标记开始之前需要先将所有的根确定下来。

根对象包括四个来源：

* bss 段
* data 段
* goroutine 栈
* finalizer 关联的 special 类对象

### gcDrain

gcDrain 是标记的核心流程

#### markroot

根标记的流程很简单，就是根据 gcMarkrootPrepare 中计算出的索引值，遍历使用的根，执行 scanblock。

这些全局变量、goroutine 栈变量被扫描后，会被推到 gcw 队列中，成为灰色对象。

#### 标记过程中的队列 gcw && wbBuf && work.full

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="540" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D201%253A808%26node-id%3D212%253A20%26viewport%3D241%252C48%252C0.13%26scaling%3Dcontain%26starting-point-node-id%3D212%253A20" allowfullscreen></iframe>
{{</rawhtml>}}

#### 排空本地 gcw 和全局 work.full

#### 标记终止流程

## mutator 与 marker 并发执行时的问题

### 对象丢失问题

GC 标记过程与 mutator 是并发执行的，所以在标记过程中，堆上对象的引用关系也会被动态修改，这时候可能有下面这种情况：

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D16%253A150%26node-id%3D16%253A151%26viewport%3D241%252C48%252C0.17%26scaling%3Dcontain%26starting-point-node-id%3D16%253A151" allowfullscreen></iframe>
{{</rawhtml>}}

丢失的对象会被认为是垃圾而被回收掉，这样在 mutator 后续访问该对象时便会发生内存错误。为了解决这个问题，mutator 在 GC 标记阶段需要打开 write barrier。所谓的 write barrier，就是在堆上指针发生修改前，插入一小段代码：

![write barrier demo](/images/runtime/memory/write_barrier_demo.jpg)

每次修改堆上指针都会判断 runtime.writeBarrier.enabled 是否为 true，如果为 true，那么在修改指针前需要调用 runtime.gcWriteBarrier。

Go 语言使用的 gc write barrier 是插入和删除的混合屏障，我们先来看看插入和删除屏障是什么。

#### dijistra 插入屏障

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="540" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D201%253A0%26node-id%3D201%253A1%26viewport%3D241%252C48%252C0.03%26scaling%3Dcontain%26starting-point-node-id%3D201%253A1" allowfullscreen></iframe>
{{</rawhtml>}}

#### yuasa 删除屏障

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="540" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FtSl3CoSWKitJtvIhqLd8Ek%2Fmemory-management-%2526%2526-garbage-collection%3Fpage-id%3D201%253A293%26node-id%3D201%253A294%26viewport%3D241%252C48%252C0.03%26scaling%3Dcontain%26starting-point-node-id%3D201%253A294" allowfullscreen></iframe>
{{</rawhtml>}}

#### Go 语言使用的混合屏障

runtime.gcWriteBarrier 是汇编函数，可以看到会将指针在修改前指向的值，和修改后指向的值都 push 到 wbBuf 中。

如果 wbBuf 满，那么就会将其 push 到 gcw 中，gcw 满了会 push 到全局的 work.full 中。

```go
TEXT runtime·gcWriteBarrier<ABIInternal>(SB),NOSPLIT,$112
	......
	MOVQ	(p_wbBuf+wbBuf_next)(R13), R12
	// Increment wbBuf.next position.
	LEAQ	16(R12), R12
	MOVQ	R12, (p_wbBuf+wbBuf_next)(R13)
	CMPQ	R12, (p_wbBuf+wbBuf_end)(R13)
	// Record the write.
	MOVQ	AX, -16(R12)	// Record value
	MOVQ	(DI), R13
	MOVQ	R13, -8(R12)	// Record *slot
	// Is the buffer full? (flags set in CMPQ above)
	JEQ	flush
ret:
	MOVQ	96(SP), R12
	MOVQ	104(SP), R13
	// Do the write.
	MOVQ	AX, (DI)
	RET

flush:
	......
	CALL	runtime·wbBufFlush(SB)
	......
	JMP	ret
	......
```

## 清扫流程 sweep

标记完成后，在 gcMarkTermination 中调用 gcSweep 会唤醒后台清扫 goroutine。该 goroutine 循环遍历所有 mspan，sweepone -> sweep 的主要操作为：

* mspan.allocBits = mspan.gcMarkBits
* mspan.gcMarkBits clear

清扫完成后会有三种情况：
* 该 mspan 全空了，那么调用 freeSpan 释放该 mspan 使其回归 arena，等待 scavenge 最终将这些 page 归还给操作系统
* 尽管清扫了，但该 mspan 还是满的，那么将该 mspan 从 full 的 Unswept 链表移动到 full 的 Swept 部分
* 清扫后 mspan 中出现了空槽，那么将该 mspan 从 full/partial 的 Unswept 链表移动到 partial 的 Swept 部分

### 协助清扫

TODO

## 归还内存流程 scavenge

bgscavenge -> pageAlloc.scavenge -> pageAlloc.scavengeOne -> pageAlloc.scavengeRangeLocked -> sysUnused -> madvise


### GOGC 及 GC 调步算法


## debug.FreeOsMemory

TODO