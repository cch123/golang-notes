# map

```
          ┌─────────────┐                                                                                                                                                                                                           
          │    hmap     │                                                                                                                                                                                                           
          ├─────────────┴──────────────────┐           ┌───────────────┐                                        ┌─────────┐                               ┌─────────┐                                                               
          │           count int            │           │               │                     ┌─────────────────▶│  bmap   │                          ┌───▶│  bmap   │                                                               
          │                                │           │               ▼                     │                  ├─────────┴─────────────────────┐    │    ├─────────┴─────────────────────┐                                         
          ├────────────────────────────────┤           │    ────────┬─────┐                  │                  │   tophash [bucketCnt]uint8    │    │    │   tophash [bucketCnt]uint8    │                                         
          │          flags uint8           │           │       ▲    │  0  │                  │                  │                               │    │    │                               │                                         
          │                                │           │       │    │     │──────────────────┘                  ├──────────┬────────────────────┤    │    ├──────────┬────────────────────┤                                         
          ├────────────────────────────────┤           │       │    ├─────┤                                     │   keys   │                    │    │    │   keys   │                    │                                         
          │            B uint8             │           │       │    │  1  │                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                         
          │                                │           │       │    │     │──────────────────┐                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  ├───┴───┴──┬┴───┴───┴───┴───┴───┤    │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                         
          │        noverflow uint16        │           │       │    │  2  │                  │                  │  values  │                    │    │    │  values  │                    │                                         
          │                                │           │       │    │     │                  │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                         
          │          hash0 uint32          │           │       │    │  3  │                  │                  ├───┴───┴───┴───┴───┴───┴───┴───┤    │    ├───┴───┴───┴───┴───┴───┴───┴───┤                                         
          │                                │           │       │    │     │                  │                  │        overflow *bmap         │    │    │        overflow *bmap         │                                         
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │                               │────┘    │                               │                                         
          │     buckets unsafe.Pointer     │           │       │    │  4  │                  │                  ├─────────┬─────────────────────┘         └───────────────────────────────┘                                         
          │                                │───────────┘       │    │     │                  └─────────────────▶│  bmap   │                                                                                                         
          ├────────────────────────────────┤                        ├─────┤                                     ├─────────┴─────────────────────┐                                                                                   
          │   oldbuckets unsafe.Pointer    │                        │  5  │                                     │   tophash [bucketCnt]uint8    │                                                                                   
          │                                │                        │     │                                     │                               │                                                                                   
          ├────────────────────────────────┤         size = 2 ^ B   ├─────┤                                     ├──────────┬────────────────────┤                                                                                   
          │       nevacuate uintptr        │                        │  6  │                                     │   keys   │                    │                                                                                   
          │                                │                        │     │                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
          ├────────────────────────────────┤                   │    ├─────┤                                     │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
          │        extra *mapextra         │                   │    │  7  │                                     ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                                                                   
       ┌──│                                │                   │    │     │                                     │  values  │                    │                                                                                   
       │  └────────────────────────────────┘                   │    └─────┘                                     ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
       │                                                       │      ....                                      │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
       │                                                       │                                                ├───┴───┴───┴───┴───┴───┴───┴───┤                                                                                   
       │                                                       │    ┌─────┐                                     │        overflow *bmap         │                                                                                   
       │                                                       │    │ 61  │                                     │                               │                                                                                   
       │                                                       │    │     │                                     └───────────────────────────────┘                                                                                   
       ▼                                                       │    ├─────┤                                               ............                                                                                              
┌─────────────┐                                                │    │ 62  │                                     ┌─────────┐                               ┌─────────┐                              ┌─────────┐                      
│  mapextra   │                                                │    │     │────────────────────────────────────▶│  bmap   │                          ┌───▶│  bmap   │                         ┌───▶│  bmap   │                      
├─────────────┴──────────────┐                                 │    ├─────┤                                     ├─────────┴─────────────────────┐    │    ├─────────┴─────────────────────┐   │    ├─────────┴─────────────────────┐
│     overflow *[]*bmap      │                                 │    │ 63  │                                     │   tophash [bucketCnt]uint8    │    │    │   tophash [bucketCnt]uint8    │   │    │   tophash [bucketCnt]uint8    │
│                            │                                 ▼    │     │──────────────────┐                  │                               │    │    │                               │   │    │                               │
├────────────────────────────┤                              ────────┴─────┘                  │                  ├──────────┬────────────────────┤    │    ├──────────┬────────────────────┤   │    ├──────────┬────────────────────┤
│    oldoverflow *[]*bmap    │                                                               │                  │   keys   │                    │    │    │   keys   │                    │   │    │   keys   │                    │
│                            │                                                               │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤   │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤
├────────────────────────────┤                                                               │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
│     nextoverflow *bmap     │                                                               │                  ├───┴───┴──┬┴───┴───┴───┴───┴───┤    │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤   │    ├───┴───┴──┬┴───┴───┴───┴───┴───┤
│                            │                                                               │                  │  values  │                    │    │    │  values  │                    │   │    │  values  │                    │
└────────────────────────────┘                                                               │                  ├───┬───┬──┴┬───┬───┬───┬───┬───┤    │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤   │    ├───┬───┬──┴┬───┬───┬───┬───┬───┤
                                                                                             │                  │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │    │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   │    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
                                                                                             │                  ├───┴───┴───┴───┴───┴───┴───┴───┤    │    ├───┴───┴───┴───┴───┴───┴───┴───┤   │    ├───┴───┴───┴───┴───┴───┴───┴───┤
                                                                                             │                  │        overflow *bmap         │    │    │        overflow *bmap         │   │    │        overflow *bmap         │
                                                                                             │                  │                               │────┘    │                               │───┘    │                               │
                                                                                             │                  ├─────────┬─────────────────────┘         └───────────────────────────────┘        └───────────────────────────────┘
                                                                                             └─────────────────▶│  bmap   │                                                                                                         
                                                                                                                ├─────────┴─────────────────────┐                                                                                   
                                                                                                                │   tophash [bucketCnt]uint8    │                                                                                   
                                                                                                                │                               │                                                                                   
                                                                                                                ├──────────┬────────────────────┤                                                                                   
                                                                                                                │   keys   │                    │                                                                                   
                                                                                                                ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
                                                                                                                │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
                                                                                                                ├───┴───┴──┬┴───┴───┴───┴───┴───┤                                                                                   
                                                                                                                │  values  │                    │                                                                                   
                                                                                                                ├───┬───┬──┴┬───┬───┬───┬───┬───┤                                                                                   
                                                                                                                │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │                                                                                   
                                                                                                                ├───┴───┴───┴───┴───┴───┴───┴───┤                                                                                   
                                                                                                                │        overflow *bmap         │                                                                                   
                                                                                                                │                               │                                                                                   
                                                                                                                └───────────────────────────────┘                                                                                   
```

