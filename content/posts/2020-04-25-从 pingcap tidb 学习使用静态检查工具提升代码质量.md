---
title: '从 pingcap tidb 学习使用静态检查工具提升代码质量'
date: 2020-04-25
description: '如何使用一些 golang 的静态代码检查工具来提升代码质量'
tags: ["go", "pingcap", "tidb"]
categories: ["Go"]
---

大家好，我是 [Dongpo](https://github.com/0xPoe) 。今天想跟大家简单介绍一下如何使用一些 golang 的静态代码检查工具来提升代码质量！

此博客在 [GitHub](https://github.com/0xPoe/blog) 上公开发布. 如果您有任何问题或疑问，请在此处打开一个 [issue](https://github.com/0xPoe/blog/issues)。

## 简介

从去年接触到 TiDB 就开始尝试在社区帮忙修复一些简单的 Bug。最近，我在阅读代码的过程中发现 TiDB 的代码库中有大量的没有必要的类型转换，我就用 GoLand 分析检查出大部分的无效的类型转化，
然后提了一个 [PR](https://github.com/pingcap/tidb/pull/16262) （CEO 半夜 review 代码，哈哈哈）修复。在这个 PR 中 [zz-jason 大神](https://github.com/zz-jason) 评论希望能够通过静态检查工具来检测无效的类型转换。

我经过一些研究，决定使用 [unconvert](https://github.com/mdempsky/unconvert) 来检测无效的类型转换，然后在这个 [PR](https://github.com/pingcap/tidb/pull/16549) 解决了这个问题。
**最近我终于有机会在公司写 Go了，所以我也想在公司的项目上配置和使用一些静态检查工具来提升代码质量。** 在经过一下午的努力之后终于把 TiDB 的大部分检查工具移植到了公司项目上，并且在 github 上创建了一个模板项目
[go-boilerplate](https://github.com/0xPoe/go-boilerplate) 。下面我就简单介绍一下这个模板的构建过程和使用的方式。

## init 项目，添加代码

我最近使用的 Go 版本 1.13.8，所以就使用 go mod 来初始化和管理项目。

```shell
 go mod init github.com/0xPoe/go-boilerplate
```

然后我添加了 main 文件和一个用作例子的 foo 文件，目录结构如下所示：
```text
.
├── foo.go
├── foo_test.go
├── go.mod
├── go.sum
├── LICENSE
├── main.go
├── Makefile
├── README.md
```

我在 main 文件中只是简单的输出一句话：

```go
// main.go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("I love Rust!")
}
```

在 foo 函数我为了测试 go tidy 功能，专门引入了一个第三方的随机生成测试数据的库：go get github.com/Pallinder/go-randomdata。在代码中简单的生成一个随机数并测试：

```go
// foo.go
package main

import (
	"github.com/Pallinder/go-randomdata"
)

func foo() int {
	return randomdata.Number(20)
}
```

```go
// foo_test.go
package main

import "testing"

func TestFoo(t *testing.T) {
	testFoo := foo()
	if testFoo < 0 || testFoo > 20 {
		t.Error("The value should more than 0 and less than 20!")
	}
}
```


## 添加 Makefile，开始构建

在初始化完项目之后，我们需要添加一个 Makefile 来构建项目，我接下来所有的检查工具都是通过 Makefile 来组织和构建的。在 Makefile 中我们要先定义一些基础的通用的变量：

```text
# 定义项目名称
PROJECT=go-boilerplate
GOPATH ?= $(shell go env GOPATH)
P=8

# 确保 GOPATH 已经设置好了
ifeq "$(GOPATH)" ""
  $(error Please set the environment variable GOPATH before running `make`)
endif
# 校验输出是否正常。
FAIL_ON_STDOUT := awk '{ print } END { if (NR > 0) { exit 1 } }'
# 设置环境变量
path_to_add := $(addsuffix /bin,$(subst :,/bin:,$(GOPATH))):$(PWD)/tools/bin
export PATH := $(path_to_add):$(PATH)
# 一些 Go 相关的命令
GO              := GO111MODULE=on go
GOBUILD         := $(GO) build $(BUILD_FLAG) -tags codes
GOTEST          := $(GO) test -p $(P)

# 获取项目的包和包中的文件
PACKAGE_LIST  := go list ./...
PACKAGES  := $$($(PACKAGE_LIST))
PACKAGE_DIRECTORIES := $(PACKAGE_LIST) | sed 's|github.com/0xPoe/$(PROJECT)||'
FILES     := $$(find $$($(PACKAGE_DIRECTORIES)) -name "*.go")
```

这些通用的变量中有两个地方需要注意：
   - 我们定义了一个 FAIL_ON_STDOUT 的 [awk](https://zh.wikipedia.org/zh-hans/AWK) 命令，该命令会检测是否有错误信息输出，我们在后面会多次使用到该命令。NR 是内置的变量表示 number of record。如果我们检测到其他输出信息就失败退出（**输出信息也就是错误信息，我们会做输出重定向**）。
   - 我们匹配和查找出了包下的所有 go 文件，因为我这只是个简单的模板项目没有其他的 package，所以我在 [sed](https://zh.wikipedia.org/wiki/Sed) 中只替换和匹配了第一层 package：`sed 's|github.com/0xPoe/$(PROJECT)||'`。如果你有很多子 package，就需要修改这个替换规则。


## 添加工具，创建命令

我们需要用 go get 获取静态检查工具，但是为了不污染我们项目的依赖（因为我们在代码中实际上并没有用到这些包），我们可以创建一个 tools/check 的目录并且在其中创建一个 mod 来管理这些静态检查工具。

```shell
mkdir tools/check
cd tools/check
go mod init github.com/0xPoe/go-boilerplate/_tools
```
初始化 tools 模块之后目录结构如下：

```text
.
├── foo.go
├── foo_test.go
├── go.mod
├── go.sum
├── LICENSE
├── main.go
├── Makefile
├── README.md
└── tools
    └── check
        └── go.mod
```

有了该模块，我们就可以将工具直接编译到其目录下来使用。下面我就开始介绍目前我的模板项目中用到的一些很有帮助的检查工具：

1.gofmt

```text
fmt:
	@echo "gofmt (simplify)"
	@gofmt -s -l -w $(FILES) 2>&1 | $(FAIL_ON_STDOUT)
```

首先就是 Go 自带的 fmt 检查代码格式并输出错误，**通过 2>&1 将标准错误输出重定向到标准输出**，然后利用我们上面定义的 FAIL_ON_STDOUT 命令来检测结果。

2.[goword](https://github.com/chzchzchz/goword)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/goword: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/goword github.com/chzchzchz/goword

# 先调用上面的编译命令，再调用进行检测
goword:tools/bin/goword
	tools/bin/goword $(FILES) 2>&1 | $(FAIL_ON_STDOUT)
```

该工具主要是检测你代码中 godoc 函数注释的拼写错误，在这里我们检测了包下的所有文件。

3.[gosec](https://github.com/securego/gosec)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/gosec: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/gosec github.com/securego/gosec/cmd/gosec

# 先调用上面的编译命令，再调用进行检测
gosec:tools/bin/gosec
	tools/bin/gosec ./...
```

该工具主要是检测你代码中可能的安全问题，在这里我们使用了 ./.. 匹配所有文件。

4.[golangci-lint](https://github.com/golangci/golangci-lint)

```text
# 使用官方提供的安装脚本安装
tools/bin/golangci-lint:
	curl -sfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh| sh -s -- -b ./tools/bin v1.21.0

# 先调用上面的安装命令，再调用进行检测
check-static: tools/bin/golangci-lint
	tools/bin/golangci-lint run -v --disable-all --deadline=3m \
	  --enable=misspell \
	  --enable=ineffassign \
	  $$($(PACKAGE_DIRECTORIES))
```

该工具非常强大并且它是插件化的，可以集成使用很多其他的检查工具，在这里我们用 `--disable-all` 先关掉了所有的插件，然后用 `--enable=misspell --enable=ineffassign` 只打开这两个检测工具去检测拼写错误和无效的变量分配和赋值。

这个工具实际上能集成几乎所有的静态检查工具，而且还可以做 CI 支持。**如果你的项目足够复杂建议直接启用所有插件，使用这一个工具就能搞定大部分问题。**

5.[errcheck](https://github.com/kisielk/errcheck)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/errcheck: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/errcheck github.com/kisielk/errcheck

# 先调用上面的编译命令，再调用进行检测
errcheck:tools/bin/errcheck
	@echo "errcheck"
	@GO111MODULE=on tools/bin/errcheck -exclude ./tools/check/errcheck_excludes.txt -ignoretests -blank $(PACKAGES)
```
该工具主要是做错误处理的检测，强制要求你处理错误。可以看到我们在命令参数中还排除了一些错误处理的检测。**如果有些错误你不想处理即可将其放入其中。注意我们在这里检测的是 PACKAGES。**

```text
# errcheck_excludes.txt

fmt.Fprintf
fmt.Fprint
fmt.Sscanf

```

6.[unconvert](https://github.com/mdempsky/unconvert)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/unconvert: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/unconvert github.com/mdempsky/unconvert

# 先调用上面的编译命令，再调用进行检测
unconvert:tools/bin/unconvert
	@echo "unconvert check"
	@GO111MODULE=on tools/bin/unconvert ./...
```

这就是我给 TiDB 添加用于检测无效类型转换的工具。

7.[revive](https://github.com/mgechev/revive)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/revive: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/revive github.com/mgechev/revive

# 先调用上面的编译命令，再调用进行检测
lint:tools/bin/revive
	@echo "linting"
	@tools/bin/revive -formatter friendly -config tools/check/revive.toml $(FILES)
```

该工具是一个代码风格格式化工具，可以看到我们定义了一个 revite.toml 规则文件来确定标准。在这里我们检测了所有文件。

```text
# revive.toml

ignoreGeneratedHeader = false
severity = "error"
confidence = 0.8
errorCode = -1
warningCode = -1

[rule.blank-imports]
[rule.context-as-argument]
[rule.dot-imports]
[rule.error-return]
[rule.error-strings]
[rule.error-naming]
[rule.exported]
[rule.if-return]
[rule.var-naming]
[rule.package-comments]
[rule.range]
[rule.receiver-naming]
[rule.indent-error-flow]
[rule.superfluous-else]
[rule.modifies-parameter]
[rule.unreachable-code]
```

8.evt

```text
vet:
	@echo "vet"
	$(GO) vet -all $(PACKAGES) 2>&1 | $(FAIL_ON_STDOUT)
```
我们也使用 Go 自带的检测工具来检测一些语法错误。

9.[staticcheck](https://github.com/dominikh/go-tools)

```text
# 编译到 tool/bin 目录下以供使用
tools/bin/staticcheck: tools/check/go.mod
	cd tools/check; \
	$(GO) build -o ../bin/staticcheck honnef.co/go/tools/cmd/staticcheck

# 先调用上面的编译命令，再调用进行检测
staticcheck:tools/bin/staticcheck
	@echo "static checking"
	@GO111MODULE=on tools/bin/staticcheck ./...
```

该工具主要检测一些无用代码和变量，并且会提供一些简化和优化代码的建议。

10.tidy

```text
tidy:
	@echo "go mod tidy"
	./tools/check/check-tidy.sh
```

我们也利用 Go 自带的 tidy 功能来防止依赖被污染，我们创建了一个脚本来检测依赖是否正常：

```shell
set -euo pipefail

# go mod tidy do not support symlink
cd -P .

cp go.sum /tmp/go.sum.before
GO111MODULE=on go mod tidy
diff -q go.sum /tmp/go.sum.before
```

**该脚本会执行 [tidy](https://golang.org/cmd/go/) 命令并且和你原来的 sum 文件比较。** 我编写 foo 函数时引入第三方库就是为了测试该脚本。

最终的目录结构：

```text
.
├── foo.go
├── foo_test.go
├── go.mod
├── go.sum
├── LICENSE
├── main.go
├── Makefile
├── README.md
└── tools
    ├── bin
    │   ├── errcheck
    │   ├── golangci-lint
    │   ├── gosec
    │   ├── goword
    │   ├── revive
    │   ├── staticcheck
    │   └── unconvert
    └── check
        ├── check-tidy.sh
        ├── errcheck_excludes.txt
        ├── go.mod
        ├── go.sum
        └── revive.toml
```

## 整合命令，快速检测

在上面我们定义好了编译和检测的命令，你可以在这里找到完整的 [Makefile](https://github.com/0xPoe/go-boilerplate/blob/master/Makefile) 文件。

除了检查的命令，我们也可以定义一些常用的开发命令：
```text
# 构建
build:
	$(GOBUILD)
# 清理
clean:
	$(GO) clean -i ./...
	rm -rf *.out
# 测试
test:
	$(GOTEST)
	@>&2 echo "Great, all tests passed."
```

现在我们所有的命令都就绪了，就可以开始封装整合命令：

```text
dev: check test

check: fmt errcheck unconvert lint tidy check-static vet staticcheck goword
```

我们整合出了两个命令，一个是 check 它会执行所有的检测任务，另外一个是 dev 它不仅可以进行检查还跑了单元测试。我们在开发完成之后就可以进行检测并提交，甚至将其作为 CI 任务运行。

**到此为止，我们就基本完善了项目的静态检查工具链。该项目在 github 整理作为 [template 项目](https://github.com/0xPoe/go-boilerplate) 开源，大家可以直接使用 [github template](https://github.com/0xPoe/go-boilerplate/generate) 功能初始化你的项目。**
希望这篇文章对你集成静态代码分析工具有帮助！

---

### 参考链接
[tidb](https://github.com/pingcap/tidb)
