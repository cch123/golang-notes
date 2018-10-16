# 内存管理

## 数据结构

```go
// 内存分配器基于页工作。
// 小内存分配 (最多到 32 kB，且包含 32 kB) 会被四舍五入到 70 个大小类型
// 这 70 个大小类型，每一种都有相应的 free 内存块集合
// 任意空闲的内存页都可以被 split 为某一种大小类型的一系列对象集合
// 这些集合使用一个 bitmap 来进行管理
//
// 分配器的数据结构:
//
//    fixalloc: a free-list allocator for fixed-size off-heap objects,
//        used to manage storage used by the allocator.
//    mheap: the malloc heap, managed at page (8192-byte) granularity.
//    mspan: a run of pages managed by the mheap.
//    mcentral: collects all spans of a given size class.
//    mcache: a per-P cache of mspans with free space.
//    mstats: allocation statistics.
//
```

```
┌─────────────────┐                                     ┌─────────────────┐                                     ┌─────────────────┐                                
│      mheap      │                                     │    mSpanList    │                                     │    fixalloc     │                                
├─────────────────┴───────────────────────────────┐     ├─────────────────┴───────────────────────────────┐     ├─────────────────┴───────────────────────────────┐
│                   lock mutex                    │     │                  first *mspan                   │     │                  size uintptr                   │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│          free [_MaxMHeapList]mSpanList          │     │                  last  *mspan                   │     │        first func(arg, p unsafe.Pointer)        │
├─────────────────────────────────────────────────┤     └─────────────────────────────────────────────────┘     ├─────────────────────────────────────────────────┤
│                freelarge mTreap                 │                                                             │               arg unsafe.Pointer                │
├─────────────────────────────────────────────────┤     ┌─────────────────┐                                     ├─────────────────────────────────────────────────┤
│          busy [_MaxMHeapList]mSpanList          │     │      mspan      │                                     │                   list *mlink                   │
├─────────────────────────────────────────────────┤     ├─────────────────┴───────────────────────────────┐     ├─────────────────────────────────────────────────┤
│               busylarge mSpanList               │     │                   next *mspan                   │     │                  chunk uintptr                  │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│                sweepgen  uint32                 │     │                   prev *mspan                   │     │                  nchunk uint32                  │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│                sweepdone uint32                 │     │            list *mSpanList // debug             │     │                  inuse uintptr                  │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│                sweepers  uint32                 │     ├─────────────────────────────────────────────────┤     │                  stat *uint64                   │
├─────────────────────────────────────────────────┤     │                startAddr uintptr                │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                    zero bool                    │
│                allspans []*mspan                │     │                 npages uintptr                  │     └─────────────────────────────────────────────────┘
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤                                                        
│                 spans []*mspan                  │     ├─────────────────────────────────────────────────┤     ┌─────────────────┐                                
├─────────────────────────────────────────────────┤     │            manualFreeList gclinkptr             │     │      mlink      │                                
│            sweepSpans [2]gcSweepBuf             │     ├─────────────────────────────────────────────────┤     ├─────────────────┴───────────────────────────────┐
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                   next *mlink                   │
│                    _ uint32                     │     │                freeindex uintptr                │     └─────────────────────────────────────────────────┘
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤                                                        
├─────────────────────────────────────────────────┤     │                 nelems uintptr                  │     ┌─────────────────┐                                
│                pagesInUse uint64                │     ├─────────────────────────────────────────────────┤     │    mcentral     │                                
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────┴───────────────────────────────┐
│                pagesSwept uint64                │     │                allocCache uint64                │     │                   lock mutex                    │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│             pagesSweptBasis uint64              │     │               allocBits  *gcBits                │     │               spanclass spanClass               │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│            sweepHeapLiveBasis uint64            │     │               gcmarkBits *gcBits                │     │               nonempty  mSpanList               │
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤
│            sweepPagesPerByte float64            │     ├─────────────────────────────────────────────────┤     │                 empty mSpanList                 │
├─────────────────────────────────────────────────┤     │                 sweepgen uint32                 │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                 nmalloc uint64                  │
│                largealloc uint64                │     │                  divMul uint16                  │     └─────────────────────────────────────────────────┘
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤                                                        
│               nlargealloc uint64                │     │                 baseMask uint16                 │     ┌─────────────────┐                                
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │   gcSweepBuf    │                                
│               largefree   uint64                │     │                allocCount uint16                │     ├─────────────────┴───────────────────────────────┐
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                 spineLock mutex                 │
│               nlargefree  uint64                │     │               spanclass spanClass               │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │              spine unsafe.Pointer               │
│       nsmallfree [_NumSizeClasses]uint64        │     │                  incache bool                   │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                spineLen uintptr                 │
├─────────────────────────────────────────────────┤     │                state mSpanState                 │     ├─────────────────────────────────────────────────┤
│                 bitmap uintptr                  │     ├─────────────────────────────────────────────────┤     │                spineCap uintptr                 │
├─────────────────────────────────────────────────┤     │                 needzero uint8                  │     ├─────────────────────────────────────────────────┤
│              bitmap_mapped uintptr              │     ├─────────────────────────────────────────────────┤     │                  index uint32                   │
├─────────────────────────────────────────────────┤     │                 divShift uint8                  │     └─────────────────────────────────────────────────┘
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤                                                        
│               arena_start uintptr               │     │                 divShift2 uint8                 │     ┌─────────────────┐                                
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │     mcache      │                                
│               arena_used  uintptr               │     │                elemsize uintptr                 │     ├─────────────────┴───────────────────────────────┐
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                next_sample int32                │
│               arena_alloc uintptr               │     │                unusedsince int64                │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │               local_scan uintptr                │
│                arena_end uintptr                │     │               npreleased uintptr                │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │                  tiny uintptr                   │
│               arena_reserved bool               │     │                  limit uintptr                  │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │               tinyoffset uintptr                │
│                    _ uint32                     │     │                speciallock mutex                │     ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤     ├─────────────────────────────────────────────────┤     │            local_tinyallocs uintptr             │
├─────────────────────────────────────────────────┤     │                specials *special                │     ├─────────────────────────────────────────────────┤
│        central [numSpanClasses]struct {         │     └─────────────────────────────────────────────────┘     │          alloc [numSpanClasses]*mspan           │
│                  mcentral mcentral              │                                                             ├─────────────────────────────────────────────────┤
│            pad      [sys.CacheLineSize -        │                                                             │    stackcache [_NumStackOrders]stackfreelist    │
│unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte │                                                             ├─────────────────────────────────────────────────┤
├─────────────────────────────────────────────────┤                                                             │              local_nlookup uintptr              │
├─────────────────────────────────────────────────┤                                                             ├─────────────────────────────────────────────────┤
│               spanalloc fixalloc                │                                                             │             local_largefree uintptr             │
├─────────────────────────────────────────────────┤                                                             ├─────────────────────────────────────────────────┤
│               cachealloc fixalloc               │                                                             │            local_nlargefree uintptr             │
├─────────────────────────────────────────────────┤                                                             ├─────────────────────────────────────────────────┤
│               treapalloc fixalloc               │                                                             │    local_nsmallfree [_NumSizeClasses]uintptr    │
├─────────────────────────────────────────────────┤                                                             └─────────────────────────────────────────────────┘
│         specialfinalizeralloc fixalloc          │                                                                                                                
├─────────────────────────────────────────────────┤                                                                                                                
│          specialprofilealloc fixalloc           │                                                                                                                
├─────────────────────────────────────────────────┤                                                                                                                
│                speciallock mutex                │                                                                                                                
├─────────────────────────────────────────────────┤                                                                                                                
├─────────────────────────────────────────────────┤                                                                                                                
│            unused *specialfinalizer             │                                                                                                                
└─────────────────────────────────────────────────┘                                                                                                                
```

