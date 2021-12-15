---
title: Go 语言的网络抽象
weight: 2
---

# netpoller

从网络编程基础篇我们知道，网络编程中涉及到的主要系统调用是下面这些:

* socket
* bind
* listen
* accept
* epoll_create
* epoll_wait
* epoll_ctl
* read
* write

因此在学习 Go 对网络层的抽象时，我们也是重点关注 Go 的 netpoller 中，这些 syscall 被封装进了哪个具体的流程里。
