# map

```
          ┌─────────────┐                                                                                                                         
          │    hmap     │                                                                                       ┌─────────┐                       
          ├─────────────┴──────────────────┐           ┌───────────────┐                     ┌─────────────────▶│  bmap   │                       
          │           count int            │           │               │                     │                  ├─────────┴──────────────────────┐
          │                                │           │               ▼                     │                  │    tophash [bucketCnt]uint8    │
          ├────────────────────────────────┤           │    ────────┬─────┐                  │                  │                                │
          │          flags uint8           │           │       ▲    │  0  │                  │                  ├────────────────────────────────┤
          │                                │           │       │    │     │──────────────────┘                  │      keys unsafe.Pointer       │
          ├────────────────────────────────┤           │       │    ├─────┤                                     │                                │
          │            B uint8             │           │       │    │  1  │                                     ├────────────────────────────────┤
          │                                │           │       │    │     │──────────────────┐                  │     values unsafe.Pointer      │
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │                                │
          │        noverflow uint16        │           │       │    │  2  │                  │                  ├────────────────────────────────┤
          │                                │           │       │    │     │                  │                  │         overflow *bmap         │
          ├────────────────────────────────┤           │       │    ├─────┤                  │                  │                                │
          │          hash0 uint32          │           │       │    │  3  │                  │                  ├─────────┬──────────────────────┘
          │                                │           │       │    │     │                  └─────────────────▶│  bmap   │                       
          ├────────────────────────────────┤           │       │    ├─────┤                                     ├─────────┴──────────────────────┐
          │     buckets unsafe.Pointer     │           │       │    │  4  │                                     │    tophash [bucketCnt]uint8    │
          │                                │───────────┘       │    │     │                                     │                                │
          ├────────────────────────────────┤                        ├─────┤                                     ├────────────────────────────────┤
          │   oldbuckets unsafe.Pointer    │                        │  5  │                                     │      keys unsafe.Pointer       │
          │                                │                        │     │                                     │                                │
          ├────────────────────────────────┤         size = 2 ^ B   ├─────┤                                     ├────────────────────────────────┤
          │       nevacuate uintptr        │                        │  6  │                                     │     values unsafe.Pointer      │
          │                                │                        │     │                                     │                                │
          ├────────────────────────────────┤                   │    ├─────┤                                     ├────────────────────────────────┤
          │        extra *mapextra         │                   │    │  7  │                                     │         overflow *bmap         │
       ┌──│                                │                   │    │     │                                     │                                │
       │  └────────────────────────────────┘                   │    └─────┘                                     └────────────────────────────────┘
       │                                                       │      ....                                                   ........             
       │                                                       │                                                ┌─────────┐                       
       │                                                       │    ┌─────┐                  ┌─────────────────▶│  bmap   │                       
       │                                                       │    │ 61  │                  │                  ├─────────┴──────────────────────┐
       │                                                       │    │     │                  │                  │    tophash [bucketCnt]uint8    │
       ▼                                                       │    ├─────┤                  │                  │                                │
┌─────────────┐                                                │    │ 62  │                  │                  ├────────────────────────────────┤
│  mapextra   │                                                │    │     │──────────────────┘                  │      keys unsafe.Pointer       │
├─────────────┴──────────────┐                                 │    ├─────┤                                     │                                │
│     overflow *[]*bmap      │                                 │    │ 63  │                                     ├────────────────────────────────┤
│                            │                                 ▼    │     │──────────────────┐                  │     values unsafe.Pointer      │
├────────────────────────────┤                              ────────┴─────┘                  │                  │                                │
│    oldoverflow *[]*bmap    │                                                               │                  ├────────────────────────────────┤
│                            │                                                               │                  │         overflow *bmap         │
├────────────────────────────┤                                                               │                  │                                │
│     nextoverflow *bmap     │                                                               │                  ├─────────┬──────────────────────┘
│                            │                                                               └─────────────────▶│  bmap   │                       
└────────────────────────────┘                                                                                  ├─────────┴──────────────────────┐
                                                                                                                │    tophash [bucketCnt]uint8    │
                                                                                                                │                                │
                                                                                                                ├────────────────────────────────┤
                                                                                                                │      keys unsafe.Pointer       │
                                                                                                                │                                │
                                                                                                                ├────────────────────────────────┤
                                                                                                                │     values unsafe.Pointer      │
                                                                                                                │                                │
                                                                                                                ├────────────────────────────────┤
                                                                                                                │         overflow *bmap         │
                                                                                                                │                                │
                                                                                                                └────────────────────────────────┘
```

形如:

```go
make(map[k]v, hint)
```

的代码，在 hint <= 7 时，会调用 makemap_small 来进行初始化，如果 hint > 7，则调用 makemap。

```go
make(map[k]v)
```

不提供 hint 的代码，编译器始终会调用 makeap_small 来初始化。

```go
// make(map[k]v, hint)
// 如果编译器认为 map 和第一个 bucket 可以直接创建在栈上，h 和 bucket 可能都是非空
// h != nil，可以直接在 h 内创建 map
// 如果 h.buckets != nil，其指向的 bucket 可以作为第一个 bucket 来使用
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 在 64 位系统上 hmap 大小为 48 字节
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
