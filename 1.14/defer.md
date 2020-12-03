# 1.14 defer

## 正常处理流程

在 Go 1.14 中，增加了一种新的 defer 实现：open coded defer。当函数内 defer 不超过 8 个时，则会使用这种实现。

形如下面这样的代码:

```go
defer f1(a)
if cond {
 defer f2(b)
}
body...
```

会被翻译为:

```go
deferBits |= 1<<0  // 因为函数主 block 有 defer 语句，所以这里要设置第 0 位为 1
tmpF1 = f1 // 将函数地址搬运到预留的栈上空间
tmpA = a // 将函数参数复制到预留的栈上空间
if cond {
 deferBits |= 1<<1 // 因为进入了 if block，所以这里要设置第 1 位为 1
 tmpF2 = f2 // 将函数地址搬运到预留的栈上空间
 tmpB = b // 将函数参数复制到预留的栈上空间
}
body...
exit: // 退出函数 body
if deferBits & 1<<1 != 0 { // 检查第 1 个 bit，如果是 1，执行保存好的函数
 deferBits &^= 1<<1
 tmpF2(tmpB)
}
if deferBits & 1<<0 != 0 { // 检查第 0 个 bit，如果是 1，执行保存好的函数
 deferBits &^= 1<<0
 tmpF1(tmpA)
}
```

这样看我们不一定知道他具体是怎么实现的，可以写个 block 更多的例子来看看实际生成的代码是什么样的:

```go
     1	package main
     2
     3	import "fmt"
     4
     5	var i = 100
     6
     7	func main() {
     8		if i == 0 {
     9			defer fmt.Println("1")
    10		}
    11
    12		if i == 1 {
    13			defer fmt.Println("2")
    14		}
    15
    16		if i == 2 {
    17			defer fmt.Println("3")
    18		}
    19
    20		if i == 3 {
    21			defer fmt.Println("4")
    22		}
    23
    24		if i == 4 {
    25			defer fmt.Println("5")
    26		}
    27
    28		if i == 5 {
    29			defer fmt.Println("6")
    30		}
    31	}
```

下面是 go tool compile -S 生成的汇编代码，简单划分一下 block 来理解一下:

