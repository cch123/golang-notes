# slice

要说 slice，那实在是太让人熟悉了，从功能上讲 slice 支持追加，按索引引用，按索引范围生成新的 slice，自动扩容等，和 C++ 或 Java 中的 Vector 有些类似，但也有一些区别。

不过 Go 里的 slice 还有一个底层数组的概念，这一点和其它语言不同。

```go
runtime/slice.go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

slice 的底层结构定义非常直观，指向底层数组的指针，当前长度 len 和当前slice 的 cap。

```
                                                                   
       []int{1,3,4,5}                                              
                                                                   
       struct {                                                    
         array unsafe.Pointer --------------+                      
         len int                            |                      
         cap int                            |                      
       }                                    |                      
                                            |                      
                                            v                      
                                                                   
                               +------|-------|------|------+-----+
                               |      |  1    |  3   | 4    |  5  |
                               |      |       |      |      |     |
                               +------|-------|------|------+-----+
                                                 [5]int            
                                                                   
                                                                   
                                                                   

```



> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjE0MDkxOTY4OF19
-->