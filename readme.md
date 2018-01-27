## node-prune

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)

> 清除 不必要 文件 在 node_modules (.md, .ts, ...) 中


Explanation

> "version": "1.0.1"

[github source](https://github.com/tj/node-prune)

~~[english](./README.en.md)~~

---

## 本目录

- [import-库的导入](#import-库的导入)

- [go-运行-or-说明](#go-运行-or-说明)

- [init-初始化](#init-初始化)

- [main-命令分析](#main-命令分析)

- [prune-新建](#prune-新建)

- [output-详细信息](#output-详细信息)


---

## 命令行 node-prune

go 语言中，一个库能被命令方式使用，必然需要 

[node-prune/cmd/node-prune/main.go](./node-prune/cmd/node-prune/main.go)

``` go
package main // 作为编译-命令-程序的入口
```

我们就从 `main.go` 开始

---

## import-库的导入

代码 1-12

``` go
package main

import (
	"flag" // 原生-命令解析 :> $ node-prune --help
	"fmt"  // 原生-终端屏幕输出 :> 就像 js{console} py{print()}
	"time" // 原生-时间 :>

	"github.com/apex/log" // 作者所写，日志记录
	"github.com/apex/log/handlers/cli"
	"github.com/dustin/go-humanize"

	"github.com/tj/node-prune" // 引入-node-prune-主库
)
```

### go-运行-or-说明

- go 语言在，库与库之间的应用上

- 决心，铲除异己，一个库只能存在一个版本

- 而 所有库都存放在，`GOPATH` 的 环境变量路径中，

- 这就决定了，库的`唯一版本`且存放的`体积`, 而不是 `node_modules` 的无尽深渊

-

如果 .zshrc | .bashrc

``` zsh
export GOPATH='path/to/gopath/ 
```

-

那么 	`"github.com/apex/log"` 就在

``` go
GOPATH/src/github.com/apex/log //中
// 类推
//"github.com/apex/log/handlers/cli"
GOPATH/src/github.com/apex/log/handlers/cli
// ...
```

> 当然，GOPATH 的可以是多个路径，一个路径一个集体

---

❕下文，有些例子，那么你需要 `clone` 这 `explain` 到你的 `GOPATH`

```
go get -v github.com/chinanf-boy/explain-node-prune
```
---

## init-初始化

代码 15-17

``` go
func init() {
	log.SetHandler(cli.Default)
	log.SetLevel(log.WarnLevel)
}
```

1. log

> 很明显, `log` 从哪里来, 那么我们需要看下去👀

``` go
import (
    // ...
    "github.com/apex/log"
    // ...
)
```

- ❓ 这样的导入, 意味着什么

- ℹ️ 意味着- `import` -> 导入 `package log` -> 🉐️`log`

- ㊙️ 这个 `log` 代表了，整个库的内容，不过只有 `开头大写` 才是公有的

``` go
log.SetHandler(cli.Default) // S 大写
```

2. log.SetHandler(cli.Default)

> 日志设置出口

cli.go

``` go
var Default = New(os.Stderr)  //写入标准错误以防止混乱和崩溃
```

3. log.SetLevel(log.WarnLevel)

> 日志设置等级

- log.WarnLevel

``` go
const (
    InvalidLevel Level = iota - 1 // iota==0
    // 用一次 + 1
	DebugLevel // 0
	InfoLevel // 1
	WarnLevel // 2
	ErrorLevel // 3
	FatalLevel // 4
)
```

> [`log`库点到为止-更多内容](#apex-log)

---

## main-命令分析

> 使用 `flag` 对 用户的输出解析

代码 20-35

``` go
func main() {
    debug := flag.Bool("verbose", false, "Verbose log output.")
    // 在所有flag都注册之后
	flag.Parse() // 来解析命令行参数写入注册的flag里。
	dir := flag.Arg(0) // 第一个命令选项

	start := time.Now() // 现在时间
    // : 2009-11-10 23:00:00 +0000 UTC m=+0.000000001

	if *debug {
		log.SetLevel(log.DebugLevel)
	}

	var options []prune.Option
    // 一个 特定-容器

	if dir != "" {
		options = append(options, prune.WithDir(dir))
    }
    // prune.WithDir(dir) 返回 一个 特定-结构 被 特定容器装下

    // append : options-数组->推入-> prune.WithDir(dir)-数值
```

1. flag.Bool(a,b,c)
    
- a == `verbose`

``` zsh
$node-prune verbose # 触发
```

- b == `false`

> 默认值

- c == "Verbose log output."

>$node-prune --help 

```
Usage of node-prune:
  -verbose
        Verbose log output.
```

---

2. prune.WithDir(dir)

``` go
func WithDir(s string) Option {
	return func(v *Pruner) {
		v.dir = s
	}
}
```

> go 语言的类型与接口是固定程序的强大工具

> 但是，一般不熟悉，相关类型与接口的，却可以说疲于奔命

``` go
type Pruner struct {
	dir   string
	log   log.Interface
	dirs  map[string]struct{}
	exts  map[string]struct{}
	files map[string]struct{}
	ch    chan func()
	wg    sync.WaitGroup
}

// Option function.
type Option func(*Pruner)
```

在这一段，说明白对谁都好👀

- `func WithDir(s string) Option `

    - `s string` | s-函数变量名, 变量类型-string

    - `Option` | 返回 Option 类型

- `type Option func(*Pruner) `

    - `type` | 定义类型-关键字

    - `Option` | 类型名字

    - `func(*Pruner)` | 对应类型 - 函数 - 其中变量需要是`*Pruner`

- `type Pruner struct `

    > 如果说上面的类型-只不过是**一对一**指代关系

    - `struct` | 那么 struct 就是**一对多**的关系

    - 这里就需要称 `Pruner` 为 `结构` 啦

``` go
type Pruner struct { // Pruner 结构
	dir   string // dir-类型名， string-类型
	log   log.Interface
	dirs  map[string]struct{}
	exts  map[string]struct{}
	files map[string]struct{}
	ch    chan func()
	wg    sync.WaitGroup
}
```

#### prune.WithDir(dir) 小结

``` go
func WithDir(s string) Option {
	return func(v *Pruner) {
		v.dir = s
	}
}
```

- `prune.WithDir(dir)`返回一个 函数类型 `Option`

- 又因为 `type Option func(*Pruner)`

- 所以其实 `Option` == `func(*Pruner)`

- `func(*Pruner)`函数的变量是 `Pruner` 结构

- `Pruner` 结构中成分- `*Pruner.dir` = `dir`

>  `dir` 作为第一层`WithDir`的变量`s`, 附值给第二层**func**`的`变量`v *Prune`中的结构成分-`v.dir`

---

🦊 真累啊, 想不到怎么难说 😢 , 继续前进吧

---

## prune-新建

> 新建 `prune` 实例并运行

代码 37-42

``` go
	p := prune.New(options...)

	stats, err := p.Prune()
	if err != nil {
		log.Fatalf("error: %s", err)
	}

```

>[ 因为 `prune`库 太长，请移步-prune.readme.md](./prune.readme.md)

---

## output-详细信息

> 展示运行后数据

代码 44-56

``` go
	println() 
	defer println() // 1

	output("files total", humanize.Comma(stats.FilesTotal))
	output("files removed", humanize.Comma(stats.FilesRemoved))
	output("size removed", humanize.Bytes(uint64(stats.SizeRemoved)))
	output("duration", tim·e.Since(start).Round(time.Millisecond).String())
}

func output(name, val string) {
	fmt.Printf("\x1b[1m%20s\x1b[0m %s\n", name, val)
}

```

- `defer` : 总是在函数退出前运行

> 可用来，数据库的断开，文件关闭，等等

- `humanize`

> 转类型

---

## 其他

### apex-log

[apex/log tj大神的组织公司-apex](https://github.com/apex/log)

### flag

[flag-中文 go原生命令解析](https://studygolang.com/pkgdoc)