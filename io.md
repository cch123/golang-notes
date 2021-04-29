# directio
## page cache
页面缓存（Page Cache）是 Linux 内核中针对文件I/O的一项优化, 众所周知磁盘 I/O 的成本远比内存访问来得高, 如果每次进行文件读写都需要直接进行磁盘操作, 那成本会是非常高的, 因此, kernel 针对于文件 I/O 设计了 page cache, 简单来说就是将目标读写的文件页缓存在内存中, 而后操作这块缓存进行读写 (而且例如针对机械磁盘来说, 为了降低磁头寻道的耗时, page cache 通常会采用预读的机制), 写入新数据后该页变为脏页, 等待刷盘, 刷脏的操作可由用户主动请求 (fsync) 或者由内核在合适的时机进行操作

### 总结下 page cache 的好处:
- 缓存最近被访问的数据, 提高文件 I/O 的效率
- 预读功能减少磁头寻道损耗

## 那么 page cache 的设计一定是能提高服务效率的么?

来考虑下一个场景:
> 某服务正在正常工作, 存放了许许多多的静态资源等待访问, 大小为小到几十 kb, 大到几百 GB （大到无法全都加载到内存, 只能存放在本地磁盘）的大文件不等。小文件为热点文件, 大文件为冷门资源, 某天, 突然有用户进行了大文件的访问。

这时候会发生什么？

正常工作的情况下假设内存足够缓存所有小文件, 服务无需进行磁盘 I/O, 而这时候突然来了个大文件的缓存, 直接跑满了大部分的 page cache, 造成内核不得不通过淘汰策略将 page cache 置换了出来(先暂不考虑各种内存拷贝的损耗), 那么接下去再去访问那一堆热点小文件就不得不去进行磁盘 I/O, 然后写入 page cache, 两者之间的矛盾无法调和不断重复, 尤其是大量的小文件基本都为随机读写。从而服务的压力增加, 效率降低。

### 如何优化这种场景?
> 各架构机器支持程度不一, 此处仅讨论 linux x86

```shell
dd if=/dev/zero of=test bs=1M count=1000 # 生成 1000MB 文件以供测试
```
先来测试下正常情况下文件读写的情况:
```go
package main

import (
	"log"
	"os"
	"syscall"
	"time"
	"unsafe"
)

const (
	// Size to align the buffer to
	AlignSize = 4096

	// Minimum block size
	BlockSize = 4096
)

func alignment(block []byte, AlignSize int) int {
	return int(uintptr(unsafe.Pointer(&block[0])) & uintptr(AlignSize-1))
}

func AlignedBlock(BlockSize int) []byte {
	block := make([]byte, BlockSize+AlignSize)
	if AlignSize == 0 {
		return block
	}
	a := alignment(block, AlignSize)
	offset := 0
	if a != 0 {
		offset = AlignSize - a
	}
	block = block[offset : offset+BlockSize]
	// Can't check alignment of a zero sized block
	if BlockSize != 0 {
		a = alignment(block, AlignSize)
		if a != 0 {
			log.Fatal("Failed to align block")
		}
	}
	return block
}
func main() {
	fd, err := os.OpenFile("/disk/data/tmp/test", os.O_RDWR|syscall.O_DIRECT, 0666)
	block := AlignedBlock(BlockSize)
	if err != nil {
		panic(err)
	}
	defer fd.Close()
	for i := range block {
		block[i] = 1
	}
	for {
		n, err := fd.Write(block)
		_ = n
		if err != nil {
			panic(err)
		}
		fd.Seek(0, 0)
		time.Sleep(time.Nanosecond * 10)
	}
}
```
```shell
iotop
Total DISK READ :       0.00 B/s
Actual DISK READ:       0.00 B/s
```
可以观察到除了初次 I/O 的时候产生磁盘读写, 待测试代码稳定下后是没有产生磁盘读写的, 再看看文件 page cache 的情况
```shell
vmtouch /disk/data/tmp/test
           Files: 1
     Directories: 0
  Resident Pages: 4/256000  16K/1000M  0.00156%
         Elapsed: 0.007374 seconds
```
与预期中的一样缓存了前 16K 文件页
```shell
# 强刷 page cache
echo 3 > /proc/sys/vm/drop_caches
```
再来测试下绕过 page cache 进行文件读写的方法
```go
package main

import (
	"log"
	"os"
	"syscall"
	"time"
	"unsafe"
)

const (
	// Size to align the buffer to
	AlignSize = 4096

	// Minimum block size
	BlockSize = 4096
)

func alignment(block []byte, AlignSize int) int {
	return int(uintptr(unsafe.Pointer(&block[0])) & uintptr(AlignSize-1))
}

func AlignedBlock(BlockSize int) []byte {
	block := make([]byte, BlockSize+AlignSize)
	if AlignSize == 0 {
		return block
	}
	a := alignment(block, AlignSize)
	offset := 0
	if a != 0 {
		offset = AlignSize - a
	}
	block = block[offset : offset+BlockSize]
	// Can't check alignment of a zero sized block
	if BlockSize != 0 {
		a = alignment(block, AlignSize)
		if a != 0 {
			log.Fatal("Failed to align block")
		}
	}
	return block
}
func main() {
        // syscall.O_DIRECT fsfd 直接 I/O 选项
	fd, err := os.OpenFile("/disk/data/tmp/test", os.O_RDWR|syscall.O_DIRECT, 0666)
	block := AlignedBlock(BlockSize)
	if err != nil {
		panic(err)
	}
	defer fd.Close()
	for i := range block {
		block[i] = 1
	}
	for {
		n, err := fd.Read(block)
		_ = n
		if err != nil {
			panic(err)
		}
		fd.Seek(0, 0)
		time.Sleep(time.Nanosecond * 10)
	}
}
```
> 注: 要使用 O_DIRECT 的方式进行文件 I/O 的话, 文件每次操作的大小得进行文件 lock size 以及 memory address 的对齐

