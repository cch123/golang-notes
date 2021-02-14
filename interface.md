# iface 和 eface

Go 使用 iface 和 eface 来表达 interface。iface 其实就是 interface，这种接口内部是有函数的。

eface 表示 empty interface。

非空接口:
```go
// runtime2.go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

空接口:

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

所以我们可以比较容易地推断出来下面两种 interface 在 Go 内部是怎么表示的：

iface:

```go
type Reader interface {
    Read([]byte) (int, error)
}
```

eface:

```go
var x interface{}
```

## iface

### iface 适用场景

在 Go 语言中，我们可以借助 iface，实现传统 OOP 编程中的 DIP(dependency inversion principle)，即依赖反转。

```go
                               ┌─────────────────┐                                
                               │ business logic  │                                
                               └─────────────────┘                                
                                        │                                         
                                        │                                         
                                        │                                         
        ┌───────────────────────────────┼───────────────────────────────┐         
        │                               │                               │         
        │                               │                               │         
        │                               │                               │         
        ▼                               ▼                               ▼         
┌─────────────────┐            ┌─────────────────┐             ┌─────────────────┐
│       RPC       │            │    DATABASE     │             │ CONFIG_SERVICE  │
└─────────────────┘            └─────────────────┘             └─────────────────┘
```

上面这样的控制流关系在企业软件中很常见，但如果我们让软件的依赖方向与实际控制流方向一致，很容易导致 RPC、CONFIG_SERVER，DATABASE 的抽象被泄露到上层。例如在业务逻辑中有形如下面的结构体定义:

```go
type Person struct {
    Age int `db:"age"`
    Name string `db:"name"`
}
```

或者在业务逻辑中有类似下面的字眼:

```go
func CreateOrder(order OrderStruct) {
    config := GetConfigFromETCD() // abstraction leak
    CalcOrderMinusByConfig(config, order)
    SaveOrder()
}
```

业务逻辑不应该知道底层的实现，不应该有形如 db 的 tag，不应该知道配置是从 ETCD 中取出的。

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│                        business logic                        │
├────────────────────┬────────────────────┬────────────────────┤
│  configInterface   │   storeInterface   │    getInterface    │
└────────────────────┴────────────────────┴────────────────────┘
           ▲                    ▲                    ▲          
           │                    │                    │          
           │                    │                    │          
           │                    │                    │          
           │                    │                    │          
           │                    │                    │          
           │                    │                    │          
           │                    │                    │          
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ 
  │    etcd Impl    │  │    DATABASE     │  │       RPC       │ 
  └─────────────────┘  └─────────────────┘  └─────────────────┘ 
```

## eface

从定义来看，eface 其实是 iface 的一个子集:

```
┌─────────┐                                 ┌─────────┐                    
│  iface  │                          ┌─────▶│  itab   │                    
├─────────┴───────────────────┐      │      ├─────────┴───────────────────┐
│         tab  *itab          │──────┘      │    inter *interfacetype     │
┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫             ├─────────────────────────────┤
┃     data unsafe.Pointer     ┃             ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛             ┃        _type *_type         ┃
                                            ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫
                                            ├─────────────────────────────┤
                                            │        hash  uint32         │
                                            ├─────────────────────────────┤
                                            │      fun   [1]uintptr       │
                                            └─────────────────────────────┘
┌─────────┐                                                                
│  eface  │                                                                
├─────────┴───────────────────┐                                            
│        _type *_type         │                                            
├─────────────────────────────┤                                            
│     data unsafe.Pointer     │                                            
└─────────────────────────────┘                                            
```

既然是个子集，那已经有了 iface 为什么还需要 eface？显然是为了优化。

### eface 适用场景

举个标准库里的例子:

```go
func Slice(slice interface{}, less func(i, j int) bool)
```

标准库的 container package，因为不知道用户会在 container 中存储什么类型，所以大部分 API 也是用 interface{} 的。

除了标准库，我们在 fmt.Print 系列和 json.Unmarshal 中也会见到 interface{}。

```go
func Println(a ...interface{}) (n int, err error)
```

```go
func Unmarshal(data []byte, v interface{}) error
```

总结一下，使用 interface 主要是我们在不知道用户的输入类型信息的前提下，希望能够实现一些通用数据结构或函数。这时候便会将空 interface{} 作为函数的输入/输出参数。在其它语言里，解决这种问题一般使用泛型。

## 类型转换

### 普通类型转换为 eface

```go
 1	package main
 2
 3	var i interface{}
 4
 5	func main() {
 6		var y = 999
 7		i = y
 8		var x = i.(int)
 9		println(x)
10	}
```

老规矩，看汇编知一切，主要是第 7 行:

```go
0x0021 00033 (interface.go:7)	MOVQ	$999, (SP)          // 把 999 作为参数挪到栈底
0x0029 00041 (interface.go:7)	CALL	runtime.convT64(SB) // 以 999 作为参数调用 runtime.convT64，内部会为该变量在堆上分配额外空间，并返回其地址
0x002e 00046 (interface.go:7)	MOVQ	8(SP), AX           // 返回值移动到 AX 寄存器
0x0033 00051 (interface.go:7)	LEAQ	type.int(SB), CX    // 将 type.int 值赋值给 CX 寄存器
0x003a 00058 (interface.go:7)	MOVQ	CX, "".i(SB)        // 将 type.int 赋值给 eface._type
0x004a 00074 (interface.go:7)	MOVQ	AX, "".i+8(SB)      // 将数据 999 的地址赋值给 eface.data
```

