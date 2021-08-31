# atomic 详解
> golang 中 atomic 类操作最终是使用 assembly 进行 cpu 指令调用实现的。本章将会对相应 api 及对应 assembly 进行分析
> 本章所有函数分析均以 amd64 架构为例

sync/atomic/doc.go
```go
// Copyright 2011 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package atomic provides low-level atomic memory primitives
// useful for implementing synchronization algorithms.
//
// These functions require great care to be used correctly.
// Except for special, low-level applications, synchronization is better
// done with channels or the facilities of the sync package.
// Share memory by communicating;
// don't communicate by sharing memory.
//
// The swap operation, implemented by the SwapT functions, is the atomic
// equivalent of:
//
//	old = *addr
//	*addr = new
//	return old
//
// The compare-and-swap operation, implemented by the CompareAndSwapT
// functions, is the atomic equivalent of:
//
//	if *addr == old {
//		*addr = new
//		return true
//	}
//	return false
//
// The add operation, implemented by the AddT functions, is the atomic
// equivalent of:
//
//	*addr += delta
//	return *addr
//
// The load and store operations, implemented by the LoadT and StoreT
// functions, are the atomic equivalents of "return *addr" and
// "*addr = val".
//
package atomic

import (
	"unsafe"
)

// BUG(rsc): On x86-32, the 64-bit functions use instructions unavailable before the Pentium MMX.
//
// On non-Linux ARM, the 64-bit functions use instructions unavailable before the ARMv6k core.
//
// On ARM, x86-32, and 32-bit MIPS,
// it is the caller's responsibility to arrange for 64-bit
// alignment of 64-bit words accessed atomically. The first word in a
// variable or in an allocated struct, array, or slice can be relied upon to be
// 64-bit aligned.

// SwapInt32 atomically stores new into *addr and returns the previous *addr value.
func SwapInt32(addr *int32, new int32) (old int32)

// SwapInt64 atomically stores new into *addr and returns the previous *addr value.
func SwapInt64(addr *int64, new int64) (old int64)

// SwapUint32 atomically stores new into *addr and returns the previous *addr value.
func SwapUint32(addr *uint32, new uint32) (old uint32)

// SwapUint64 atomically stores new into *addr and returns the previous *addr value.
func SwapUint64(addr *uint64, new uint64) (old uint64)

// SwapUintptr atomically stores new into *addr and returns the previous *addr value.
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)

// SwapPointer atomically stores new into *addr and returns the previous *addr value.
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)

// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

// CompareAndSwapInt64 executes the compare-and-swap operation for an int64 value.
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

// CompareAndSwapUint32 executes the compare-and-swap operation for a uint32 value.
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

// CompareAndSwapUint64 executes the compare-and-swap operation for a uint64 value.
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)

// CompareAndSwapUintptr executes the compare-and-swap operation for a uintptr value.
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)

// CompareAndSwapPointer executes the compare-and-swap operation for a unsafe.Pointer value.
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)

// AddInt32 atomically adds delta to *addr and returns the new value.
func AddInt32(addr *int32, delta int32) (new int32)

// AddUint32 atomically adds delta to *addr and returns the new value.
// To subtract a signed positive constant value c from x, do AddUint32(&x, ^uint32(c-1)).
// In particular, to decrement x, do AddUint32(&x, ^uint32(0)).
func AddUint32(addr *uint32, delta uint32) (new uint32)

// AddInt64 atomically adds delta to *addr and returns the new value.
func AddInt64(addr *int64, delta int64) (new int64)

// AddUint64 atomically adds delta to *addr and returns the new value.
// To subtract a signed positive constant value c from x, do AddUint64(&x, ^uint64(c-1)).
// In particular, to decrement x, do AddUint64(&x, ^uint64(0)).
func AddUint64(addr *uint64, delta uint64) (new uint64)

// AddUintptr atomically adds delta to *addr and returns the new value.
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)

// LoadInt32 atomically loads *addr.
func LoadInt32(addr *int32) (val int32)

// LoadInt64 atomically loads *addr.
func LoadInt64(addr *int64) (val int64)

// LoadUint32 atomically loads *addr.
func LoadUint32(addr *uint32) (val uint32)

// LoadUint64 atomically loads *addr.
func LoadUint64(addr *uint64) (val uint64)

// LoadUintptr atomically loads *addr.
func LoadUintptr(addr *uintptr) (val uintptr)

// LoadPointer atomically loads *addr.
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

// StoreInt32 atomically stores val into *addr.
func StoreInt32(addr *int32, val int32)

// StoreInt64 atomically stores val into *addr.
func StoreInt64(addr *int64, val int64)

// StoreUint32 atomically stores val into *addr.
func StoreUint32(addr *uint32, val uint32)

// StoreUint64 atomically stores val into *addr.
func StoreUint64(addr *uint64, val uint64)

// StoreUintptr atomically stores val into *addr.
func StoreUintptr(addr *uintptr, val uintptr)

// StorePointer atomically stores val into *addr.
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)

// Helper for ARM.  Linker will discard on other systems
func panic64() {
	panic("sync/atomic: broken 64-bit atomic operations (buggy QEMU)")
}

```
> go 文件内只进行了函数的定义，具体实现为在 asm 文件中

