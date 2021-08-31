# pprof
> 本章节没有介绍具体 pprof 以及周边工具的使用, 而是进行了 runtime pprof 实现原理的分析, 旨在提供给读者一个使用方面的参考
在进行深入本章节之前, 让我们来看三个问题, 相信下面这几个问题也是大部分人在使用 pprof 的时候对它最大的困惑, 那么可以带着这三个问题来进行接下去的分析
- 开启 pprof 会对 runtime 产生多大的压力?
- 能否选择性在合适阶段对生产环境的应用进行 pprof 的开启 / 关闭操作?
- pprof 的原理是什么?

go 内置的 `pprof API` 在 `runtime/pprof` 包内, 它提供给了用户与 `runtime` 交互的能力, 让我们能够在应用运行的过程中分析当前应用的各项指标来辅助进行性能优化以及问题排查, 当然也可以直接加载 `_ "net/http/pprof"` 包使用内置的 `http 接口` 来进行使用, `net` 模块内的 pprof 即为 go 替我们封装好的一系列调用 `runtime/pprof` 的方法, 当然也可以自己直接使用
```go
// src/runtime/pprof/pprof.go
// 可观察类目
profiles.m = map[string]*Profile{
        "goroutine":    goroutineProfile,
        "threadcreate": threadcreateProfile,
        "heap":         heapProfile,
        "allocs":       allocsProfile,
        "block":        blockProfile,
        "mutex":        mutexProfile,
    }
```

## allocs
```go

var allocsProfile = &Profile{
  name:  "allocs",
  count: countHeap, // identical to heap profile
  write: writeAlloc,
}
```
- writeAlloc (主要涉及以下几个 api)
  - ReadMemStats(m *MemStats)
  - MemProfile(p []MemProfileRecord, inuseZero bool)

```go
// ReadMemStats populates m with memory allocator statistics.
//
// The returned memory allocator statistics are up to date as of the
// call to ReadMemStats. This is in contrast with a heap profile,
// which is a snapshot as of the most recently completed garbage
// collection cycle.
func ReadMemStats(m *MemStats) {
  // STW 操作
  stopTheWorld("read mem stats")
  // systemstack 切换
  systemstack(func() {
    // 将 memstats 通过 copy 操作复制给 m
    readmemstats_m(m)
  })

  startTheWorld()
}
```

```go
// MemProfile returns a profile of memory allocated and freed per allocation
// site.
//
// MemProfile returns n, the number of records in the current memory profile.
// If len(p) >= n, MemProfile copies the profile into p and returns n, true.
// If len(p) < n, MemProfile does not change p and returns n, false.
//
// If inuseZero is true, the profile includes allocation records
// where r.AllocBytes > 0 but r.AllocBytes == r.FreeBytes.
// These are sites where memory was allocated, but it has all
// been released back to the runtime.
//
// The returned profile may be up to two garbage collection cycles old.
// This is to avoid skewing the profile toward allocations; because
// allocations happen in real time but frees are delayed until the garbage
// collector performs sweeping, the profile only accounts for allocations
// that have had a chance to be freed by the garbage collector.
//
// Most clients should use the runtime/pprof package or
// the testing package's -test.memprofile flag instead
// of calling MemProfile directly.
func MemProfile(p []MemProfileRecord, inuseZero bool) (n int, ok bool) {
  lock(&proflock)
  // If we're between mProf_NextCycle and mProf_Flush, take care
  // of flushing to the active profile so we only have to look
  // at the active profile below.
  mProf_FlushLocked()
  clear := true
  /* 
   * 记住这个 mbuckets -- memory profile buckets 
   * allocs 的采样都是记录在这个全局变量内, 下面会进行详细分析
   * -------------------------------------------------
   * (gdb) info variables mbuckets
   * All variables matching regular expression "mbuckets":

   * File runtime:
   * runtime.bucket *runtime.mbuckets;
   * (gdb)
   */
  for b := mbuckets; b != nil; b = b.allnext {
    mp := b.mp()
    if inuseZero || mp.active.alloc_bytes != mp.active.free_bytes {
      n++
    }
    if mp.active.allocs != 0 || mp.active.frees != 0 {
      clear = false
    }
  }
  if clear {
    // Absolutely no data, suggesting that a garbage collection
    // has not yet happened. In order to allow profiling when
    // garbage collection is disabled from the beginning of execution,
    // accumulate all of the cycles, and recount buckets.
    n = 0
    for b := mbuckets; b != nil; b = b.allnext {
      mp := b.mp()
      for c := range mp.future {
        mp.active.add(&mp.future[c])
        mp.future[c] = memRecordCycle{}
      }
      if inuseZero || mp.active.alloc_bytes != mp.active.free_bytes {
        n++
      }
    }
  }
  if n <= len(p) {
    ok = true
    idx := 0
    for b := mbuckets; b != nil; b = b.allnext {
      mp := b.mp()
      if inuseZero || mp.active.alloc_bytes != mp.active.free_bytes {
        // mbuckets 数据拷贝
        record(&p[idx], b)
        idx++
      }
    }
  }
  unlock(&proflock)
  return
}
```