```shell
iotop
Total DISK READ :      17.45 M/s 
Actual DISK READ:      17.42 M/s
```
再来看看 page cache 的情况:
```shell
vmtouch /disk/data/tmp/test
           Files: 1
     Directories: 0
  Resident Pages: 0/256000  0/1000M  0%
         Elapsed: 0.006608 seconds
```

## 回到 golang
当前标准库如 http.ServeFile, os.Open 等默认采用的访问静态资源的方式均为非直接 I/O, 因此如果有特定场景需要用户自己进行这方面的考量及优化
## 参考资料
- http://nginx.org/en/docs/http/ngx_http_core_module.html#directio
- https://tech.meituan.com/2017/05/19/about-desk-io.html
- https://github.com/ncw/directio
# DMA
# Zero-Copy

## splice
在介绍 splice 及其在 golang 中的应用之前, 先从一段简单的网络代理代码开始入手
#### read & write
```go
var (
	p sync.Pool
)

func init() {
	p.New = func() interface{} {
		return make([]byte, DEFAULTSIZE)
	}
}
// src 客户端 tcp 链接
// dst mock server tcp 链接
func normal(src, dst net.Conn) {
	var bts []byte = p.Get().([]byte)
	var bts2 []byte = p.Get().([]byte)
	defer p.Put(bts)
	defer p.Put(bts2)
	// mock server to client
	go func() {
		for {
			num, err := dst.Read(bts2[:])
			if err != nil {
				break
			}
			var num_write int
			for num > 0 {
				num_write, err = src.Write(bts2[num_write:num])
				if err != nil {
					return
				}
				num -= num_write
			}
		}
	}()
	// client to mock server
	for {
		num, err := src.Read(bts[:])
		if err != nil {
			break
		}
		var num_write int
		for num > 0 {
			num_write, err = dst.Write(bts[num_write:num])
			if err != nil {
				return
			}
			num -= num_write
		}
	}
}
```
以上片段实现了一个简单的功能: 将客户端请求的 tcp 数据通过`read`系统调用读出放入本地用户空间 缓存, 而后再调用`write`发送给目标服务器,反之亦然

整个过程如下图所示（暂不考虑 IO 模型调度以及 DMA 等细节部分）

