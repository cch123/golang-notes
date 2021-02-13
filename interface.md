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

### eface 实现原理

## 类型转换

### 普通类型转换为 eface

### 普通类型转换为 iface

### eface 转换为普通类型

### iface 转换为普通类型

### 类型转换图