## 内存分配器初始化

```go

// OS-defined helpers:
//
// sysAlloc obtains a large chunk of zeroed memory from the
// operating system, typically on the order of a hundred kilobytes
// or a megabyte.
// NOTE: sysAlloc returns OS-aligned memory, but the heap allocator
// may use larger alignment, so the caller must be careful to realign the
// memory obtained by sysAlloc.
//
// SysUnused notifies the operating system that the contents
// of the memory region are no longer needed and can be reused
// for other purposes.
// SysUsed notifies the operating system that the contents
// of the memory region are needed again.
//
// SysFree returns it unconditionally; this is only used if
// an out-of-memory error has been detected midway through
// an allocation. It is okay if SysFree is a no-op.
//
// SysReserve reserves address space without allocating memory.
// If the pointer passed to it is non-nil, the caller wants the
// reservation there, but SysReserve can still choose another
// location if that one is unavailable. On some systems and in some
// cases SysReserve will simply check that the address space is
// available and not actually reserve it. If SysReserve returns
// non-nil, it sets *reserved to true if the address space is
// reserved, false if it has merely been checked.
// NOTE: SysReserve returns OS-aligned memory, but the heap allocator
// may use larger alignment, so the caller must be careful to realign the
// memory obtained by sysAlloc.
//
// SysMap maps previously reserved address space for use.
// The reserved argument is true if the address space was really
// reserved, not merely checked.
//
// SysFault marks a (already sysAlloc'd) region to fault
// if accessed. Used only for debugging the runtime.

func mallocinit() {
    if class_to_size[_TinySizeClass] != _TinySize {
        throw("bad TinySizeClass")
    }

    testdefersizes()

    // Copy class sizes out for statistics table.
    for i := range class_to_size {
        memstats.by_size[i].size = uint32(class_to_size[i])
    }

    // Check physPageSize.
    if physPageSize == 0 {
        // The OS init code failed to fetch the physical page size.
        throw("failed to get system page size")
    }
    if physPageSize < minPhysPageSize {
        print("system page size (", physPageSize, ") is smaller than minimum page size (", minPhysPageSize, ")\n")
        throw("bad system page size")
    }
    if physPageSize&(physPageSize-1) != 0 {
        print("system page size (", physPageSize, ") must be a power of 2\n")
        throw("bad system page size")
    }

    // The auxiliary regions start at p and are laid out in the
    // following order: spans, bitmap, arena.
    var p, pSize uintptr
    var reserved bool

    // The spans array holds one *mspan per _PageSize of arena.
    var spansSize uintptr = (_MaxMem + 1) / _PageSize * sys.PtrSize
    spansSize = round(spansSize, _PageSize)
    // The bitmap holds 2 bits per word of arena.
    var bitmapSize uintptr = (_MaxMem + 1) / (sys.PtrSize * 8 / 2)
    bitmapSize = round(bitmapSize, _PageSize)

    // Set up the allocation arena, a contiguous area of memory where
    // allocated data will be found.
    if sys.PtrSize == 8 {
        // On a 64-bit machine, allocate from a single contiguous reservation.
        // 512 GB (MaxMem) should be big enough for now.
        //
        // The code will work with the reservation at any address, but ask
        // SysReserve to use 0x0000XXc000000000 if possible (XX=00...7f).
        // Allocating a 512 GB region takes away 39 bits, and the amd64
        // doesn't let us choose the top 17 bits, so that leaves the 9 bits
        // in the middle of 0x00c0 for us to choose. Choosing 0x00c0 means
        // that the valid memory addresses will begin 0x00c0, 0x00c1, ..., 0x00df.
        // In little-endian, that's c0 00, c1 00, ..., df 00. None of those are valid
        // UTF-8 sequences, and they are otherwise as far away from
        // ff (likely a common byte) as possible. If that fails, we try other 0xXXc0
        // addresses. An earlier attempt to use 0x11f8 caused out of memory errors
        // on OS X during thread allocations.  0x00c0 causes conflicts with
        // AddressSanitizer which reserves all memory up to 0x0100.
        // These choices are both for debuggability and to reduce the
        // odds of a conservative garbage collector (as is still used in gccgo)
        // not collecting memory because some non-pointer block of memory
        // had a bit pattern that matched a memory address.
        //
        // Actually we reserve 544 GB (because the bitmap ends up being 32 GB)
        // but it hardly matters: e0 00 is not valid UTF-8 either.
        //
        // If this fails we fall back to the 32 bit memory mechanism
        //
        // However, on arm64, we ignore all this advice above and slam the
        // allocation at 0x40 << 32 because when using 4k pages with 3-level
        // translation buffers, the user address space is limited to 39 bits
        // On darwin/arm64, the address space is even smaller.
        arenaSize := round(_MaxMem, _PageSize)
        pSize = bitmapSize + spansSize + arenaSize + _PageSize
        for i := 0; i <= 0x7f; i++ {
            switch {
            case GOARCH == "arm64" && GOOS == "darwin":
                p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
            case GOARCH == "arm64":
                p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
            default:
                p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
            }
            p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
            if p != 0 {
                break
            }
        }
    }

    if p == 0 {
        // On a 32-bit machine, we can't typically get away
        // with a giant virtual address space reservation.
        // Instead we map the memory information bitmap
        // immediately after the data segment, large enough
        // to handle the entire 4GB address space (256 MB),
        // along with a reservation for an initial arena.
        // When that gets used up, we'll start asking the kernel
        // for any memory anywhere.

        // We want to start the arena low, but if we're linked
        // against C code, it's possible global constructors
        // have called malloc and adjusted the process' brk.
        // Query the brk so we can avoid trying to map the
        // arena over it (which will cause the kernel to put
        // the arena somewhere else, likely at a high
        // address).
        procBrk := sbrk0()

        // If we fail to allocate, try again with a smaller arena.
        // This is necessary on Android L where we share a process
        // with ART, which reserves virtual memory aggressively.
        // In the worst case, fall back to a 0-sized initial arena,
        // in the hope that subsequent reservations will succeed.
        arenaSizes := []uintptr{
            512 << 20,
            256 << 20,
            128 << 20,
            0,
        }

        for _, arenaSize := range arenaSizes {
            // SysReserve treats the address we ask for, end, as a hint,
            // not as an absolute requirement. If we ask for the end
            // of the data segment but the operating system requires
            // a little more space before we can start allocating, it will
            // give out a slightly higher pointer. Except QEMU, which
            // is buggy, as usual: it won't adjust the pointer upward.
            // So adjust it upward a little bit ourselves: 1/4 MB to get
            // away from the running binary image and then round up
            // to a MB boundary.
            p = round(firstmoduledata.end+(1<<18), 1<<20)
            pSize = bitmapSize + spansSize + arenaSize + _PageSize
            if p <= procBrk && procBrk < p+pSize {
                // Move the start above the brk,
                // leaving some room for future brk
                // expansion.
                p = round(procBrk+(1<<20), 1<<20)
            }
            p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
            if p != 0 {
                break
            }
        }
        if p == 0 {
            throw("runtime: cannot reserve arena virtual address space")
        }
    }

    // PageSize can be larger than OS definition of page size,
    // so SysReserve can give us a PageSize-unaligned pointer.
    // To overcome this we ask for PageSize more and round up the pointer.
    p1 := round(p, _PageSize)
    pSize -= p1 - p

    spansStart := p1
    p1 += spansSize
    mheap_.bitmap = p1 + bitmapSize
    p1 += bitmapSize
    if sys.PtrSize == 4 {
        // Set arena_start such that we can accept memory
        // reservations located anywhere in the 4GB virtual space.
        mheap_.arena_start = 0
    } else {
        mheap_.arena_start = p1
    }
    mheap_.arena_end = p + pSize
    mheap_.arena_used = p1
    mheap_.arena_alloc = p1
    mheap_.arena_reserved = reserved

    if mheap_.arena_start&(_PageSize-1) != 0 {
        println("bad pagesize", hex(p), hex(p1), hex(spansSize), hex(bitmapSize), hex(_PageSize), "start", hex(mheap_.arena_start))
        throw("misrounded allocation in mallocinit")
    }

    // Initialize the rest of the allocator.
    mheap_.init(spansStart, spansSize)
    _g_ := getg()
    _g_.m.mcache = allocmcache()
}
```

