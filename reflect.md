# reflect

reflect.TypeOf => 具体类型，例如 main.Person(struct)，map[string]int

reflect.TypeOf.Kind => 总体类型，例如是 struct，还是 map，还是 int 什么的。

reflect.ValueOf => 获取 Value，并进行修改

reflect.ValueOf.Elem => 获取指针指向的对象、获取 map 的 val，获取 interface 中的具体元素，

// It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.

## 基本类型

bool, int, uint, uintptr, float, complex

```go
var a = 1

// change a
reflect.ValueOf(a).SetInt(1) // panic, ValueOf(a) is not addressable
reflect.ValueOf(&a).Elem().SetInt(2) // correct
```

## map type

```go
var a map[string]int

fmt.Println(reflect.TypeOf(a)) // reflect.Type : map[string]int
fmt.Println(reflect.TypeOf(a).Kind()) // reflect.Kind : map
fmt.Println(reflect.TypeOf(a).Elem()) // reflect.Type : int
fmt.Println(reflect.TypeOf(a).Key())  // reflect.Type : string

// 使用反射初始化 map
reflect.ValueOf(a).Set(reflect.MakeMap(reflect.TypeOf(a))) // error ! ValueOf(a) is not addressable
reflect.ValueOf(&a).Elem().Set(reflect.MakeMap(reflect.TypeOf(a))) // correct

// set key value
reflect.ValueOf(a).SetMapIndex(reflect.ValueOf("abc"), reflect.ValueOf(1))

```

reflect.Type 是个 interface{} 类型，如果要按照类型做 switch，一定要按照 reflect.Kind 来做，因为 Kind 是枚举值。

当然，也可以使用 type switch，具体参考 elasticsql。

## struct type

```go
```

注意，time.Time 也是一个 struct 类型，所以通过 Kind 来判断是会有问题的。

```go
var a time.Time
fmt.Println(reflect.TypeOf(a).Kind()) // struct
fmt.Println(reflect.TypeOf(a))        // time.Time
fmt.Println(reflect.TypeOf(a) == reflect.TypeOf(time.Time{}))        // true
```

## array/slice type

slice :

```go
var a []int
var b []int
reflect.ValueOf(&a).Elem().Set(reflect.MakeSlice(reflect.TypeOf(a), 0, 10))
reflect.ValueOf(&b).Elem().Set(reflect.ValueOf([]int{1, 2}))
reflect.ValueOf(&a).Elem().Set(reflect.Append(reflect.ValueOf(a), reflect.ValueOf(1)))
fmt.Printf("%#v\n", a)
fmt.Printf("%#v\n", b)
```

array :

```go
```

## ptr type
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjg5ODA4NDEwLDY4MTA3NjE0OSw2ODk4MD
g0MTBdfQ==
-->