```shell
[ user space ]

              --------------------------------------------
             |             application                    |
              --------------------------------------------
 ····················|·································|··················
                     | read()                          | write()
[ kernel space ]     |                                 |
              -----------------               -----------------
             |  socket buffer  |             |  socket buffer  |
              -----------------               -----------------
                     | copy                            |
 ····················|·································|··················
[ hardware sapce ]   |                                 |
              ------------------------------------------------------
             |                  network interface                  |
              ------------------------------------------------------

```
对于透传或者部分透传(例如七层头部解析后透明代理请求体)之类的需求场景来说, 这种流程的成本无疑是很高的, 可以总结下涉及的几个浪费点

- 数据需要从内核态拷贝到用户态
- 应用层在 read 及 write 的过程中对这部分 byte 的操作开销(申请、释放、对象池维护等)
  
#### splice 介绍
```c
/*
       splice() moves data between two file descriptors without copying
       between kernel address space and user address space.  It
       transfers up to len bytes of data from the file descriptor fd_in
       to the file descriptor fd_out, where one of the file descriptors
       must refer to a pipe.
*/
ssize_t splice(int fd_in, off64_t *off_in, int fd_out,
                      off64_t *off_out, size_t len, unsigned int flags);
```
一句话概括就是, `splice` 不需要从内核空间复制这部分数据到用户空间就可以支持将数据从两个文件描述符之间进行转移, 不过两个描述符至少得有一个是 `pipe`, 以下列举如何利用`splice`完成 `socket->socket` 的数据代理

example:
```go
func example(src, dst net.Conn) {
	// 由于 src 及 dst 都是 socket, 没法直接使用 splice, 因此先创建临时 pipe
	const flags = syscall.O_CLOEXEC | syscall.O_NONBLOCK
	var fds [2]int // readfd, writefd
	if err := syscall.Pipe2(fds[:], flags); err != nil {
		panic(err)
	} 
	// 使用完后关闭 pipe
	defer syscall.Close(fds[0])
	defer syscall.Close(fds[1])
	// 获取 src fd
	srcfile, err := src.(*net.TCPConn).File()
	if err != nil {
		panic(err)
	}
	srcfd := int(srcfile.Fd())
	syscall.SetNonblock(srcfd, true)
	...
	// 从 srcfd 读出, 写入 fds[1] (pipe write fd)
	num, err := syscall.Splice(srcfd, nil, fds[1], nil, DEFAULTSIZE/* size to splice */, SPLICE_F_NONBLOCK)
	...
}
```
此时的调用过程变为:

```shell
[ user space ]

              -----------------------------------------------------------------------------------------------
             |                                           application                                         |
              -----------------------------------------------------------------------------------------------
 ········································|····················|·····················|··································
                                         | splice()           | pipe()              | splice()
[ kernel space ]                         |                    |                     |
              -----------------        ”copy“       -----------------------      ”copy“       -----------------
             |  socket buffer  |· · · · · · · · · >| pipe writefd & readfd |· · · · · · · · >|  socket buffer  |
              -----------------                     -----------------------                   -----------------
                     | copy                                                                         |
 ····················|··············································································|
[ hardware sapce ]   |                                                                              |
              -----------------------------------------------------------------------------------------------
             |                                         network interface                                     |
              -----------------------------------------------------------------------------------------------
```
此时产生的系统调用为
- 首先 `pipe()` 调用, 创建临时管道
- 调用 `splice()` 将 `srcfd` 数据 ”拷贝“ 到 `pipe`
- 调用 `splice()` 将 `pipe` 中的数据 ”拷贝“ 到 `dstfd`

可以注意到图中以及总结部分的”拷贝“给加了引号, 具体了解过`pipe`底层实现的小伙伴应该理解, 在这边简单表述下, `splice` 是基于 `pipe buffer` 实现的, 本质上在数据传输的时候并没有进行数据的拷贝, 而是仅仅将数据的内存地址等信息塞进了`pipe_buffer`中对应的字段

至此, 完成了 kernel-user space 的拷贝优化, 不过可能细心的人会发现, 这种方式虽然减少了数据的拷贝, 但是同样额外增加了系统调用(create pipe & close pipe), 接下来就关于这部分的取舍与具体场景进行具体讨论

#### splice 还是 read & write?
如何取舍使用哪种方式?

