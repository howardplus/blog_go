+++
date = "2016-07-26T02:53:05Z"
draft = false
title = "Using Map"

+++

A **map** is a key and value store, where each key can direct you to a certain value. It's incredibly useful in programming, and it's no surprise that Go provides **map** as a built-in data type.

A map is declared using the keyword **map** followed by the key type in **[]** and the value type. A **map** with key of type **string**, and value of type **int** will be declared as follows:

{{<highlight go>}}
var m map[string]int
{{</highlight>}}

A map can be initialized either using the built-in function **make()**, or by assigning a map literal. They are functionally the same, although with **make()**, you may optionally give it a **preallocation size**:

{{<highlight go>}}
// m1 is an empty map of string->int
m1 := map[string]int{}

// m2 is an empty map of string->int,
// preallocated to hold 10 items
m2 := make(map[string]int, 10)
{{</highlight>}}

In most practical purposes, the first style using a map literal is sufficient. It is worth noting that even though map can be allocated with **make()** with an initial capacity, the built-in **cap()** function does not work with a **map**.

Once a map has been created, you may assign a value to a map using square brackets:

{{<highlight go>}}
m1["foo"] = 100
{{</highlight>}}

This assigns the value "100" with the key "foo". Since each key maps to one and only one value, a duplicate assignment replaces the old value:

{{<highlight go>}}
m1["foo"] = 200 // replace original 100 with 200
{{</highlight>}}

The square bracket can be used as either l-value or r-value. When used as r-value, it returns the value mapped to this particular key:

{{<highlight go>}}
fmt.Printf("%d\n", m1["foo"])
{{</highlight>}}

In this case, as we last set the value of key "foo" to 200, the output is "200".

What if a key is not found? The return value of an unfound key is 0 on an integer type, empty string "" for a string, or nil for a pointer.

But how do I distinguish between "key not found" and "key gives me a 0 value"? Fortunately, **map** returns 2 values instead of 1. The second return value is a boolean indicating whether the key is found:

{{<highlight go>}}
value, found := m1["foo1"]
if !found {
	fmt.Printf("key foo1 is not found\n")
} else {
	fmt.Printf("value = %d\n", value)
}
{{</highlight>}}

In this case, since key "foo1" has not been assigned to a value, the second return value "found" returns **false**. The output is therefore:

	key foo1 is not found
---
Map supports the **range** built-in function, which iterates through all of its key/value pairs. In the following example, we first create a map literal that maps an abbreviated month string to its numeric value. We then iterate through every item and print it out:

{{<highlight go>}}
months := map[string]int{
    "Jan": 1,
    "Feb": 2,
    "Mar": 3,
    "Apr": 4,
    "May": 5,
    "Jun": 6,
    "Jul": 7,
    "Aug": 8,
    "Sep": 9,
    "Oct": 10,
    "Nov": 11,
    "Dec": 12}

for k, v := range months {
    fmt.Printf("%s : %d\n", k, v)
}
{{</highlight>}}

The result of this code is the following:

	May : 5
	Sep : 9
	Dec : 12
	Jan : 1
	Feb : 2
	Mar : 3
	Aug : 8
	Oct : 10
	Nov : 11
	Apr : 4
	Jun : 6
	Jul : 7

Notice that the order of the iteration does not conform to the order during assignment. In fact, running this code the second time will result in completely different order. In Go, **map** is an **unordered map**, where data are not sorted in any particular order.

An item can be removed from a map using the built-in **delete** function with its key:

{{<highlight go>}}
delete(months, "Dec")
{{</highlight>}}

This function removes "Dec" from the map, leaving it with just 11 items.
