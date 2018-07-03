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

## 初始化

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

## 元素访问

元素访问有 mapaccess1，mapaccess2，mapaccessK，但几个方法都差不多，只差别在返回内容上，我们来看看 mapaccess2:

```go
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
    // map 为空，或者元素数为 0，直接返回未找到
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0]), false
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    alg := t.key.alg
    // 不同类型的 key，所用的 hash 算法是不一样的
    // 具体可以参考 algarray
    hash := alg.hash(key, uintptr(h.hash0))
    // 如果 B = 3，那么结果用二进制表示就是 111
    // 如果 B = 4，那么结果用二进制表示就是 1111
    m := bucketMask(h.B)
    // 按位 &，可以 select 出对应的 bucket
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
    // 会用到 h.oldbuckets 时，说明 map 发生了扩容
    // 这时候，新的 buckets 里可能还没有老的内容
    // 所以一定要在老的里面找，否则有可能发生“消失”的诡异现象
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            // 说明之前只有一半的 bucket，需要除 2
            m >>= 1
        }
        oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }
    // tophash 取其高 8bit 的值
    top := tophash(hash)
    for ; b != nil; b = b.overflow(t) {
        // 一个 bucket 在存储满 8 个元素后，就再也放不下了
        // 这时候会创建新的 bucket
        // 挂在原来的 bucket 的 overflow 指针成员上
        for i := uintptr(0); i < bucketCnt; i++ {
            // 循环对比 bucket 中的 tophash 数组
            // 如果找到了相等的 tophash，那说明就是这个 bucket 了
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

    // 所有 bucket 都没有找到，返回零值和 false
    return unsafe.Pointer(&zeroVal[0]), false
}
```

显然，这就是我们平常在写如下代码时:

```go
v, ok := m[k]
```

所使用的底层函数。

## 赋值

```go
// 和 mapaccess 函数差不多，但在没有找到 key 时，会为 key 分配一个新的槽位
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil {
        // nil map 不能进行赋值操作
        panic(plainError("assignment to entry in nil map"))
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }

    // 调用对应类型的 hash 算法
    alg := t.key.alg
    hash := alg.hash(key, uintptr(h.hash0))

    // 调用 alg.hash 设置 hashWriting 的 flag，因为 alg.hash 可能会 panic
    // 这时候我们没法完成一次写操作
    h.flags |= hashWriting

    if h.buckets == nil {
        // 分配第一个 buckt
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
    }

again:
    // 计算低 8 位 hash，根据计算出的 bucketMask 选择对应的 bucket
    // mask : 1111111
    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork(t, h, bucket)
    }
    // 计算出存储的 bucket 的内存位置
    // pos = start + bucketNumber * bucetsize
    // 这里的命名不太好，bucket 其实是 bucketNumber
    b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    // 计算高 8 位 hash
    top := tophash(hash)

    var inserti *uint8
    var insertk unsafe.Pointer
    var val unsafe.Pointer
    for {
        for i := uintptr(0); i < bucketCnt; i++ {
            // 遍历 8 个 bucket 中的元素
            // 这里的 bucketCnt 是全局常量
            // 其实叫 bucketElemCnt 更合适
            if b.tophash[i] != top {
                if b.tophash[i] == empty && inserti == nil {
                    // 如果这个槽位没有被占，说明可以往这里塞 key 和 value
                    inserti = &b.tophash[i] // tophash 的插入位置
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
                    val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey {
                k = *((*unsafe.Pointer)(k))
            }
            // 如果相同的 hash 位置的 key 和要插入的 key 字面上不相等
            // 注意，hash 碰撞的时候就可能会走到这里的
            if !alg.equal(key, k) {
                continue
            }
            // 对应的位置已经有 key 了，直接更新就行
            if t.needkeyupdate {
                typedmemmove(t.key, k, key)
            }
            val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
            goto done
        }
        // bucket 的 8 个槽没有满足条件的能插入或者能更新的，去 overflow 里继续找
        ovf := b.overflow(t)
        // 如果 overflow 为 nil，说明到了 overflow 链表的末端了
        if ovf == nil {
            break
        }
        // 赋值为链表的下一个元素，继续循环
        b = ovf
    }

    // 没有找到 key，分配新的空间

    // 如果触发了最大的 load factor，或者已经有太多 overflow buckets
    // 并且这个时刻没有在进行 growing 的途中，那么就开始 growing
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again // Growing the table invalidates everything, so try again
    }

    if inserti == nil {
        // 前面在桶里找的时候，没有找到能塞这个 tophash 的位置
        // 说明当前所有 buckets 都是满的，分配一个新的 bucket
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        val = add(insertk, bucketCnt*uintptr(t.keysize))
    }

    // 把新的 key 和 value 存储到应插入的位置
    if t.indirectkey {
        kmem := newobject(t.key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    if t.indirectvalue {
        vmem := newobject(t.elem)
        *(*unsafe.Pointer)(val) = vmem
    }
    typedmemmove(t.key, insertk, key)
    *inserti = top
    h.count++

done:
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    h.flags &^= hashWriting
    if t.indirectvalue {
        val = *((*unsafe.Pointer)(val))
    }
    return val
}
```

这里有件比较奇怪的事情，mapassign 并没有把用户 `m[k] = v` 时的 v 写入到 map 的 value 区域，而是直接返回了这个值所应该在的内存地址。那么把 v 拷贝到该内存区域的操作是在哪里做的呢？

```go
    var a = make(map[int]int, 7)
    for i := 0; i < 1000; i++ {
        a[i] = 99999
    }
```

看看生成的汇编部分:

```go
    0x003f 00063 (m.go:9)    MOVQ    DX, (SP) // 第一个参数
    0x0043 00067 (m.go:9)    MOVQ    AX, 8(SP) // 第二个参数
    0x0048 00072 (m.go:9)    MOVQ    CX, 16(SP) // 第三个参数
    0x004d 00077 (m.go:9)    PCDATA    $0, $1 // GC 相关
    0x004d 00077 (m.go:9)    CALL    runtime.mapassign_fast64(SB) // 调用函数
    0x0052 00082 (m.go:9)    MOVQ    24(SP), AX // 返回值，即 value 应该存放的内存地址
    0x0057 00087 (m.go:9)    MOVQ    $99999, (AX) // 把 99999 放入该地址中
```

赋值的最后一步实际上是编译器额外生成的汇编指令来完成的，可见靠 runtime 有些工作是没有做完的。这里和 go 在函数调用时插入 prologue 和 epilogue 是类似的。编译器和 runtime 配合，才能完成一些复杂的工作。

## 删除

## 扩容

## 其它

针对 32 位、64 位 和 string 类型的 map 元素的访问、赋值、删除、扩容，Go 内部有都有对应的优化函数，比如 mapaccess1 对应有 mapaccess1_fast64，mapaccess1_fast32，mapaccess1_faststr。mapassign 对应有 mapassign_fast64，mapassign_fast32 和 mapassign_faststr。

但这些优化函数长得都差不多，但不知为何官方没有用脚本来做这些优化函数的生成工作。可见有时候 Go team 的复制粘贴功力也是蛮强的。
