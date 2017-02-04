+++
title = "Hello World"
draft = false
date = "2016-07-11T04:36:09Z"

+++

Go shows a lot of promise to be a serious contender to C. The first things that catches my attention, although not necessarily the comprehensive list, are the followings:

1. No more messy makefiles
* Github integration
* Testing framework
* Built-in unicode support

These are the obvious benefits that I can immediately sense based on my limited exposure. I am sure after more in-depth time with Go, I will discover other pros and cons.

The first thing I need is to find something I can build with Go. The first example is none other than **"Hello World!"**. This takes about 5 minutes, and achieves the followings: 

1. Know where to get Go
* Know how to build and run Go

### Getting Go ###

To get your hands on Go, proceed to download the Go binary distributions [here](https://golang.org/dl/). Go is open source, so if you are feeling adventurous, you have access to the complete Go compiler's [source code](https://github.com/golang/go).

Once Go is installed on the system, it's usually a good idea to set the GOROOT environment variable to the installation path so that the "go" binary can be found under $GOROOT/bin/go.

A programming language is not complete without all the supporting libraries. In the world of Go, libraries are called **packages**. The list of built-in packages are listed [here](https://golang.org/pkg/).

Since I am already familiar with VIM for C development, I decided to explore the list of VIM plugins to make Go programming easier. The main plugin I am currently using is "vim-go" plugin available at https://github.com/fatih/vim-go. "vim-go" gives me great syntax highlighting capability, auto completion and Go syntax error checking.

To facilitate Go development for myself, I have packaged the golang binary and vim plugins into a Vagrant file that I can bring up any time I need a fresh VM for development. This Vagrant file can be found on [my github](https://github.com/howardplus/go-devel).

### Building and Run Go ###

The first application is to print out "Hello World!" to the screen. A standalone application always use the special package **main**. Analagous to the **stdio.h** header file in C, we need to import the **fmt** package in Go, which defines the method **Printf()** to print formatted string. Lastly, the entry function of Go is also called **main()** as in C, and it is a function that takes and returns no argument.

That's all that's needed to create our first "Hello World!" application:

{{<highlight go >}}
package main

import (
	"fmt"
)

func main() {
	// code to run
	fmt.Printf("Hello World!\n")
}
{{</highlight>}}

Now, I save this file as **helloworld.go** and exit the editor. In Go, all the source files have the extension **go**. 

The easiest way to test Go code is to do **go run** followed by the file name. In this example, I can do the following to run it:

	# go run helloworld.go
	Hello World!

This executes the specified source code without explicitly building it. To build the code into a binary executable, use the **build** command:

	# go build helloworld.go
	
It generates a Go binary named after the file name, which is **helloworld** in this case.

Both **run** and **build** can take wildcard, or when there is nothing specified, run or build every Go source files in the current directory.

For example, if I have 3 files in the current directory:

	# ls
	foo.go  helloworld2.go  helloworld.go

I can selectively build **helloworld.go** and **helloworld2.go** together this way:

	# go build helloworld*.go

Or if I want to build everything inside the current directory, which include all 3 files, I can omit the last parameter and just do:

	# go build

This creates a binary that is named after the directory name. 

The recommended Go project directory layout can be found [here](https://golang.org/doc/code.html#Organization).

