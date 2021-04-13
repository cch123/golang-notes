# Generics

在泛型出现之前，社区里也有一些伪泛型方案，例如 [genny](https://github.com/cheekybits/genny)。

genny 本质是基于文本替换的，比如 example 里的例子：

```go
package queue

import "github.com/cheekybits/genny/generic"

// NOTE: this is how easy it is to define a generic type
type Something generic.Type

// SomethingQueue is a queue of Somethings.
type SomethingQueue struct {
  items []Something
}

func NewSomethingQueue() *SomethingQueue {
  return &SomethingQueue{items: make([]Something, 0)}
}
func (q *SomethingQueue) Push(item Something) {
  q.items = append(q.items, item)
}
func (q *SomethingQueue) Pop() Something {
  item := q.items[0]
  q.items = q.items[1:]
  return item
}
```

执行替换命令：

```shell
cat source.go | genny gen "Something=string"
```

然后 genny 会将代码中所有 Something 替换成 string，同时确保大小写与原来一致，不影响字段/类型的导出特征。

没有官方的泛型支持，社区怎么搞都是邪道。2021 年 1 月，官方的方案已经基本上成型，并释出了 [draft design](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md)。

