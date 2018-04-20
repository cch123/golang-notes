## plan9 assembly 完全解析

众所周知，Go 使用了 Unix 老古董(误 们发明的 plan9 汇编。就算你对 x86 汇编有所了解，在 plan9 里还是有些许区别。说不定你在看代码的时候偶然发现代码里的寄存器他写着 SP 但不是 SP 的时候就被恶心到一下什么的。

本文将对 plan9 汇编进行全面的介绍，同时解答你在接触 plan9 汇编时可能遇到的大部分问题。

本文所使用的平台是 linux amd64，因为不同的平台指令集和寄存器都不一样，所以没有办法共同探讨。这也是由汇编本身的性质决定的。
### 指令构成

```go
```

### 寄存器
#### 通用寄存器
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

在使用这些真实寄存器时，可以直接忽略前缀 `r`，例如要使用 rax 寄存器时，只要写 AX 即可，例如:

```go
MOVQ $101, AX
```

和

```asm
mov rax, 101
```
是等价的。其它寄存器以此类推。

#### 伪寄存器
Go 的汇编还引入了 4 个伪寄存器，援引官方文档的描述:
>-   `FP`: Frame pointer: arguments and locals.
>-   `PC`: Program counter: jumps and branches.
>-   `SB`: Static base pointer: global symbols.
>-   `SP`: Stack pointer: top of stack.

如果没有 FP 和 SP(注意这里的 SP 不是硬件的那个 SP) 的情况下，例如在 intel 汇编中，我们只能使用 bp 或者 sp + offset 来找我们的变量位置。而有了 FP 和 SP，我们可以直接以其为基准进行参数查找和局部变量引用，即使编译之后他们的相对位置变化了，对手写代码也是无感知的。至于它们的相对位置为什么会变化，在后文中会进行说明。

实际上这里官方文档中，对这几个伪寄存器的描述并不精确，虽然文档之后还有一点补充说明说明，但由于是拿很多硬件平台泛泛而谈，细节并没有讲明白。

我们来画一个通常情况下被调用的函数的栈结构图:

TODO，这里有图

图上的 caller BP，指的是 caller 的 BP 寄存器值，有些人把 caller BP 叫作 caller 的 frame pointer，实际上这个习惯是从 x86 架构沿袭来的。虽然在 Go 的 asm 文档中把伪寄存器 FP 也称为 frame pointer，但是这两个 frame pointer 根本不是一回事。

此外需要注意的是，caller BP 是在编译期由编译器插入的，用户手写代码时，计算 frame size 时是不包括这个 caller BP 部分的。图上可以看到，FP 伪寄存器指向函数的传入参数的开始位置，因为栈是朝低地址方向增长，为了通过寄存器引用参数时方便，所以参数的摆放方向和栈的增长方向是相反的，即：
```
                              FP
high ----------------------> low
argN, ... arg3, arg2, arg1, arg0
```
假设所有参数均为 8 字节，这样我们就可以用 symname+0(FP) 访问第一个 参数，symname+8(FP) 访问第二个参数，以此类推。

我们来看看一个典型的 plan9 的汇编函数的定义：
```go
// func add(a, b int) int
TEXT (pkgname)·add(SB), NOSPLIT, $0-8
	MOVQ a+0(FP), AX
	MOVQ a+8(FP), BX
	ADDQ AX, BX
	MOVQ BX, ret+16(FP)
	RET
```
为什么要以 TEXT 来开头呢？如果对程序数据在文件中和内存中的分段稍有了解的同学应该知道，我们的代码在二进制文件中，是存储在 .text 段中的，这里也就是一种约定俗成的使用方法。

定义中的 pkgname 部分是可以省略的，如果你有强迫症，那写上也没有什么问题。

中点 `·` 比较特殊，是一个 unicode 的中点，该点在 mac 下的输入方法是 `option+shift+9`。在程序被链接之后，所有的中点`·` 都会被替换为`.`，比如你的方法是 `runtime·main`，在编译之后的程序里的符号则是 `runtime.main`。嗯，看起来很变态。

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


### 变量声明
在汇编里的qf
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE1ODUwODE3NjMsLTIwNjgxMzI5NTMsMT
A2ODQ1MzkwMywtMzcwNzYzODQ3LDk4NDcwNTI4Myw5NjI2NDcz
MCwxMzg5ODU1MjEzLC0xODIyODQwNjc2LDcxMDUwMzQzMSwtNj
M5NDg5MTE2LC0yMTY1NjQ3ODUsMTI0MDU3ODcyN119
-->