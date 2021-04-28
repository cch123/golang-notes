# pprof
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
# 参考资料
https://go-review.googlesource.com/c/go/+/299671