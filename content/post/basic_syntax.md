+++
date = "2016-07-23T06:59:26Z"
draft = false
title = "Basic Syntax"

+++

The syntax of Go aims to be simple to the developers. Many decisions in the Go syntax are made with the users in mind, rather than ease of parsing for the compiler. Here we introduce some significant differences between Go and C syntax.

### No semi-colon at the end of line

Each Go statement resides in a single line. Unlike C, it is forbidden to combine statements in a single line in Go. This renders the line-ending semi-colon useless. As mentioned in the Go FAQ, the semi-colon is **"for the parser, not for people"**, and therefore omitted from the language to delimit statements.

The semi-colon, however, is still significant in other context such as the for loop, which we have already seen in the previous post.

### Variable declaration

To declare a variable in C, we start with the type of the variable followed by the variable name, like this:

{{<highlight c>}}
int i = 0;
{{</highlight>}}

In Go, variable declaration requires the keyword **var**, and its order is reversed:

{{<highlight go>}}
var x int = 0
{{</highlight>}}

If the type of the local-scoped variable can be deduced, both the keyword **var** and the type **int** may be omitted:

{{<highlight go>}}
func foo() {
	x := 0
}
{{</highlight>}}

The ":=" assignment operator deduces the type of x to be **int**, therefore avoiding the redundant type and keyword **var**. This only works for local variable, but not global variables.

To assign a number other than the default **int** type, we can simply use type casting:

{{<highlight go>}}
x := uint64(0)
{{</highlight>}}

### The open brace '{' cannot start on its own line

Go uses the same braces '{' and '}' like C to denote the starting and ending position of a code block. The difference is that Go does not support having the open brace to be on its own line. The reason is noted in [Go FAQ](https://golang.org/doc/faq#semicolons). To summarize, the decision is two-fold:

1. Since Go compiler automatically add semi-colon to the end of every line, causing an open brace on its own line a headache to the compiler's parser implementation.

2. Enforcing a coding style can be a good thing.

### Variables needs to be used if declared

Unused variables used to be just a compiler warning in C, which most people choose to ignore. In Go, this is a build error, which you cannot ignore. In some rare cases where an unused variable needs to be there, or should be there for clarity of the code, you need to specifically tell Go that a variable is declared but not used.

For example, the following code is a compile error:

{{<highlight go>}}
func foo() {
	x := 0
}
{{</highlight>}}

The variable x is declared with an initial value of 0, but never used again in the function scope. If you run this code, you will get:

	x declared and not used

The compile error should force to rethink why **x** is needed in the code, and remove it if necessary. However, if you really need the variable there for any reason, the placeholder can be used to avoid this error:

{{<highlight go>}}
func foo() {
	x := 0
	_ = x	// assign x to a placeholder to avoid unused variable error
}
{{</highlight>}}

### Multiple return values from a function

In C, returning multiple values from a function is painful. In many cases, an extra data structure is created with its sold purpose of existence is to allow us to return multiple values from a function. Other times, you need to pass in pointers in order to return values. This makes the function declarations messy.

Go allows multiple return values, where each return value is separated by commas. For example, the built-in **range** operator on a string returns both the index and the character at that index:

{{<highlight go >}}
p := "test"
for i, c := range p {
	fmt.Printf("%d: %c\n", i, c)
}
{{</highlight>}}

Remember that unused variable is a compile error. If **i** is declared but not used, you need to use a placeholder to absorb that return value:

{{<highlight go >}}
p := "test"
for _, c := range p {
	fmt.Printf("%c\n", c)
}
{{</highlight>}}

This is only necessary if you are skipping a return value in sequence. In other words, if you are using just the index, but not the character at that index, there is no need to use a placeholder for **c**:

{{<highlight go >}}
p := "test"
for i := range p {
	fmt.Printf("%d\n", i)
}
{{</highlight>}}

### Code is not compiled from top to bottom

Go does not require a data structure to be defined earlier in a file before they can be used. This is very different from C where code is compiled from top to bottom, and forward declarations are required to work around inter-dependent data structures. 

For example, you may write your main function that utilitizes a data structure that is defined later:

{{<highlight go>}}
func main() {
	s := SomeType{} // this is defined at the bottom
}

type SomeType struct {
	val SomeOtherType
}

type SomeOtherType struct {
	val int
}
{{</highlight>}}

There is no need to forward-declare **SomeOtherType** before it can be used in **SomeType**, which occurs earlier.
