# memory barrier

> There are only two hard things in Computer Science: cache invalidation and naming things.

> -- Phil Karlton

https://martinfowler.com/bliki/TwoHardThings.html

memory barrier 也称为 memory fence。

## CPU 架构

```
 ┌─────────────┐                ┌─────────────┐   
 │    CPU 0    │                │    CPU 1    │   
 └───────────┬─┘                └───────────┬─┘   
   ▲         │                     ▲        │     
   │         │                     │        │     
   │         │                     │        │     
   │         │                     │        │     
   │         ▼                     │        ▼     
   │    ┌────────┐                 │    ┌────────┐
   │◀───┤ Store  │                 │◀───┤ Store  │
   ├───▶│ Buffer │                 ├───▶│ Buffer │
   │    └────┬───┘                 │    └───┬────┘
   │         │                     │        │     
   │         │                     │        │     
   │         │                     │        │     
   │         │                     │        │     
   │         ▼                     │        ▼     
┌──┴────────────┐               ┌──┴────────────┐ 
│               │               │               │ 
│     Cache     │               │     Cache     │ 
│               │               │               │ 
└───────┬───────┘               └───────┬───────┘ 
        │                               │         
        │                               │         
        │                               │         
 ┌──────┴──────┐                 ┌──────┴──────┐  
 │ Invalidate  │                 │ Invalidate  │  
 │    Queue    │                 │    Queue    │  
 └──────┬──────┘                 └──────┬──────┘  
        │                               │         
        │         Interconnect          │         
        └───────────────┬───────────────┘         
                        │                         
                        │                         
                        │                         
                        │                         
                ┌───────┴───────┐                 
                │               │                 
                │    Memory     │                 
                │               │                 
                └───────────────┘                 
```

## CPU 导致乱序

使用 litmus 进行形式化验证:

```
cat sb.litmus

X86 SB
{ x=0; y=0; }
 P0          | P1          ;
 MOV [x],$1  | MOV [y],$1  ;
 MOV EAX,[y] | MOV EAX,[x] ;
locations [x;y;]
exists (0:EAX=0 /\ 1:EAX=0)
```

```shell
~ ❯❯❯ bin/litmus7 ./sb.litmus
%%%%%%%%%%%%%%%%%%%%%%%%%%%
% Results for ./sb.litmus %
%%%%%%%%%%%%%%%%%%%%%%%%%%%
X86 SB

{x=0; y=0;}

 P0          | P1          ;
 MOV [x],$1  | MOV [y],$1  ;
 MOV EAX,[y] | MOV EAX,[x] ;

locations [x; y;]
exists (0:EAX=0 /\ 1:EAX=0)
Generated assembler
	##START _litmus_P0
	movl	$1, -4(%rbx,%rcx,4)
	movl	-4(%rsi,%rcx,4), %eax
	##START _litmus_P1
	movl	$1, -4(%rsi,%rcx,4)
	movl	-4(%rbx,%rcx,4), %eax

Test SB Allowed
Histogram (4 states)
96    *>0:EAX=0; 1:EAX=0; x=1; y=1;
499878:>0:EAX=1; 1:EAX=0; x=1; y=1;
499862:>0:EAX=0; 1:EAX=1; x=1; y=1;
164   :>0:EAX=1; 1:EAX=1; x=1; y=1;
Ok

Witnesses
Positive: 96, Negative: 999904
Condition exists (0:EAX=0 /\ 1:EAX=0) is validated
Hash=2d53e83cd627ba17ab11c875525e078b
Observation SB Sometimes 96 999904
Time SB 0.11
```

## mesi 协议

| | p0 | p1 | p2 | p3 |
|---|:----:|:----:|:--:| :--: |
| initial state | I | I | I | I |
| p0 reads X | E | I | I | I |
| p1 reads X | S | S | I | I |
| p2 reads X | S | S | S | I |
| p3 writes X | I | I | I | M |
| p0 readx X | S | I | I | S |

