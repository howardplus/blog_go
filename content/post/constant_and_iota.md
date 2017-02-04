+++
date = "2016-07-25T23:26:21Z"
title = "Constant and Iota"

+++

### Constant

Constants are values that are not altered by the program. To declare a constant in Go, use the **const** keyword:

{{<highlight go>}}
const charConstant = 'c'
const stringConst = "string"
const boolConst = true
const intConst = 25
const floatConst = 5.1
{{</highlight>}}

A constant always deduce its type from the value. In case of ambiguity such as declaring an **uint** data type from an integer, you may cast the value:

{{<highlight go>}}
const uintConst = uint(25)
fmt.Printf("type of uintConst = %T\n", uintConst)
{{</highlight>}}

Go's fmt package provides a handy specifier "%T" that prints out its type. This works on either variable or constant. This code will print out:

	type of uintConst = uint

Typically there are more than one constant to be declared at once. Constants can be grouped together using parenthesis:

{{<highlight go>}}
const (
	charConstant = 'c'
	stringConst = "string"
	boolConst = true
	intConst = 25
	floatConst = 5.1
)
{{</highlight>}}

With the **const** block, a constant that does not take on any value will inherit the type and value from the previous constant:

{{<highlight go>}}
const (
	stringConst = "string"
	stringConst2
	intConst = 25
	intConst2
)
{{</highlight>}}

In this case, **stringConst2** and **intConst2** do not have any specified value, therefore **stringConst2** will be set to the same value as **stringConst**, which is of type string, and value "string"; likewise, **intConst22** will be set to type int with value "25".

A constant does not have to be a single value; it can be composed of a mathematical expression:

{{<highlight go>}}
const twoTimeFive = 2*5
{{</highlight>}}

This will produce a constant of type **int** and value of 10.

### Iota

C provides an **enum** type, which takes on integer type and automatically increment values based on its position in the enum. 

Go does not have an **enum** type; a special keyword value **iota** is used instead to automatically increment an integer constant:

{{<highlight go>}}
const (
	zero = iota
	one = iota
	two = iota
)
{{</highlight>}}

Recall that if a constant does not take on any value, it inherits the previous element. This is especially useful in the case of iota, where the keyword **iota** can be assigned to the first one, and the rest of the constants just inherit from its previous element:

{{<highlight go>}}
const (
	zero = iota
	one
	two
)
{{</highlight>}}

In both of these cases, the constants **zero**, **one**, and **two** takes on the value 0, 1, and 2.

To skip a value in **iota**, you may put a placeholder:

{{<highlight go>}}
const (
	zero = iota
	one
	_ 		// skipping value 2
	three	// value is 3
)
{{</highlight>}}

In this case, the value 2 has been skipped, and the constant **three** will take on the value 3.

What happens if you need to skip to, say 1000? Remember that **iota** is just a special constant value that increments; you may apply mathematical expression to an **iota**:

{{<highlight go>}}
const (
	oneThousand = iota + 1000
	oneThousandOne
	oneThousandTwo
)
{{</highlight>}}

At each line, the **iota** value increments by 1, but the value 1000 stays constant, resulting in 1000, 1001 and 1002.

You may also mix expressions in the same **const** block:

{{<highlight go>}}
const (
	oneThousand = iota + 1000
	oneThousandOne
	oneThousandTwo
	oneHundredThree = iota + 100
	oneHundredFour
)
{{</highlight>}}

In this case, the constant **oneHundredThree** no longer inherits the value from the previous value, but **iota** continues to increment to 3, resulting in the value "103". The only value that increments is **iota**, not the expression.

Lastly, the **iota** value resets to 0 on a new **const** block:

{{<highlight go>}}
const (
	zero = iota
	one
)
const (
	zeroAgain = iota
)
{{</highlight>}}

Here, the value of **zeroAgain** is 0, since it is the first **iota** in its **const** block.
