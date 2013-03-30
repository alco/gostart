## Table of contents ##

* [Motivation](#motivation)
* [The Go way](#canonical)
* [Problems with the current Go way](#problems)
* [FAQ](#faq)
  * [How do I start writing Go code?](#faq1)
  * [I've written some code. How do I run it?](#faq2)
  * [How do I split my main package into multiple files?](#faq3)
  * [How do I split my main package into multiple subpackages?](#faq4)
  * [How do I write packages for others to use (i.e. a non-main package)?](#faq5)
  * [How do I set up multiple workspaces?](#faq6)
  * [Can I create a package outside of $GOPATH?](#faq7)
  * [How do I download remote packages?](#faq8)
  * [How do I distinguish between library packages and main packages?](#faq9)
  * [Can I import commands in my code?](#faq10)
  * [Package naming and file naming](#faq11)
  * [What if I want to hack on some (possibly throw-away) code outside of $GOPATH?](#faq12)
  * [What if I don't want to use code hosting domains in my import paths?](#faq13)

<a name="motivation"/>
## Motivation ##

The **go tool** is bundled with Go distribution by default and it's convenient for automating common tasks such as getting dependencies, building, and testing your code. It's easy to use and provides a consistent command-line interface, but it also enforces a set of strict conventions that introduce a slight learning curve for some and require a bit of getting used to.

While the conventions imposed by go tool might seem natural for a hardcore gopher, it takes effort for a newcomer to get up to speed with it. If you hit a wall trying to make it work for you and ask for help on #go-nuts channel or [golang-nuts group][2] showing your code layout and error messages `go get` or `go build` produces, you will most likely be told to first learn how to use go tool properly before you start coding in Go. Its documentation is actually pretty good, but it's not easy to absorb it all at once.

My experience was such that neither the recommended [initial reading][1], nor discussions on the mailing list cleared up the picture completely for me. I was only able to eventually learn the go way by gathering tidbits from the net, through experimentation, and by looking at the go tool's source code.

In this article I'm going to explain the go way from an outsider's point of view. Assuming you're likely to stumble into the same problems I had, this guide should answer your questions and help you understand go tool's conventions. There is also a FAQ with code samples at the bottom.

  [1]: http://golang.org/doc/code.html
  [2]: http://groups.google.com/group/golang-nuts

<a name="canonical"/>
## The Go way ##

### 1. Go tool is only compatible with the code that resides in your workspace

This is a general rule. There is an exception for the simplest case, when you'd like to build a single file that has no remote imports (imports only packages from the standard distribution). But once you start writing your own packages or importing remote packages, go tool won't work well for you unless all of your code resides in workspace.

So let's take a closer look at workspaces and find out what they'r all about.

### 2. Go tool does not allow you to depend on specific versions of external packages

If you really need to have a specific version of a certain package, you'll need to fork that package and check it out at the specific version you desire. If you need to use different versions for different packages, you'll need to create a separate fork for each of the versions.

Obviously, this approach doesn't scale and quickly gets tedious once you need more than one version of any given package. Go's reasoning behind this is that versioning is damn hard, so you should keep your dependencies at the bleeding edge and depend on as little specific version is possible. While this is a good advice, it doesn't stand the test of reality. In practice, you _will_ need to depend on specific versions. The reasons are countless: private svn repo, private own repo, etc.

On other issue with this is that go tool does not provide any way to create a reproducible development environment. If you tested your code locally, you can never be sure that it'll work during your next deploy, because one of the dependency might introduce a breaking change during the time period between your testing and deployment. The only apparent solution to this is to package up your downloaded dependencies and copy them over to your production environment. Again, this will have to be done manually.

### 3. Go tool forces you to use remote imports and always build imports paths from the package root

There is no such thing as local packages in Go. While local imports are supported, they're not documented and are discouraged from use. Anything you import is relative to your $GOPATH/src. Thus, if you have a directory structure like the following one:

```
.
└── src
    └── gopher
        ├── main.go
        └── sub
            └── sub.go
```

in your main.go file you'll need to import sub as follows:

```
package main

import "gopher/sub"

func main() {
    sub.ExportedFunction()
}
```


<a name="problems"/>
## Problems with the current Go way ##

As mentioned before:

  * no freedom to write go code anywhere on your file system
  * no support for setting up a reproducible dev environment or packaging app locally set up environment
  * no support for managing dependency versions
  * URL-ish imports in your code. This is a feature, in fact, but it's rather opinionated.

So, this is the go way. There's nothing wrong with it choosing certain conventions and forcing them on its users. However, even with those conventions there are going to be problems when theory meets practice, it's better to have remedy for that than not.

I'm not advocating changing the go way in any way, but there certainly exists justification for a 3rd party tool that provides more flexible workflow, automates mandane tasks that are inevitable in practice and solves some of the problems with go's approach.


<a name="faq"/>
## FAQ ##

<a name="faq1"/>
### How do I start writing Go code? ###

Before you start any coding, you should pick a directory that will become your Go workspace. All your Go code will reside there. Set the `GOPATH` environment variable to the path to that directory in your .bashrc or similar file for your shell.

```
export GOPATH=/Users/alco/go
```

You'll also need to create a subdirectory named `src` inside your `$GOPATH`, this is where you'll be keeping your Go packages and commands. Here's what your initial directory structure is going to look like:

```
$ tree -L 2 $GOPATH
/Users/alco/go
└── src
    ├── example
    ├── gopher
    └── testy
```

Each of the subdirectories inside `src` represents a separate package or a command. Each one will contain .go files and may also have subdirectories of its own.

For more information about GOPATH and workspace directory structure, run `go help gopath`.


<a name="faq2"/>
### I've written some code. How do I run it? ###

Navigate to your package's directory and use the go tool to build and run your code.

```
$ cd $GOPATH/src/example
$ ls
main.go

$ cat main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello world!")
}

$ go run main.go
Hello world!

$ go build
$ ls
example main.go

$ ./example
Hello world!

$ go install
$ $GOPATH/bin/example
Hello world!
```

Here we defined a main package which has a main function that is the starting point of our Go program. Any Go program must have one main function. By convention, this kind of packages are called commands. In other words, we have built a command named _example_.

You can also define packages with other names, those are simply called packages. They are not intended to produce executable programs, but rather be included as part of some command that will provide a main package with a single main function defined in it.

See also: `go help build`, `go help install`.


<a name="faq3"/>
### How do I split my main package into multiple files? ###

Go treats files in a single directory as belonging to one package as long as they all have the same name in their `package` declarations.

Let's continue working on our example command. We'll add a second file named helper.go and define a helper function inside it.

```
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

Now we cannot simply `go run main.go` because main.go references a function defined in another file. We can either pass all files as arguments to go run or build the current package and then run the produced binary.

```
$ go run main.go
# command-line-arguments
./main.go:6: undefined: privateHelperFunc

$ go run *.go
Hello world! My lucky number is 25

$ go build
$ ./example
Hello world! My lucky number is 25
```

Private (non-exported) functions and data are accessible in all files that belong to a single package.

The main package allows only one main function to be defined, so you'll need to choose one single file from your main package to put it in.


<a name="faq4"/>
### How do I split my main package into multiple subpackages? ###

Subpackages are just separate packages that happen to reside in another package's subdirectory. Go doesn't treat them in any special way, so import paths for subpackages are relative to your `$GOPATH/src`. Use subpackages only their functionality is tied to the main package which contains then and when it doesn't make sense to put that package on one level with other top-level packages.

Let's create a subdirectory in our example project called `math` and create a file there named `math.go`.

```
$ cat math/math.go
package math

func Mul2(x int) int {
    return x * 2
}
```

Let's also edit `main.go` to call Mul2().

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

Note that if you were to replace `package math` in math.go with `package main`, this would not make math.go part of the same main package that both main.go and helper.go belong to. It would be treated as another main package and you would get an error when trying to run or import it because it doesn't define a main function.


<a name="faq5"/>
### How do I write packages for others to use (i.e. a non-main package)? ###

In the previous questions we were looking at writing Go's so called commands — packages that declare `package main` and are meant to be built to produce an executable binary.

The other flavor of Go packages is used as libraries or modules in other languages, you can't build them into an executable. Their purpose is to be imported into another package (not necesarilly main package) to provide useful functionality to that package.

You create a package the same way as you would create a command. The only difference is that instead of `package main` you write `package <some other name>`. All other rules described in previous questions apply to these packages as well: you may split one package into multiple files, but all files belonging to a package reside in a single directory.

Let's say we have created a package called util that resides at `$GOPATH/src/util`.

```
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

In the context of a simple (non-main) package, `go build` is used to verify that the package compiles without errors. It doesn't produce any binary. To precompile the package for use by other package, you can run `go install`. This will put `util.a` into `$GOPATH/pkg/<arch>/util.a`. Read more about the pkg directory by running `go help gopath`.

In our util package, there are two exported functions (Square and Circle) and one private function (cube). The provide function is only visible in files that are part of the package. Other packages can only call exported functions.

<a name="faq6"/>
### How do I set up multiple workspaces? ###

Short answer — you don't. Go tool does expect you to work in a single workspace. Even if you add two paths to your GOPATH, go get will always download new packages into the first path.

Using two workspaces can be only sometimes marginally useful when GOPATH is updated automatically and temporarilly by some kind of project management tool.

<a name="faq7"/>
### Can I create a package outside of $GOPATH? ###

No. And there is no easy workaround either. You'll need to modify Go's toolchain to achieve that.

<a name="faq8"/>
### How do I download remote packages? ###

To get all dependencies for the current package:

```
go get ./...
```

To download a particular remote package:

```
go get <package import path>  # see go help packages for details

# for instance,
go get github.com/user/package
```

All downloaded packages end up in $GOPATH/src. They are also automatically built and installed in $GOPATH/pkg. You can skip installation by passing -d flag to go get:

```
go get -d github.com/user/package
```

<a name="faq9"/>
### How do I distinguish between library packages and main packages? ###

They live side by side in your src directory, so there's no distinction in file system. There is a convention to call the former ones simply packages and the latter ones commands. So, if your package's first line reads `package main`, it's a command. Otherwise, it's just called a package.

<a name="faq10"/>
### Can I import commands in my code? ###

Sure, but you'll need to provided an alias during import so that the package's name is not main.

```
import chef "github.com/user/chef"
```

<a name="faq11"/>
### Package naming and file naming ###

<a name="faq12"/>
### What if I want to hack on some (possibly throw-away) code outside of $GOPATH? ###

If you're not going to import anything outside of standard library or have one level of local imports, then it'll work for you with the go tool as it current is. If, however, you need to use fully qualified imports, you have to move your code to Go workspace.

Workarounds are possible for particular cases and those can be provided by a 3rd party tool. In general, however, you have to stick to Go's conventions to make it work for you.

<a name="faq13"/>
### What if I don't want to use code hosting domains in my import paths? ###
