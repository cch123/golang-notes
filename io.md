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
tmp vmtouch /disk/data/tmp/test
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
http://nginx.org/en/docs/http/ngx_http_core_module.html#directio
https://tech.meituan.com/2017/05/19/about-desk-io.html
https://github.com/ncw/directio
# DMA
# Zero-Copy