```
"".main STEXT size=1125 args=0x0 locals=0x160
if i == 0:
	0x0055 00085 (open-coded-defer.go:8)	CMPQ	"".i(SB), $0
	0x005d 00093 (open-coded-defer.go:8)	JNE	1075 // 1075 是条废指令，其实本质是跳过这条 if 的 DEFER PREPARE BLOCK，去地址 188

DEFER PREPARE BLOCK 1，主要工作是将 fmt.Println 的参数和函数地址搬运到栈上该 defer 对应的区域:
	0x0063 00099 (open-coded-defer.go:9)	XORPS	X0, X0
	0x0066 00102 (open-coded-defer.go:9)	MOVUPS	X0, ""..autotmp_3+72(SP)
	0x006b 00107 (open-coded-defer.go:9)	LEAQ	type.string(SB), AX
	0x0072 00114 (open-coded-defer.go:9)	MOVQ	AX, ""..autotmp_3+72(SP)
	0x0077 00119 (open-coded-defer.go:9)	LEAQ	""..stmp_0(SB), CX
	0x007e 00126 (open-coded-defer.go:9)	MOVQ	CX, ""..autotmp_3+80(SP)
	0x0083 00131 (open-coded-defer.go:9)	LEAQ	fmt.Println·f(SB), CX
	0x008a 00138 (open-coded-defer.go:9)	MOVQ	CX, ""..autotmp_25+192(SP)
	0x0092 00146 (open-coded-defer.go:9)	LEAQ	""..autotmp_3+72(SP), DX
	0x0097 00151 (open-coded-defer.go:9)	MOVQ	DX, ""..autotmp_26+320(SP)
	0x009f 00159 (open-coded-defer.go:9)	MOVQ	$1, ""..autotmp_26+328(SP)
	0x00ab 00171 (open-coded-defer.go:9)	MOVQ	$1, ""..autotmp_26+336(SP)
	0x00b7 00183 (open-coded-defer.go:9)	MOVB	$1, ""..autotmp_24+55(SP)
	
	// 根据 zero flag 设置值，如果 zero flag = 1，那么 DI = 1，如果 zero flag  = 0，那么 DI = 0
	// 详情参见 intel 手册的 sete/setz 指令
	// 也就是说，只要走了上面的 DEFER PREPARE BLOCK，那么这里就 DI = 1，否则 DI = 0
	0x00bc 00188 (open-coded-defer.go:8)	SETEQ	DL

if i == 1:
	0x00bf 00191 (open-coded-defer.go:12)	CMPQ	"".i(SB), $1
	0x00c7 00199 (open-coded-defer.go:12)	JNE	278

DEFER PREPARE BLOCK 2, 和 BLOCK 1 工作差不多，是把第二个 defer 的参数和函数地址保存在栈上:
	0x00c9 00201 (open-coded-defer.go:13)	XORPS	X0, X0
	0x00cc 00204 (open-coded-defer.go:13)	MOVUPS	X0, ""..autotmp_7+56(SP)
	0x00d1 00209 (open-coded-defer.go:13)	MOVQ	AX, ""..autotmp_7+56(SP)
	0x00d6 00214 (open-coded-defer.go:13)	LEAQ	""..stmp_1(SB), BX
	0x00dd 00221 (open-coded-defer.go:13)	MOVQ	BX, ""..autotmp_7+64(SP)
	0x00e2 00226 (open-coded-defer.go:13)	MOVQ	CX, ""..autotmp_27+184(SP)
	0x00ea 00234 (open-coded-defer.go:13)	LEAQ	""..autotmp_7+56(SP), BX
	0x00ef 00239 (open-coded-defer.go:13)	MOVQ	BX, ""..autotmp_28+296(SP)
	0x00f7 00247 (open-coded-defer.go:13)	MOVQ	$1, ""..autotmp_28+304(SP)
	0x0103 00259 (open-coded-defer.go:13)	MOVQ	$1, ""..autotmp_28+312(SP)

	0x010f 00271 (open-coded-defer.go:13)	ORL	$2, DX  // 设置 DX 的第二个标记位为 1
	0x0112 00274 (open-coded-defer.go:13)	MOVB	DL, ""..autotmp_24+55(SP) // TODO

if i == 2:
	0x0116 00278 (open-coded-defer.go:16)	CMPQ	"".i(SB), $2
	0x011e 00286 (open-coded-defer.go:16)	JNE	377

DEFER PREPARE BLOCK 3，和上面类似，就不解释了:
	0x0120 00288 (open-coded-defer.go:17)	XORPS	X0, X0
	0x0123 00291 (open-coded-defer.go:17)	MOVUPS	X0, ""..autotmp_11+136(SP)
	0x012b 00299 (open-coded-defer.go:17)	MOVQ	AX, ""..autotmp_11+136(SP)
	0x0133 00307 (open-coded-defer.go:17)	LEAQ	""..stmp_2(SB), BX
	0x013a 00314 (open-coded-defer.go:17)	MOVQ	BX, ""..autotmp_11+144(SP)
	0x0142 00322 (open-coded-defer.go:17)	MOVQ	CX, ""..autotmp_29+176(SP)
	0x014a 00330 (open-coded-defer.go:17)	LEAQ	""..autotmp_11+136(SP), BX
	0x0152 00338 (open-coded-defer.go:17)	MOVQ	BX, ""..autotmp_30+272(SP)
	0x015a 00346 (open-coded-defer.go:17)	MOVQ	$1, ""..autotmp_30+280(SP)
	0x0166 00358 (open-coded-defer.go:17)	MOVQ	$1, ""..autotmp_30+288(SP)

	0x0172 00370 (open-coded-defer.go:17)	ORL	$4, DX // 设置 DX 的第三个标记位为 1
	0x0175 00373 (open-coded-defer.go:17)	MOVB	DL, ""..autotmp_24+55(SP)

if i == 3
	0x0179 00377 (open-coded-defer.go:20)	CMPQ	"".i(SB), $3
	0x0181 00385 (open-coded-defer.go:20)	JNE	467

DEFER PREPARE BLOCK 4:
	0x0183 00387 (open-coded-defer.go:21)	XORPS	X0, X0
	0x0186 00390 (open-coded-defer.go:21)	MOVUPS	X0, ""..autotmp_15+120(SP)
	0x018b 00395 (open-coded-defer.go:21)	MOVQ	AX, ""..autotmp_15+120(SP)
	0x0190 00400 (open-coded-defer.go:21)	LEAQ	""..stmp_3(SB), BX
	0x0197 00407 (open-coded-defer.go:21)	MOVQ	BX, ""..autotmp_15+128(SP)
	0x019f 00415 (open-coded-defer.go:21)	MOVQ	CX, ""..autotmp_31+168(SP)
	0x01a7 00423 (open-coded-defer.go:21)	LEAQ	""..autotmp_15+120(SP), BX
	0x01ac 00428 (open-coded-defer.go:21)	MOVQ	BX, ""..autotmp_32+248(SP)
	0x01b4 00436 (open-coded-defer.go:21)	MOVQ	$1, ""..autotmp_32+256(SP)
	0x01c0 00448 (open-coded-defer.go:21)	MOVQ	$1, ""..autotmp_32+264(SP)

	0x01cc 00460 (open-coded-defer.go:21)	ORL	$8, DX // 设置 DX 的第四个标记位为 1
	0x01cf 00463 (open-coded-defer.go:21)	MOVB	DL, ""..autotmp_24+55(SP)

if i == 4:
	0x01d3 00467 (open-coded-defer.go:24)	CMPQ	"".i(SB), $4
	0x01db 00475 (open-coded-defer.go:24)	JNE	554

DEFER PREPARE BLOCK 5:
	0x01dd 00477 (open-coded-defer.go:25)	XORPS	X0, X0
	0x01e0 00480 (open-coded-defer.go:25)	MOVUPS	X0, ""..autotmp_19+104(SP)
	0x01e5 00485 (open-coded-defer.go:25)	MOVQ	AX, ""..autotmp_19+104(SP)
	0x01ea 00490 (open-coded-defer.go:25)	LEAQ	""..stmp_4(SB), BX
	0x01f1 00497 (open-coded-defer.go:25)	MOVQ	BX, ""..autotmp_19+112(SP)
	0x01f6 00502 (open-coded-defer.go:25)	MOVQ	CX, ""..autotmp_33+160(SP)
	0x01fe 00510 (open-coded-defer.go:25)	LEAQ	""..autotmp_19+104(SP), BX
	0x0203 00515 (open-coded-defer.go:25)	MOVQ	BX, ""..autotmp_34+224(SP)
	0x020b 00523 (open-coded-defer.go:25)	MOVQ	$1, ""..autotmp_34+232(SP)
	0x0217 00535 (open-coded-defer.go:25)	MOVQ	$1, ""..autotmp_34+240(SP)

	0x0223 00547 (open-coded-defer.go:25)	ORL	$16, DX // 设置 DX 的第五个标记位为 1
	0x0226 00550 (open-coded-defer.go:25)	MOVB	DL, ""..autotmp_24+55(SP)

if i == 5:
	0x022a 00554 (open-coded-defer.go:28)	CMPQ	"".i(SB), $5
	0x0232 00562 (open-coded-defer.go:28)	JNE	641

DEFER PREPARE BLOCK 6:
	0x0234 00564 (open-coded-defer.go:29)	XORPS	X0, X0
	0x0237 00567 (open-coded-defer.go:29)	MOVUPS	X0, ""..autotmp_23+88(SP)
	0x023c 00572 (open-coded-defer.go:29)	MOVQ	AX, ""..autotmp_23+88(SP)
	0x0241 00577 (open-coded-defer.go:29)	LEAQ	""..stmp_5(SB), AX
	0x0248 00584 (open-coded-defer.go:29)	MOVQ	AX, ""..autotmp_23+96(SP)
	0x024d 00589 (open-coded-defer.go:29)	MOVQ	CX, ""..autotmp_35+152(SP)
	0x0255 00597 (open-coded-defer.go:29)	LEAQ	""..autotmp_23+88(SP), AX
	0x025a 00602 (open-coded-defer.go:29)	MOVQ	AX, ""..autotmp_36+200(SP)
	0x0262 00610 (open-coded-defer.go:29)	MOVQ	$1, ""..autotmp_36+208(SP)
	0x026e 00622 (open-coded-defer.go:29)	MOVQ	$1, ""..autotmp_36+216(SP)

	0x027a 00634 (open-coded-defer.go:29)	ORL	$32, DX // 设置 DX 的第六个标志位为 1
	0x027d 00637 (open-coded-defer.go:29)	MOVB	DL, ""..autotmp_24+55(SP)

FUNCTION END, FLAG CHECKS, 这里要检查上面设置的所有的 open coded defer bits 了，可以看到与 flag 的设置顺序是相反的:
	0x0281 00641 (open-coded-defer.go:31)	TESTB	$32, DL
	0x0284 00644 (open-coded-defer.go:31)	JNE	1011
	0x028a 00650 (open-coded-defer.go:31)	TESTB	$16, DL
	0x028d 00653 (open-coded-defer.go:31)	JNE	947
	0x0293 00659 (open-coded-defer.go:31)	TESTB	$8, DL
	0x0296 00662 (open-coded-defer.go:31)	JNE	883
	0x029c 00668 (open-coded-defer.go:31)	TESTB	$4, DL
	0x029f 00671 (open-coded-defer.go:31)	JNE	819
	0x02a5 00677 (open-coded-defer.go:31)	TESTB	$2, DL
	0x02a8 00680 (open-coded-defer.go:31)	JNE	755
	0x02aa 00682 (open-coded-defer.go:31)	TESTB	$1, DL
	0x02ad 00685 (open-coded-defer.go:31)	JNE	703
	0x02af 00687 (open-coded-defer.go:31)	MOVQ	344(SP), BP
	0x02b7 00695 (open-coded-defer.go:31)	ADDQ	$352, SP
	0x02be 00702 (open-coded-defer.go:31)	RET

DEFER 逻辑执行部分
    // defer fmt.Println("6"):
	0x02bf 00703 (open-coded-defer.go:31)	ANDL	$-2, DX
	0x02c2 00706 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x02c6 00710 (open-coded-defer.go:31)	MOVQ	""..autotmp_26+320(SP), AX
	0x02ce 00718 (open-coded-defer.go:31)	MOVQ	""..autotmp_26+328(SP), CX
	0x02d6 00726 (open-coded-defer.go:31)	MOVQ	""..autotmp_26+336(SP), DX
	0x02de 00734 (open-coded-defer.go:31)	MOVQ	AX, (SP)
	0x02e2 00738 (open-coded-defer.go:31)	MOVQ	CX, 8(SP)
	0x02e7 00743 (open-coded-defer.go:31)	MOVQ	DX, 16(SP)
	0x02ec 00748 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x02f1 00753 (open-coded-defer.go:31)	JMP	687

    // defer fmt.Println("5"):
	0x02f3 00755 (open-coded-defer.go:31)	ANDL	$-3, DX
	0x02f6 00758 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_37+54(SP)
	0x02fa 00762 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x02fe 00766 (open-coded-defer.go:31)	MOVQ	""..autotmp_28+312(SP), AX
	0x0306 00774 (open-coded-defer.go:31)	MOVQ	""..autotmp_28+304(SP), CX
	0x030e 00782 (open-coded-defer.go:31)	MOVQ	""..autotmp_28+296(SP), BX
	0x0316 00790 (open-coded-defer.go:31)	MOVQ	BX, (SP)
	0x031a 00794 (open-coded-defer.go:31)	MOVQ	CX, 8(SP)
	0x031f 00799 (open-coded-defer.go:31)	MOVQ	AX, 16(SP)
	0x0324 00804 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x0329 00809 (open-coded-defer.go:31)	MOVBLZX	""..autotmp_37+54(SP), DX
	0x032e 00814 (open-coded-defer.go:31)	JMP	682

    // defer fmt.Println("4"):
	0x0333 00819 (open-coded-defer.go:31)	ANDL	$-5, DX
	0x0336 00822 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_37+54(SP)
	0x033a 00826 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x033e 00830 (open-coded-defer.go:31)	MOVQ	""..autotmp_30+288(SP), AX
	0x0346 00838 (open-coded-defer.go:31)	MOVQ	""..autotmp_30+280(SP), CX
	0x034e 00846 (open-coded-defer.go:31)	MOVQ	""..autotmp_30+272(SP), BX
	0x0356 00854 (open-coded-defer.go:31)	MOVQ	BX, (SP)
	0x035a 00858 (open-coded-defer.go:31)	MOVQ	CX, 8(SP)
	0x035f 00863 (open-coded-defer.go:31)	MOVQ	AX, 16(SP)
	0x0364 00868 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x0369 00873 (open-coded-defer.go:31)	MOVBLZX	""..autotmp_37+54(SP), DX
	0x036e 00878 (open-coded-defer.go:31)	JMP	677

    // defer fmt.Println("3"):
	0x0373 00883 (open-coded-defer.go:31)	ANDL	$-9, DX
	0x0376 00886 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_37+54(SP)
	0x037a 00890 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x037e 00894 (open-coded-defer.go:31)	MOVQ	""..autotmp_32+264(SP), AX
	0x0386 00902 (open-coded-defer.go:31)	MOVQ	""..autotmp_32+256(SP), CX
	0x038e 00910 (open-coded-defer.go:31)	MOVQ	""..autotmp_32+248(SP), BX
	0x0396 00918 (open-coded-defer.go:31)	MOVQ	BX, (SP)
	0x039a 00922 (open-coded-defer.go:31)	MOVQ	CX, 8(SP)
	0x039f 00927 (open-coded-defer.go:31)	MOVQ	AX, 16(SP)
	0x03a4 00932 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x03a9 00937 (open-coded-defer.go:31)	MOVBLZX	""..autotmp_37+54(SP), DX
	0x03ae 00942 (open-coded-defer.go:31)	JMP	668

    // defer fmt.Println("2"):
	0x03b3 00947 (open-coded-defer.go:31)	ANDL	$-17, DX
	0x03b6 00950 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_37+54(SP)
	0x03ba 00954 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x03be 00958 (open-coded-defer.go:31)	MOVQ	""..autotmp_34+224(SP), AX
	0x03c6 00966 (open-coded-defer.go:31)	MOVQ	""..autotmp_34+240(SP), CX
	0x03ce 00974 (open-coded-defer.go:31)	MOVQ	""..autotmp_34+232(SP), BX
	0x03d6 00982 (open-coded-defer.go:31)	MOVQ	AX, (SP)
	0x03da 00986 (open-coded-defer.go:31)	MOVQ	BX, 8(SP)
	0x03df 00991 (open-coded-defer.go:31)	MOVQ	CX, 16(SP)
	0x03e4 00996 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x03e9 01001 (open-coded-defer.go:31)	MOVBLZX	""..autotmp_37+54(SP), DX
	0x03ee 01006 (open-coded-defer.go:31)	JMP	659

    // defer fmt.Println("1"):
	0x03f3 01011 (open-coded-defer.go:31)	ANDL	$-33, DX
	0x03f6 01014 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_37+54(SP)
	0x03fa 01018 (open-coded-defer.go:31)	MOVB	DL, ""..autotmp_24+55(SP)
	0x03fe 01022 (open-coded-defer.go:31)	MOVQ	""..autotmp_36+216(SP), AX
	0x0406 01030 (open-coded-defer.go:31)	MOVQ	""..autotmp_36+200(SP), CX
	0x040e 01038 (open-coded-defer.go:31)	MOVQ	""..autotmp_36+208(SP), BX
	0x0416 01046 (open-coded-defer.go:31)	MOVQ	CX, (SP)
	0x041a 01050 (open-coded-defer.go:31)	MOVQ	BX, 8(SP)
	0x041f 01055 (open-coded-defer.go:31)	MOVQ	AX, 16(SP)
	0x0424 01060 (open-coded-defer.go:31)	CALL	fmt.Println(SB)
	0x0429 01065 (open-coded-defer.go:31)	MOVBLZX	""..autotmp_37+54(SP), DX
	0x042e 01070 (open-coded-defer.go:31)	JMP	650

如果第一个 if 不满足，会跳到这里:
	0x0433 01075 (open-coded-defer.go:31)	LEAQ	type.string(SB), AX
	0x043a 01082 (open-coded-defer.go:31)	LEAQ	fmt.Println·f(SB), CX
	0x0441 01089 (open-coded-defer.go:8)	JMP	188

这段代码没什么用，只有在 panic 时，才可能跳到这里，但本例中无 panic
	0x0446 01094 (open-coded-defer.go:8)	CALL	runtime.deferreturn(SB)
	0x044b 01099 (open-coded-defer.go:8)	MOVQ	344(SP), BP
	0x0453 01107 (open-coded-defer.go:8)	ADDQ	$352, SP
	0x045a 01114 (open-coded-defer.go:8)	RET

函数 prologue 栈检查的时候可能会跳到这里
	0x045b 01115 (open-coded-defer.go:8)	NOP
	0x045b 01115 (open-coded-defer.go:7)	CALL	runtime.morestack_noctxt(SB)
	0x0460 01120 (open-coded-defer.go:7)	JMP	0
```

