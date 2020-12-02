# 1.14 defer

在 Go 1.14 中，增加了一种新的 defer 实现：open coded defer。当函数内 defer 不超过 8 个时，则会使用这种实现。

```go
package main

import "fmt"

var i = 100

func main() {
	if i == 0 {
		defer fmt.Println("1")
	}

	if i == 1 {
		defer fmt.Println("2")
	}

	if i == 2 {
		defer fmt.Println("3")
	}

	if i == 3 {
		defer fmt.Println("4")
	}

	if i == 4 {
		defer fmt.Println("5")
	}

	if i == 5 {
		defer fmt.Println("6")
	}
}

```

可以分析上面的代码生成的汇编来理解这个新版的 open coded defer 到底是怎么实现的：

```
```
