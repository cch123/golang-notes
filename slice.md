# slice 和 array

要说 slice，那实在是太让人熟悉了，从功能上讲 slice 支持追加，按索引引用，按索引范围生成新的 slice，自动扩容等，和 C++ 或 Java 中的 Vector 有些类似，但也有一些区别。

不过 Go 里的 slice 还有一个底层数组的概念，这一点和其它语言不同。

```go
runtime/slice.go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

slice 的底层结构定义非常直观，指向底层数组的指针，当前长度 len 和当前 slice 的 cap。

数据指针不一定就是指向底层数组的首部，也可以指腰上:

```
                                                                   
       []int{1,3,4,5}                                              
                                                                   
       struct {                                                    
         array unsafe.Pointer --------------+                      
         len int                            |                      
         cap int                            |                      
       }                                    |                      
                                            |                      
                                            v                      
                                                                   
                               +------|-------|------|------+-----+
                               |      |  1    |  3   | 4    |  5  |
                               |      |       |      |      |     |
                               +------|-------|------|------+-----+
                                                 [5]int            
                                                                   
                                                                   
                                                                   

```

我们可以轻松地推断出，是可以有多个 slice 指向同一个底层数组的。一般情况下，一个 slice 的 cap 取决于其底层数组的长度。如果在元素追加过程中，底层数组没有更多的空间了，那么这时候就需要申请更大的底层数组，并发生数据拷贝。这时候的 slice 的底层数组的指针地址也会发生改变，务必注意。

## len 和 cap

Go 语言虽然将 len 和 cap 作为 slice 和 array 附带的 builtin 函数，但对这两个函数的调用实际上最终会被编译器直接计算出结果，并将值填到代码运行的位置上。所以 len 和 cap 更像是宏一样的东西，在 slice 和 array 的场景，会被直接展开为 sl->len 和 sl->cap 这样的结果。

## 源码分析

形如:

```go
var a = make([]int, 10, 20)
```

的代码，会被编译器翻译为 runtime.makeslice。

```go
func makeslice(et *_type, len, cap int) slice {
    maxElements := maxSliceCap(et.size)
    if len < 0 || uintptr(len) > maxElements {
        panic(errorString("makeslice: len out of range"))
    }

    if cap < len || uintptr(cap) > maxElements {
        panic(errorString("makeslice: cap out of range"))
    }

    p := mallocgc(et.size*uintptr(cap), et, true)
    return slice{p, len, cap}
}
```

如果是

```go
var a = new([]int)
```

这样的代码，则会被翻译为:

```go
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}
```

实在是太简单了，没啥可说的。mallocgc 函数会根据申请的内存大小，去对应的内存块链表上找合适的内存来进行分配，是 Go 自己改造的 tcmalloc 那一套。

内存拷贝:

```go
func slicecopy(to, fm slice, width uintptr) int {
    if fm.len == 0 || to.len == 0 {
        return 0
    }

    n := fm.len
    if to.len < n {
        n = to.len
    }

    if width == 0 {
        return n
    }

    size := uintptr(n) * width
    if size == 1 { // common case worth about 2x to do here
        // TODO: is this still worth it with new memmove impl?
        *(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
    } else {
        memmove(to.array, fm.array, size)
    }
    return n
}
```

最终最关键的就是 memmove，所有平台的 memmove 都是用汇编实现的，每个平台会针对 memmove 的目标的长度，选择对应平台的优化指令来进行内存移动。比如 intel 平台就有大量的 VMOVDQU，VMOVDQA 指令。不过别人都帮我们实现好了，大概瞟一眼就行，这里截个片段:

```go
.... 前面有很多看不懂的代码
    VMOVDQU    -0x40(SI), Y1
    VMOVDQU    -0x60(SI), Y2
    VMOVDQU    -0x80(SI), Y3
    SUBQ    $0x80, SI
....总之就是弃疗
    VMOVNTDQ    Y0, -0x20(DI)
    VMOVNTDQ    Y1, -0x40(DI)
    VMOVNTDQ    Y2, -0x60(DI)
....后面也有很多看不懂的代码
```

逻辑上稍微有点复杂的就只有 growslice，在对 slice 执行 append 操作时，如果 cap 不够用了，会导致 slice 扩容:

```go
func growslice(et *_type, old slice, cap int) slice {

    if et.size == 0 {
        if cap < old.cap {
            panic(errorString("growslice: cap out of range"))
        }
        return slice{unsafe.Pointer(&zerobase), old.len, cap}
    }

    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        // 注意这里的 1024 阈值
        if old.len < 1024 {
            newcap = doublecap
        } else {
            // Check 0 < newcap to detect overflow
            // and prevent an infinite loop.
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {
                newcap = cap
            }
        }
    }

    var overflow bool
    var lenmem, newlenmem, capmem uintptr
    const ptrSize = unsafe.Sizeof((*byte)(nil))
    switch et.size {
    case 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))
        overflow = uintptr(newcap) > _MaxMem
        newcap = int(capmem)
    case ptrSize:
        lenmem = uintptr(old.len) * ptrSize
        newlenmem = uintptr(cap) * ptrSize
        capmem = roundupsize(uintptr(newcap) * ptrSize)
        overflow = uintptr(newcap) > _MaxMem/ptrSize
        newcap = int(capmem / ptrSize)
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem = roundupsize(uintptr(newcap) * et.size)
        overflow = uintptr(newcap) > maxSliceCap(et.size)
        newcap = int(capmem / et.size)
    }

    if cap < old.cap || overflow || capmem > _MaxMem {
        panic(errorString("growslice: cap out of range"))
    }

    var p unsafe.Pointer
    if et.kind&kindNoPointers != 0 {
        p = mallocgc(capmem, nil, false)
        memmove(p, old.array, lenmem)
        // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
        // Only clear the part that will not be overwritten.
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        // Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
        p = mallocgc(capmem, et, true)
        if !writeBarrier.enabled {
            memmove(p, old.array, lenmem)
        } else {
            for i := uintptr(0); i < lenmem; i += et.size {
                typedmemmove(et, add(p, i), add(old.array, i))
            }
        }
    }

    return slice{p, old.len, newcap}
}
```

扩容时会判断 slice 的 cap 是不是已经大于 1024，如果在 1024 之内，会按二倍扩容。超过的话就是 1.25 倍扩容了。

slice 扩容必然会导致内存拷贝，如果是性能敏感的系统中，尽可能地提前分配好 slice 是较好的选择。

```go
var arr = make([]int, 0, 10)
```

## 值还是引用传递

网上有很多鬼扯的结论说 Go 的 slice 是按引用传递的，证据是类似下面这样的代码:

```go
func main() {
    var a = make([]int, 10)
    doSomeHappyThings(a)
    fmt.Println(a)
}

