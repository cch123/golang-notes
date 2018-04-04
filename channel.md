## Channel
### 数据结构

```go
// channel 在 runtime 中的结构体

type hchan struct {

	// 队列中目前的元素计数

	qcount uint // total data in the queue

	// 环形队列的总大小

	dataqsiz uint // size of the circular queue

	// void * 的内存 buffer 区域

	buf unsafe.Pointer // points to an array of dataqsiz elements

	// sizeof chan 中的数据

elemsize uint16

// 是否已被关闭

closed uint32

// runtime._type，代表 channel 中的元素类型的 runtime 结构体

elemtype *_type // element type

// 发送索引

sendx uint // send index

// 接收索引

recvx uint // receive index

// 接收 goroutine 对应的 sudog 队列

recvq waitq // list of recv waiters

// 发送 goroutine 对应的 sudog 队列

sendq waitq // list of send waiters

// lock protects all fields in hchan, as well as several

// fields in sudogs blocked on this channel.

//

// Do not change another G's status while holding this lock

// (in particular, do not ready a G), as this can deadlock

// with stack shrinking.

lock mutex

}
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2NTg2NDAwMTQsMTM3ODc1Mjg4M119
-->