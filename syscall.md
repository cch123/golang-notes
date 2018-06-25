# 系统调用

```go
// func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
    MOVQ    a1+8(FP), DI
    MOVQ    a2+16(FP), SI
    MOVQ    a3+24(FP), DX
    MOVQ    $0, R10
    MOVQ    $0, R8
    MOVQ    $0, R9
    MOVQ    trap+0(FP), AX    // syscall entry
    SYSCALL
    CMPQ    AX, $0xfffffffffffff001
    JLS    ok1
    MOVQ    $-1, r1+32(FP)
    MOVQ    $0, r2+40(FP)
    NEGQ    AX
    MOVQ    AX, err+48(FP)
    RET
ok1:
    MOVQ    AX, r1+32(FP)
    MOVQ    DX, r2+40(FP)
    MOVQ    $0, err+48(FP)
    RET
```

vdso:

```go
// func gettimeofday(tv *Timeval) (err uintptr)
TEXT ·gettimeofday(SB),NOSPLIT,$0-16
    MOVQ    tv+0(FP), DI
    MOVQ    $0, SI
    MOVQ    runtime·__vdso_gettimeofday_sym(SB), AX
    CALL    AX

    CMPQ    AX, $0xfffffffffffff001
    JLS    ok7
    NEGQ    AX
    MOVQ    AX, err+8(FP)
    RET
ok7:
    MOVQ    $0, err+8(FP)
    RET
```