整体流程还好，不过比起以前堆上分配一个 `_defer` 链表，函数执行完之后直接遍历链表还是复杂太多了。

超过 8 个 defer 时会退化回以前的 defer 链，也可以观察一下:

```go
package main

func main() {
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
	defer println(1)
}
```

编译出的汇编就不贴了，和以前的没什么区别。

结合代码分析过程和 proposal 中的描述，总结:

* open coded defer 在函数内部总 defer 数量少于 8 时才会使用，大于 8 时会退化回老的 defer 链，这个权衡是考虑到程序文件的体积(个人觉得应该也有栈膨胀的考虑在)
* 使用 open coded defer 时，在进入函数内的每个 block 时，都会设置这个 block 相应的 bit(ORL 指令)，在执行到相应的 defer 语句时，把 defer 语句的参数和调用函数地址推到栈的相应位置
* 在栈上需要为这些 defer 调用的参数、函数地址，以及这个用来记录 defer bits 的 byte 预留空间，所以毫无疑问，函数的 framesize 会变大

## panic 处理部分

proposal 有两部分，一部分是讲 open coded defer 的实现，另一部分是说 panic 的处理，现在在函数中发生 panic 时，我们没有办法获得原来的 `_defer` 结构体了，所以必须在编译期，给每个函数准备一项 FUNCDATA，存储每一个 defer block 的地址。这样 runtime 就可以通过 FUNCDATA 和偏移找到需要执行的 defer block 在什么地方。