两种方法各有各的好处, 往往选择层面的考虑在于应用层的具体策略, 如是否进行透传(/部分), 饥饿问题, 对象池策略等等

下面提供几种场景下的测试以供参考
benchmark 代码:
```go
/*
 * 测试环境: go 1.14.3 centos7
 */
package main

import (
	"bytes"
	"io"
	"net"
	"net/http"
	"sync"
	"sync/atomic"
	"testing"
	"time"
)

var (
	p sync.Pool
)

func init() {
	p.New = func() interface{} {
		return make([]byte, DEFAULTSIZE)
	}
}

const (
	// mock http 请求体大小
	REQUESTBYTESIZE = 0
	// 应用层对象池 byte 大小
	DEFAULTSIZE     = 1 << 10

	SPLICE_F_MOVE     = 0x1
	SPLICE_F_NONBLOCK = 0x2
	SPLICE_F_MORE     = 0x4
	SPLICE_F_GIFT     = 0x8
)
// io.Copy 该场景下内部调用 splice syscall, 感兴趣的自行查看源码
func gosplice(src, dst net.Conn) {
	go func() {
		io.Copy(src, dst)
	}()
	io.Copy(dst, src)
}

func normal(src, dst net.Conn) {
	var bts []byte = p.Get().([]byte)
	var bts2 []byte = p.Get().([]byte)
	defer p.Put(bts)
	defer p.Put(bts2)
	go func() {
		for {
			num, err := dst.Read(bts2[:])
			if err != nil {
				break
			}
			var num_write int
			for num > 0 {
				num_write, err = src.Write(bts2[num_write:num])
				num -= num_write
				if err != nil {
					return
				}
			}
		}
	}()
	// local to mock serve
	for {
		num, err := src.Read(bts[:])
		if err != nil {
			break
		}
		var num_write int
		for num > 0 {
			num_write, err = dst.Write(bts[num_write:num])
			num -= num_write
			if err != nil {
				return
			}
		}
	}
}

// Server http server
var Server *http.Server

type s struct{}

func (ss s) ServeHTTP(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(200)
	return
}
func TestMain(m *testing.M) {
	// mock tcp server
	var ss s
	go func() {
		Server = &http.Server{
			Addr:         "0.0.0.0:9610",
			Handler:      ss,
			WriteTimeout: 60 * time.Second,
			ReadTimeout:  60 * time.Second,
		}
		err := Server.ListenAndServe()
		if err != nil {
			panic(err)
		}
	}()
	go func() { // normal 9611
		l, err := net.ListenTCP("tcp4", &net.TCPAddr{
			IP:   net.ParseIP("0.0.0.0"),
			Port: 9611,
		})
		if err != nil {
			panic(err)
		}
		for {
			n, err := l.Accept()
			if err != nil {
				continue
			}
			defer n.Close()
			remote, err := net.DialTCP("tcp4", &net.TCPAddr{
				IP: net.ParseIP("0.0.0.0"), Port: 0,
			}, &net.TCPAddr{
				IP: net.ParseIP("0.0.0.0"), Port: 9610,
			})
			if err != nil {
				continue
			}
			go normal(n, remote)
		}
	}()
	go func() { // gosplice 9612
		l, err := net.ListenTCP("tcp4", &net.TCPAddr{
			IP:   net.ParseIP("0.0.0.0"),
			Port: 9612,
		})
		if err != nil {
			panic(err)
		}
		for {
			n, err := l.Accept()
			if err != nil {
				continue
			}
			defer n.Close()
			remote, err := net.DialTCP("tcp4", &net.TCPAddr{
				IP: net.ParseIP("0.0.0.0"), Port: 0,
			}, &net.TCPAddr{
				IP: net.ParseIP("0.0.0.0"), Port: 9610,
			})
			if err != nil {
				continue
			}
			go gosplice(n, remote)
		}
	}()
	m.Run()
}
func BenchmarkNormalReadWrite(b *testing.B) {
	// normal 9611
	c := http.Client{
		Timeout: time.Minute,
	}
	var total, success uint32
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		atomic.AddUint32(&total, 1)
		req, err := http.NewRequest("POST", "http://0.0.0.0:9611", bytes.NewReader(make([]byte, REQUESTBYTESIZE)))
		if err != nil {
			b.Fatalf("%s", err.Error())
		}
		res, err := c.Do(req)
		if err == nil && res.StatusCode == 200 {
			atomic.AddUint32(&success, 1)
		}
	}
	b.Logf("test:%s,total: %d,rate: %.2f%%\n", b.Name(), total, float64(success*100/total))
}

func BenchmarkGoSplice(b *testing.B) {
	// normal 9612
	c := http.Client{
		Timeout: time.Minute,
	}
	var total, success uint32
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		atomic.AddUint32(&total, 1)
		req, err := http.NewRequest("POST", "http://0.0.0.0:9612", bytes.NewReader(make([]byte, REQUESTBYTESIZE)))
		if err != nil {
			b.Fatalf("%s", err.Error())
		}
		res, err := c.Do(req)
		if err == nil && res.StatusCode == 200 {
			atomic.AddUint32(&success, 1)
		}
	}
	b.Logf("test:%s, total: %d, success rate: %.2f%%\n", b.Name(), total, float64(success*100/total))
}
```
- 场景一: 单次请求数据量较少, 应用维护单个 buffer 较小
```go
REQUESTBYTESIZE = 0 // http request body
DEFAULTSIZE = 1 << 10 // buffer size 1kb
```
```shell
RRunning tool: /usr/local/bin/go test -benchmem -run=^$ -bench ^(BenchmarkNormalReadWrite|BenchmarkGoSplice)$ barrier/t

goos: linux
goarch: amd64
pkg: barrier/t
BenchmarkNormalReadWrite-4   	    6348	    179699 ns/op	    4847 B/op	      62 allocs/op
--- BENCH: BenchmarkNormalReadWrite-4
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 1,rate: 100.00%
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 100,rate: 100.00%
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 6348,rate: 100.00%
BenchmarkGoSplice-4          	    6652	    179622 ns/op	    4852 B/op	      62 allocs/op
--- BENCH: BenchmarkGoSplice-4
    test_test.go:194: test:BenchmarkGoSplice, total: 1, success rate: 100.00%
    test_test.go:194: test:BenchmarkGoSplice, total: 100, success rate: 100.00%
    test_test.go:194: test:BenchmarkGoSplice, total: 6652, success rate: 100.00%
PASS
ok  	barrier/t	2.391s
```
两种方式无明显性能差异
- 场景二: 单次请求数据量多, 应用维护单个 buffer 较小
```go
REQUESTBYTESIZE = 1 << 20 // 1 MB
DEFAULTSIZE = 1 << 10 // buffer size 1kb
```
```shell
Running tool: /usr/local/bin/go test -benchmem -run=^$ -bench ^(BenchmarkNormalReadWrite|BenchmarkGoSplice)$ barrier/t

goos: linux
goarch: amd64
pkg: barrier/t
BenchmarkNormalReadWrite-4   	     465	   2329209 ns/op	 1073956 B/op	     163 allocs/op
--- BENCH: BenchmarkNormalReadWrite-4
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 1,rate: 100.00%
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 100,rate: 100.00%
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 376,rate: 100.00%
    test_test.go:173: test:BenchmarkNormalReadWrite,total: 465,rate: 100.00%
BenchmarkGoSplice-4          	     963	   1555386 ns/op	 1070506 B/op	     157 allocs/op
--- BENCH: BenchmarkGoSplice-4
    test_test.go:194: test:BenchmarkGoSplice, total: 1, success rate: 100.00%
    test_test.go:194: test:BenchmarkGoSplice, total: 100, success rate: 100.00%
    test_test.go:194: test:BenchmarkGoSplice, total: 963, success rate: 100.00%
PASS
ok  	barrier/t	4.056s
```
当链接需要处理的数据量较多而应用层每次处理的 buffer 相比起来较小, 以至于需要 read & write 的次数更多的时候, 差异就会比较明显