总结一下 `pprof/allocs` 所涉及的操作
- 短暂的 `STW` 以及 `systemstack` 切换来获取 `runtime` 相关信息
- 拷贝全局对象 `mbuckets` 值返回给用户

### mbuckets
上文提到, `pprof/allocs` 的核心在于对 `mbuckets` 的操作, 下面用一张图来简单描述下 `mbuckets` 的相关操作
```go
var mbuckets  *bucket // memory profile buckets
type bucket struct {
  next    *bucket
  allnext *bucket
  typ     bucketType // memBucket or blockBucket (includes mutexProfile)
  hash    uintptr
  size    uintptr
  nstk    uintptr
}
```


```shell
                                                  ---------------
                                                 |  user access  |
                                                  ---------------
                                                         |
 ------------------                                      |
|   mbuckets list  |              copy                   |
|     (global)     | -------------------------------------  
 ------------------
       |
       |
       | create_or_get && insert_or_update bucket into mbuckets
       |
       |
 --------------------------------------
|  func stkbucket & typ == memProfile  |
 --------------------------------------
                |
         ----------------
        |  mProf_Malloc  | // 堆栈等信息记录
         ----------------
                |
         ----------------
        |  profilealloc  | // next_sample 计算
         ----------------
                |      
                |       /*
                |       * if rate := MemProfileRate; rate > 0 {
                |       *   if rate != 1 && size < c.next_sample {
                |       *     c.next_sample -= size
                | 采样   *   } else {
                | 记录   *     mp := acquirem()
                |       *     profilealloc(mp, x, size)
                |       *     releasem(mp)
                |       *   }
                |       * }
                |       */
                |
           ------------    不采样
          |  mallocgc  |-----------...
           ------------
```

由上图我们可以清晰的看见, `runtime` 在内存分配的时候会根据一定策略进行采样, 记录到 `mbuckets` 中让用户得以进行分析, 而采样算法有个重要的依赖 `MemProfileRate`

```go
// MemProfileRate controls the fraction of memory allocations
// that are recorded and reported in the memory profile.
// The profiler aims to sample an average of
// one allocation per MemProfileRate bytes allocated.
//
// To include every allocated block in the profile, set MemProfileRate to 1.
// To turn off profiling entirely, set MemProfileRate to 0.
//
// The tools that process the memory profiles assume that the
// profile rate is constant across the lifetime of the program
// and equal to the current value. Programs that change the
// memory profiling rate should do so just once, as early as
// possible in the execution of the program (for example,
// at the beginning of main).
var MemProfileRate int = 512 * 1024
```
默认大小是 512 KB, 可以由用户自行配置.

