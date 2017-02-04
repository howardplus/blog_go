+++
date = "2016-07-24T07:14:19Z"
draft = false
title = "Basic Data Types"

+++

Go supports the basic data types that C supports, with a few additional data types that facilitate modern computing environment, such as unicode string support. Let's visit the basic data types:

### integer types

Go supports all the signed and unsigned integer types from 8 to 64 bits:

* int, uint
* int8, uint8
* int16, uint16
* int32, uint32
* int64, uint64

**int** and **uint** are 32 bits on a 32-bit system, and 64 bits on a 64-bit system. When a variable is declared using **:=** and given an integer constant, its type is always **int** unless explicitly casted.

### bool

Unlike C, Go natively supports a boolean data type named **bool**, which takes the values **true** and **false**. An integer may not be implicitly converted to a **bool** , which means that an **if** condition only works on **bool**, not **int**.

The following is a compile error:

{{<highlight go>}}
v := 0
if v {
	fmt.Printf("v = %d\n", v)
}
{{</highlight>}}

The result of compiling this gives the following error message:

	non-bool v (type int) used as if condition

To correct this, either explicitly compare the value **v** to its intended target value, or use a **bool** value in the condition of **if**:

{{<highlight go>}}
v := 0
if v != 0 {
	fmt.Printf("v = %d\n", v)
}
{{</highlight>}}

### pointer integer type

A pointer integer type **uintptr** holds enough memory to store a pointer on the system. 

Go is strongly typed. This means that casting from a pointer to **uintptr** is disallowed. Also, Go explicitly disallows pointer arithmetic. So both of the followings are compile errors:

{{<highlight go>}}
v := 0
p1 := uintptr(&v) // error, cannot assign pointer to uintptr type
p2 := &v
p2++	// error, pointer arithmetic not allowed
{{</highlight>}}

The **uintptr** type is used in rare cases where an integer representation of a pointer is absolutely necessary. But how do we convert a pointer to **uintptr** if explicit cast is not allowed?

The answer lies in the **unsafe** package. The **unsafe** package contains a list of low-level operations C provides. It is **unsafe** because it is designed to beat the strongly-typed system Go is designed with. Specifically, the **unsafe** package provides the **Pointer** type, which fundamentally decays a Go pointer into a C-style pointer. This allows you to manipulate pointers in the following ways:

1. Cast from one type to another
2. Pointer arithmetics

In fact, the pointer arithmetic is achieved by first casting a **unsafe.Pointer** to **uintptr**, perform the arithmetics on **uintptr**, and cast it back to **unsafe.Pointer**. The following code is now valid:

{{<highlight go>}}
v := 0
p1 := uintptr(unsafe.Pointer(&v)) // ok, unsafe.Pointer can be cast to uintptr
fmt.Printf("before: %p\n", &v)
vv := (*int)(unsafe.Pointer(p1+4)) // ok, arithmetic allowed on uintptr
fmt.Printf("after: %p\n", vv)
{{</highlight>}}

Run this, and you will get something like the following result:

	before: 0xc420056190
	after: 0xc420056194

The new pointer after the arithmetic is 4 bytes after the original location of **v**. It is not hard to see why the package is called the **unsafe** package now; this operation defeats the purpose of making Go strongly typed, and dereferencing **vv** may not be safe at this point.

### string

In C, a string typically represented by a char array. When working with unicode characters, it is fundamentally the developer's job to make sure that there are enough storage to hold the unicode characters. One way to achieve that is to allocate more **char** per character you intend to hold, which depends greatly on the encoding format.

In Go, a string is a byte slice. A slice in Go describes an array segment. It can though of as a structure that contains the following information:

* pointer to the start of array segment
* length
* capacity

We will go into details of slice later, but know that a **string** is a byte slice that can grow as needed. What makes Go's **string** powerful is that the built-in **string** type already supports unicode character. You can declare a string directly using Chinese characters and **string** can automatically grow to accomodate the storage requirement:

{{<highlight go>}}
str := "六個字的字串"
fmt.Printf("len of str = %d\n", len(str))
{{</highlight>}}

If you can't read Chinese, it literally means "a string of 6 characters". The built-in **len()** function gives you the number of bytes of the string; in this case, it returns 18, meaning 18 bytes were used to represent this Chinese string.

The string, in Chinese characters, is only 6 characters long. To take into account the string is utf-8 encoded, we need to import the **unicode/utf8** package and use the **RuneCountInString()** function:

{{<highlight go>}}
str := "六個字的字串"
fmt.Printf("rune count of str = %d\n", utf8.RuneCountInString(str))
{{</highlight>}}

This does not return the byte count, which is 18, but the number of character in the string, which is 6.

### byte and rune

We have seen the use of **byte** and **rune** in the **string** data type discussion. In fact, **byte** is simply an alias of **uint8**; while **rune** is an alias of **int32**.

A **byte** is synonymous with **unsigned char** in C. When dealing with multi-byte characters, a **rune** is used internally in a **string** to hold each character. 

When iterating over all the characters in a string, the **string** data type recognizes the encoding format and iterate over each character, not byte. Consider the following code:

{{<highlight go>}}
str := "六個字的字串"
for i, c := range str {
	fmt.Printf("%d: %c", i, c)
}
{{</highlight>}}

The for loop will run 6 iterations, corresponding to 6 Chinese characters, not bytes. Each time, the index **i** is not sequential; it represents the byte offset of each character. The output of this code is therefore:

	0: 六
	3: 個
	6: 字
	9: 的
	12: 字
	15: 串

It is important to note that in order to support utf-8 character encoding in the source code, all Go source files should be utf-8 encoded.

### float32/float 64 and complex64/complex128

Go supports floating point number in 2 precisions: **float32** and **float64**, equivalent to **float** and **double** precision numbers in C. The default floating point constnat is **float64*.

In addition, Go has built-in data type to support complex numbers. A complex number consists of a real and imaginary part, with each part represented by either a **float32** or **float64**, resulting in either 64 or 128 bits number.

A complex number is constructed by using the built-in **complex()** function, and is default to **complex128**. Complex number arithmetics are supported by the built-in operations. For example, we can multiple two complex numbers directly using '*' operator this way:

{{<highlight go>}}
c1 := complex(0, 1)
c2 := c1 * c1
fmt.Printf("%g\n", c2)
{{</highlight>}}

Here, we multiple (0+j) by (0+j), which yields -1. We can use the **%g** format specifier to print complex numbers. In this case, the result is:

	(-1+0i)

More complicated math operations on complex number are supporte by the **math/cmplx** package described [here](https://golang.org/pkg/math/cmplx/).