![](images/mesi_1.jpg)

下面是不同类型的处理器请求和总线侧的请求:

处理器向 Cache 发请求包括下面这些操作:

PrRd: 处理器请求读取一个 Cache block
PrWr: 处理器请求写入一个 Cache block

总线侧的请求包含:

BusRd: 监听到请求，表示当前有其它处理器正在发起对某个 Cache block 的读请求

BusRdX: 监听到请求，表示当前有其它处理器正在发起对某个其未拥有的 Cache block 的写请求

BusUpgr: 监听到请求，表示有另一个处理器正在发起对某 Cache block 的写请求，该处理器已经持有此 Cache block 块

Flush: 监听到请求，表示整个 cache 块已被另一个处理器写入到主存中

FlushOpt: 监听到请求，表示一个完整的 cache 块已经被发送到总线，以提供给另一个处理器使用(Cache 到 Cache 传数)

Cache 到 Cache 的传送可以降低 read miss 导致的延迟，如果不这样做，需要先将该 block 写回到主存，再读取，延迟会大大增加，在基于总线的系统中，这个结论是正确的。但在多核架构中，coherence 是在 L2 caches 这一级保证的，从 L3 中取可能要比从另一个 L2 中取还要快。

## store buffer 和 invalidate queue

Store Buffer
A store buffer is used when writing to an invalid cache line. Since the write will proceed anyway, the CPU issues a read-invalid message (hence the cache line in question and all other CPUs' cache lines which store that memory address are invalidated) and then pushes the write into the store buffer, to be executed when the cache line finally arrives in the cache.

A direct consequence of the store buffer's existence is that when a CPU commits a write, that write is not immediately written in the cache. Therefore, whenever a CPU needs to read a cache line, it first has to scan its own store buffer for the existence of the same line, as there is a possibility that the same line was written by the same CPU before but hasn't yet been written in the cache (the preceding write is still waiting in the store buffer). Note that while a CPU can read its own previous writes in its store buffer, other CPUs cannot see those writes before they are flushed from the store buffer to the cache - a CPU cannot scan the store buffer of other CPUs.

Invalidate Queues

With regard to invalidation messages, CPUs implement invalidate queues, whereby incoming invalidate requests are instantly acknowledged but not in fact acted upon. Instead, invalidation messages simply enter an invalidation queue and their processing occurs as soon as possible (but not necessarily instantly). Consequently, a CPU can be oblivious to the fact that a cache line in its cache is actually invalid, as the invalidation queue contains invalidations which have been received but haven't yet been applied. Note that, unlike the store buffer, the CPU can't scan the invalidation queue, as that CPU and the invalidation queue are physically located on opposite sides of the cache.

As a result, memory barriers are required. A store barrier will flush the store buffer, ensuring all writes have been applied to that CPU's cache. A read barrier will flush the invalidation queue, thus ensuring that all writes by other CPUs become visible to the flushing CPU. Furthermore, memory management units do not scan the store buffer, causing similar problems. This effect is already visible in single threaded processors.

## 编译器导致乱序

## lfence, sfence, mfence

## acquire/release 抽象

## write barrier, read barrier

## memory order

std::memory_order specifies how memory accesses, including regular, non-atomic memory accesses, are to be ordered around an atomic operation. Absent any constraints on a multi-core system, when multiple threads simultaneously read and write to several variables, one thread can observe the values change in an order different from the order another thread wrote them. Indeed, the apparent order of changes can even differ among multiple reader threads. Some similar effects can occur even on uniprocessor systems due to compiler transformations allowed by the memory model.

The default behavior of all atomic operations in the library provides for sequentially consistent ordering (see discussion below). That default can hurt performance, but the library's atomic operations can be given an additional std::memory_order argument to specify the exact constraints, beyond atomicity, that the compiler and processor must enforce for that operation.

## cache coherency vs memory consistency

The MESI protocol makes the memory caches effectively invisible. This means that multithreaded programs don't have to worry about a core reading stale data from them or two cores writing to different parts of a cache line and getting half of one write and half of the other sent to main memory.

However, this doesn't help with read-modify-write operations such as increment, compare and swap, and so on. The MESI protocol won't stop two cores from each reading the same chunk of memory, each adding one to it, and then each writing the same value back, turning two increments into one.

On modern CPUs, the LOCK prefix locks the cache line so that the read-modify-write operation is logically atomic. These are oversimplified, but hopefully they'll give you the idea.

Unlocked increment:

1. Acquire cache line, shareable is fine. Read the value.
2. Add one to the read value.
3. Acquire cache line exclusive (if not already E or M) and lock it.
4. Write the new value to the cache line.
5. Change the cache line to modified and unlock it.

Locked increment:

1. Acquire cache line exclusive (if not already E or M) and lock it.
2. Read value.
3. Add one to it.
4. Write the new value to the cache line.
5. Change the cache line to modified and unlock it.

Notice the difference? In the unlocked increment, the cache line is only locked during the write memory operation, just like all writes. In the locked increment, the cache line is held across the entire instruction, all the way from the read operation to the write operation and including during the increment itself.

Also, some CPUs have things other than memory caches that can affect memory visibility. For example, some CPUs have a read prefetcher or a posted write buffer that can result in memory operations executing out of order. Where needed, a LOCK prefix (or equivalent functionality on other CPUs) will also do whatever needs to be done to handle memory operation ordering issues.

mesi 协议使得各核心的写操作可以序列化地进行观测，但是却没有办法解决先读后写的问题。而先读后写又是我们在写程序时非常常见的操作，比如对用户访问进行计数。甚至在实现用户逻辑层的 sync 库时，也需要底层提供原子性的 check and set 即 CAS 操作，所以 CPU 一般还会提供一个 lock 指令来应对这种窘境:

https://stackoverflow.com/questions/29880015/lock-prefix-vs-mesi-protocol


## atomic/lock 操作成本 in Go

```go
package main

import (
	"sync/atomic"
	"testing"
)

var a int64

func BenchmarkAtomic(b *testing.B) {
	for i := 0; i < b.N; i++ {
		atomic.StoreInt64(&a, int64(i))
	}
}

func BenchmarkNormal(b *testing.B) {
	for i := 0; i < b.N; i++ {
		a = 1
	}
}

```

```shell
go test -bench=.
goos: darwin
goarch: amd64
BenchmarkAtomic-4   	200000000	         7.00 ns/op
BenchmarkNormal-4   	2000000000	         0.39 ns/op
PASS
ok  	_/Users/xx/test/go/atomic_bench	2.925s
```

## false sharing / true sharing

参考资料：

https://homes.cs.washington.edu/~bornholt/post/memory-models.html

https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html

https://monkeysayhi.github.io/2017/12/28/%E4%B8%80%E6%96%87%E8%A7%A3%E5%86%B3%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C/

https://blog.csdn.net/zhangxiao93/article/details/42966279

http://kaiyuan.me/2017/09/22/memory-barrier/

https://blog.csdn.net/dd864140130/article/details/56494925

https://preshing.com/20120515/memory-reordering-caught-in-the-act/

https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/

https://preshing.com/20120625/memory-ordering-at-compile-time/

https://preshing.com/20120612/an-introduction-to-lock-free-programming/

https://preshing.com/20130922/acquire-and-release-fences/

https://webcourse.cs.technion.ac.il/234267/Spring2016/ho/WCFiles/tirgul%205%20mesi.pdf

https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESIHelp.htm

http://15418.courses.cs.cmu.edu/spring2017/lectures
https://software.intel.com/en-us/articles/how-memory-is-accessed

https://software.intel.com/en-us/articles/detect-and-avoid-memory-bottlenecks#_Move_Instructions_into

https://stackoverflow.com/questions/29880015/lock-prefix-vs-mesi-protocol

https://github.com/torvalds/linux/blob/master/Documentation/memory-barriers.txt