#### go 中的 splice
在上面的介绍过程中简单说了下 `io.Copy` 在 `socket` 之间操作的时候, 当机器架构支持的时候会采取 `splice`, 接下来就此进行详细分析来介绍下 `runtime` 在 `splice` 上的一些决策以及当前`runtime`在 `splice` 上的一些不足
```go
/*
 * src/net/spice_linux.go
 */
// splice transfers data from r to c using the splice system call to minimize
// copies from and to userspace. c must be a TCP connection. Currently, splice
// is only enabled if r is a TCP or a stream-oriented Unix connection.
//
// If splice returns handled == false, it has performed no work.
func splice(c *netFD, r io.Reader) (written int64, err error, handled bool) {
	/*
	* 因为前面介绍过 splice 是通过 pipe buffer 实现的
	* 在调用的时候 kernel无需进行数据拷贝, 仅操作数据原信息(基础字段的指针等)
	* 所以这边默认 splice 的 len 开得比较大, 读到 EOF 为止
	*/
	var remain int64 = 1 << 62 // by default, copy until EOF
	lr, ok := r.(*io.LimitedReader)
	if ok {
		remain, r = lr.N, lr.R
		if remain <= 0 {
			return 0, nil, true
		}
	}

	var s *netFD
	if tc, ok := r.(*TCPConn); ok {
		s = tc.fd
	} else if uc, ok := r.(*UnixConn); ok {
		if uc.fd.net != "unix" {
			return 0, nil, false
		}
		s = uc.fd
	} else {
		return 0, nil, false
	}

	written, handled, sc, err := poll.Splice(&c.pfd, &s.pfd, remain)
	if lr != nil {
		lr.N -= written
	}
	return written, wrapSyscallError(sc, err), handled
}
```
```go
/*
 * go 1.14.3
 * src/internal/poll/splice_linux.go
 */
// Splice transfers at most remain bytes of data from src to dst, using the
// splice system call to minimize copies of data from and to userspace.
//
// Splice creates a temporary pipe, to serve as a buffer for the data transfer.
// src and dst must both be stream-oriented sockets.
//
// If err != nil, sc is the system call which caused the error.
func Splice(dst, src *FD, remain int64) (written int64, handled bool, sc string, err error) {
	// dst 以及 src 均为 socket.fd, 因此若想使用 splice 则需要借助 pipe
	// 创建临时 pipe
	prfd, pwfd, sc, err := newTempPipe()
	if err != nil {
		return 0, false, sc, err
	}
	defer destroyTempPipe(prfd, pwfd)
	var inPipe, n int
	for err == nil && remain > 0 {
		max := maxSpliceSize
		if int64(max) > remain {
			max = int(remain)
		}
		// spliceDrain 调用 splice syscall
		inPipe, err = spliceDrain(pwfd, src, max)
		// The operation is considered handled if splice returns no
		// error, or an error other than EINVAL. An EINVAL means the
		// kernel does not support splice for the socket type of src.
		// The failed syscall does not consume any data so it is safe
		// to fall back to a generic copy.
		//
		// spliceDrain should never return EAGAIN, so if err != nil,
		// Splice cannot continue.
		//
		// If inPipe == 0 && err == nil, src is at EOF, and the
		// transfer is complete.
		handled = handled || (err != syscall.EINVAL)
		if err != nil || (inPipe == 0 && err == nil) {
			break
		}
		// splicePump 调用 splice syscall
		n, err = splicePump(dst, prfd, inPipe)
		if n > 0 {
			written += int64(n)
			remain -= int64(n)
		}
	}
	if err != nil {
		return written, handled, "splice", err
	}
	return written, true, "", nil
}
```

相信上面简短的分析大家也可以看到, 每次在进行 `splice` 的时候都会利用临时 `pipe`, 频繁的创建、销毁, 用户态-内核态的切换会带来非常多不必要的开销, 当前社区内也有关于 `splice temp-pipe` 生命周期的[讨论](https://go-review.googlesource.com/c/go/+/271537/)。

再者, 因为当前关联到 `socket` 的 `splice` 实现在 `runtime` 层面和内置 `io 模型(epoll 等)`高度耦合, 基本无法解耦单独应用, 而如果想自己来实现 `splice（syscall.Splice)` 的话则不得不顺带在用户层面实现自己的`io 模型`再来使用, 会比较繁琐(上面测试用例使用内置 `splice api` 也是因为这个原因)

## 参考资料

- https://go-review.googlesource.com/c/go/+/271537/
- https://zhuanlan.zhihu.com/p/308054212
