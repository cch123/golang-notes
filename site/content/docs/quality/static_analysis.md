---
title: 静态分析
weight: 1
---

# 静态分析

静态分析是通过扫描并解析用户代码，寻找代码中的潜在 bug 的一种手段。

静态分析一般会集成在项目上线的 CI 流程中，如果分析过程找到了 bug，会直接阻断上线，避免有问题的代码被部署到线上系统。从而在部署早期发现并修正潜在的问题。

## 社区常见 linter

时至今日，社区已经有了丰富的 linter 资源供我们使用，本文会挑出一些常见 linter 进行说明。

### go lint

go lint 是官方出的 linter，是 Go 语言最早期的 linter 了，其可以检查：

* 导出函数是否有注释
* 变量、函数、包命名不符合 Go 规范，有下划线
* receiver 命名是否不符合规范

但这几年社区的 linter 蓬勃发展，所以这个项目也被官方 deprecated 掉了。其主要功能被另外一个 linter：revive[^1] 完全继承了。

### go vet

go vet 也是官方提供的静态分析工具，其内置了锁拷贝检查、循环变量捕获问题、printf 参数不匹配等工具。

比如新手老手都很容易犯的 loop capture 错误：

```go
package main

func main() {
	var a = map[int]int {1 : 1, 2: 3}
	var b = map[int]*int{}
	for k, r := range a {
		go func() {
			b[k] = &r
		}()
	}
}
```

go vet 会直接把你骂醒：

```shell
~/test git:master ❯❯❯ go vet ./clo.go
# command-line-arguments
./clo.go:8:6: loop variable k captured by func literal
./clo.go:8:12: loop variable r captured by func literal
```

执行 go tool vet help 可以看到 go vet 已经内置的一些 linter。

```shell
~ ❯❯❯ go tool vet help
vet is a tool for static analysis of Go programs.

vet examines Go source code and reports suspicious constructs,
such as Printf calls whose arguments do not align with the format
string. It uses heuristics that do not guarantee all reports are
genuine problems, but it can find errors not caught by the compilers.

Registered analyzers:

    asmdecl      report mismatches between assembly files and Go declarations
    assign       check for useless assignments
    atomic       check for common mistakes using the sync/atomic package
    bools        check for common mistakes involving boolean operators
    buildtag     check that +build tags are well-formed and correctly located
    cgocall      detect some violations of the cgo pointer passing rules
    composites   check for unkeyed composite literals
    copylocks    check for locks erroneously passed by value
    errorsas     report passing non-pointer or non-error values to errors.As
    httpresponse check for mistakes using HTTP responses
    loopclosure  check references to loop variables from within nested functions
    lostcancel   check cancel func returned by context.WithCancel is called
    nilfunc      check for useless comparisons between functions and nil
    printf       check consistency of Printf format strings and arguments
    shift        check for shifts that equal or exceed the width of the integer
    stdmethods   check signature of methods of well-known interfaces
    structtag    check that struct field tags conform to reflect.StructTag.Get
    tests        check for common mistaken usages of tests and examples
    unmarshal    report passing non-pointer or non-interface values to unmarshal
    unreachable  check for unreachable code
    unsafeptr    check for invalid conversions of uintptr to unsafe.Pointer
    unusedresult check for unused results of calls to some functions
```

默认情况下这些 linter 都是会跑的，当前很多 IDE 在代码修改时会自动执行 go vet，所以我们在写代码的时候一般就能发现这些错了。

但 `go vet` 还是应该集成到线上流程中，因为有些程序员的下限实在太低。

### errcheck

Go 语言中的大多数函数返回字段中都是有 error 的：

```go
func sayhello(wr http.ResponseWriter, r *http.Request) {
	io.WriteString(wr, "hello")
}

func main() {
	http.HandleFunc("/", sayhello)
	http.ListenAndServe(":1314", nil) // 这里返回的 err 没有处理
}
```

这个例子中，我们没有处理 `http.ListenAndServe` 函数返回的 error 信息，这会导致我们的程序在启动时发生静默失败。

程序员往往会基于过往经验，对当前的场景产生过度自信，从而忽略掉一些常见函数的返回错误，这样的编程习惯经常为我们带来意外的线上事故。例如，规矩的写法是下面这样的：

```go
data, err := getDataFromRPC()
if err != nil {
	return nil, err
}

// do business logic
age := data.age
```

而自信的程序员可能会写成这样：

```go
data, _ := getDataFromRPC()

// do business logic
age := data.age
```

如果底层 RPC 逻辑出错，上层的 data 是个空指针也是很正常的，如果底层函数返回的 err 非空时，我们不应该对其它字段做任何的假设。这里 data 完全有可能是个空指针，造成用户程序 panic。

errcheck 会强制我们在代码中检查并处理 err。

### gocyclo

gocyclo 主要用来检查函数的圈复杂度。圈复杂度可以参考下面的定义：

> 圈复杂度(Cyclomatic complexity)是一种代码复杂度的衡量标准，在 1976 年由 Thomas J. McCabe, Sr. 提出。在软件测试的概念里，圈复杂度用来衡量一个模块判定结构的复杂程度，数量上表现为线性无关的路径条数，即合理的预防错误所需测试的最少路径条数。圈复杂度大说明程序代码可能质量低且难于测试和维护，根据经验，程序的可能错误和高的圈复杂度有着很大关系。

看定义较为复杂但计算还是比较简单的，我们可以认为：