```
	0x004a 00074 (defer.go:8)	FUNCDATA	$2, gclocals·951c056c1b0a4d6097d387fbe928bbbf(SB)
	0x004a 00074 (defer.go:8)	FUNCDATA	$3, "".main.stkobj(SB)
	0x004a 00074 (defer.go:8)	FUNCDATA	$5, "".main.opendefer(SB)

	....

	"".main.opendefer SRODATA dupok size=29
	0x0000 30 c1 01 04 30 80 01 01 60 18 00 30 78 01 48 18  0...0...`..0x.H.
	0x0010 00 30 70 01 30 18 00 30 68 01 18 18 00           .0p.0..0h....
```

这里的 main.opendefer 在编译为二进制之后，可能会被优化掉。不过我们理解基本的执行流程就可以了。

跟踪 runtime.gopanic 的流程，可以看到现在的 defer 上有 openDefer 的 bool 值:

```
(dlv) p d
*runtime._defer {
	siz: 48,
	started: true,
	heap: false,
	openDefer: false,
	sp: 824634391712,
	pc: 17590156,
	fn: *runtime.funcval {fn: 17565744},
	_panic: *runtime._panic nil,
	link: *runtime._defer {
		siz: 8,
		started: false,
		heap: true,
		openDefer: true,
		sp: 824634392456,
		pc: 17001293,
		fn: *runtime.funcval nil,
		_panic: *runtime._panic nil,
		link: *runtime._defer nil,
		fd: unsafe.Pointer(0x10fea48),
		varp: 824634392528,
		framepc: 17000984,},
	fd: unsafe.Pointer(0x0),
	varp: 0,
	framepc: 0,}
