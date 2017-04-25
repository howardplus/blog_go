+++
title = "C Binding"
date = "2017-03-14T23:46:37-07:00"

+++

Writing code in native Go is fun. But there are times where some code has already been written in C, and there is no reason to rewrite that code. It would be nice for Go to call C function, or access variables in C.

And that's where C binding comes in.

### C Binding

C binding is the ability for Go code to access C code. To do so, all you need to is put the C code in the **comment** right above the special "C" package:

{{<highlight go>}}
/*
#include <stdio.h>

const char *str = "String in C";

int foo()
{
    printf("Function foo in C\n");
    return 0;
}
*/
import "C"
{{</highlight>}}

C binding is made possible by the "cgo" facility. Everything are included in the comment section is available to the Go code via "C" package. The "cgo" facility automatically compiles the commented C code, and made the function and variables available to Go.

In this example, the C function **foo()** can be called via **C.foo()**; while the global variable **str** can be accessed via **C.str**.

{{<highlight go>}}
func main() {
    val := C.foo()
}
{{</highlight>}}

The return value **0** is passed from the function **C.foo()** to the variable **val** in Go.

There is a problem with accessing **C.str**. The type **const char*** is different from Go's **string** type. To convert C's **char *** to Go's **string**, we use a conversion function **C.GoString()**:

{{<highlight go>}}
func main() {
    fmt.Printf("%s\n", C.GoString( C.str ))
}
{{</highlight>}}

Conversely, a Go string can be converted to C string using **C.CString()**:

{{<highlight go>}}
func main() {
    C.str = C.CString("modified string")
    fmt.Printf("%s\n", C.GoString(C.str))
}
{{</highlight>}}

There is just one problem. **C.CString** makes an additional allocation in C to hold the string "modified string" from Go. And since C code does not have garbage collection, it's your responsibility to free the memory:

{{<highlight go>}}
/*
#include <stdlib.h>
const char *str = "String in C";
 */
import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    C.str = C.CString("modified string")
    fmt.Printf("%s\n", C.GoString(C.str))
    // need to free this string back to the C world
    C.free(unsafe.Pointer(C.str))
}
{{</highlight>}}

Couple things to note here. The additional **#include** is necessary so that the function **C.free()** is available. Also, function **C.free()** acts on pointers, which in Go is only provided via **unsafe.Pointer** type.

A list of special conversion functions are listed here for reference:

{{<highlight go>}}
// Go string to C string
func C.CString(string) *C.char

// Go []byte slice to C array
func C.CBytes([]byte) unsafe.Pointer

// C string to Go string
func C.GoString(*C.char) string

// C data with explicit length to Go string
func C.GoStringN(*C.char, C.int) string

// C data with explicit length to Go []byte
func C.GoBytes(unsafe.Pointer, C.int) []byte
{{</highlight>}}

### Compile Flags

With cgo, the comments before **import "C"** is compiled and loaded into the "C" package. The compilation flags can be further controlled using **#cgo** primitives. 

For example, since cgo pass in "CFLAGS" during the build process, you can optionally include preprocessor definitions. For example, to define "DEBUGGING" definition, we can do the following:

{{<highlight go>}}
/*
#cgo CFLAGS: -DDEBUGGING
#ifdef DEBUGGING
char * str = "String in C (Debug)";
#else
char * str = "String in C";
#endif
*/
func main() {
    fmt.Printf("%s\n", C.GoString(C.str))
}
{{</highlight>}}

since "DEBUGGING" is defined via CFLAGS, and subsequently the C code is compiled with DEBUGGING defined, the output is "String in C (Debug)".