形如:

```go
make(map[k]v, hint)
```

的代码，在 hint <= 7 时，会调用 makemap_small 来进行初始化，如果 hint > 7，则调用 makemap。

```go
make(map[k]v)
```

不提供 hint 的代码，编译器始终会调用 makemap_small 来初始化。

```go
// make(map[k]v, hint)
// 如果编译器认为 map 和第一个 bucket 可以直接创建在栈上，h 和 bucket 可能都是非空
// h != nil，可以直接在 h 内创建 map
// 如果 h.buckets != nil，其指向的 bucket 可以作为第一个 bucket 来使用
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 在 64 位系统上 hmap 结构体大小为 48 字节
    // 32 位系统上是 28 字节
    if sz := unsafe.Sizeof(hmap{}); sz != 8+5*sys.PtrSize {
        println("runtime: sizeof(hmap) =", sz, ", t.hmap.size =", t.hmap.size)
        throw("bad hmap size")
    }

    if hint < 0 || hint > int(maxSliceCap(t.bucket.size)) {
        hint = 0
    }

    // 初始化 hmap
    if h == nil {
        h = (*hmap)(newobject(t.hmap))
    }
    h.hash0 = fastrand()

    // 按照提供的元素个数，找一个可以放得下这么多元素的 B 值
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配初始的 hash table
    // 如果 B == 0，buckets 字段会由 mapassign 来 lazily 分配
    // 因为如果 hint 很大的话，对这部分内存归零会花比较长时间
    if h.B != 0 {
        var nextOverflow *bmap
        h.buckets, nextOverflow = makeBucketArray(t, h.B)
        if nextOverflow != nil {
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }

    return h
}
```

元素访问有 mapaccess1，mapaccess2，mapaccessK，但几个方法都差不多，只差别在返回内容上，我们来看看 mapaccess2:

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
    if raceenabled && h != nil {
        callerpc := getcallerpc()
        pc := funcPC(mapaccess2)
        racereadpc(unsafe.Pointer(h), callerpc, pc)
        raceReadObjectPC(t.key, key, callerpc, pc)
    }
    if msanenabled && h != nil {
        msanread(key, t.key.size)
    }
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0]), false
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            // There used to be half as many buckets; mask down one more power of two.
            m >>= 1
        }
        oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }
    top := tophash(hash)
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey {
                k = *((*unsafe.Pointer)(k))
            }
            if alg.equal(key, k) {
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                if t.indirectvalue {
                    v = *((*unsafe.Pointer)(v))
                }
                return v, true
            }
        }
    }
    return unsafe.Pointer(&zeroVal[0]), false
}
```

显然，这就是我们平常在写如下代码时:

```go
v, ok := m[k]
```

所使用的底层函数。
