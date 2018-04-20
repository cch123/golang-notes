plan9 assembly 完全解析

众所周知，Go 使用了 Unix 老古董(误 们发明的 plan9 汇编。就算你对 x86 汇编有所了解，在 plan9 里还是有些许区别。说不定你在看代码的时候偶然发现代码里的寄存器他写着 SP 但不是 SP 的时候就被恶心到一下什么的。

本文将对 plan9 汇编进行全面的介绍，同时解答你在接触 plan9 汇编时可能遇到的大部分问题。

本文所使用的平台是 linux amd64，因为不同的平台指令集和寄存器都不一样，所以没有办法共同探讨。这也是由汇编本身的性质jy


<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEyNzk2Mjk2MDYsLTIxNjU2NDc4NSwxMj
QwNTc4NzI3XX0=
-->