还是比较简单的。

### eface 转换为普通类型

空接口转成普通类型，其实就是断言。我们继续使用上面的 demo:

```go
 1	package main
 2
 3	var i interface{}
 4
 5	func main() {
 6		var y = 999
 7		i = y
 8		var x = i.(int) // 这次关注断言的实现
 9		println(x)
10	}
```

看看第 8 行的输出:

```go
0x0033 00051 (interface.go:7)	LEAQ	type.int(SB), CX  // 在第 7 行把 type.int 搬到 CX 寄存器了，注意下面要用到
.....
0x0051 00081 (interface.go:8)	MOVQ	"".i+8(SB), AX    // AX = eface.data
0x0058 00088 (interface.go:8)	MOVQ	"".i(SB), DX      // DX = eface._type
0x005f 00095 (interface.go:8)	CMPQ	DX, CX            // 比较 _type 和 type.int 是否相等
0x0062 00098 (interface.go:8)	JNE	161               // 如果不等，说明断言失败，跳到 161 位置，开始执行 panic 流程
0x0064 00100 (interface.go:8)	MOVQ	(AX), AX          // 把 AX 地址里的内容搬到 AX 寄存器，现在 AX 里存的是 999 了
0x0067 00103 (interface.go:8)	MOVQ	AX, "".x+24(SP)   // 将 999 赋值给变量 x

// 下面这部分是实现 panic 的，不是我们的重点
0x00a1 00161 (interface.go:8)	MOVQ	DX, (SP)
0x00a5 00165 (interface.go:8)	MOVQ	CX, 8(SP)
0x00aa 00170 (interface.go:8)	LEAQ	type.interface {}(SB), AX
0x00b1 00177 (interface.go:8)	MOVQ	AX, 16(SP)
0x00b6 00182 (interface.go:8)	CALL	runtime.panicdottypeE(SB)
0x00bb 00187 (interface.go:8)	XCHGL	AX, AX
0x00bc 00188 (interface.go:8)	NOP
```

也挺简单的，这里我们用的是 int 来进行实验，对于稍微复杂一些的复合类型，也只是多出了一些步骤，本质上没有什么区别，感兴趣的读者可以自行研究。

### 普通类型转换为 iface

```go
 1	package main
 2
 3	import (
 4		"io"
 5		"os"
 6	)
 7
 8	var j io.Reader
 9
10	func main() {
11		var i, _ = os.Open("ab.go")
12		j = i
13		var k = j.(*os.File)
14		println(k)
15	}
```

看看第 12 行的汇编:

```go
0x0050 00080 (interface2.go:12)	LEAQ	go.itab.*os.File,io.Reader(SB), CX
0x0057 00087 (interface2.go:12)	MOVQ	CX, "".j(SB)
0x0067 00103 (interface2.go:12)	MOVQ	AX, "".j+8(SB)
```

和 eface 的赋值其实差不多，稍微有区别的是这里的 go.itab.*os.File,io.Reader(SB)，这个符号(理解成一个全局变量，类型是 runtime._type 就行)是编译器帮我们生成的:

```go
go.itab.*os.File,io.Reader SRODATA dupok size=32
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 44 b5 f3 33 00 00 00 00 00 00 00 00 00 00 00 00  D..3............
	rel 0+8 t=1 type.io.Reader+0
	rel 8+8 t=1 type.*os.File+0
	rel 24+8 t=1 os.(*File).Read+0
```

并且是按需生成的，如果你把 `*os.File` 赋值给 io.Writer，在编译结果中也能找到类似的符号:

```go
go.itab.*os.File,io.Writer SRODATA dupok size=32
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 44 b5 f3 33 00 00 00 00 00 00 00 00 00 00 00 00  D..3............
	rel 0+8 t=1 type.io.Writer+0
	rel 8+8 t=1 type.*os.File+0
	rel 24+8 t=1 os.(*File).Write+0
```

看起来就容易理解了。

### iface 转换为普通类型

```go
 1	package main
 2
 3	import (
 4		"io"
 5		"os"
 6	)
 7
 8	var j io.Reader
 9
10	func main() {
11		var i, _ = os.Open("ab.go")
12		j = i
13		var k = j.(*os.File)
14		println(k)
15	}
```

主要关心第 13 行的汇编结果:

```go
// 数据准备在这里，AX 和 CX 寄存器 13 行会用到
0x0050 00080 (interface2.go:12)	LEAQ	go.itab.*os.File,io.Reader(SB), CX
0x0057 00087 (interface2.go:12)	MOVQ	CX, "".j(SB)
0x0067 00103 (interface2.go:12)	MOVQ	AX, "".j+8(SB)

0x006e 00110 (interface2.go:13)	MOVQ	"".j+8(SB), AX  // AX = j.data
0x0075 00117 (interface2.go:13)	MOVQ	"".j(SB), DX    // DX = j.itab
0x007c 00124 (interface2.go:13)	CMPQ	DX, CX          // 比较 iface.itab 和  go.itab.*os.File,io.Reader(SB) 是否一致
0x007f 00127 (interface2.go:13)	JNE	187             // 不一致，说明断言失败，跳到 panic 流程

// 下面就是正常流程了
```