* 一个 if，那么函数的圈复杂度要 + 1
* 一个 switch 的 case，函数的圈复杂度要 + 1
* 一个 for 循环，圈复杂度 + 1
* 一个 && 或 ||，圈复杂度 + 1

在大多数语言中，若函数的圈复杂度超过了 10，那么我们就认为该函数较为复杂，需要做拆解或重构。部分场景可以使用表驱动的方式进行重构。

由于在 Go 语言中，我们使用 `if err != nil` 来处理错误，所以在一个函数中出现多个 `if err != nil` 是比较正常的，因此 Go 中函数复杂度的阈值可以稍微调高一些，15 是较为合适的值。

下面是在个人项目 elasticsql 中执行 gocyclo 的结果，输出 top 10 复杂的函数：

```shell
~/g/s/g/c/elasticsql git:master ❯❯❯ gocyclo -top 10  ./
23 elasticsql handleSelectWhere select_handler.go:289:1
16 elasticsql handleSelectWhereComparisonExpr select_handler.go:220:1
16 elasticsql handleSelect select_handler.go:11:1
9 elasticsql handleGroupByFuncExprDateHisto select_agg_handler.go:82:1
9 elasticsql handleGroupByFuncExprDateRange select_agg_handler.go:154:1
8 elasticsql buildComparisonExprRightStr select_handler.go:188:1
7 elasticsql TestSupported select_test.go:80:1
7 elasticsql Convert main.go:28:1
7 elasticsql handleGroupByFuncExpr select_agg_handler.go:215:1
6 elasticsql handleSelectWhereOrExpr select_handler.go:157:1
```

### bodyclose

使用 bodyclose[^2] 可以帮我们检查在使用 HTTP 标准库时忘记关闭 http body 导致连接一直被占用的问题。

```go
resp, err := http.Get("http://example.com/") // Wrong case
if err != nil {
	// handle error
}
body, err := ioutil.ReadAll(resp.Body)
```

像上面这样的例子是不对的，使用标准库很容易犯这样的错。bodyclose 可以直接检查出这个问题：

```shell
# command-line-arguments
./httpclient.go:10:23: response body must be closed
```

所以必须要把 Body 关闭：

```go
resp, err := http.Get("http://example.com/")
if err != nil {
	// handle error
}
defer resp.Body.Close() // OK
body, err := ioutil.ReadAll(resp.Body)
```

HTTP 标准库的 API 设计的不太好，这个问题更好的避免方法是公司内部将 HTTP client 封装为 SDK，防止用户写出这样不 Close HTTP body 的代码。

### sqlrows

与 HTTP 库设计类似，我们在面向数据库编程时，也会碰到 sql.Rows 忘记关闭的问题，导致连接大量被占用。sqlrows[^3] 这个 linter 能帮我们避免这个问题，先来看看错误的写法：

```go
rows, err := db.QueryContext(ctx, "SELECT * FROM users")
if err != nil {
    return nil, err
}

for rows.Next() {
	err = rows.Scan(...)
	if err != nil {
		return nil, err // NG: this return will not release a connection.
	}
}
```

正确的写法需要在使用完后关闭 sql.Rows：

```go
rows, err := db.QueryContext(ctx, "SELECT * FROM users")
if err != nil {
    return nil, err
}
defer rows.Close()
```

与 HTTP 同理，公司内也应该将 DB 查询封装为合理的 SDK，不要让业务使用标准库中的 API，避免上述错误发生。

### funlen

funlen[^4] 和 gocyclo 类似，但是这两个 linter 对代码复杂度的视角不太相同，gocyclo 更多关注函数中的逻辑分支，而 funlen 则重点关注函数的长度。默认函数超过 60 行和 40 条语句时，该 linter 即会报警。

## linter 集成工具

一个一个去社区里找 linter 来拼搭效率太低，当前社区里已经有了较好的集成工具，早期是 gometalinter，后来性能更好，功能更全的 golangci-lint 逐渐取而代之。目前 golangci-lint 是 Go 社区的绝对主流 linter。

### golangci-lint

golangci-lint[^5] 能够通过配置来 enable 很多 linter，基本主流的都包含在内了。

在本节开头讲到的所有 linter 都可以在 golangci-lint 中进行配置，

使用也较为简单，只要在项目目录执行 golangci-lint run . 即可。

```shell
~/g/s/g/c/elasticsql git:master ❯❯❯ golangci-lint run .
main.go:36:9: S1034: assigning the result of this type assertion to a variable (switch stmt := stmt.(type)) could eliminate type assertions in switch cases (gosimple)
	switch stmt.(type) {
	       ^
main.go:38:34: S1034(related information): could eliminate this type assertion (gosimple)
		dsl, table, err = handleSelect(stmt.(*sqlparser.Select))
		                               ^
main.go:40:23: S1034(related information): could eliminate this type assertion (gosimple)
		return handleUpdate(stmt.(*sqlparser.Update))
		                    ^
main.go:42:23: S1034(related information): could eliminate this type assertion (gosimple)
		return handleInsert(stmt.(*sqlparser.Insert))
		                    ^
select_handler.go:192:9: S1034: assigning the result of this type assertion to a variable (switch expr := expr.(type)) could eliminate type assertions in switch cases (gosimple)
	switch expr.(type) {
```

## 参考资料

[^1]:https://revive.run/

[^2]:https://github.com/timakin/bodyclose

[^3]:https://github.com/gostaticanalysis/sqlrows

[^4]:https://github.com/ultraware/funlen

[^5]:https://github.com/golangci/golangci-lint
