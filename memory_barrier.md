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

Store Buffer：

当写入到一行 invalidate 状态的 cache line 时便会使用到 store buffer。写如果要继续执行，CPU 需要先发出一条 read-invalid 消息(因为需要确保所有其它缓存了当前内存地址的 CPU 的 cache line 都被 invalidate 掉)，然后将写推入到 store buffer 中，当最终 cache line 达到当前 CPU 时再执行这个写操作。

CPU 存在 store buffer 的直接影响是，当 CPU 提交一个写操作时，这个写操作不会立即写入到 cache 中。因而，无论什么时候 CPU 需要从 cache line 中读取，都需要先扫描它自己的 store buffer 来确认是否存在相同的 line，因为有可能当前 CPU 在这次操作之前曾经写入过 cache，但该数据还没有被刷入过 cache(之前的写操作还在 store buffer 中等待)。需要注意的是，虽然 CPU 可以读取其之前写入到 store buffer 中的值，但其它 CPU 并不能在该 CPU 将 store buffer 中的内容 flush 到 cache 之前看到这些值。即 store buffer 是不能跨核心访问的，CPU 核心看不到其它核心的 store buffer。

Invalidate Queues：

为了处理 invalidation 消息，CPU 实现了 invalidate queue，借以处理新达到的 invalidate 请求，在这些请求到达时，可以马上进行响应，但可以不马上处理。取而代之的，invalidation 消息只是会被推进一个 invalidation 队列，并在之后尽快处理(但不是马上)。因此，CPU 可能并不知道在它 cache 里的某个 cache line 是 invalid 状态的，因为 invalidation 队列包含有收到但还没有处理的 invalidation 消息，这是因为 CPU 和 invalidation 队列从物理上来讲是位于 cache 的两侧的。

从结果上来讲，memory barrier 是必须的。一个 store barrier 会把 store buffer flush 掉，确保所有的写操作都被应用到 CPU 的 cache。一个 read barrier 会把 invalidation queue flush 掉，也就确保了其它 CPU 的写入对执行 flush 操作的当前这个 CPU 可见。再进一步，MMU 没有办法扫描 store buffer，会导致类似的问题。这种效果对于单线程处理器来说已经是会发生的了。

## lfence, sfence, mfence

https://stackoverflow.com/questions/27595595/when-are-x86-lfence-sfence-and-mfence-instructions-required

## acquire/release 抽象

https://preshing.com/20130922/acquire-and-release-fences/

## write barrier, read barrier

## memory order

std::memory_order specifies how memory accesses, including regular, non-atomic memory accesses, are to be ordered around an atomic operation. Absent any constraints on a multi-core system, when multiple threads simultaneously read and write to several variables, one thread can observe the values change in an order different from the order another thread wrote them. Indeed, the apparent order of changes can even differ among multiple reader threads. Some similar effects can occur even on uniprocessor systems due to compiler transformations allowed by the memory model.

The default behavior of all atomic operations in the library provides for sequentially consistent ordering (see discussion below). That default can hurt performance, but the library's atomic operations can be given an additional std::memory_order argument to specify the exact constraints, beyond atomicity, that the compiler and processor must enforce for that operation.

## sequential consistency

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

## 编译器导致乱序

snippet 1
```python
X = 0
for i in range(100):
    X = 1
    print X
```

snippet 2
```python
X = 1
for i in range(100):
    print X

```

snippet 1 和 snippet 2 从逻辑上等价的。

如果这时候，假设有 Processor 2 同时在执行一条指令：

```python
X = 0
```

P2 中的指令和 snippet 1 交错执行时，可能产生的结果是：111101111..

P2 中的指令和 snippet 2 交错执行时，可能产生的结果是：11100000…​

多核心下，编译器对代码的优化理论上就是重排。

有人说这个例子不够有说服力，我们看看参考资料中的另一个例子:

```c
int a, b;
int foo()
{
    a = b + 1;
    b = 0; 
    return 1;
}
```

输出汇编:
```
mov eax, DWORD PTR b[rip]
add eax, 1
mov DWORD PTR a[rip], eax    // --> store to a
mov DWORD PTR b[rip], 0      // --> store to b
```

开启 O2 优化后，输出汇编:
```
mov eax, DWORD PTR b[rip]
mov DWORD PTR b[rip], 0      // --> store to b
add eax, 1
mov DWORD PTR a[rip], eax    // --> store to a
```

可见 compiler 也是可能会修改赋值的顺序的。

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