sync/atomic/asm.s
```assembly
// Copyright 2011 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// +build !race

#include "textflag.h"

TEXT ·SwapInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg(SB)

TEXT ·SwapUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg(SB)

TEXT ·SwapInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg64(SB)

TEXT ·SwapUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg64(SB)

TEXT ·SwapUintptr(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchguintptr(SB)

TEXT ·CompareAndSwapInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas(SB)

TEXT ·CompareAndSwapUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas(SB)

TEXT ·CompareAndSwapUintptr(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Casuintptr(SB)

TEXT ·CompareAndSwapInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas64(SB)

TEXT ·CompareAndSwapUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas64(SB)

TEXT ·AddInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadd(SB)

TEXT ·AddUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadd(SB)

TEXT ·AddUintptr(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadduintptr(SB)

TEXT ·AddInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadd64(SB)

TEXT ·AddUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadd64(SB)

TEXT ·LoadInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Load(SB)

TEXT ·LoadUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Load(SB)

TEXT ·LoadInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Load64(SB)

TEXT ·LoadUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Load64(SB)

TEXT ·LoadUintptr(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Loaduintptr(SB)

TEXT ·LoadPointer(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Loadp(SB)

TEXT ·StoreInt32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Store(SB)

TEXT ·StoreUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Store(SB)

TEXT ·StoreInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Store64(SB)

TEXT ·StoreUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Store64(SB)

TEXT ·StoreUintptr(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Storeuintptr(SB)

```
可以看到这边函数的实现仅为 JMP 跳转指令，统一跳转到 runtime/internal/atomic 下的各个函数进行


runtime/internal/atomic/stubs.go
```go
// Copyright 2015 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// +build !wasm

package atomic

import "unsafe"

//go:noescape
func Cas(ptr *uint32, old, new uint32) bool

// NO go:noescape annotation; see atomic_pointer.go.
func Casp1(ptr *unsafe.Pointer, old, new unsafe.Pointer) bool

//go:noescape
func Casuintptr(ptr *uintptr, old, new uintptr) bool

//go:noescape
func Storeuintptr(ptr *uintptr, new uintptr)

//go:noescape
func Loaduintptr(ptr *uintptr) uintptr

//go:noescape
func Loaduint(ptr *uint) uint

// TODO(matloob): Should these functions have the go:noescape annotation?

//go:noescape
func Loadint64(ptr *int64) int64

//go:noescape
func Xaddint64(ptr *int64, delta int64) int64

```
#### func Cas(ptr *uint32, old, new uint32) bool
- 入参为 uint32 指针，旧值，新值
- 返回值为是否 cas 成功

go 文件内同样只进行了函数的定义，具体实现为在各平台的 asm 文件中：

runtime/internal/atomic/asm_amd64.s
```assembly
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17    //  17 = sizeof(*uint32 + sizeof(uint32) + sizeof(uint32) + sizeof(uint8/bool))
	MOVQ	ptr+0(FP), BX                         // 入参 1 ，8 字节 uint32 指针
	MOVL	old+8(FP), AX                         // 入参 2 ，4 字节 uint32
	MOVL	new+12(FP), CX                        // 入参 3 ，4 字节 uint32
	LOCK                                        // lock 指令前缀
        /*
         * CMPXCHGL r, [m]
         * if AX == [m] {
         *   ZF = 1;
         *   [m] = r;
         * } else {
         *   ZF = 0;
         *   AX = [m];
         * }
         */
	CMPXCHGL	CX, 0(BX)                     // 比较并交换指令, ZF set to 1 if success         
	SETEQ	ret+16(FP)                            // 1 if ZF set to 1
	RET
```
> ZF: 标志寄存器的一种，零标志：用于判断结果是否为0。运算结果0，ZF置1，否则置0。

#### func SwapInt32(addr *int32, new int32) (old int32)
> cmd/compile/internal/gc/ssa.go
> alias("sync/atomic", "SwapInt32", "runtime/internal/atomic", "Xchg", all...)
> 以上可知 SwapInt32 函数为 runtime/internal/atomic·Xchg 函数的一个别名。

runtime/internal/atomic/asm_amd64.s
```assembly
TEXT runtime∕internal∕atomic·Xchg(SB), NOSPLIT, $0-20
	MOVQ	ptr+0(FP), BX
	MOVL	new+8(FP), AX
	XCHGL	AX, 0(BX)     // 交换指令
	MOVL	AX, ret+16(FP) // 交换后的 AX(old value) 写入 FP 返回值位
	RET
```

#### func AddInt32(addr *int32, new int32) (old int32)
> alias("sync/atomic", "AddInt32", "runtime/internal/atomic", "Xadd", all...)

runtime/internal/atomic/asm_amd64.s
```assembly
TEXT runtime∕internal∕atomic·Xadd(SB), NOSPLIT, $0-20
	MOVQ	ptr+0(FP), BX
	MOVL	delta+8(FP), AX
	MOVL	AX, CX
	LOCK
	XADDL	AX, 0(BX)  // Exchange and Add
	ADDL	CX, AX     // AX += CX
	MOVL	AX, ret+16(FP)
	RET
```
#### func StoreInt32(addr *int32, val int32)
> alias("sync/atomic", "StoreInt32", "runtime/internal/atomic", "Store", all...)

runtime/internal/atomic/asm_amd64.s

```assembly
TEXT runtime∕internal∕atomic·Store(SB), NOSPLIT, $0-12
	MOVQ	ptr+0(FP), BX
	MOVL	val+8(FP), AX
	XCHGL	AX, 0(BX) // 交换指令
	RET
```

<img width="330px"  src="https://xargin.com/content/images/2021/05/wechat.png">