(dlv) n
```

执行具体的 defer 逻辑在 runtime.runOpenDeferFrame，也没啥神奇的:

```go
func runOpenDeferFrame(gp *g, d *_defer) bool {
	done := true
	fd := d.fd

	// Skip the maxargsize
	_, fd = readvarintUnsafe(fd)
	deferBitsOffset, fd := readvarintUnsafe(fd)
	nDefers, fd := readvarintUnsafe(fd)
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))

	for i := int(nDefers) - 1; i >= 0; i-- {
		// read the funcdata info for this defer
		var argWidth, closureOffset, nArgs uint32
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
		if deferBits&(1<<i) == 0 {
			for j := uint32(0); j < nArgs; j++ {
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
			}
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure
		deferArgs := deferArgs(d)
		// If there is an interface receiver or method receiver, it is
		// described/included as the first arg.
		for j := uint32(0); j < nArgs; j++ {
			var argOffset, argLen, argCallOffset uint32
			argOffset, fd = readvarintUnsafe(fd)
			argLen, fd = readvarintUnsafe(fd)
			argCallOffset, fd = readvarintUnsafe(fd)
			memmove(unsafe.Pointer(uintptr(deferArgs)+uintptr(argCallOffset)),
				unsafe.Pointer(d.varp-uintptr(argOffset)),
				uintptr(argLen))
		}
		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		// These args are just a copy, so can be cleared immediately
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		if d._panic != nil && d._panic.recovered {
			done = deferBits == 0
			break
		}
	}

	return done

```

参考资料:

1. [open coded defer](https://github.com/golang/proposal/blob/master/design/34481-opencoded-defers.md)
