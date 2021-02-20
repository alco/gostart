# gostart

## 目录 ##

* [动机](#motivation)
* [Go 的方式](#canonical)
* [Go 工具并不能完成所有工作](#missing-features)
* [FAQ](#faq)
  * [0. 什么是 GOPATH 以及我该怎么做?](#faq0)
  * [1. 如何开始写 Go 代码?](#faq1)
  * [2. 我写了一些代码，我如何运行它?](#faq2)
  * [3. 如何将我的包分割成多个文件?](#faq3)
  * [4. 如何将我的包分成多个子包?](#faq4)
  * [5. 如何创建一个供他人使用的软件包（即非主包）?](#faq5)
  * [6. 运行 `go build` 和 `go install` 的输出去哪里去了?](#faq6)
  * [7. 如何设置多个工作区?](#faq7)
  * [8. 我可以在 `$GOPATH` 之外创建一个包吗？](#faq8)
  * [9. 如果我想破解 `$GOPATH` 之外的某些代码（可能是丢弃），该怎么办?](#faq9)
  * [10. 如何下载远程软件包?](#faq10)
  * [11. 如何区分库包和主包?](#faq11)
  * [12. 将命令和程序包保存在不同的工作区中有意义吗？?](#faq12)
  * [13. 我可以在我的代码中导入命令吗?](#faq13)
  * [14. 如果我不想在我的导入路径中使用代码托管域名，该怎么办?](#faq14)
  * [15. 如何管理软件包版本?](#faq15)
  * [16. 如何在部署时冻结软件包?](#faq16)
  * [17. 我在哪里可以找到更多关于学习 Go 的信息?](#faq17)
  * [18. 当前有哪些编辑器支持 Go ?](#faq18)

<a name="motivation"/>

## 动机

**go tool**默认与 Go 发行版绑定在一起，便于自动执行常见任务，如获取依赖关系，构建和测试代码。它很容易使用，并提供一个一致的命令行界面，它也希望你尊重一堆的约定，其中一些我觉得很奇特的，认为他们为一些用户引入了一点点学习曲线，需要一些习惯。

当公约被 go tool 强制执行，对于一个死忠 gopher 来说似乎很自然，这需要新手加快速度。如果你视图让它为你工作，并寻求 `#go-nuts` 频道或 [golang-nuts group][2] 显示你的代码布局和错误信息 `go get` 或 `go build` 产生，你很可能会被告知在 Go 中编码之前首先要学习如何正确使用 Go 工具。它的文档事实上是相当不错的，但是一下子吸收它并不容易。

我的经历是这样的，推荐的[初始阅读][1]和邮件列表上的讨论都不能完全帮我理清楚。我只能通过网上收集案例，通过试验，以及查看 Go 工具的源码来最终掌握方向。

在这篇文章中，我将从外部人的视角来解释一下 Go 的方法。假设你会遇到类似的障碍，本指南将回答你的问题，并帮助你了解 Go 工具的约定。底部还有一个 FAQ 代码示例。

<a name="canonical"/>

## Go 的方式

以下是使用 Go 工具时需要注意的一些基础知识。

### 1. Go 工具仅与在工作区中代码兼容

这是编写 Go 代码的基本规则。有一些例外情况适用于小程序和丢弃代码，但是一般来说，你在 Go 中编写的所有内容都将在你的工作区中。那么什么是工作区呢？

工作区是文件系统的一个目录，其路径存储在环境变量 `GOPATH`中。这个目录有一个特定的结构，最基本的说，它应该有一个名为 `src` 的子目录。在 `src` 下创建新的子目录，每一个都是一个单独的 Go package。

这在上述[文章][1]和本文件末尾的 FAQ 中有更详细的描述。为了简单起见，当你开始一个新的 Go 项目时，总是在 `$GOPATH/src` 里创建一个新的目录。

### 2. Go 工具不允许你依赖于特定版本的外部软件包

如果你确实需要一个特定版本的特定软件包，你需要把这个软件包分叉出来，然后按照你想要的特定版本进行检查。如果你需要使用单个软件包的不同版本，则需要为每个版本创建一个单独的分支。

很显然，一旦你需要给定的软件包不止一个版本，这种方法就不会缩小和变得单调乏味。Go 背后的推理是版本控制非常困难，以至于你的依赖关系应该更好的在你的最新版本和最新版本的各自依赖关系上递归的工作。虽然这是一个很好的建议，但在实践中通常是不可能的。你将需要依赖你的直接依赖项的具体版本或依赖项的依赖项等。

简而言之，不要试图用 Go 工具来模拟版本控制，它并不能很好的发挥作用。

### 3. 使用 Go 工具时，请使用完全限定的导入，并始终从 `$GOPATH` 构建导入路径

Go 中没有本地包。尽管本地导入在一定程度上是可能的，但对于 Go 开发者来说，他们比 Go 用户更重要。

任何你导入的都是相对于你的 `$GOPATH/src` 。因此，如果你有一个像下面这样的目录结构：

```sh
.
└── src
    └── gopher
        ├── main.go
        └── sub
            └── sub.go
```

在你的 _main.go_ 文件中，你将需要如下所示去导入 sub:

```go
package main

import "gopher/sub"

func main() {
    sub.ExportedFunction()
}
```

如果你想使用某个代码托管站点公开提供的 Go 包，推荐的方式是像下面这样导入：

```go
import "codehosting.com/path/to/package"

func main() {
    package.ExportedFunction()
}
```

Go 工具让你下载远程包，或者通过他们的导入路径传递给 `go get` 或者在你的项目目录中调用 `go get ./...` 来递归的获得所有的远程依赖。

```sh
go get codehosting.com/path/to/package
```

下载的软件包将会放到 `$GOPATH/src/codehosting.com/path/to/package`. 
Go 工具也将自动的构建一个静态库放到 `$GOPATH/pkg/...`. 要了解更多，运行 `go help get` 和 `go help gopath`. 另请参阅[问题10](#faq10).

<a name="missing-features"/>

## Go 工具并不能完成所有工作

来自其他语言/环境，你可能会期望 `go` 是一个完整的包管理解决方案。它不是！ 以下 FAQ 可能对你有用：

  * [如何设置多个工作区？](#faq7)
  * [我可以在 `$GOPATH` 之外创建一个包吗？](#faq8)
  * [如何管理软件包版本？](#faq15)
  * [如何在部署时冻结软件包？](#faq16)

<a name="faq"/>

## FAQ

<a name="faq0"/>

### 0. 什么是 GOPATH 和我该怎么做?

对于初学者来说，`GOPATH` 最好的办法就是设置它并且忘记它。
如果你刚开始使用 Go，请执行以下操作：

1. 安装 Go.
2. 选择一个你要保存 Go 代码的目录，并设置为 `GOPATH`，例如：

```sh
# in your .bashrc or similar file
export GOPATH=$HOME/go
```

3. 忘掉它。在你的 `$GOPATH/src` 启动每一个新的项目，然后用 Go 工具来构建，测试，获取依赖关系。

如果你想了解更多 `GOPATH` ，在本 FAQ 中经常提到。另外，请看下面的资源。

* 官方 [如何编写 Go 代码][1] 向导
* `build` 包 [文档][3]
* [GOPATH][4] (社区 wiki)
* [排除你的 Go 安装问题][5] (社区 wiki)

<a name="faq1"/>

### 1. 如何开始写 Go 代码?

在开始写代码之前，你应该选择一个目录成为你的 Go 工作区的目录。你所有的 Go 代码将放在那里。将 `GOPATH` 环境变量设置到 `.bashrc` 或类似的文件中。

```sh
export GOPATH=/Users/alco/go
```

你还需要在 `$GOPATH` 下创建一个 `src` 子目录，这是保存你 Go 包和命令的地方。以下是你的初始目录结构：

```sh
$ tree -L 2 $GOPATH
/Users/alco/go
└── src
    ├── example
    ├── gopher
    └── testy
```

`src` 中的每个子目录代表一个单独的包或命令（看[问题11](#faq11)，描述包和命令之间的区别）。每个文件至少包含一个 _.go_ 文件，也可能包含自己的子目录。

有关 `GOPATH` 和工作区目录结构的更多信息，运行 `go help gopath`。

<a name="faq2"/>

### 2. 我写了一些代码。我如何运行它？

导航到你的软件包目录，并试用 Go 工具来构建和运行你的代码。

假设你在 `$GOPATH/src/example` 目录下创建了一个程序:

```sh
$ cd $GOPATH/src/example
$ ls
main.go

$ cat main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello world!")
}
```

快速运行它的方法是使用 `go run`:

```sh
$ go run main.go
Hello world!
```

如果你的主包被分成多个文件（参见[问题3](#faq3)），那么你需要将它们全部作为参数传递给 `go run`.

然而，不久，你将需要从 Go 源生成一个二进制文件，你可以将其作为独立的可执行文件运行，而无需使用 Go 工具。使用 `go build`，它会在当前目录下创建一个可执行文件。

你也能运行 `go install`，它将构建代码并生成一个可执行文件放到 `$GOPATH/bin`。你可能希望将后者添加到 `PATH` 环境变量中，以便能够在文件系统中的任何位置都可以直接运行 Go 程序。如果你设置了 `GOBIN` 环境变量，那么运行 `go install` 的结构将被放置在那里。

```sh
$ go build
$ ls
example main.go

$ ./example
Hello world!

$ go install
$ $GOPATH/bin/example
Hello world!
```

在这里，我们定义了一个主函数，它是我们的 Go 程序的起点。任何 Go 程序都必须有一个 `main` 函数。按照惯例，这些包（声明 `package main` 的包）被称为命令。 换句话说，我们已经建立了一个名为 _example_ 的命令。

你也可以定义其他名称的包，这些名称就是简单的包。它们不是为了生成可执行的程序，而是作为一些命令的一部分被包含在内，这些命令将提供一个在其中定义了一个 `main` 函数的主包。

也可以看看: `go help run`, `go help build`, `go help install`。

<a name="faq3"/>

### 3. 如何将我的包分割成多个文件?

只要在 `package` 声明中具有相同的名称，Go 将单个目录中的文件视为属于一个包。

让我们继续处理我们的示例命令，我们将添加一个名为 _helper.go_ 的文件，并在其中定义一个辅助函数。

```go
$ cd $GOPATH/src/example
$ cat helper.go
package main

func privateHelperFunc() int {
    return 25
}

# now edit main.go to call this function
$ tail -3 main.go
func main() {
    fmt.Println("Hello world! My lucky number is", privateHelperFunc())
}
```

现在我们不能简单地 `go run main.go`，因为 _main.go_ 调用另一个文件中定义的函数。 我们需要传递所有文件作为参数来运行或生成当前包，然后运行生成的二进制文件。

```sh
$ go run main.go
# command-line-arguments
./main.go:6: undefined: privateHelperFunc

$ go run *.go
Hello world! My lucky number is 25

$ go build
$ ./example
Hello world! My lucky number is 25
```

私有（非导出）函数和数据可以在属于单个软件包的所有文件中访问。

一个主包只允许定义一个 `main` 函数，所以你需要从你的主包中选择一个文件来放入它。

最后，你也可以将其他软件包分成多个文件，而不是 main 文件一个。Go 的标准软件包可以普遍使用这个功能。

<a name="faq4"/>

### 4. 如何将我的包分成多个子包?

子包只是发生在另一个包的目录中的独立包。 Go 不会以任何特殊的方式处理它们，所以子包的导入路径是相对于你的 `$GOPATH/src`。只有当它的功能绑定到包含它的主包时，以及当把这个包与其他顶层包放在一个层次上没有意义时，才使用子包。

我们在我们的示例项目中创建一个名为 _math_ 的子目录，并在那里创建一个名为 _math.go_ 的文件。

```sh
$ cat math/math.go
package math

func Mul2(x int) int {
    return x * 2
}
```

我们编辑 _main.go_ 来调用 `Mul2()` 。

```
$ cat main.go
package main

import (
    "fmt"
    "example/math"  // just "math" would not work, it would import std package math
)

func main() {
    fmt.Println("Hello world! My lucky number is", math.Mul2(privateHelperFunc()))
}

$ go run *.go
Hello world! My lucky number is 50
```

请注意，如果要用 `package main` 替换 _math.go_ 中的 `package math`，这不会使 _main.go_ 和 _helper.go_ 属于同一主包的 _math.go_ 部分。 它将被视为另一个主包，并且在尝试运行或导入时会遇到错误，因为它没有定义主函数。

<a name="faq5"/>

### 5. 如何创建一个供他人使用的软件包（即非主包）?

在前面的问题中，我们主要是看编写所谓的命令 - 声明 `package main` 的包，并且是为了生成可执行的二进制文件而构建的。

Go 包的其他风格被用作其他语言的库或模块，你不能将它们构建到可执行文件中。 他们的目的是导入到另一个包（不一定是主包）中，为这个包提供有用的功能。

你可以像创建命令一样创建一个包。唯一的区别是，不是 `package main` ，而是 `package <some other name>` 。在前面的问题中描述的所有其他规则也适用于这些软件包：你可以将一个软件包分成多个文件，但属于软件包的所有文件都驻留在一个目录中。

假设我们已经创建了一个名为 _util_ 的包，位于 `$GOPATH/src/util` 中。

```sh
$ cd $GOPATH/src/util
$ cat main.go   # file name is arbitrary and doesn't make any significance to Go
package util

import "math"

func Square(x float32) float32 {
    return x * x
}

func Circle(r float32) float32 {
    return math.Pi * r * r
}

func cube(x float32) float32 {
    return x * x * x
}

$ go build
# no output
```

在一个简单的（非主要）包中，`go build` 被用来验证这个包编译没有错误。它不会产生任何二进制。 要预编译包以供其他包使用，可以运行 `go install` 。这会将 _util.a_ 放到 `$GOPATH/pkg/<arch>/util.a` 中。 通过运行 `go help gopath` 来了解更多有关 _pkg_ 目录的信息。

在我们的 util 包中，有两个导出函数（ `Square` 和 `Circle` ）和一个私有函数（ `cube` ）。 私有函数仅在作为包的一部分的文件中可见。其他软件包只能调用导出的函数。

另见这个[wiki页面][7]，它描述了发布和使用远程包的过程。

<a name="faq6"/>

### 6. 运行 `go build` 和 `go install` 的输出去哪里去了?

这两个行为有点不同，取决于你是否是建立一个命令（一个主包）或一个简单的包。

在构建主包时，生成的可执行文件将被放置在当前目录中。另一方面，运行 `go install` 将像往常一样构建源代码，并将可执行文件放在 `$GOPATH/bin` 中（除非你有 `GOBIN`环境变量设置，在这种情况下生成的二进制文件将被放置在那里）。这也在[第二个问题](#faq2)中讨论过。

在简单包的目录中运行 `go build` 不会产生任何二进制文件。 用这种方式构建一个包来验证它是干净地编译的。为了从包源创建一个二进制文件，在包的目录中运行 `go install` 。 正如[上一个问题](#faq5)中所讨论的，这将在 `$GOPATH/pkg/<arch>` 目录内创建一个 _.a_ 文件。当建立其他导入当前包的包时，Go 工具将能够选择这个二进制文件。

也可以看看: `go help build`, `go help gopath`。

<a name="faq7"/>

### 7. 我如何设置多个工作区？

简短的回答 - 你不需要。Go 工具确实希望你能够在单个工作区中工作。你可以在你的 `GOPATH` 环境变量中添加多个，但是有一个问题：`go get` 会一直下载新的包到 `GOPATH` 中列出的第一个位置。

因此，为 `GOPATH` 添加多个路径很少有用，但你可能仍然需要使用多个工作区。 你只需要根据你当前所在的工作区来更改 `GOPATH` 。这里有一些可以自动标记的构建工具。

也可以看看 [问题14](#faq14) 有一个例子介绍使用多个工作区。

<a name="faq8"/>

### 8. 我可以在 `$GOPATH` 之外创建一个包吗？

不行，你可以改变你的 `GOPATH` ，如前面的答案所述。但请记住 `$GOPATH` 指向一个工作区。 你就需要进入工作区的 _src_ 目录打包。

<a name="faq9"/>

### 9. 如果我想破解 `$GOPATH` 之外的某些代码（可能是丢弃），该怎么办？

如果你不打算导入标准库之外的任何东西或者只有一个本地导入级别的，那么它将为你工作。但是，如果你需要使用完全限定的导入，则必须将代码移至工作区，否则在尝试 `go get` 你的依赖关系时可能会遇到问题，并且还可能发生其他不好的事情。

对于特定情况可能的解决办法是可能的，那些可以由另一个工具提供。 但是，一般来说，你必须坚持 Go 的惯例来使它适合你。

<a name="faq10"/>

### 10. 如何下载远程软件包?

获取当前包的所有依赖关系：

```sh
go get ./...
```

要下载特定的远程软件包：

```sh
go get <package import path>  # 运行 'go help packages' 了解更多详情

# 示例：
go get github.com/user/package
```

所有下载的包都将放在 `$GOPATH/src` 。它们也被自动构建并安装到 `$GOPATH/pkg` 中。 您可以通过将 `-d` 标志传递给 `go get` 来跳过安装：

```sh
go get -d github.com/user/package
```

<a name="faq11"/>

### 11. 如何区分库包和主包?

这两种包在你的工作区的 _src_ 目录中并排存放，因此在文件系统级别上没有区别。 有一个惯例叫前者简单的包和后者的命令。所以，如果你的包的第一行读取`package main`，那么这是一个命令。否则，它只是被称为一个包。

<a name="faq12"/>

### 12. 将命令和程序包保存在不同的工作区中有意义吗?

我是反对的。使用多个工作区是棘手的，每次在它们之间切换时都需要记得调整 `GOPATH`。另请参阅[问题7](#faq7)了解更多详情。

<a name="faq13"/>

### 13. 我可以在我的代码中导入命令吗?

当然，但是在导入过程中你需要提供一个别名，这样如果你把它导入到一个主包中，这个包的 `main` 函数就不会和你的主函数相冲突。

```go
import chef "github.com/user/chef"
```

<a name="faq14"/>

### 14. 如果我不想在我的导入路径中使用代码托管域名？

不要有这种想法，这是Go社区已经广泛采用的做法。它也适用于 `go get` 自动获取或更新您的依赖关系。沿用下去，你就不会在使用其他人的包有麻烦以及共享自己的包也不会有麻烦。

如果你仍然好奇，在 `go get` 下载你的依赖关系之后，你可以在 `$GOPATH/src`里面移动东西。尽管如此，这将会违背 Go 方式。而且，目前唯一可以区分别人软件包的方法是通过查看 `$GOPATH/src` 中的目录结构：远程软件包将驻留在 _github.com_ 或 _code.google.com_ 之类的目录中，这样很方便。

将远程软件包保存在另一个目录中，与你自己的软件包分开，肯定会很棒。它可以通过在 `GOPATH`中指定多个路径来模拟。运行 `go get` 将总是将软件包下载到 `GOPATH`中列出的第一个目录中，然后可以在 `GOPATH` 路径列表中的第二个目录中创建自己的软件包。

使用多个工作区也有它的问题。有关更多详细信息，请参阅[问题7](＃faq7)

<a name="faq15"/>

### 15. 我如何管理软件包版本？

目前还没有广泛采用的软件包版本管理解决方案。Go 工具不知道包的版本；通过这个文档[does choose package versions (vcs tags)](http://golang.org/cmd/go/#hdr-Download_and_install_packages_and_dependencies) 基于你当前的Go版本.

目前被接受的方法是建立你的所有依赖主从。 运行`go get -u ./...`来更新本地缓存的依赖关系是 _helpful_ 以避免麻烦，尽管并不理想。

在你准备编写自己的软件包版本管理器之前，一定要阅读以前在这方面的尝试的讨论：

* [gonuts](https://groups.google.com/d/topic/golang-nuts/cyt-xteBjr8/discussion)
* [gopkg](https://groups.google.com/d/topic/golang-nuts/-65WPrNcT3U/discussion)
* [packaging as debs](https://groups.google.com/d/topic/golang-nuts/-UG-lF_Ey-o/discussion)
* [gopack](https://groups.google.com/d/topic/golang-nuts/c1V9PjKhOz4/discussion)

<a name="faq16"/>

### 16. 部署时如何冻结 packages ?

go 工具不提供任何方法来创建一个傻瓜式部署的可重现环境。

如果你在本次测试你的代码，你永远不能保证它会在你的下一次部署中工作，因为其中一个依赖可能在测试和部署之间的时间段内引入了一个重大的变化。

用 [dep](https://github.com/golang/dep) 来管理和锁定你的依赖。它不是工具链的一部分，但将包含在你未来的版本中。

<a name="faq17"/>

### 17. 我从哪里可以找到关于学习 Go 的更多信息？

[http://golang.org/doc/](http://golang.org/doc/)

看看[Go talks][6]官方列表. 对于刚开始进入 Go 程序员的思维是很有帮助的。希望他们在 Go 官方网站上更突出地提到。

<a name="faq18"/>

### 18. 当前有哪些编辑器支持 Go ?

看这个 [wiki page](http://code.google.com/p/go-wiki/wiki/IDEsAndTextEditorPlugins).

[1]: http://golang.org/doc/code.html
[2]: http://groups.google.com/group/golang-nuts
[3]: http://golang.org/pkg/go/build
[4]: http://code.google.com/p/go-wiki/wiki/GOPATH
[5]: http://code.google.com/p/go-wiki/wiki/InstallTroubleshooting#Tips
[6]: http://code.google.com/p/go-wiki/wiki/GoTalks
[7]: http://code.google.com/p/go-wiki/wiki/GithubCodeLayout
