plan9 assembly 完全解析

众所周知，Go 使用了 Unix 老古董(误 们发明的 plan9 汇编。就算你对 x86 汇编有所了解，在 plan9 里还是有些许区别。说不定你在看代码的时候偶然发现代码里的寄存器他写着 SP 但不是 SP 的时候就被恶心到一下什么的。

本文将对 plan9 汇编进行全面的介绍，同时解答你在接触 plan9 汇编时可能遇到的大部分问题。

本文所使用的平台是 linux amd64，因为不同的平台指令集和寄存器都不一样，所以没有办法共同探讨。这也是由汇编本身的性质决定的。

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
在 plan9
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyNTAxMDI4NjksNzEwNTAzNDMxLC02Mz
k0ODkxMTYsLTIxNjU2NDc4NSwxMjQwNTc4NzI3XX0=
-->