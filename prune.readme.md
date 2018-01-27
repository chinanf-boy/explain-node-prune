## node-prune

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)

> 清除 不必要 文件 在 node_modules (.md, .ts, ...) 中


Explanation

> "version": "1.0.1"

[github source](https://github.com/tj/node-prune)

~~[english](./README.en.md)~~

---

让我们翻开-解释📖-之旅 😊

不过让我们回忆一下-node-prune/cmd/node-prune/main.go 使用[node-prune/prune.go](./node-prune/prune.go) 的内容

``` go
// ..
var options []prune.Option
// ..
	p := prune.New(options...)

	stats, err := p.Prune()
	if err != nil {
		log.Fatalf("error: %s", err)
    }
// ..
```

---

本目录

- 

- [`prune.New(options...)`](#prune-new)
---

## 模块与导入-定义

代码 1-12

``` go
// package prune provides node_modules pruning of unnecessary files.
package prune // 定义模块

import (
	"os"
	"path/filepath"
	"runtime"
	"sync"
	"sync/atomic"

	"github.com/apex/log"
)
```

## 文件去除-定义

代码 14-65

``` go
// DefaultFiles pruned.
//
// Copied from yarn (mostly).
var DefaultFiles = []string{
	"Makefile",
	"Gulpfile.js",
	"Gruntfile.js",
	"gulpfile.js",
	".DS_Store",
	".tern-project",
	".gitattributes",
	".editorconfig",
	".eslintrc",
	"eslint",
	".eslintrc.js",
	".eslintrc.json",
	".eslintignore",
	".stylelintrc",
	"stylelint.config.js",
	".stylelintrc.json",
	".stylelintrc.yaml",
	".stylelintrc.yml",
	".stylelintrc.js",
	".htmllintrc",
	"htmllint.js",
	".lint",
	".npmignore",
	".jshintrc",
	".flowconfig",
	".documentup.json",
	".yarn-metadata.json",
	".travis.yml",
	"appveyor.yml",
	".gitlab-ci.yml",
	"circle.yml",
	".coveralls.yml",
	"CHANGES",
	"LICENSE.txt",
	"LICENSE",
	"license",
	"AUTHORS",
	"CONTRIBUTORS",
	".yarn-integrity",
	".yarnclean",
	"_config.yml",
	".babelrc",
	".yo-rc.json",
	"jest.config.js",
	"karma.conf.js",
	".appveyor.yml",
	"tsconfig.json",
}
```

## 文件夹去除-定义

代码 67-88

``` go
// DefaultDirectories pruned.
//
// Copied from yarn (mostly).
var DefaultDirectories = []string{
	"__tests__",
	"test",
	"tests",
	"powered-test",
	"docs",
	"doc",
	".idea",
	".vscode",
	"website",
	"images",
	"assets",
	"example",
	"examples",
	"coverage",
	".nyc_output",
	".circleci",
	".github",
}
```

## 扩展去掉-定义

代码 91-99

``` go
// DefaultExtensions pruned.
var DefaultExtensions = []string{
	".markdown",
	".md",
	".ts",
	".jst",
	".coffee",
	".tgz",
	".swp",
}
```

## prune-type

代码 101-120

``` go
// Stats for a prune.
type Stats struct { // 状态-结构定义
	FilesTotal   int64
	FilesRemoved int64
	SizeRemoved  int64
}

// Pruner is a module pruner.
type Pruner struct { // 去除文件-主
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


## prune-New

prune包的新建函数

> `main.go`例子使用

``` go
	var options []prune.Option

	if dir != "" {
		options = append(options, prune.WithDir(dir))
	}

	p := prune.New(options...)
```

代码 122-138

prune.go

``` go
// New with the given options.
func New(options ...Option) *Pruner {
	v := &Pruner{
		dir:   "node_modules",
		log:   log.Log, // 记录
		exts:  toMap(DefaultExtensions), // 扩展
		dirs:  toMap(DefaultDirectories), // 文件夹
		files: toMap(DefaultFiles), // 文件
		ch:    make(chan func()), // 通道
	}

	for _, o := range options {
		// o 是 func(*Pruner) 类型
		o(v)
	}

	return v
}
```

## prune-With

设置-配置

代码 140-166

``` go
// WithDir option.
// 设置-路径
func WithDir(s string) Option {
	return func(v *Pruner) {
		v.dir = s
	}
}

// WithExtensions option.
// 设置-扩展
func WithExtensions(s []string) Option {
	return func(v *Pruner) {
		v.exts = toMap(s)
	}
}

// WithDirectories option.
// 设置-文件夹
func WithDirectories(s []string) Option {
	return func(v *Pruner) {
		v.dirs = toMap(s)
	}
}

// WithFiles option.
// 设置-文件
func WithFiles(s []string) Option {
	return func(v *Pruner) {
		v.files = toMap(s)
	}
}
```

- [toMap](#tomap)

> [Hello] --> map[Hello:{}]

---

## 运行-Prune

运行去除

``` go
// Prune performs the pruning.
func (p *Pruner) Prune() (*Stats, error) {
	var stats Stats

	p.startN(runtime.NumCPU()) // -- 1 开始多协程-并-等待 通道传输
	defer p.stop() // -- 2 结束运行

	// -- 3 文件目录
	err := filepath.Walk(p.dir, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}

		stats.FilesTotal++ // 准备文件加1

		ctx := p.log.WithFields(log.Fields{ // -- 记录
			"path": path,
			"size": info.Size(),
			"dir":  info.IsDir(),
		})

		// keep
		if !p.prune(path, info) { // -- 5 // 是否需要
			ctx.Debug("keep")
			return nil
		}

		// prune
		ctx.Info("prune")
		atomic.AddInt64(&stats.FilesRemoved, 1) // -- 6 // 文件移除数量
		atomic.AddInt64(&stats.SizeRemoved, info.Size()) // 文件移除大小

		// remove and skip dir
		if info.IsDir() { // 是否是文件夹
			p.ch <- func() { // -- 7 传输
				s, _ := dirStats(path) // -- 8

				atomic.AddInt64(&stats.FilesTotal, s.FilesTotal)
				atomic.AddInt64(&stats.FilesRemoved, s.FilesRemoved)
				atomic.AddInt64(&stats.SizeRemoved, s.SizeRemoved)

				if err := os.RemoveAll(path); err != nil {
					ctx.WithError(err).Error("removing directory")
				}
			}
			return filepath.SkipDir
		}

		// remove file
		p.ch <- func() { // -- 9
			if err := os.Remove(path); err != nil {
				ctx.WithError(err).Error("removing file")
			}
		}

		return nil
	})

	return &stats, err
}

