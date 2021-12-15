---
title: 调用规约
weight: 3
draft: true
---

# 调用规约

早期版本的 Go 在向函数传递参数时，是使用栈的。

从 1.17 开始，调用函数时会尽量使用寄存器传参。

"".add STEXT size=121 args=0x10 locals=0x18 funcid=0x0
	0x0000 00000 (add.go:8)	TEXT	"".add(SB), ABIInternal, $24-16
	0x0000 00000 (add.go:8)	CMPQ	SP, 16(R14)
	0x0004 00004 (add.go:8)	JLS	94
	0x0006 00006 (add.go:8)	SUBQ	$24, SP
	0x005d 00093 (add.go:11)	RET
	0x005e 00094 (add.go:11)	NOP
	0x005e 00094 (add.go:8)	MOVQ	AX, 8(SP)
	0x0063 00099 (add.go:8)	MOVQ	BX, 16(SP)
	0x0068 00104 (add.go:8)	CALL	runtime.morestack_noctxt(SB)
	0x006d 00109 (add.go:8)	MOVQ	8(SP), AX
	0x0072 00114 (add.go:8)	MOVQ	16(SP), BX
	0x0077 00119 (add.go:8)	JMP	0
