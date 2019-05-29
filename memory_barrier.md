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

![](./images/mesi_1.jpg)

![](./images/mesi_2.jpg)

## 编译器导致乱序

## lfence, sfence, mfence

## write barrier, read barrier

## memory order

## cache coherency vs memory consistency

```
The MESI protocol makes the memory caches effectively invisible. This means that multithreaded programs don't have to worry about a core reading stale data from them or two cores writing to different parts of a cache line and getting half of one write and half of the other sent to main memory.

However, this doesn't help with read-modify-write operations such as increment, compare and swap, and so on. The MESI protocol won't stop two cores from each reading the same chunk of memory, each adding one to it, and then each writing the same value back, turning two increments into one.

On modern CPUs, the LOCK prefix locks the cache line so that the read-modify-write operation is logically atomic. These are oversimplified, but hopefully they'll give you the idea.

Unlocked increment:

Acquire cache line, shareable is fine. Read the value.
Add one to the read value.
Acquire cache line exclusive (if not already E or M) and lock it.
Write the new value to the cache line.
Change the cache line to modified and unlock it.
Locked increment:

Acquire cache line exclusive (if not already E or M) and lock it.
Read value.
Add one to it.
Write the new value to the cache line.
Change the cache line to modified and unlock it.
Notice the difference? In the unlocked increment, the cache line is only locked during the write memory operation, just like all writes. In the locked increment, the cache line is held across the entire instruction, all the way from the read operation to the write operation and including during the increment itself.

Also, some CPUs have things other than memory caches that can affect memory visibility. For example, some CPUs have a read prefetcher or a posted write buffer that can result in memory operations executing out of order. Where needed, a LOCK prefix (or equivalent functionality on other CPUs) will also do whatever needs to be done to handle memory operation ordering issues.
```

引用自 [这里](https://stackoverflow.com/questions/29880015/lock-prefix-vs-mesi-protocol)。

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
