# plan9 assembly 完全解析

众所周知，Go 使用了 Unix 老古董(误 们发明的 plan9 汇编。就算你对 x86 汇编有所了解，在 plan9 里还是有些许区别。说不定你在看代码的时候，偶然发现代码里的 SP 看起来是 SP，但它实际上不是 SP 的时候就抓狂了哈哈哈。

本文将对 plan9 汇编进行全面的介绍，同时解答你在接触 plan9 汇编时可能遇到的大部分问题。

本文所使用的平台是 linux amd64，因为不同的平台指令集和寄存器都不一样，所以没有办法共同探讨。这也是由汇编本身的性质决定的。

## 基本指令

### 栈调整

在 intel 或 AT&T 汇编中，栈帧调整一般是通过对 rbp 和 rsp 进行 push  和 pop 操作来完成的。plan9 没有像 intel IA64 那样的 push 和 pop 指令，栈的调整是通过对硬件 SP 寄存器进行运算来实现的，例如:

```go
SUBQ $0x18, SP // 对 SP 做减法，为函数分配函数栈帧
...			   // 省略无用代码
ADDQ $0x18, SP // 对 SP 做加法，清除函数栈帧
```

通用的指令和 IA64 平台差不多，比如:

### 数据搬运

常数在 plan9 汇编用 $num 表示，可以为负数，默认情况下为十进制。可以用 $0x123 的形式来表示十六进制数。

```go
MOVB $1, DI      // 1 byte
MOVW $0x10, BX   // 2 bytes
MOVD $1, DX      // 4 bytes
MOVQ $10, AX     // 8 bytes
```
可以看到，搬运的长度是由 MOV 的后缀决定的，这一点与 intel 汇编稍有不同，看看类似的 IA64 汇编:
```asm
mov rax, 0x1   // 8 bytes
mov eax, 0x100 // 4 bytes
mov ax, 0x22   // 2 bytes
mov ah, 0x33   // 1 byte
mov al, 0x44   // 1 byte
```
plan9 的汇编的操作数的方向是和 intel 汇编相反的，与 AT&T 类似。
```go
MOVQ $0x10, AX ===== mov rax, 0x10
       |    |------------|      |
       |------------------------|
```
### 常见计算指令

```go
ADDQ  AX, BX   // BX += AX
SUBQ  AX, BX   // BX -= AX
IMULQ AX, BX   // BX *= AX
```
类似数据搬运指令，同样可以通过修改指令的后缀来对应不同长度的操作数。例如 ADDQ/ADDW/ADDL/ADDB。

### 条件跳转/无条件跳转
```go
// 无条件跳转
JMP addr   // 跳转到地址，地址可为代码中的地址，不过实际上手写不会出现这种东西
JMP label  // 跳转到标签，可以跳转到同一函数内的标签位置
JMP 2(PC)  // 以当前指令为基础，向前/后跳转 x 行
JMP -2(PC) // 同上

// 有条件跳转
JNZ target // 如果 zero flag 被 set 过，则跳转

```

### 指令集
可以参考源代码的 [arch](https://github.com/golang/arch/blob/master/x86/x86.csv) 部分。

## 寄存器

### 通用寄存器

amd64 的通用寄存器:
```
(lldb) reg read
General Purpose Registers:
       rax = 0x0000000000000005
       rbx = 0x000000c420088000
       rcx = 0x0000000000000000
       rdx = 0x0000000000000000
       rdi = 0x000000c420088008
       rsi = 0x0000000000000000
       rbp = 0x000000c420047f78
       rsp = 0x000000c420047ed8
        r8 = 0x0000000000000004
        r9 = 0x0000000000000000
       r10 = 0x000000c420020001
       r11 = 0x0000000000000202
       r12 = 0x0000000000000000
       r13 = 0x00000000000000f1
       r14 = 0x0000000000000011
       r15 = 0x0000000000000001
       rip = 0x000000000108ef85  int`main.main + 213 at int.go:19
    rflags = 0x0000000000000212
        cs = 0x000000000000002b
        fs = 0x0000000000000000
        gs = 0x0000000000000000
```
在 plan9 汇编里都是可以使用的，应用代码层面会用到的通用寄存器主要是: rax, rbx, rcx, rdx, rdi, rsi, r8~r15 这 14 个寄存器，虽然 rbp 和 rsp 也可以用，不过 bp 和 sp 会被用来管理栈顶和栈底，最好不要拿来进行运算。

plan9 中使用寄存器不需要带 r 或 e 的前缀，例如 rax，只要写 AX 即可:

```go
MOVQ $101, AX = mov rax, 101
```
下面是通用通用寄存器的名字在 IA64 和 plan9 中的对应关系:

| IA64 | rax | rbx| rcx | rdx | rdi | rsi | rbp | rsp | r8 | r9 | r10 | r11 | r12 | r13 | r14 | rip|
|--|--|--|--| --| --|--| --|--|--|--|--|--|--|--|--|--|
| Plan9 | AX | BX | CX | DX | DI | SI | BP | SP | R8 | R9 | R10 | R11 | R12 | R13 | R14 | PC |



### 伪寄存器
Go 的汇编还引入了 4 个伪寄存器，援引官方文档的描述:
>-   `FP`: Frame pointer: arguments and locals.
>-   `PC`: Program counter: jumps and branches.
>-   `SB`: Static base pointer: global symbols.
>-   `SP`: Stack pointer: top of stack.

官方的描述稍微有一些问题，我们对这些说明进行一点扩充:
- FP:  使用形如 `symbol+offset(FP)` 的方式，引用函数的输入参数。例如 `arg0+0(FP)`，`arg1+8(FP)`，使用 FP 不加 symbol 时，无法通过编译，在汇编层面来讲，symbol 并没有什么用，加 symbol 主要是为了提升代码可读性。另外，官方文档虽然将伪寄存器 FP 称之为 frame pointer，实际上它根本不是 frame pointer，按照传统的 x86 的习惯来讲，frame pointer 是指向整个 stack frame 底部的 BP 寄存器。假如当前的 callee 函数是 add，在 add 的代码中引用 FP，该 FP 指向的位置不在 callee 的 stack frame 之内，而是在 caller 的 stack frame 上。具体可参见之后的 **栈结构** 一章。
- PC: 实际上就是在体系结构的知识中常见的 pc 寄存器，在 x86 平台下对应 ip 寄存器，amd64 上则是 rip。除了个别跳转之外，手写 plan9 代码与 PC 寄存器打交道的情况较少。
- SB: 全局静态基指针，一般用来声明函数或全局变量，在之后的函数知识和示例部分会看到具体用法。
- SP: plan9 的这个 SP 寄存器指向当前栈帧的局部变量的开始位置，使用形如 `symbol+offset(SP)` 的方式，引用函数的局部变量。offset 的合法取值是 [-framesize, 0)，注意是个左闭右开的区间。假如局部变量都是 8 字节，那么第一个局部变量就可以用 `localvar0-8(SP)` 来表示。这也是一个词不表意的寄存器。与硬件寄存器 SP 是两个不同的东西，在栈帧 size 为 0 的情况下，伪寄存器 SP 和硬件寄存器 SP 指向同一位置。手写汇编代码时，如果是 `symbol+offset(SP)` 形式，则表示伪寄存器 SP。如果是 `offset(SP)` 则表示硬件寄存器 SP。务必注意。对于编译输出(go tool compile -S / go tool objdump)的代码来讲，目前所有的 SP 都是硬件寄存器 SP，无论是否带 symbol。

我们这里对容易混淆的几点简单进行说明：
1. 伪 SP 和硬件 SP 不是一回事，在手写代码时，伪 SP 和硬件 SP 的区分方法是看该 SP 前是否有 symbol。如果有 symbol，那么即为伪寄存器，如果没有，那么说明是硬件 SP 寄存器。
2. SP 和 FP 的相对位置是会变的，所以不应该尝试用伪 SP 寄存器去找那些用 FP + offset 来引用的值，例如函数的入参和返回值。
3. 官方文档中说的伪 SP 指向 stack 的 top，是有问题的。其指向的局部变量位置实际上是整个栈的栈底(除 caller BP 之外)，所以说 bottom 更合适一些。
4. 在 go tool objdump/go tool compile -S 输出的代码中，是没有伪 SP 和 FP 寄存器的，我们上面说的区分伪 SP 和硬件 SP 寄存器的方法，对于上述两个命令的输出结果是没法使用的。在编译和反汇编的结果中，只有真实的 SP 寄存器。
5. FP 和 Go 的官方源代码里的 framepointer 不是一回事，源代码里的 framepointer 指的是 caller BP 寄存器的值，在这里和 caller 的伪 SP 是值是相等的。

以上说明看不懂也没关系，在熟悉了函数的栈结构之后再反复回来查看应该就可以明白了。个人意见，这些是 Go 官方挖的坑。。

## 变量声明

在汇编里所谓的变量，一般是存储在 .rodata 或者 .data 段中的只读值。对应到应用层的话，就是已初始化过的全局的 const、var、static 变量/常量。

使用 DATA 结合 GLOBL 来定义一个变量。DATA 的用法为:
```go
DATA	symbol+offset(SB)/width, value
```
大多数参数都是字面意思，不过这个 offset 需要稍微注意。其含义是该值相对于符号 symbol 的偏移，而不是相对于全局某个地址的偏移。

使用 GLOBL 指令将变量声明为 global，额外接收两个参数，一个是 flag，另一个是变量的总大小。

```go
GLOBL divtab(SB), RODATA, $64
```

GLOBL 必须跟在 DATA 指令之后，下面是一个定义了多个 readonly 的全局变量的完整例子:

```go
DATA age+0x00(SB)/4, $18  // forever 18
GLOBL age(SB), RODATA, $4

DATA pi+0(SB)/8, $3.1415926
GLOBL pi(SB), RODATA, $8

DATA birthYear+0(SB)/4, $1988
GLOBL birthYear(SB), RODATA, $4
```
正如之前所说，所有符号在声明时，其 offset 一般都是 0。

有时也可能会想在全局变量中定义数组，或字符串，这时候就需要用上非 0 的 offset 了，例如:
```go
DATA bio<>+0(SB)/8, $"oh yes i"
DATA bio<>+8(SB)/8, $"am here "
GLOBL bio<>+0(SB), RODATA, $16
```
大部分都比较好理解，不过这里我们又引入了新的标记 `<>`，这个跟在符号名之后，表示该全局变量只在当前文件中生效，类似于 C 语言中的 static。如果在另外文件中引用该变量的话，会报 `relocation target not found` 的错误。

本小节中提到的 flag，还可以有其它的取值:
>-   `NOPROF`  = 1  
    (For  `TEXT`  items.) Don't profile the marked function. This flag is deprecated.
>-   `DUPOK`  = 2  
    It is legal to have multiple instances of this symbol in a single binary. The linker will choose one of the duplicates to use.
>-   `NOSPLIT`  = 4  
    (For  `TEXT`  items.) Don't insert the preamble to check if the stack must be split. The frame for the routine, plus anything it calls, must fit in the spare space at the top of the stack segment. Used to protect routines such as the stack splitting code itself.
>-   `RODATA`  = 8  
    (For  `DATA`  and  `GLOBL`  items.) Put this data in a read-only section.
>-   `NOPTR`  = 16  
    (For  `DATA`  and  `GLOBL`  items.) This data contains no pointers and therefore does not need to be scanned by the garbage collector.
>-   `WRAPPER`  = 32  
    (For  `TEXT`  items.) This is a wrapper function and should not count as disabling  `recover`.
>-   `NEEDCTXT`  = 64  
    (For  `TEXT`  items.) This function is a closure so it uses its incoming context register.

当使用这些 flag 的字面量时，需要在汇编文件中 `#include "textflag.h"`。

## 函数声明

我们来看看一个典型的 plan9 的汇编函数的定义：
```go
// func add(a, b int) int
//   => 该声明定义在同一个 package 下的任意 .go 文件中
//   => 只有函数头，没有实现
TEXT pkgname·add(SB), NOSPLIT, $0-8
	MOVQ a+0(FP), AX
	MOVQ a+8(FP), BX
	ADDQ AX, BX
	MOVQ BX, ret+16(FP)
	RET
```
为什么要叫 TEXT ？如果对程序数据在文件中和内存中的分段稍有了解的同学应该知道，我们的代码在二进制文件中，是存储在 .text 段中的，这里也就是一种约定俗成的起名方式。实际上在 plan9 中 TEXT 是一个指令，用来定义一个函数。除了 TEXT 之外还有前面变量声明说到的 DATA/GLOBL。

定义中的 pkgname 部分是可以省略的，如果你有强迫症，那写上也没有什么问题。

中点 `·` 比较特殊，是一个 unicode 的中点，该点在 mac 下的输入方法是 `option+shift+9`。在程序被链接之后，所有的中点`·` 都会被替换为`.`，比如你的方法是 `runtime·main`，在编译之后的程序里的符号则是 `runtime.main`。嗯，看起来很变态。简单总结一下:

```go

                              参数及返回值大小
                                  | 
 TEXT pkgname·add(SB),NOSPLIT,$32-32
       |        |               |
      包名     函数名         栈帧大小(局部变量+可能需要的额外调用函数的参数空间以及 ret address 的总大小)

```

## 栈结构

下面是一个典型的函数的栈结构图:
```
                                                                                   
                       -----------------                                           
                       current func arg0                                           
                       ----------------- <----------- FP(pseudo FP)                
                        caller ret addr                                            
                       +---------------+                                           
                       | caller BP(*)  |                                           
                       ----------------- <----------- SP(pseudo SP，实际上是当前栈帧的 BP 位置)
                       |   Local Var0  |                                           
                       -----------------                                           
                       |   Local Var1  |                                           
                       -----------------                                           
                       |   Local Var2  |                                           
                       -----------------                -                          
                       |   ........    |                                           
                       -----------------                                           
                       |   Local VarN  |                                           
                       -----------------                                           
                       |               |                                           
                       |               |                                           
                       |  temporarily  |                                           
                       |  unused space |                                           
                       |               |                                           
                       |               |                                           
                       -----------------                                           
                       |  call retn    |                                           
                       -----------------                                           
                       |  call ret(n-1)|                                           
                       -----------------                                           
                       |  ..........   |                                           
                       -----------------                                           
                       |  call ret1    |                                           
                       -----------------                                           
                       |  call argn    |                                           
                       -----------------                                           
                       |   .....       |                                           
                       -----------------                                           
                       |  call arg3    |                                           
                       -----------------                                           
                       |  call arg2    |                                           
                       |---------------|                                           
                       |  call arg1    |                                           
                       -----------------                                           
                       | return addr   |                                           
                       +---------------+  <------------  hardware SP 位置
```

图上的 caller BP，指的是 caller 的 BP 寄存器值，有些人把 caller BP 叫作 caller 的 frame pointer，实际上这个习惯是从 x86 架构沿袭来的。Go 的 asm 文档中把伪寄存器 FP 也称为 frame pointer，但是这两个 frame pointer 根本不是一回事。

此外需要注意的是，caller BP 是在编译期由编译器插入的，用户手写代码时，计算 frame size 时是不包括这个 caller BP 部分的。是否插入 caller BP 的主要判断依据是:

1. 函数的栈帧大小大于 0
2. 下述函数返回 true

```go
func Framepointer_enabled(goos, goarch string) bool {
	return framepointer_enabled != 0 && goarch == "amd64" && goos != "nacl"
}
```

如果编译器在最终的汇编结果中没有插入 caller BP(源代码中所称的 frame pointer)的情况下，伪 SP 和伪 FP 之间只有 8 个字节的 caller 的 return address，而插入了 BP 的话，就会多出额外的 8 字节。也就说伪 SP 和伪 FP 的相对位置是不固定的，有可能是间隔 8 个字节，也有可能间隔 16 个字节。并且判断依据会根据平台和 Go 的版本有所不同。

图上可以看到，FP 伪寄存器指向函数的传入参数的开始位置，因为栈是朝低地址方向增长，为了通过寄存器引用参数时方便，所以参数的摆放方向和栈的增长方向是相反的，即：
```
                              FP
high ----------------------> low
argN, ... arg3, arg2, arg1, arg0
```
假设所有参数均为 8 字节，这样我们就可以用 symname+0(FP) 访问第一个 参数，symname+8(FP) 访问第二个参数，以此类推。用伪 SP 来引用局部变量，原理上来讲差不多，不过因为伪 SP 指向的是局部变量的底部，所以 symname-8(SP) 表示的是第一个局部变量，symname-16(SP)表示第二个，以此类推。当然，这里假设局部变量都占用 8 个字节。

图的最上部的 caller return address 和 current func arg0 都是由 caller 来分配空间的。不算在当前的栈帧内。

因为官方文档本身较模糊，我们来一个函数调用的全景图，来看一下这些真假 SP/FP/BP 到底是个什么关系:
```
                                                                                                                              
                                       caller                                                                                 
                                 +------------------+                                                                         
                                 |                  |                                                                         
       +---------------------->  --------------------                                                                         
       |                         |                  |                                                                         
       |                         | caller parent BP |                                                                         
       |           BP(pseudo SP) --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   Local Var0     |                                                                         
       |                         --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   .......        |                                                                         
       |                         --------------------                                                                         
       |                         |                  |                                                                         
       |                         |   Local VarN     |                                                                         
                                 --------------------                                                                         
 caller stack frame              |                  |                                                                         
                                 |   callee arg2    |                                                                         
       |                         |------------------|                                                                         
       |                         |                  |                                                                         
       |                         |   callee arg1    |                                                                         
       |                         |------------------|                                                                         
       |                         |                  |                                                                         
       |                         |   callee arg0    |                                                                         
       |                         ----------------------------------------------+   FP(virtual register)                       
       |                         |                  |                          |                                              
       |                         |   return addr    |  parent return address   |                                              
       +---------------------->  +------------------+---------------------------    <-------------------------------+         
                                                    |  caller BP               |                                    |         
                                                    |  (caller frame pointer)  |                                    |         
                                     BP(pseudo SP)  ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |     Local Var0           |                                    |         
                                                    ----------------------------                                    |         
                                                    |                          |                                              
                                                    |     Local Var1           |                                              
                                                    ----------------------------                            callee stack frame
                                                    |                          |                                              
                                                    |       .....              |                                              
                                                    ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |     Local VarN           |                                    |         
                                  SP(Real Register) ----------------------------                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    |                          |                                    |         
                                                    +--------------------------+    <-------------------------------+         
                                                                                                                              
                                                              callee
```

## argsize 和 framesize 计算规则

### argsize

在函数声明中:
```go
 TEXT pkgname·add(SB),NOSPLIT,$16-32
```
前面已经说过 $16-32 表示 $framesize-argsize。Go 在函数调用时，参数和返回值都需要由 caller 在其栈帧上备好空间。callee 在声明时仍然需要知道这个 argsize。argsize 的计算方法是，参数大小求和+返回值大小求和，例如入参是 3 个 int64 类型，返回值是 1 个 int64 类型，那么这里的 argsize =  sizeof(int64) * 4。

不过真实世界永远没有我们假设的这么美好，函数参数往往混合了多种类型，还需要考虑内存对齐问题。

如果不确定自己的函数签名需要多大的 argsize，可以通过简单实现一个相同签名的空函数，然后 go tool objdump 来逆向查找应该分配多少空间。

### framesize
函数的 framesize 就稍微复杂一些了，手写代码的 framesize 不需要考虑由编译器插入的 caller BP，要考虑：

1. 局部变量，及其每个变量的 size。
2. 在函数中是否有对其它函数调用时，如果有的话，调用时需要将 callee 的参数、返回值、以及在调用 callee 时需要保存的 PC 寄存器的值，保存在当前函数栈顶，所以这些值的 size 也需要考虑在内。
3. 原则上来说，只要调用函数时只要不把局部变量覆盖掉就可以了。稍微多分配几个字节的 framesize 也不会死。
4. 在确保逻辑没有问题的前提下，你愿意覆盖局部变量也没有问题。只要保证进入和退出汇编函数时的 caller 和 callee 能正确拿到返回值就可以。

## 示例

###  add/sub/mul

math.go: 
```go
package main

import "fmt"

func add(a, b int) int // 汇编函数声明

func sub(a, b int) int // 汇编函数声明

func mul(a, b int) int // 汇编函数声明

func main() {
	fmt.Println(add(10, 11))
	fmt.Println(sub(99, 15))
	fmt.Println(mul(11, 12))
}
```
math.s
```go
#include "textflag.h" // 因为我们声明函数用到了 NOSPLIT 这样的 flag，所以需要将 textflag.h 包含进来

// func add(a, b int) int
TEXT ·add(SB), NOSPLIT, $0-24
	MOVQ a+0(FP), AX // 参数 a
	MOVQ b+8(FP), BX // 参数 b
	ADDQ BX, AX    // AX += BX
	MOVQ AX, ret+16(FP) // 返回
	RET

// func sub(a, b int) int
TEXT ·sub(SB), NOSPLIT, $0-24
	MOVQ a+0(FP), AX
	MOVQ b+8(FP), BX
	SUBQ BX, AX    // AX -= BX
	MOVQ AX, ret+16(FP)
	RET

// func mul(a, b int) int
TEXT ·mul(SB), NOSPLIT, $0-24
	MOVQ  a+0(FP), AX
	MOVQ  b+8(FP), BX
	IMULQ BX, AX    // AX *= BX
	MOVQ  AX, ret+16(FP)
	RET
    // 最后一行的空行是必须的，否则可能报 unexpected EOF
```
把这两个文件放在任意目录下，执行 `go build` 并运行就可以看到效果了。

### 伪寄存器 SP 、伪寄存器 FP 和硬件寄存器 SP 
来写一段简单的代码证明伪 SP、伪 FP 和硬件 SP 的位置关系。
spspfp.s
```go
#include "textflag.h"

// func output(int) (int, int, int)
TEXT ·output(SB), $8-48
	MOVQ 24(SP), DX // 不带 symbol，这里的 SP 是硬件寄存器 SP
	MOVQ DX, ret3+24(FP) // 第三个返回值
	MOVQ perhapsArg1+16(SP), BX // 当前函数栈大小 > 0，所以 FP 在 SP 的上方 16 字节处
	MOVQ BX, ret2+16(FP) // 第二个返回值
	MOVQ arg1+0(FP), AX
	MOVQ AX, ret1+8(FP)  // 第一个返回值
	RET

```
spspfp.go:
```go
package main

import (
	"fmt"
)

func output(int) (int, int, int) // 汇编函数声明

func main() {
	a, b, c := output(987654321)
	fmt.Println(a, b, c)
}
```
执行上面的代码，可以得到输出:

```shell
987654321 987654321 987654321
```
和代码结合思考，可以知道我们当前的栈结构是这样的:
```
------
ret2 (8 bytes)
------
ret1 (8 bytes)
------
ret0 (8 bytes)
------
arg0 (8 bytes)
------ FP
ret addr (8 bytes)
------ 
caller BP (8 bytes)
------ pseudo SP
frame content (8 bytes)
------ hardware SP
```
 本小节例子的 framesize 是大于 0 的，读者可以尝试修改 framesize 为 0，然后修改伪 SP 和硬件 SP 的偏移量，来使该程序也输出同样的结果。
### 汇编调用非汇编函数


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTE2MDk0NzcyNSwtMzQ4MTA0NjIzLDIwOD
QwNjM3MjAsLTE1NTYyODU0NDAsMTI2MTcwMTYyMyw3NTI0MDk2
NTUsMTg4NDQ5NTE5MF19
-->