## 堆内存分配流程

```mermaid
```

```go
// 分配一个小对象会穿过几个层次的 cache:
//
//    1. 四舍五入 size 到合适的 size class 之一
//       然后在当前 P 的 mcache 中查找对应的 mspan。
//       扫描 mspan 的 free bitmap 寻找空闲的 slot。
//       如果有空闲 slot，分配之。
//       这个过程全程无需获取锁。
//
//    2. 如果 mspan 没有空闲 slot。从 mcentral 的 mspan 列表中
//       获取一个大小至少为所需 size class 大小的新 mspan。
//       作为获取一个完整的 span 的代价会对 mcentral 加锁。
//
//    3. 如果 mcentral 的 mspan 列表为空，从 mheap 中获取一系列
//       页，来为 mspan 服务。
//
//    4. 如果 mheap 是空，或者没有足够大的页，
//       向操作系统要求分配一组新的页(至少 1MB)。
//       分配一大笔页的成本主要在于和操作系统交互。
//
// Sweeping 一个 mspan 并释放空闲对象走的是类似的逻辑层次:
//
//    1. 如果 mspan 由于响应分配动作，而正在被 swept，该 mspan 会被返回到 mcache
//       以满足内存分配的需求。
//
//    2. 否则的话，如果 mspan 中仍然有分配过的内存对象，
//       则该 mspan 会被放进 mcentral 的对应 mspan size class 大小的空闲列表。
//
//    3. 否则的话，如果 mspan 中所有对象均为空，mspan 则为 "idle" 状态，
//       这时候会把 mspan 返回给 mheap 并不再对应任何 size class。
//       这一步可能会把该 mspan 和其相邻的空闲 mspan 进行合并。
//
//    4. 如果一个 mspan 抽象 idle 了很长一段时间的话，会将其所包含的页返还给操作系统。
//
// 分配和释放大对象会越过 mcache 和 mcentral，直接使用全局 mheap，
//
// mspan 中的空闲对象槽只有在 mspan.needzero 为 false 时才会被置零。
// 如果 needzero 是 true，则会在分配对象时再将其置零。延迟置零的操作有很多好处:
//
//    1. 栈帧分配能够避免一次性全置零。
//
//    2. 在空间局部性上更合理，因为程序很可能马上就要对置零的内存进行写入操作。
//
//    3. 如果内存页不被复用，那么我们永远不需要清零操作。
```

