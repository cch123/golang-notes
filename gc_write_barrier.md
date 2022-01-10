# GC write barrier 详解

在垃圾回收领域所讲的 barrier 包括 read barrier 与 write barrier，无论是哪一种，都与并发编程领域的 memory barrier 不是一回事。

在 GC 中的 barrier 其本质是 : snippet of code insert before pointer modify。

所以在阅读相关材料时，请注意不要将两者混淆。

在当前 Go 语言的实现中，GC 只有 write barrier，没有 read barrier。

在应用进入 GC 标记阶段前的 stw 阶段，会将全局变量 runtime.writeBarrier.enabled 修改为 true，当应用从 STW 中恢复，重新开始执行，垃圾回收的标记阶段便与应用逻辑开始并发执行，这时所有的堆上指针修改操作在修改之前会额外调用 runtime.gcWriteBarrier：

![](./images/garbage_collection/barrier_asm.png)

我们随便找找这些反汇编结果在代码中对应的行：

![](./images/garbage_collection/barrier_code.png)

Go 语言当前使用了混合 barrier 来实现 gc 标记期间的被修改对象动态跟踪，早期只使用了 Dijistra 插入 barrier，但 Dijistra barrier 需要在标记完成之后进行栈重扫，因此在 1.8 时修改为混合 barrier。

Dijistra 插入屏障伪代码如下：

![](http://xargin.com/content/images/2021/12/image-42.png)

堆上指针修改时，新指向的对象要标灰。

但是因为 Go 的栈上对象不加 barrier，所以会存在对象丢失的问题：

![](http://xargin.com/content/images/2021/12/djb.gif)

还有一种非常有名的 barrier，Yuasa 删除屏障，与 Dijistra 屏障相反，它是在堆指针指向的对象发生变化时，将之前指向的对象标灰：

![](http://xargin.com/content/images/2021/12/image-43.png)

和 Dijistra 类似，也存在对象漏标问题：

![](http://xargin.com/content/images/2021/12/yb.gif)
