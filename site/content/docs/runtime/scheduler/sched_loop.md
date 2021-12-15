---
title: 调度流程(WIP)
weight: 1
---

## 组件大图

![](/images/runtime/schedule/sche_big.png)

## Go 的调度流程

我们可以认为 goroutine 的创建与调度循环是一个生产-消费流程。整个 go 程序的运行就是在不断地执行 goroutine 的生产与消费流程。

创建 goroutine 即是在创建任务，这些生产出来的 goroutine 可能会有三个去处，分别是：

* p.runnext
* p.localrunq
* schedt.global runq

按照执行权来讲，优先级是逐渐降低的。

调度循环会不断地从上面讲的三个目标中消费 goroutine，并执行。

## goroutine 生产

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FgByIPDf4nRr6No4dNYjn3e%2Fbootstrap%3Fpage-id%3D242%253A7%26node-id%3D242%253A323%26viewport%3D241%252C48%252C0.17%26scaling%3Dscale-down-width%26starting-point-node-id%3D242%253A9" allowfullscreen></iframe>
{{</rawhtml>}}

## goroutine 消费

{{<rawhtml>}}
<iframe style="border: 1px solid rgba(0, 0, 0, 0.1);" width="720" height="450" src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Fproto%2FgByIPDf4nRr6No4dNYjn3e%2Fbootstrap%3Fpage-id%3D143%253A212%26node-id%3D143%253A213%26viewport%3D241%252C48%252C0.06%26scaling%3Dscale-down-width%26starting-point-node-id%3D143%253A213" allowfullscreen></iframe>
{{</rawhtml>}}