func doSomeHappyThings(sl []int) {
    if len(sl) > 0 {
        sl[0] = 1
    }
}
```

把 a 传入到 doSomeHappyThings，然后 a 的第一个元素就被修改了，进而认为在 Go 中，slice 是引用传递的。

但实际上并不是这样的，从汇编层面来讲，Go 的 slice 实际上是把三个参数传到函数内部了，这就类似于我们写一段 c 代码:

```c
void doSomeHappyThings(int * arr, int len, int cap) {
    if(len > 0) {
        arr[0] = 1
    }
}
```

所以如果你在函数内对这个 slice 进行 append 时导致了 slice 的扩容，那理论上外部是不受影响的，哪怕不扩容，也可能只影响底层数组，而不影响传入的 slice。举个例子:

```go
func main() {
    var arr = make([]int,0,10)
    doSomeHappyThings(arr)
    fmt.Println(arr, len(arr), cap(arr), "after return")
}

func doSomeHappyThings(arr []int) {
    arr = append(arr, 1)
    fmt.println(arr, "after append")
}
```

脏活编译器帮你做了一些，但也就导致有些结论不那么直观了。

当然，你可以尝试用汇编来写一个处理 slice 的函数。

a.s:

```go
#include "textflag.h"

// func sum(sl []int64) int64
TEXT ·sum(SB),NOSPLIT, $0-32
    MOVQ $0, SI
    MOVQ sl+0(FP), BX // &sl[0], addr of the first elem
    MOVQ sl+8(FP), CX // len(sl)
  INCQ CX

start:
    DECQ CX       // CX--
    JZ   done
    ADDQ (BX), SI // SI += *BX
    ADDQ $8, BX   // 指针移动
    JMP  start

done:
    // 返回地址是 24 是怎么得来的呢？
    // 可以通过 go tool compile -S math.go 得知
    // 在调用 sum1 函数时，会传入三个值，分别为:
    // slice 的首地址、slice 的 len， slice 的 cap
    // 不过我们这里的求和只需要 len，但 cap 依然会占用参数的空间
    // 就是 16(FP)
    MOVQ SI, ret+24(FP)
    RET
```

a.go:

```go
package main

import "fmt"

func sum(s []int64) int64

func main() {
    arr := []int64{1, 2, 3, 4, 10}
    sux := sum(arr)
    fmt.Println(sux)
}
```

## reflect 与 slice
对于 slice 的操作，除了计算 slice 的内存布局来获取 len, cap, data 以外，同样也可以利用 reflect 来实现
```go
    var b []byte = []byte("test")
    bh := (*reflect.SliceHeader)(unsafe.Pointer(&b))
    _ = bh.Len // len
    _ = bh.Cap // cap
    _ = bh.Data // 底层数组指针

    // 案例: 0 拷贝转换 slice 为 string
    var str string
    sh := (*reflect.StringHeader)(unsafe.Pointer(&str))
    // 从头转换 test
    sh.Data = bh.Data
    sh.Len = bh.Len
    // 从中间截取转换 est
    sh.Data = (uintptr)(unsafe.Pointer(&b[1]))
    sh.Len = bh.Len - 1
```

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
