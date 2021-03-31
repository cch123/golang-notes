# interface
## eface
#### 从类型定义说起
首先来看一段代码
```go
package main

type eface_struct struct {
	f float32
	i int
}

func main() {
	var test_interface interface{}
	test_interface = eface_struct{}
	_ = test_interface
}
```
在这段简单的代码中我们声明了 `test_interface` 是一个不含任何函数的基础 `interface` 结构，并把 `eface_struct` 类型结构体赋予了它

那么我们来看看编译器是如何处理`声明`这一过程的
> go tool compile  -l -S -N ./test.go
注意这边将编译器优化关闭`-N`，以便我们更好的看看编译器的工作流程

```assembly
(... 以下已去除多余代码)
"".main STEXT nosplit size=91 args=0x0 locals=0x38
	0x0000 00000 (./test.go:8)	TEXT	"".main(SB), NOSPLIT|ABIInternal, $56-0
	0x0000 00000 (./test.go:8)	SUBQ	$56, SP
	0x0004 00004 (./test.go:8)	MOVQ	BP, 48(SP)
	0x0009 00009 (./test.go:8)	LEAQ	48(SP), BP
	0x000e 00014 (./test.go:8)	PCDATA	$0, $-2
	0x000e 00014 (./test.go:8)	PCDATA	$1, $-2
	0x000e 00014 (./test.go:8)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x000e 00014 (./test.go:8)	FUNCDATA	$1, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x000e 00014 (./test.go:8)	FUNCDATA	$2, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x000e 00014 (./test.go:9)	PCDATA	$0, $0
	0x000e 00014 (./test.go:9)	PCDATA	$1, $0
	0x000e 00014 (./test.go:9)	XORPS	X0, X0
	0x0011 00017 (./test.go:9)	MOVUPS	X0, "".test_interface+32(SP)
	0x0016 00022 (./test.go:10)	XORPS	X0, X0
	0x0019 00025 (./test.go:10)	MOVSS	X0, ""..autotmp_1+16(SP)
	0x001f 00031 (./test.go:10)	MOVQ	$0, ""..autotmp_1+24(SP)
	0x0028 00040 (./test.go:10)	MOVSS	""..autotmp_1+16(SP), X0
	0x002e 00046 (./test.go:10)	MOVSS	X0, ""..autotmp_2(SP)
	0x0033 00051 (./test.go:10)	MOVQ	$0, ""..autotmp_2+8(SP)
	0x003c 00060 (./test.go:10)	PCDATA	$0, $1
	0x003c 00060 (./test.go:10)	LEAQ	type."".eface_struct(SB), AX
	0x0043 00067 (./test.go:10)	PCDATA	$0, $0
	0x0043 00067 (./test.go:10)	MOVQ	AX, "".test_interface+32(SP)
	0x0048 00072 (./test.go:10)	PCDATA	$0, $1
	0x0048 00072 (./test.go:10)	LEAQ	""..autotmp_2(SP), AX
	0x004c 00076 (./test.go:10)	PCDATA	$0, $0
	0x004c 00076 (./test.go:10)	MOVQ	AX, "".test_interface+40(SP)
	0x0051 00081 (./test.go:12)	MOVQ	48(SP), BP
	0x0056 00086 (./test.go:12)	ADDQ	$56, SP
	0x005a 00090 (./test.go:12)	RET
	0x0000 48 83 ec 38 48 89 6c 24 30 48 8d 6c 24 30 0f 57  H..8H.l$0H.l$0.W
	0x0010 c0 0f 11 44 24 20 0f 57 c0 f3 0f 11 44 24 10 48  ...D$ .W....D$.H
	0x0020 c7 44 24 18 00 00 00 00 f3 0f 10 44 24 10 f3 0f  .D$........D$...
	0x0030 11 04 24 48 c7 44 24 08 00 00 00 00 48 8d 05 00  ..$H.D$.....H...
	0x0040 00 00 00 48 89 44 24 20 48 8d 04 24 48 89 44 24  ...H.D$ H..$H.D$
	0x0050 28 48 8b 6c 24 30 48 83 c4 38 c3                 (H.l$0H..8.
	rel 63+4 t=16 type."".eface_struct+0                                ...i
type."".eface_struct SRODATA size=144
	0x0000 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 ca b4 29 a7 07 08 08 19 00 00 00 00 00 00 00 00  ..).............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00  ........@.......
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	rel 24+8 t=1 type..eqfunc."".eface_struct+0
	rel 32+8 t=1 runtime.gcbits.+0
	rel 40+4 t=5 type..namedata.*main.eface_struct-+0
	rel 44+4 t=5 type.*"".eface_struct+0
	rel 48+8 t=1 type..importpath."".+0
	rel 56+8 t=1 type."".eface_struct+96
	rel 80+4 t=5 type..importpath."".+0
	rel 96+8 t=1 type..namedata.f-+0
	rel 104+8 t=1 type.float32+0
	rel 120+8 t=1 type..namedata.i-+0
	rel 128+8 t=1 type.int+0
```
那么其中的
```assembly
  0x003c 00060 (./test.go:10)	LEAQ	type."".eface_struct(SB), AX
  0x0043 00067 (./test.go:10)	PCDATA	$0, $0
  0x0043 00067 (./test.go:10)	MOVQ	AX, "".test_interface+32(SP)
  0x0048 00072 (./test.go:10)	PCDATA	$0, $1
  0x0048 00072 (./test.go:10)	LEAQ	""..autotmp_2(SP), AX
  0x004c 00076 (./test.go:10)	PCDATA	$0, $0
  0x004c 00076 (./test.go:10)	MOVQ	AX, "".test_interface+40(SP) 
```
就是对 `test_interface` 变量的赋值过程了，可以看到首先将`type."".eface_struct`放到了该变量的首字节地址上

