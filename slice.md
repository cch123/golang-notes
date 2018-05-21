# slice 和 array

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

slice 的底层结构定义非常直观，指向底层数组的指针，当前长度 len 和当前 slice 的 cap。

数据指针不一定就是指向底层数组的首部，也可以指腰上:

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

我们可以轻松地推断出，是可以有多个 slice 指向同一个底层数组的。一般情况下，一个 slice 的 cap 取决于其底层数组的长度。如果在元素追加过程中，底层数组没有更多的空间了，那么这时候就需要申请更大的底层数组，并发生数据拷贝。这时候的 slice 的底层数组的指针地址也会发生改变，务必注意。

## len 和 cap

Go 语言虽然将 len 和 cap 作为 slice 和 array 附带的 builtin 函数，但对这两个函数的调用实际上最终会被编译器直接计算出结果，并将值填到代码运行的位置上。所以 len 和 cap 更像是宏一样的东西，在 slice 和 array 的场景，会被直接展开为 sl->len 和 sl->cap 这样的结果。

## 源码分析

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU1MTU3OTMwOSwyMTQwOTE5Njg4XX0=
-->