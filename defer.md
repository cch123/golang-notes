# defer

defer 可以被拆分为两个步骤 `runtime.deferproc` 和 `runtime.deferreturn`。在用户函数中调用:

```go
func f() int {
    defer println(1)
}
```

`defer println(1)` 被翻译两个过程，先执行 `runtime.deferproc` 生成 println 函数及其相关参数的描述结构体，然后将其挂载到当前 g 的 `_defer` 指针上。先来看看 deferproc 的过程:

## deferproc

```go
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
    sp := getcallersp(unsafe.Pointer(&siz))
    argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
    callerpc := getcallerpc() // 存储的是 caller 中，call deferproc 的下一条指令的地址

    d := newdefer(siz)
    if d._panic != nil {
        throw("deferproc: d.panic != nil after newdefer")
    }
    d.fn = fn
    d.pc = callerpc
    d.sp = sp
    switch siz {
    case 0:
        // Do nothing.
    case sys.PtrSize:
        *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
    default:
        memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
    }

    return0()
    // No code can go here - the C return register has
    // been set and must not be clobbered.
}
```

比较关键的就是 newdefer:

```go
func newdefer(siz int32) *_defer {
    var d *_defer
    sc := deferclass(uintptr(siz))
    gp := getg()
    if sc < uintptr(len(p{}.deferpool)) {
        // 从 p 结构体的 deferpool 中获取可用的 defer struct
        // 代码比较简单，省略
    }
    if d == nil {
        // 上面没有成功获取到可用的 defer struct
        // 因此需要切换到 g0 生成新的 defer struct
        systemstack(func() {
            total := roundupsize(totaldefersize(uintptr(siz)))
            d = (*_defer)(mallocgc(total, deferType, true))
        })
    }
    // defer func 的参数大小
    d.siz = siz
    // 链表链接
    // 后 defer 的在前，类似一个栈结构
    d.link = gp._defer
    // 修改当前 g 的 defer 结构体，指向新的 defer struct
    gp._defer = d
    return d
}

```

`newdefer` 函数返回的结构体为 `_defer`，简单看看其结构:

```go
type _defer struct {
    siz     int32 // 函数的参数总大小
    started bool  // TODO defer 是否已开始执行?
    sp      uintptr // 存储调用 defer 函数的函数的 sp 寄存器值
    pc      uintptr  // 存储 call deferproc 的下一条汇编指令的指令地址
    fn      *funcval // 描述函数的变长结构体，包括函数地址及参数
    _panic  *_panic // 正在执行 defer 的 panic 结构体
    link    *_defer // 链表指针
}
```

## deferreturn

```go
// 编译器会在调用过 defer 的函数的末尾插入对 deferreturn 的调用
// 如果有被 defer 的函数的话，这里会调用 runtime·jmpdefer 跳到对应的位置
// 实际效果是会一遍遍地调用 deferreturn 直到 _defer 链表被清空
// 这里不能进行栈分裂，因为我们要该函数的栈来调用 defer 函数

//go:nosplit
func deferreturn(arg0 uintptr) {
    gp := getg()
    d := gp._defer
    if d == nil {
        return
    }
    sp := getcallersp(unsafe.Pointer(&arg0))
    if d.sp != sp {
        return
    }

    switch d.siz {
    case 0:
        // Do nothing.
    case sys.PtrSize:
        *(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
    default:
        memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
    }

    // 把 defer 中的函数信息提取出来，清空链表上的该 _defer 节点
    fn := d.fn
    d.fn = nil
    // 指向 defer 链表下一个节点
    gp._defer = d.link
    // 清空 defer 结构体信息，并将该结构体存储到 p 的 deferpool 中
    freedefer(d)
    // 跳转
    jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

对链表的持续遍历是用 jmpdefer 实现的，看看 jmpdefer 的代码:

```go
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
    MOVQ    fv+0(FP), DX    // defer 的函数的地址
    MOVQ    argp+8(FP), BX    // caller sp
    LEAQ    -8(BX), SP    // caller sp after CALL
    MOVQ    -8(SP), BP    // 在 framepointer enable 的时候，不影响函数栈的结构
    SUBQ    $5, (SP)    // call 指令长度为 5，因此通过将 ret addr 减 5，能够使 deferreturn 自动被反复调用
    MOVQ    0(DX), BX
    JMP    BX    // 调用被 defer 的函数
```

在 jmpdefer 所调用的函数返回时，会回到调用 deferreturn 的函数，并重新执行 deferreturn，每次执行都会使 g 的 defer 链表表头被消耗掉，直到进入 deferreturn 时 `d == nil` 并返回。至此便完成了整个 defer 的流程。

这里比较粗暴的是直接把栈上存储的 pc 寄存器的值减了 5，注释中说是因为 call deferreturn 这条指令长度为 5，这是怎么算出来的呢:

```go
  defertest.go:8        0x104aca4               e82754fdff              CALL runtime.deferreturn(SB)
  defertest.go:8        0x104aca9               488b6c2418              MOVQ 0x18(SP), BP
```

0x104aca9 - 0x104aca4 = 5。所以这里理论上就是一个用汇编实现的非常 trick 的 for 循环。。。

Q && A:

Q: deferreturn + jmpdefer 就可以使 _defer 链表被消耗完毕，为什么还需要编译出多次 deferreturn 调用？

A: deferproc 和 deferreturn 是成对出现的，对于编译器的实现来说，这样做应该稍微简单一些。

参考资料：

https://ieevee.com/tech/2017/11/23/go-panic.html



<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