值的注意的是, 由于开启了 pprof 会产生一些采样的额外压力及开销, go 团队已经在较新的编译器中有选择地进行了这个变量的配置以[改变](https://go-review.googlesource.com/c/go/+/299671/8/src/runtime/mprof.go)默认开启的现状

具体方式为代码未进行相关引用则编译器将初始值配置为 0, 否则则为默认(512 KB)

(本文讨论的基于 1.14.3 版本, 如有差异请进行版本确认)

#### pprof/allocs 总结
- 开启后会对 runtime 产生额外压力, 采样时会在 `runtime malloc` 时记录额外信息以供后续分析
- 可以人为选择是否开启, 以及采样频率, 通过设置 `runtime.MemProfileRate` 参数, 不同 go 版本存在差异(是否默认开启), 与用户代码内是否引用(linker)相关模块/变量有关, 默认大小为 512 KB

`allocs` 部分还包含了 `heap` 情况的近似计算, 放在下一节分析
## heap
>allocs: A sampling of all past memory allocations

>heap: A sampling of memory allocations of live objects. You can specify the gc GET parameter to run GC before taking the heap sample.

对比下 `allocs` 和 `heap` 官方说明上的区别, 一个是分析所有内存分配的情况, 一个是当前 `heap` 上的分配情况. `heap` 还能使用额外参数运行一次 `GC` 后再进行分析

看起来两者差别很大。。。不过实质上在代码层面两者除了一次 `GC` 可以人为调用以及生成的文件类型不同之外 (debug == 0 的时候) 之外没啥区别.

### heap 采样(伪)
```go
// p 为上文提到过的 MemProfileRecord 采样记录
for _, r := range p {
    hideRuntime := true
    for tries := 0; tries < 2; tries++ {
      stk := r.Stack()
      // For heap profiles, all stack
      // addresses are return PCs, which is
      // what appendLocsForStack expects.
      if hideRuntime {
        for i, addr := range stk {
          if f := runtime.FuncForPC(addr); f != nil && strings.HasPrefix(f.Name(), "runtime.") {
            continue
          }
          // Found non-runtime. Show any runtime uses above it.
          stk = stk[i:]
          break
        }
      }
      locs = b.appendLocsForStack(locs[:0], stk)
      if len(locs) > 0 {
        break
      }
      hideRuntime = false // try again, and show all frames next time.
    }
    // rate 即为 runtime.MemProfileRate
    values[0], values[1] = scaleHeapSample(r.AllocObjects, r.AllocBytes, rate)
    values[2], values[3] = scaleHeapSample(r.InUseObjects(), r.InUseBytes(), rate)
    var blockSize int64
    if r.AllocObjects > 0 {
      blockSize = r.AllocBytes / r.AllocObjects
    }
    b.pbSample(values, locs, func() {
      if blockSize != 0 {
        b.pbLabel(tagSample_Label, "bytes", "", blockSize)
      }
    })
  }
```
```go
// scaleHeapSample adjusts the data from a heap Sample to
// account for its probability of appearing in the collected
// data. heap profiles are a sampling of the memory allocations
// requests in a program. We estimate the unsampled value by dividing
// each collected sample by its probability of appearing in the
// profile. heap profiles rely on a poisson process to determine
// which samples to collect, based on the desired average collection
// rate R. The probability of a sample of size S to appear in that
// profile is 1-exp(-S/R).
func scaleHeapSample(count, size, rate int64) (int64, int64) {
  if count == 0 || size == 0 {
    return 0, 0
  }

  if rate <= 1 {
    // if rate==1 all samples were collected so no adjustment is needed.
    // if rate<1 treat as unknown and skip scaling.
    return count, size
  }

  avgSize := float64(size) / float64(count)
  scale := 1 / (1 - math.Exp(-avgSize/float64(rate)))

  return int64(float64(count) * scale), int64(float64(size) * scale)
}
```

为什么要在标题里加个伪? 看上面代码片段也可以注意到, 实质上在 `pprof` 分析的时候并没有扫描所有堆上内存进行分析 (想想也不现实) , 而是通过之前采样出的数据, 进行计算 (现有对象数量, 大小, 采样率等) 来估算出 `heap` 上的情况, 当然给我们参考一般来说是足够了

## goroutine
- debug >= 2 的情况, 直接进行堆栈输出, 详情可以查看 [stack](runtime_stack.md) 章节

```go
// fetch == runtime.GoroutineProfile
func writeRuntimeProfile(w io.Writer, debug int, name string, fetch func([]runtime.StackRecord) (int, bool)) error {
  // Find out how many records there are (fetch(nil)),
  // allocate that many records, and get the data.
  // There's a race—more records might be added between
  // the two calls—so allocate a few extra records for safety
  // and also try again if we're very unlucky.
  // The loop should only execute one iteration in the common case.
  var p []runtime.StackRecord
  n, ok := fetch(nil)
  for {
    // Allocate room for a slightly bigger profile,
    // in case a few more entries have been added
    // since the call to ThreadProfile.
    p = make([]runtime.StackRecord, n+10)
    n, ok = fetch(p)
    if ok {
      p = p[0:n]
      break
    }
    // Profile grew; try again.
  }

  return printCountProfile(w, debug, name, runtimeProfile(p))
}
```

```go
// GoroutineProfile returns n, the number of records in the active goroutine stack profile.
// If len(p) >= n, GoroutineProfile copies the profile into p and returns n, true.
// If len(p) < n, GoroutineProfile does not change p and returns n, false.
//
// Most clients should use the runtime/pprof package instead
// of calling GoroutineProfile directly.
func GoroutineProfile(p []StackRecord) (n int, ok bool) {
  gp := getg()

  isOK := func(gp1 *g) bool {
    // Checking isSystemGoroutine here makes GoroutineProfile
    // consistent with both NumGoroutine and Stack.
    return gp1 != gp && readgstatus(gp1) != _Gdead && !isSystemGoroutine(gp1, false)
  }
  // 熟悉的味道, STW 又来了
  stopTheWorld("profile")
  // 统计有多少 goroutine
  n = 1
  for _, gp1 := range allgs {
    if isOK(gp1) {
      n++
    }
  }
  // 当传入的 p 非空的时候, 开始获取各个 goroutine 信息, 整体姿势和 stack api 几乎一模一样
  if n <= len(p) {
    ok = true
    r := p

    // Save current goroutine.
    sp := getcallersp()
    pc := getcallerpc()
    systemstack(func() {
      saveg(pc, sp, gp, &r[0])
    })
    r = r[1:]

    // Save other goroutines.
    for _, gp1 := range allgs {
      if isOK(gp1) {
        if len(r) == 0 {
          // Should be impossible, but better to return a
          // truncated profile than to crash the entire process.
          break
        }
        saveg(^uintptr(0), ^uintptr(0), gp1, &r[0])
        r = r[1:]
      }
    }
  }

  startTheWorld()

  return n, ok
}
```
总结下 `pprof/goroutine`
- STW 操作, 如果需要观察详情的需要注意这个 API 带来的风险
- 整体流程基本就是 stackdump 所有协程信息的流程, 差别不大没什么好讲的, 不熟悉的可以去看下 stack 对应章节

## pprof/threadcreate
可能会有人想问, 我们通常只关注 `goroutine` 就够了, 为什么还需要对线程的一些情况进行追踪? 例如无法被抢占的阻塞性[系统调用](syscall.md), `cgo` 相关的线程等等, 都可以利用它来进行一个简单的分析, 当然大多数情况考虑的线程问题(诸如泄露等), 一般都是上层的使用问题所导致的(线程泄露等)
```go
// 还是用之前用过的无法被抢占的阻塞性系统调用来进行一个简单的实验
package main

import (
  "fmt"
  "net/http"
  _ "net/http/pprof"
  "os"
  "syscall"
  "unsafe"
)

const (
  SYS_futex           = 202
  _FUTEX_PRIVATE_FLAG = 128
  _FUTEX_WAIT         = 0
  _FUTEX_WAKE         = 1
  _FUTEX_WAIT_PRIVATE = _FUTEX_WAIT | _FUTEX_PRIVATE_FLAG
  _FUTEX_WAKE_PRIVATE = _FUTEX_WAKE | _FUTEX_PRIVATE_FLAG
)

func main() {
  fmt.Println(os.Getpid())
  go func() {
    b := make([]byte, 1<<20)
    _ = b
  }()
  for i := 1; i < 13; i++ {
    go func() {
      var futexVar int = 0
      for {
        // Syscall && RawSyscall, 具体差别分析可自行查看 syscall 章节
        fmt.Println(syscall.Syscall6(
          SYS_futex,                          // trap AX    202
          uintptr(unsafe.Pointer(&futexVar)), // a1 DI      1
          uintptr(_FUTEX_WAIT),               // a2 SI      0
          0,                                  // a3 DX
          0,                                  //uintptr(unsafe.Pointer(&ts)), // a4 R10
          0,                                  // a5 R8
          0))
      }
    }()
  }
  http.ListenAndServe("0.0.0.0:8899", nil)
}
```
```shell
# GET /debug/pprof/threadcreate?debug=1
threadcreate profile: total 18
17 @
#  0x0

1 @ 0x43b818 0x43bfa3 0x43c272 0x43857d 0x467fb1
#  0x43b817  runtime.allocm+0x157      /usr/local/go/src/runtime/proc.go:1414
#  0x43bfa2  runtime.newm+0x42      /usr/local/go/src/runtime/proc.go:1736
#  0x43c271  runtime.startTemplateThread+0xb1  /usr/local/go/src/runtime/proc.go:1805
#  0x43857c  runtime.main+0x18c      /usr/local/go/src/runtime/proc.go:186
```
```shell
# 再结合诸如 pstack 的工具
ps -efT | grep 22298 # pid = 22298
root     22298 22298 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22299 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22300 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22301 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22302 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22303 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22304 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22305 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22306 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22307 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22308 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22309 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22310 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22311 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22312 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22316 13767  0 16:59 pts/4    00:00:00 ./mstest
root     22298 22317 13767  0 16:59 pts/4    00:00:00 ./mstest

pstack 22299
Thread 1 (process 22299):
#0  runtime.futex () at /usr/local/go/src/runtime/sys_linux_amd64.s:568
#1  0x00000000004326f4 in runtime.futexsleep (addr=0xb2fd78 <runtime.sched+280>, val=0, ns=60000000000) at /usr/local/go/src/runtime/os_linux.go:51
#2  0x000000000040cb3e in runtime.notetsleep_internal (n=0xb2fd78 <runtime.sched+280>, ns=60000000000, ~r2=<optimized out>) at /usr/local/go/src/runtime/lock_futex.go:193
#3  0x000000000040cc11 in runtime.notetsleep (n=0xb2fd78 <runtime.sched+280>, ns=60000000000, ~r2=<optimized out>) at /usr/local/go/src/runtime/lock_futex.go:216
#4  0x00000000004433b2 in runtime.sysmon () at /usr/local/go/src/runtime/proc.go:4558
#5  0x000000000043af33 in runtime.mstart1 () at /usr/local/go/src/runtime/proc.go:1112
#6  0x000000000043ae4e in runtime.mstart () at /usr/local/go/src/runtime/proc.go:1077
#7  0x0000000000401893 in runtime/cgo(.text) ()
#8  0x00007fb1e2d53700 in ?? ()
#9  0x0000000000000000 in ?? ()
```
其他的线程如果感兴趣也可以仔细查看

`pprof/threadcreate` 具体实现和 `pprof/goroutine` 类似, 无非前者遍历的对象是全局 `allm`, 而后者为 `allgs`, 区别在于 `pprof/threadcreate => ThreadCreateProfile` 时不会进行进行 `STW`

## pprof/mutex
mutex 默认是关闭采样的, 通过 `runtime.SetMutexProfileFraction(int)` 来进行 `rate` 的配置进行开启或关闭

和上文分析过的 `mbuckets` 类似, 这边用以记录采样数据的是 `xbuckets`, `bucket` 记录了锁持有的堆栈, 次数(采样)等信息以供用户查看
```go
//go:linkname mutexevent sync.event
func mutexevent(cycles int64, skip int) {
  if cycles < 0 {
    cycles = 0
  }
  rate := int64(atomic.Load64(&mutexprofilerate))
  // TODO(pjw): measure impact of always calling fastrand vs using something
  // like malloc.go:nextSample()
  // 同样根据 rate 来进行采样, 这边用以记录 rate 的是 mutexprofilerate 变量
  if rate > 0 && int64(fastrand())%rate == 0 {
    saveblockevent(cycles, skip+1, mutexProfile)
  }
}
```
```shell
                                                  ---------------
                                                 |  user access  |
                                                  ---------------
                                                         |
 ------------------                                      |
|   xbuckets list  |              copy                   |
|     (global)     | -------------------------------------  
 ------------------
       |
       |
       | create_or_get && insert_or_update bucket into xbuckets
       |
       |
 --------------------------------------
|  func stkbucket & typ == mutexProfile  |
 --------------------------------------
                 |
         ------------------
        |  saveblockevent  | // 堆栈等信息记录
         ------------------
                 |
                 |      
                 |       /*  
                 |       *   //go:linkname mutexevent sync.event
                 |       *   func mutexevent(cycles int64, skip int) {
                 |       *     if cycles < 0 {
                 |       *       cycles = 0
                 |       *     }
                 | 采样   *     rate := int64(atomic.Load64(&mutexprofilerate))
                 | 记录   *     // TODO(pjw): measure impact of always calling fastrand vs using something
                 |       *     // like malloc.go:nextSample()
                 |       *     if rate > 0 && int64(fastrand())%rate == 0 {
                 |       *       saveblockevent(cycles, skip+1, mutexProfile)
                 |       *     }
                 |       * 
                 |       */
                 |
           ------------     不采样
          | mutexevent | ----------....
           ------------
                 |
                 |
           ------------   
          | semrelease1 |
           ------------
                 |
                 |
       ------------------------  
      |   runtime_Semrelease   |
       ------------------------
                 |
                 |
           ------------   
          | unlockSlow |
           ------------
                 |
                 |
           ------------  
          |   Unlock   |
           ------------
```

## pprof/block
同上, 主要来分析下 `bbuckets` 
```shell
                                                  ---------------
                                                 |  user access  |
                                                  ---------------
                                                         |
 ------------------                                      |
|   bbuckets list  |              copy                   |
|     (global)     | -------------------------------------  
 ------------------
       |
       |
       | create_or_get && insert_or_update bucket into bbuckets
       |
       |
 --------------------------------------
|  func stkbucket & typ == blockProfile  |
 --------------------------------------
                 |
         ------------------
        |  saveblockevent  | // 堆栈等信息记录
         ------------------
                 |
                 |      
                 |       /*  
                 |       *   func blocksampled(cycles int64) bool {
                 |       *     rate := int64(atomic.Load64(&blockprofilerate))
                 |       *     if rate <= 0 || (rate > cycles && int64(fastrand())%rate > cycles) {
                 |       *       return false
                 | 采样   *     }
                 | 记录   *     return true
                 |       *   }
                 |       */
                 |
           ------------     不采样
          | blockevent | ----------....
           ------------
                 |----------------------------------------------------------------------------
                 |                                     |                                      |
           ------------          -----------------------------------------------        ------------
          | semrelease1 |       |  chansend / chanrecv &&  mysg.releasetime > 0 |      |  selectgo  |
           ------------          -----------------------------------------------        ------------
```
相比较 `mutex` 的采样, `block` 的埋点会额外存在于 `chan` 中, 每次 `block` 记录的是前后两个 `cpu 周期` 的差值 (cycles)
需要注意的是 `cputicks` 可能在不同系统上存在一些[问题](https://github.com/golang/go/issues/8976). 暂不放在这边讨论

## pprof/profile
上面分析的都属于 `runtime` 在运行的过程中自动采用保存数据后用户进行观察的, `profile` 则是用户选择指定周期内的 `CPU Profiling`

#总结
- `pprof` 的确会给 `runtime` 带来额外的压力, 压力的多少取决于用户使用的各个 `*_rate` 配置, 在获取 `pprof` 信息的时候需要按照实际情况酌情使用各个接口, 每个接口产生的额外压力是不一样的.
- 不同版本在是否默认开启上有不同策略, 需要自行根据各自的环境进行确认
- `pprof` 获取到的数据仅能作为参考, 和设置的采样频率有关, 在计算例如 `heap` 情况时会进行相关的近似预估, 非实质上对 `heap` 进行扫描

```shell
 -------------------------
|  pprof.StartCPUProfile  |
 -------------------------
            |
            |
            |
 -------------------------
|  sleep(time.Duration)   |
 -------------------------
            |
            |
            |
 -------------------------
|  pprof.StopCPUProfile  |
 -------------------------
```
`pprof.StartCPUProfile` 与 `pprof.StopCPUProfile` 核心为 `runtime.SetCPUProfileRate(hz int)` 控制 `cpu profile` 频率, 但是这边的频率设置和前面几个有差异, 不仅仅是设计 rate 的设置, 还涉及全局对象 `cpuprof` log buffer 的分配
```go
var cpuprof cpuProfile
type cpuProfile struct {
  lock mutex
  on   bool     // profiling is on
  log  *profBuf // profile events written here

  // extra holds extra stacks accumulated in addNonGo
  // corresponding to profiling signals arriving on
  // non-Go-created threads. Those stacks are written
  // to log the next time a normal Go thread gets the
  // signal handler.
  // Assuming the stacks are 2 words each (we don't get
  // a full traceback from those threads), plus one word
  // size for framing, 100 Hz profiling would generate
  // 300 words per second.
  // Hopefully a normal Go thread will get the profiling
  // signal at least once every few seconds.
  extra      [1000]uintptr
  numExtra   int
  lostExtra  uint64 // count of frames lost because extra is full
  lostAtomic uint64 // count of frames lost because of being in atomic64 on mips/arm; updated racily
}
```
`log buffer` 的大小每次分配是固定的, 无法进行调节

### cpuprof.add
将 `stack trace` 信息写入 `cpuprof` 的 `log buffer`
```go
// add adds the stack trace to the profile.
// It is called from signal handlers and other limited environments
// and cannot allocate memory or acquire locks that might be
// held at the time of the signal, nor can it use substantial amounts
// of stack.
//go:nowritebarrierrec
func (p *cpuProfile) add(gp *g, stk []uintptr) {
  // Simple cas-lock to coordinate with setcpuprofilerate.
  for !atomic.Cas(&prof.signalLock, 0, 1) {
    osyield()
  }

  if prof.hz != 0 { // implies cpuprof.log != nil
    if p.numExtra > 0 || p.lostExtra > 0 || p.lostAtomic > 0 {
      p.addExtra()
    }
    hdr := [1]uint64{1}
    // Note: write "knows" that the argument is &gp.labels,
    // because otherwise its write barrier behavior may not
    // be correct. See the long comment there before
    // changing the argument here.
    cpuprof.log.write(&gp.labels, nanotime(), hdr[:], stk)
  }

  atomic.Store(&prof.signalLock, 0)
}
```

来看下调用 `cpuprof.add` 的流程
```shell
 ------------------------
|   cpu profile start    |
 ------------------------
            |
            |
            | start timer (setitimer syscall / ITIMER_PROF)
            | 每个一段时间(rate)在向当前 P 所在线程发送一个 SIGPROF 信号量   --
            |                                                           |
            |                                                           |
 ------------------------                   loop                        |
|       sighandler       |----------------------------------------------
 ------------------------                                            |
            |                                                        |
            | /*                                                     |
            |  *  if sig == _SIGPROF {                               |
            |  *    sigprof(c.sigpc(), c.sigsp(), c.siglr(), gp, _g_.m)
            |  *    return                                           |
            |  */ }                                                  |
            |                                                        |
  ----------------------------                                       | stop
 |   sigprof(stack strace)    |                                      |
  ----------------------------                                       |
            |                                                        |
            |                                                        |
            |                                                        |
  ----------------------                                             |
 |     cpuprof.add      |                                            |
  ----------------------                                   ----------------------
           |                                              |   cpu profile stop   |
           |                                               ----------------------                  
           |            
  ----------------------
 |  cpuprof.log buffer  |                                         
  ----------------------
           |                                        ---------------------                  ---------------
           ----------------------------------------|   cpuprof.read      |----------------|  user access  |
                                                    ---------------------                  ---------------
```
由于 `GMP` 的模型设计, 在绝大多数情况下通过这种 `timer` + `sig` + `current thread` 以及当前支持的抢占式调度, 这种记录方式是能够很好进行整个 `runtime cpu profile` 采样分析的, 但也不能排除一些极端情况是无法被覆盖的, 毕竟也只是基于当前 M 而已.

# 总结
#### 可用性: 
runtime 自带的 pprof 已经在数据采集的准确性, 覆盖率, 压力等各方面替我们做好了一个比较均衡及全面的考虑

在绝大多数场景下使用起来需要考虑的性能点无非就是几个 rate 的设置

不同版本的默认开启是有差别的, 几个参数默认值可自行确认, 有时候你觉得没有开启 pprof 但是实际上已经开启了

当选择的参数合适的时候, pprof 远远没有想象中那般“重”
#### 局限性: 
得到的数据只是采样(根据 rate 决定) 或预估值

无法 cover 所有场景, 对于一些特殊的或者极端的情况, 需要各自进行优化来选择合适的手段完善
#### 安全性: 
生产环境可用 pprof, 注意接口不能直接暴露, 毕竟存在诸如 STW 等操作, 存在潜在风险点

#开源项目 pprof 参考
[nsq](https://github.com/nsqio/nsq/blob/v1.2.0/nsqd/http.go#L78-L88)
[etcd](https://github.com/etcd-io/etcd/blob/release-3.4/pkg/debugutil/pprof.go#L23) 采用的是[配置式](https://github.com/etcd-io/etcd/blob/release-3.4/etcd.conf.yml.sample#L76)选择是否开启

# 参考资料
https://go-review.googlesource.com/c/go/+/299671

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