##### type."".eface_struct 是什么?
在声明一个结构体的同时, 编译器会在只读内存段定义对应的 `*face` 结构,在平时正常使用 `struct` 的时候,编译器会通过用户定义的结构进行内存布局的计算来满足结构体的实现,这是属于普通情况的使用，而一旦涉及到`interface`,那么这个`*face`结构就开始发挥它的作用了
##### interface 的声明和赋值过程到底干了些什么？

> var test_interface interface{}
test_interface = eface_struct{}

让我们来看看对 `test_interface` 这个 `interface` 类型变量赋值的过程
对应 assembly:
```assembly
 # 将 type."".eface_struct 这个 *face 结构插入 test_interface 首地址
 x003c 00060 (./test.go:10)	LEAQ	type."".eface_struct(SB), AX
 0x0043 00067 (./test.go:10)	PCDATA	$0, $0
 0x0043 00067 (./test.go:10)	MOVQ	AX, "".test_interface+32(SP) # 不用纠结这边的 32 偏移量,与局部变量有关,可认为 test_interface 起始地址为 32(SP)
 0x0048 00072 (./test.go:10)	PCDATA	$0, $1
 0x0048 00072 (./test.go:10)	LEAQ	""..autotmp_2(SP), AX
 0x004c 00076 (./test.go:10)	PCDATA	$0, $0
 # test_interface 数据部分值插入，偏移量为 40-32 = 8
 0x004c 00076 (./test.go:10)	MOVQ	AX, "".test_interface+40(SP)
```
这边的案例首先分析的是不包含 `func` 的 `interface`,对应的 `*face` 结构为 `eface`
```go
// src/runtime/runtime2.go
type eface struct {
	_type *_type  // size 8 正好对应上述 assembly 部分 *face 结构插入
	data  unsafe.Pointer // 数据部分
}
// src/runtime/type.go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldAlign uint8
	kind       uint8
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```
再看 `_type` 结构,那么有了到目前为止的分析,我们可以试着来将该结构与编译器自动生成的`type."".eface_struct`对应着拆解看看
```assembly
type."".eface_struct SRODATA size=144
	0x0000 10 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 ca b4 29 a7 07 08 08 19 00 00 00 00 00 00 00 00  ..).............
	0x0020 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0040 02 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00  ................
	0x0050 00 00 00 00 00 00 00 00 40 00 00 00 00 00 00 00  ........@.......
	0x0060 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0070 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0080 00 00 00 00 00 00 00 00 10 00 00 00 00 00 00 00  ................
	#------------------------ 24 字节 ----------------
	rel 24+8 t=1 type..eqfunc."".eface_struct+0    # 对应 _type.equal
	rel 32+8 t=1 runtime.gcbits.+0
	rel 40+4 t=5 type..namedata.*main.eface_struct-+0
	rel 44+4 t=5 type.*"".eface_struct+0
	rel 48+8 t=1 type..importpath."".+0
	rel 56+8 t=1 type."".eface_struct+96
	rel 80+4 t=5 type..importpath."".+0
	rel 96+8 t=1 type..namedata.f-+0
	rel 104+8 t=1 type.float32+0
	rel 120+8 t=1 type..namedata.i-+0
	rel 128+8 t=1 type.int+0
```

```go
fmt.Println(reflect.TypeOf(test_interface))
/*
 *  reflect.rtype
 * 	|- size: 16									
 * 	|- ptrdata:0									
 *	|- hash 2804528330
 const (
	// tflagUncommon means that there is a pointer, *uncommonType,
	// just beyond the outer type structure.
	//
	// For example, if t.Kind() == Struct and t.tflag&tflagUncommon != 0,
	// then t has uncommonType data and it can be accessed as:
	//
	//	type tUncommon struct {
	//		structType
	//		u uncommonType
	//	}
	//	u := &(*tUncommon)(unsafe.Pointer(t)).u
	tflagUncommon tflag = 1 << 0

	// tflagExtraStar means the name in the str field has an
	// extraneous '*' prefix. This is because for most types T in
	// a program, the type *T also exists and reusing the str data
	// saves binary size.
	tflagExtraStar tflag = 1 << 1

	// tflagNamed means the type has a name.
	tflagNamed tflag = 1 << 2

	// tflagRegularMemory means that equal and hash functions can treat
	// this type as a single region of t.size bytes.
	tflagRegularMemory tflag = 1 << 3
)
 *	|- tflag:tflagUncommon|tflagExtraStar|tflagNamed (7)
 *	|- align:8
 *	|- fieldAlign:8
 *	|- kind:25
 *	|- equal:type..eq.main.eface_struct
 *	|- gcdata:<*uint8>(0x11004e6)
 *	|- str:23831	nameOff,通过该字段及 base pointer 计算偏移量来获取 struct name 相关信息
 *	|_ ptrToThis:38080
 */
```
- size:16
	- 0x 00 00 00 00 00 00 00 10
- ptrdata:0
	- 0x 00 00 00 00 00 00 00 00
- hash:2804528330
	- 0x a7 29 b4 ca

大小端的关系，和上述`assembly`中的前 20 个字节顺序相反,分析到这儿就可以知道`interface`结构整体的情况，以及起核心作用的为`*face`字段,此处分析了`eface`,而另一种带有函数实现的`interface`的`iface`结构也可以通过相同的过程拆解,值得一提的是`iface`结构首字段的`itab`结构内部也内置了 `_type`结构，所以编译器可以通过相同的赋值过程处理这两种类型的`interface`