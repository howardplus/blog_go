+++
date = "2016-09-20T07:23:33Z"
title = "Interface"

+++

One of the very essence of object oriented programming, in my opinion, is the presence of a set of functions to give an object its **personalities**. These personality functions are typically called **methods**.

Take C++ for example, we can define a class **Pet**, with methods **Eat()** and **Sleep()** to describe the behavior of this class. Omitting the constructor, destructor, the class **Pet** may be declared this way:

{{<highlight cpp>}}
class Pet {
public:
	void Eat() {
		printf("eat");
	}

	void Sleep() {
		printf("sleep");
	}
};
{{</highlight>}}

And to create another class that shares the same set of methods, but different behaviors, we use **inheritance**. This defines a **IS A** relationship; the inherited class **IS A** parent class.

{{<highlight cpp>}}
// a dog "IS A" pet
class Dog: public Pet {
	void Eat() {
		printf("dog gulps");
	}
	
	void Sleep() {
		printf("dog snores");
	}
};
{{</highlight>}}

C++ inheritance creates polymorphic behavior; the same methods **Eat()** and **Sleep()** behaves different depending on whether it is a **Dog**, or a **Pet**.

Go does not support inheritance. But C++'s polymorphism really comes in handy, what is Go's solution to this beautiful programming feature?

And this is where **interface** comes in.

### Interface, Part 1

Go supports **interface** type, which contains a set of method signatures. These signatures defines how a set of methods should be implemented, without actually implementing them.

In order for a data structure to be assigned to an interface type, either one of the properties need to be fulfilled:

1. A data structure needs to implement all methods of the interface.
2. A data structure contains an anonymous field that implements all methods of the interface.

The first property is easier to see. To make a data structure assignable to an interface, you need to implement every method. Continuing with our **Pet** example, let's create an interface type that defines two methods: **Eat()** and **Sleep()**:

{{<highlight go>}}
type Pet interface {
	Eat()
	Sleep()
}
{{</highlight>}}

The **interface** keyword tells us that **Pet** is an interface, and the list of methods do not get implemented until we create a data structure. To do that, we now create a **Dog** data structure that implements the **Pet** interface:

{{<highlight go>}}
type Dog struct {
}

func (d *Dog) Eat() {
	fmt.Printf("dog gulps")
}

func (d *Dog) Sleep() {
	fmt.Printf("dog snores")
}
{{</highlight>}}

We can test that a **Dog** can be assigned to the **Pet** interface using a placeholder:

{{<highlight go>}}
var _ Pet = &Dog{}
{{</highlight>}}

Notice that it is the **pointer to Dog** that implements the interface, not **Dog**. This means that you cannot assign **Dog{}** to a **Pet** interface type:

{{<highlight go>}}
var _ Pet = Dog{} // error, Dog does not implement Pet
{{</highlight>}}

This results in the following error:

	Dog does not implement Pet (eat method has pointer receiver)

### Interface, Part 2

A convenient property of inheritance in C++ is that a child only needs to implement the methods that require specialization. If a child's method does not need to differ from it parent, there is no need to re-implement that.

If Go requires a data structure to implement every method given by the interface, how do we achieve this basic and convenient feature of inheritance?

The answer lies in the second property described above.

A data structure can either implement all of its methods, or contain an anonymous field that implements them. This data structure is then free to specialize any method if needed.

Let's extend the **Pet** example from above. Now, consider we are creating 2 more data structures: **Labrador** and **Corgi**, which are different breeds of dog. Both **Labrador** and **Corgi** eat the same way, but a **Corgi** sleep in a slightly different way. Therefore, we need to specialize **Corgi**'s sleep method, but not eat:

{{<highlight go>}}
type Labrador struct {
	Dog
}
type Corgi struct {
	Dog
}

func (c *Corgi) sleep() {
	fmt.Printf("extend leg and sleep\n")
}
{{</highlight>}}

Both **Labrador** and **Corgi** inherits methods from the **Dog**, while only **Corgi** requires specialization on the method **sleep()**. This way, a **Corgi** type can still be assigned to the **Pet** interface type without implementing every method. 

The anonymous field does not need to directly implement every method of the interface; the anonymous field just needs to satisfy either of the 2 properties. 

But be careful here. All 3 types **Dog**, **Labrador** and **Corgi** can be assigned to the interface type **Pet**, but **Labrador** and **Corgi** do not inherit **Dog**. You may assign **Labrador** to a **Pet**, but you are not allowed to assign **Labrador** to a **Dog**. There is no direct inheritance relationship between the two data types.

### Empty Interface

An interface defines a set of method signatures. So can we define an interface that does not contain any method? The answer is **yes**. And this is a special type of interface called an **empty interface**.

Note that even though we have been describing an interface as a set of method signatures, it iself is a **type**. A type can be assigned to a particular interface type as long as it implements, directly or indirectly, all of the methods.

Since an empty interface does not define any method, it is simply used as a type. And since a data type does not have to define any method in order to satisfy the requirement of an empty interface, every data type can be assigned to an empty interface.

It's as close as you can get to a C's **void\***, or a generic pointer type.

An **empty interface** is declared this way:

{{<highlight go>}}
interface{}
{{</highlight>}}

Any composite data type may be assigned to an empty interface:

{{<highlight go>}}
var n interface{}
n = &Dog{}
n = &Labrador{}
n = &Corgi{}
{{</highlight>}}

All of the above are valid expressions.

### Type Assertion

It seems like assigning a data type to an empty interface loses all the type information.

Not quite. The **type assertion** returns the runtime data type from the interface type it is assigned to:

{{<highlight go>}}
var n interface{}
n = &Dog{} // assign Dog to empty interface

d := n.(*Dog) // runtime type assertion
d.sleep()
{{</highlight>}}

The type assertion checks for 2 things:

1. Whether n is **nil**
2. Whether n can be converted to type **\*Dog**

If either condition is not met, it throws a runtime assertion.

A type assertion may also return a second value: a boolean value indicating whether or not the type conversion succeeds. This form does not trigger a runtime assertion when conversion fails; it simply returns **false** on the second return value:

{{<highlight go>}}
if d, isDog := n.(*Dog); isDog {
	d.sleep()
}
{{</highlight>}}

This code will run the method **sleep()** if the type assertion returns **true** on the second return value.

### Type Switch

The aforementioned type assertion feature allows us to nest multiple if-else statements to act differently based on its type like this:

{{<highlight go>}}
if n == nil {
	// do something
} else if d, isDog := n.(*Dog); isDog {
	// do something
} else if d, isLab := n.(*Labrador); isLab {
	// do something 
} else if d, isCorgi := n.(*Corgi); isCorgi {
	// do something 
}
{{</highlight>}}

It looks like this can be better organized using a **switch** statement, and indeed it can be, using a **type switch**.

A **type switch** uses the keyword **type** in the form of type assertion, where each case is a different type, including **nil**. You can simply rewrite the code like this:

{{<highlight go>}}
switch d := n.(type) {
case nil:
	// do something when n is nil
case *Dog:
	// do something if type is *Dog
case *Labrador:
	// do something if type is *Labrador
case *Corgi:
	// do something if type is *Corgi
default:
	// do something if type unknown
}
{{</highlight>}}

