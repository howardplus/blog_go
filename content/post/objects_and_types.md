+++
title = "Objects and Types"
draft = false
date = "2016-09-04T06:55:53Z"

+++

### new

We have seen how to allocate memory for a slice or a map using **make()**. For generic data types, allocation can be done using the built-in function **new()**.

C does not provide a type-aware memory allocation function; the **libc**'s requires the user to provide the exact size of the memory to be allocated, and only returns one data type: "char*".

In C++, the function **new** allocate the memory and calls the constructor method. You as the developer is responsible for creating the best suited constructor for that particular object. It's not always easy to get a C++ constructor right.

Go's **new()** is similar to C++ in that it allocates memory and initializes. It just doesn't let user choose what to do with the initialization. The initialization always zero-out the entire object, no matter what it is. It works on all data types, including basic ones such as **int** or **bool**.

The effect of Go's **new()** function behavior is that every data structure you design is required to have its initialized state to be all zero's. In many ways, that is a good programming practice, too.

Let's look at a code example to use **new()**:

{{<highlight go>}}
type X struct {
	Val int
	Count int	
}

x1 := new(X)
{{</highlight>}}

In this case, both **Val** and **Count** are initialized to 0 when allocating with **new()**.

### type

The **type** keyword is new. This is analogous to the **typedef** keyword in C, except that it is required when defining a composite data type. **type** can also be used to give an alias to a basic data type, the same way **typedef** works in C. For example, the following code snippet makes **Integer** an alias of **int**:

{{<highlight go>}}
type Integer int
{{</highlight>}}

Turns out that this actually does more than making **Integer** an alias of **int**. In Go, we can create methods for any type other than the built-in basic data types. For example, we can give **X** a set of methods by defining a function on type X:

{{<highlight go>}}
type X struct {
	Val int
	Count int
}

// X's method GetVal
func (x *X) GetVal() int {
	return x.Val
}
{{</highlight>}}

Creating a method is not allowed on a basic data type such as **int**. The following code snippet is an error:

{{<highlight go>}}
func (i *int) GetVal() int {
	return *i
}
{{</highlight>}}

The compiler output is:

	cannot define new methods on non-local type int

But there is a way around it. By creating another type of the basic data type, Go allows you to define methods just like a composite type:

{{<highlight go>}}
type Integer int

func (i *Integer) GetVal() int {
	return *i
}
{{</highlight>}}

The **Integer** type is used exactly as an **int**, with the added benefit of method creation.

### Return of the Local

Go allows you to return the address of a local variable. In fact, it is a good practice in Go to allocate and return a composite data type this way. The following syntax is called a **composite literal**, and is the preferred way to initialize a composite data type, or a **struct**:

{{<highlight go>}}
func NewX() *X {
	return &X{Val: 5, Count: 1}
}
{{</highlight>}}

Or if you leave any member out, Go will initialize that to be zero for you:

{{<highlight go>}}
func NewX() *X {
	return &X{Count: 1} // Val is 0
}
{{</highlight>}}

C makes clear distinction between what is on the stack, and what is on the heap. Returning a local variable's address in C results in dire consequences. Go compiler takes a different approach. It looks at how this local variable gets used, in this case an address is returned, and decides that it should be allocated on the heap. This object can therefore be subject to garbage collection.

This convenience brought about by Go raises the question of whether the built-in function **new()** is even necessary. I haven't found a use case yet where **new()** is absolutely necessary. The zero'ing capability of **new()** can be easily achieved with an empty initializer:

{{<highlight go>}}
func NewX() *X {
	return &X{} // All members are initialized to zero
}
{{</highlight>}}

I find that using the empty initializer syntax is more intuitive than using **new()**. One little thing is that a basic data type cannot be initialized using literals:

{{<highlight go>}}
func NewInt() *int {
	return &int{}	// error, can't do that with int
}
{{</highlight>}}

The compiler will complain about using a composite literal on a basic data type:

	invalid pointer type *int for composite literal

However, you are allowed to create a **int** local variable and return its address, which basically achieve the same thing:

{{<highlight go>}}
func NewInt() *int {
	i := 0
	return &i	// ok, return address of integer
}
{{</highlight>}}

This is equivalent of using **new()** to return an address of a **int**, initialized to zero.
