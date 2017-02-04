+++
date = "2016-09-12T06:54:49Z"
title = "Control Statements"

+++

Go has 3 ways to control a program flow: **if**, **for** and **switch**.

### If statement

The **if** statement is used to control the flow based on a boolean condition:

{{<highlight go>}}
condition := true
if condition {
	// do something if true
} else {
	// do something else if false
}
{{</highlight>}}

Of course, this code is uninteresting, since the condition takes on a hard value **true**. In most cases, it takes a lot more work to reach a boolean "condition" value in order to make a decision.

For example, we may want to lookup something in a **map**, and run a piece of code based on the presence of the key:

{{<highlight go>}}
_, found := m["foo"] // m is a map
if !found {
	fmt.Printf("key foo is missing!")
}
{{</highlight>}}

This pattern of "running a statement and use the result of the statement as the condition to **if**" happens quite often; this is why Go supports the second **if** syntax:

{{<highlight go>}}
if _, found := m["foo"]; !found {
	fmt.Printf("key foo is missing!")
}
{{</highlight>}}

These code is more succinct, but they are not identical. The second code snippet limits the scope of "found" to within the **if** block; while the first code snippet allows you to use "found" outside the **if** block. 

The initialization part of the **if** statement typically involve a statement that leads up to the decision used in the second condition part. However, this is not strictly enforced by the language.

### For loop

In C, there are two ways to run loops: **for** and **while**. Both of them continuous run a statement block while a condition is true, with **for** provides initialization and increment (or decrement) in its expression. If we omit the initialization and increment/decrement part of the **for** expression, it works exactly the same way as a **while** loop.

In fact, there are 3 very popular methods of doing an infinite loop in C:

{{<highlight c>}}
for (;;) {
	// infinite loop with for
}
{{</highlight>}}

{{<highlight c>}}
while (1) {
	// infinite loop with while
}
{{</highlight>}}

{{<highlight c>}}
do {
	// infinite loop with do-while
} while (1)
{{</highlight>}}

All of these styles are correct, and all of these are widely used.

Go recognizes that a **while** loop may be completely replaced by **for** loop, but still has its place when a single exit condition is needed. Therefore, the **while** loop is phased out, and the **for** loop is expanded to include the single-condition syntax of a typical C **while** loop.

All of the followings are supported by a Go **for** loop:

* Traditional initialize, condition, increment style:

{{<highlight go>}}
for i := 0; i < 10; i++ {
	// loop through 0 to 9
}
{{</highlight>}}

* Single condition style, or **while**-like loop:

{{<highlight go>}}
cond := true
for cond {
	cond = some_func() // some_func() returns a bool
}
{{</highlight>}}

* Infinite loop:

{{<highlight go>}}
for {
	// infinite loop
}
{{</highlight>}}

These 3 styles cover all of the possibilities provided by C's **for** and **while** statements, while enforcing a single style.

Go also supports **break**, **continue** and **goto** exactly the same way C does. They are still necessary when the termination condition inside the for statement does not cover every possiblity a loop should exit or continue to the next iteration.

### Switch statement

The **switch** statement in C takes an integer constant value and jump to a location specified by the **case** tag. The **case** keyword simply provides a location where the code execution begins, and you are required to put in a **break** to exit the switch, or the program will continue to fall through.

A typical **switch** statement therefore looks like the following:

{{<highlight c>}}
switch (val) {
case 0:
	// do something when val == 0
	break;
case 1:
	// do something when val == 1
	break;
default:
	// do something when val == anything else
	break;
}
{{</highlight>}}

It is common C bug when a developer forgets to put **break**, and execution falls through to the next **case** block.

Go's **switch** statement changes that.

First, it no longer falls through, which means that you don't need to put **break** at the end of every **case** block:

{{<highlight go>}}
func PrintVal(val int) {
	switch val {
	case 0:
		fmt.Printf("val=0\n")
	case 1:
		fmt.Printf("val=1\n")
	default:
		fmt.Printf("val=%d\n", val)
	}
}
{{</highlight>}}

Calling **PrintVal(0)** results in:

	val=0

Secondly, each **case** condition may be a comma-separated list of values. For example, the following code returns whether the input is a prime number less than 30: 

{{<highlight go>}}
func IsPrimeLT30(val int) bool {
	switch val {
	case 2, 3, 5, 7, 11, 13, 17, 19, 23, 29:
		return true
	}
	return false
}
{{</highlight>}}

Thirdly, Go's **switch** statement does not have to make decision based on a single variable; it works on expressions as well. You may leave out the **switch** variable part and the **switch** statement will evaluate each **case** expression from top to bottom until a match is found. This essentially allows you to write structured **if-else** blocks.

Consider the following code snippet that tests primality of a given integer:

{{<highlight go "linenos=inline">}}
func IsPrime(n uint) bool {
	switch {
	case n == 2 || n == 3:
		return true
	case n == 0 || n == 1 || n%2 == 0 || n%3 == 0:
		return false
	}
	i := uint(5)
	w := uint(2)
	for uint(i*i) <= n {
		if n%i == 0 {
			return false
		}
		i += w
		w = 6 - w
	}
	return true
}
{{</highlight>}}

The mathematical proof of this code is outside the scope of this discussion. But notice that from line 2 through 7, the **switch** statement does not take any variable; each **case** statement is composed on a complex boolean expression. This is equivalent to writing multiple **if-else** blocks; but is generally considered a better style in Go.

Lastly, the **switch** statement can act on data types. To understand how **switch** works on data type, we need to first discuss an important concept in Go: **interface**. That's the topic we will cover in the next post.