```go
// new(type) 会被翻译为 newobject，但是也不一定，要看逃逸分析的结果
// 编译前端和 SSA 后端都知道该函数的签名
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}
```

```go
// 分配 size 个字节
// 小对象会从 per-P 缓存的 free list 中进行分配
// 大对象 > 32 KB 会直接在堆上分配
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    // _GCmarktermination 阶段不能进行内存分配
    if gcphase == _GCmarktermination {
        throw("mallocgc called with gcphase == _GCmarktermination")
    }

    // 所有大小为 0 的对象都使用同一块内存地址
    if size == 0 {
        return unsafe.Pointer(&zerobase)
    }

    // 如果正在执行 GC，那么需要有一个 assistG 负责这次内存分配
    // 如果没有在执行 GC，那么这个 G 可以是 nil
    var assistG *g
    if gcBlackenEnabled != 0 {
        // 要求当前的用户 G 负责这次分配
        assistG = getg()
        if assistG.m.curg != nil {
            assistG = assistG.m.curg
        }

        // 向当前的 G 要求付款。在 mallocgc 的最后阶段需要对这笔“债务”负责
        assistG.gcAssistBytes -= int64(size)

        if assistG.gcAssistBytes < 0 {
            // 当前的 G 还处于欠债状态。协助 GC，在分配内存前先还上这笔账
            // 这个动作必须发生在关闭抢占之前
            gcAssistAlloc(assistG)
        }
    }

    // 设置 mp.mallocing 状态，以使其避免被 GC 抢占
    mp := acquirem()
    if mp.mallocing != 0 {
        throw("malloc deadlock")
    }
    if mp.gsignal == getg() {
        throw("malloc during signal")
    }
    mp.mallocing = 1

    shouldhelpgc := false
    dataSize := size
    c := gomcache()
    var x unsafe.Pointer
    noscan := typ == nil || typ.kind&kindNoPointers != 0
    if size <= maxSmallSize {
        if noscan && size < maxTinySize {
            // Tiny allocator.
            //
            // Tiny allocator 会将几个 tiny 的内存分配请求组合为单个内存块。
            // 当该内存块中所有子对象都不可达时，便会释放这部分内存块。
            // 子对象必须是 noscan 类型(不包含指针)，这样可以确保浪费的内存是可控的。
            //
            // 用来组合小对象的内存块的大小是可调(tuning)的。
            // 当前的设置是 16 bytes，最坏情况下会浪费 2x 内存(当所有子对象中只有一个对象是可达状态时)。
            // 如果修改为 8 bytes ，虽然完全没有浪费，但是却不太可能进行组合操作。
            // 32 bytes 能够提高合并的可能性，但是最坏情况下会造成 4x 浪费。
            // 不考虑 block 大小的话，最好的情况下是 8x。 // The best case winning is 8x regardless of block size. ??
            //
            // 从 tiny allocator 中获得的内存不能被显式释放。
            // 因此当一个对象需要被显式释放时，我们要确保它的 size >= maxTinySize
            //
            // 当对象都是由 tiny allocator 分配时，会形成 SetFinalizer 有一种特殊 case，
            // 这种情况下会允许在一个内存块中设置内部字节的 finalizers
            //
            // 实现 tiny allocator 的主要目标是对那些小字符串和独立的逃逸变量。
            // 在一个 json benchmark 中，这个 allocator 会将其性能提升 ~12%
            // 并且使堆大小降低了 ~20%。
            off := c.tinyoffset
            // Align tiny pointer for required (conservative) alignment.
            if size&7 == 0 {
                off = round(off, 8)
            } else if size&3 == 0 {
                off = round(off, 4)
            } else if size&1 == 0 {
                off = round(off, 2)
            }
            if off+size <= maxTinySize && c.tiny != 0 {
                // The object fits into existing tiny block.
                x = unsafe.Pointer(c.tiny + off)
                c.tinyoffset = off + size
                c.local_tinyallocs++
                mp.mallocing = 0
                releasem(mp)
                return x
            }
            // Allocate a new maxTinySize block.
            span := c.alloc[tinySpanClass]
            v := nextFreeFast(span)
            if v == 0 {
                v, _, shouldhelpgc = c.nextFree(tinySpanClass)
            }
            x = unsafe.Pointer(v)
            (*[2]uint64)(x)[0] = 0
            (*[2]uint64)(x)[1] = 0
            // See if we need to replace the existing tiny block with the new one
            // based on amount of remaining free space.
            if size < c.tinyoffset || c.tiny == 0 {
                c.tiny = uintptr(x)
                c.tinyoffset = size
            }
            size = maxTinySize
        } else {
            // Small allocator.
            var sizeclass uint8
            if size <= smallSizeMax-8 {
                sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
            } else {
                sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
            }
            size = uintptr(class_to_size[sizeclass])
            spc := makeSpanClass(sizeclass, noscan)
            span := c.alloc[spc]
            v := nextFreeFast(span)
            if v == 0 {
                v, span, shouldhelpgc = c.nextFree(spc)
            }
            x = unsafe.Pointer(v)
            if needzero && span.needzero != 0 {
                memclrNoHeapPointers(unsafe.Pointer(v), size)
            }
        }
    } else {
        // Small allocator.
        var s *mspan
        shouldhelpgc = true
        systemstack(func() {
            s = largeAlloc(size, needzero, noscan)
        })
        s.freeindex = 1
        s.allocCount = 1
        x = unsafe.Pointer(s.base())
        size = s.elemsize
    }

    var scanSize uintptr
    if !noscan {
        // If allocating a defer+arg block, now that we've picked a malloc size
        // large enough to hold everything, cut the "asked for" size down to
        // just the defer header, so that the GC bitmap will record the arg block
        // as containing nothing at all (as if it were unused space at the end of
        // a malloc block caused by size rounding).
        // The defer arg areas are scanned as part of scanstack.
        if typ == deferType {
            dataSize = unsafe.Sizeof(_defer{})
        }
        heapBitsSetType(uintptr(x), size, dataSize, typ)
        if dataSize > typ.size {
            // Array allocation. If there are any
            // pointers, GC has to scan to the last
            // element.
            if typ.ptrdata != 0 {
                scanSize = dataSize - typ.size + typ.ptrdata
            }
        } else {
            scanSize = typ.ptrdata
        }
        c.local_scan += scanSize
    }

    // Ensure that the stores above that initialize x to
    // type-safe memory and set the heap bits occur before
    // the caller can make x observable to the garbage
    // collector. Otherwise, on weakly ordered machines,
    // the garbage collector could follow a pointer to x,
    // but see uninitialized memory or stale heap bits.
    publicationBarrier()

    // Allocate black during GC.
    // All slots hold nil so no scanning is needed.
    // This may be racing with GC so do it atomically if there can be
    // a race marking the bit.
    if gcphase != _GCoff {
        gcmarknewobject(uintptr(x), size, scanSize)
    }

    mp.mallocing = 0
    releasem(mp)

    if assistG != nil {
        // Account for internal fragmentation in the assist
        // debt now that we know it.
        assistG.gcAssistBytes -= int64(size - dataSize)
    }

    if shouldhelpgc {
        if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
            gcStart(gcBackgroundMode, t)
        }
    }

    return x
}
```

## 栈内存分配流程

TODO

## 堆外内存

```go
// notInHeap is off-heap memory allocated by a lower-level allocator
// like sysAlloc or persistentAlloc.
//
// In general, it's better to use real types marked as go:notinheap,
// but this serves as a generic type for situations where that isn't
// possible (like in the allocators).
//
// TODO: Use this as the return type of sysAlloc, persistentAlloc, etc?
//
//go:notinheap
type notInHeap struct{}

func (p *notInHeap) add(bytes uintptr) *notInHeap {
    return (*notInHeap)(unsafe.Pointer(uintptr(unsafe.Pointer(p)) + bytes))
}
```

### 堆外内存用法