```

1. [`startN`](#pruner-startn)

2. [`stop`](#pruner-stop)

3. [`filepath.Walk`](#filepath-walk)

> Walk-函数会遍历root指定的目录下的文件树，对`每一个该文件树中的目录和文件`都会调用walkFn，包括`root自身`。

5. [`p.prune(path, info)`](#pruner-prune)

> 检查是否是需要去除的内容，

> - 如果是 `return True`

> - 如果不是是 `return False`

6. [`atomic.AddInt64(`  >> 例子](https://repl.it/@chinanf_boy/AddInt64)

> [AddInt64原子性的将val的值添加到*addr并返回新值。](#atomic-addint64)

7. `if info.IsDir() { p.ch <- func() { `

> 如果是文件夹📁, 传输到 ch:  make(chan func()) 函数通道

9. `p.ch <- func() { `

> 如果是文件📃, 传输到 ch:  make(chan func()) 函数通道

8. [`dirStats`](#dirstats)

> 文件夹状态

9. [os](#os-remove-or-all)

 - `os.Remove` - Remove删除name指定的文件或目录。

 - `os.RemoveAll`- RemoveAll删除path指定的文件，或目录及它包含的任何下级对象。

---

`p.ch <- func(){}` - 带着 **func**- 去到 -> `start`

``` go
 // start loop.
func (p *Pruner) start() {
	defer p.wg.Done() // 等待⌛️-wg-做完了
	for fn := range p.ch { // <--- 这里
		fn()
	}
}
// for 就像 while 关键字 -- 永远等待-p.ch-的

```

> 直到 close

``` go
// stop loop.
func (p *Pruner) stop() {
	close(p.ch) // <--- close
	p.wg.Wait() // 等待⌛️-就等着结束
}
```

---

## Pruner-startN

代码 252-258

``` go
// startN starts n loops.
func (p *Pruner) startN(n int) {
	for i := 0; i < n; i++ {
		p.wg.Add(1)
		go p.start()
	}
}
```

- `	p.wg`    sync.WaitGroup

> WaitGroup用于等待一组线程的结束。

- `go p.start()`

> 并发-开始

## Pruner-stop

代码 260-266

``` go
// stop loop.
func (p *Pruner) stop() {
	close(p.ch)
	p.wg.Wait()
}
```

- `close(p.ch)` 

> 关闭通道

- `p.wg.Wait()`

> 等待并发结束

## Pruner-prune

检查是否是需要去除的内容，

- 如果是 `return True`

- 如果不是是 `return False`

代码 228-250

``` go
// prune returns true if the file or dir should be pruned.
func (p *Pruner) prune(path string, info os.FileInfo) bool {
	// directories
	if info.IsDir() { // 是否文件夹
		_, ok := p.dirs[info.Name()] // 在文件夹定义里面?
		return ok
	}

	// files
	if _, ok := p.files[info.Name()]; ok { //在文件定义里面?
		return true
	}

	// files exact match
	if _, ok := p.files[path]; ok { // 在文件路径定义里面?
		return true
	}

	// extensions
	ext := filepath.Ext(path) // 在扩展定义里面?
	_, ok := p.exts[ext]
	return ok
}
```
## 其他工具-or-原生函数

### filepath-Walk

[文档]((https://studygolang.com/static/pkgdoc/pkg/path_filepath.htm#Walk)

### atomic-AddInt64

[文档](https://studygolang.com/static/pkgdoc/pkg/sync.htm#WaitGroup)

## os-Remove-or-All

[文档](https://studygolang.com/static/pkgdoc/pkg/os.htm#Remove)

### dirStats

给予一个文件夹路径，返回

- 文件数总数

- 去除文件数

- 去除文件大小


代码 274-286

``` go
// dirStats returns stats for files in dir.
func dirStats(dir string) (*Stats, error) {
	var stats Stats

	err := filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
		stats.FilesTotal++ // 文件数总数
		stats.FilesRemoved++ // 去除文件数
		stats.SizeRemoved += info.Size() // 去除文件大小
		return err
	})

	return &stats, err
}
```

### toMap

[repl](https://repl.it/@chinanf_boy/node-prune-toMap)

代码 288-295

``` go
// toMap returns a map from slice.
func toMap(s []string) map[string]struct{} {
	m := make(map[string]struct{})
	for _, v := range s {
		m[v] = struct{}{}
	}
	return m
}

```