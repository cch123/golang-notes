# 定时器

## 简单用法

什么 Date，Parse，ParseInLocation，就不说了。主要关注下面几个函数：

ticker 相关:

```go
func Tick(d Duration) <-chan Time
func NewTicker(d Duration) *Ticker
func (t *Ticker) Stop()
```

timer 相关:

```go
func After(d Duration) <-chan Time
func NewTimer(d Duration) *Timer
func (t *Timer) Reset(d Duration) bool
func (t *Timer) Stop() bool
```
