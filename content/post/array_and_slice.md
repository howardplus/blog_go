+++
date = "2016-08-02T06:25:40Z"
draft = false
title = "Array and Slice"

+++

### Array

In C, an array decays into a pointer to its first element when used as l-value or passed in as a function parameter. Decaying as function parameter often leads to an extra function parameter that represents the array length, or could sometimes result in nasty buffer overflow bugs.

With Go, the length of an array is part of array data type. This means that an int array of size 3 may not be cast to an int array of size 4, or a pointer to int. They are distinct data types that are not interchangeable.

{{<highlight go>}}
func foo(a [4]int) {
	a[3] = 0
}

func main() {
	a1 := [3]int{1, 2, 3}
	foo(a1) // error, cannot assign array of size 3 to array of size 4
}
{{</highlight>}}

Running this code results in the following error:

	cannot use a1 (type [3]int) as type [4]int in assignment

With C, this is syntactically allowed, but the assignment **a[3] = 0** results in buffer overflow:

{{<highlight c>}}
void foo(int a[])
{
	a[3] = 0;
}

int main()
{
	int a1[3] = {1, 2, 3};
	foo(a1);
	return 0;
}
{{</highlight>}}

Here, the parameter **a** of function **foo** loses its length information inside the function, so it's not easy to know exactly the dimension of **a** when working inside the function.

We have said that the length of an array is part of the data type, so it is not difficult for Go compiler to detect an out-of-range array index. In fact, you are forbidden from accessing an array element outside of the allowable range.

Consider the following:

{{<highlight go>}}
a1 := [3]int{1, 2, 3}
a1[3] = 0 // error, array index out of bound.
{{</highlight>}} 

Since Go's index is 0-based, the statement **a1[3]** is accessing an array element outside of its range. The compiler catches that and spits out an error:

	invalid array index 3 (out of bounds for 3-element array)

Once an array is created, its length may not be changed. This makes array really inflexible. The common use case for an array is to create a statically defined set of constants using array literal. Since the compiler can deduce the number of array element directly from the literal, Go provides a convenient syntax:

{{<highlight go>}}
a1 := [...]int{1, 2, 3}
{{</highlight>}}

Why not just omit the number from within the square brackets? Well, that's reserved for a **slice**.

### Slice

A slice takes a segment of an array to form a dynamic array, where its length may be expanded when needed. A slice data type contains 3 components: memory location, length of slice, and the capacility of the slice. You can use the built-in function **len()** to get the length of the slice, and use **cap()** to get the capacity.

Syntactically, a slice is very similar to an array:

{{<highlight go>}}
var s []int		// a slice declaration
s1 := []int{1, 2, 3}	// a slice literal
s2 := make([]int, 5)	// using built-in make function with just length
s3 := make([]int, 5, 10)	// using built-in make function with length and capacity
{{</highlight>}}

Here **s1** creates a slice of length 3 and capacity 3, with values filled in. **s2** is a slice with length 5 and capacity 5; while **s3** is a slice with length 5 and capacity 10.

There are 3 ways to create a slice:

1. carve a slice out of an existing array
* create a slice from slice literal 
* create a slice using **make()**

In the first case, the slice uses the existing array's memory; in the second and third cases, the compiler creates a hidden array, whose memory can be referenced by the slice.

To carve a slice out of an existing array, you specify the starting and ending indices of the array that will be referenced by the slice, where the ending index is non-inclusive.

This code snippet will carve a slice out of array **a1**, with length 1:

{{<highlight go>}}
a1 := [...]int{0, 1, 2, 3, 4}
s1 := a1[2:3]	// slice starts at index 2, of length 1
{{</highlight>}}

When carving a slice out of an array, the array's memory is reused. In the example above, even though our slice ends before index 3, its capacity is extended all the way to the end of the array. This can be confirmed with the following:

{{<highlight go>}}
a1 := [...]int{0, 1, 2, 3, 4}
s1 := a1[2:3]
fmt.Printf("len=%d; cap=%d\n", len(s1), cap(s1))
{{</highlight>}}

The output of this code is:

	len=1; cap=3

The length of the slice is given by the difference of the start and end indices; the capacity is determined by the size of the array **a1**, starting from the starting index '2'.

New data can be appended to a slice using the built-in function **append()**:

{{<highlight go>}}
a1 := [...]int{0, 1, 2, 3, 4}
s1 := a1[2:3]
s1 = append(s1, 6)
for i, v := range a1 {
	fmt.Printf("%d: %d\n", i, v)
}
{{</highlight>}}

Note that **s1** is carved out of an array **a1**, which means that **s1** is actually referencing part of the memory from **a1**. Therefore, when we append a "6" after the slice, this value is overwriting the next element on the array. The output of the for loop confirms this:

	0: 1
	1: 2
	2: 3
	3: 6
	4: 5

We have mentioned that a slice can be dynamically expanded; we have also established that an array is fixed size. So if we **append()** to a slice carved out of an array, and go beyond the capacity of the array, what will happen?

Turns out when a slice goes beyond its capacity, a new array is created to accommodate the new capacity requirement.

This becomes clear in the following code snippet:

{{<highlight go "linenos=inline">}}
a1 := [...]int{0, 1, 2, 3, 4}
s1 := a1[2:4]
fmt.Printf("initial slice => %p %d %d\n", &s1[0], len(s1), cap(s1))
s1 = append(s1, 4) // still room, so slice position is unchanged
fmt.Printf("append with room => %p %d %d\n", &s1[0], len(s1), cap(s1))
s1 = append(s1, 5) // no more room, new slice location created
fmt.Printf("append no room => %p %d %d\n", &s1[0], len(s1), cap(s1))
{{</highlight>}}

Let's walk through this line by line. At first, we create an array **a1** that holds 5 elements. We carve 2 elements to form a slice **s1**, where there are room for just one more element. At line 4, we uses the last element, and at line 5, we print the address of the first slice element to confirm the location does not change. At line 6, since there are no more room for the new element, a new memory location is created.

If you run this code, the output will be something like the following:

	initial slice => 0xc420012160 2 3
	append with room => 0xc420012160 3 3
	append no room => 0xc420070000 4 6

An integer parameter is used when creating a slice using the built-in function **make()**. This parameter determines both the **length** and **capacity** of the slice. In other word, **make(type, n)** creates a full slice of type **type** and capacity **n**, with every element filled with zero values.

What if we want to create an empty slice? This can be achieved one of 2 ways:

{{<highlight go>}}
s2 := []int(nil) // nil slice is empty
s3 := []int{} // empty slice literal
{{</highlight>}}

Both **s2** and **s3** have length and capacity equal to 0.
