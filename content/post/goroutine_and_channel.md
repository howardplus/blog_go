+++
date = "2016-11-12T04:58:41Z"
title = "Goroutine, Channel and Select"

+++

In this post, we introduce three features in Go that deal with concurrrency. The three features are:

* **goroutine**: lightweight thread to spawn concurrent execution
* **channel**: a generic communication channel
* **select**: waiting on multiple channels to perform operation

### Goroutine

In C, thread creation is facilitated using the POSIX pthread library. With Go, multi-threading is supported by the language using **goroutine**.

To create a thread, you may invoke a new function using the keyword **go**:

{{<highlight go>}}
func main() {
	go runMe()
}

func runMe() {
	fmt.Printf("I have been run\n")
}
{{</highlight>}}

If you actually run this code, you may notice that it doesn't print anything. The reason is that the main thread may exit before **fmt.Printf** is executed; and the program exits when the main function exits.

We can block on user input, similar to **getchar()** in C:

{{<highlight go>}}
func main() {
	go runMe()

	// wait for \n
	r := bufio.NewReader(os.Stdin)
	r.ReadLine()
}

func runMe() {
	fmt.Printf("I have been run\n")
}
{{</highlight>}}

This way, the main thread does not exit right away, and the spawned goroutine **runMe()** executes and print out:

	I have been run

A goroutine does not have a preset function declaration; the function that follows the keyword **go** can be in any form. A caveat is that the return value of the function is ignored. In other words, you should design a goroutine function where the caller and callee do not communicate via the return value.

So how should they communicate?

The answer is **channel**.

### Channel

As the name suggests, **Channel** is used to communicate between goroutines. **Channel** is in many ways similar to a Linux pipe, except that channel is a language feature, and the data communicated via channel is strongly typed.

To see how channel works, we will rewrite the previous simple goroutine example.

The goal is to remove the wait on **os.Stdin**, and let the spawned goroutine **runMe()** notifies the main thread when it finishes. To achieve that, we create a channel that takes on a boolean value, allow the main thread to wait on any message from the channel, and have the spawned goroutine send **true** to this channel after printing is complete.

The result is the following code:

{{<highlight go "linenos=inline">}}
func main() {
	done := make(chan bool)
	go runMe(done)
	<-done // wait on channel "done"
}

func runMe(done chan bool) {
	fmt.Printf("I have been run\n")
	done <- true // send true to channel "done"
}
{{</highlight>}}

Now if we run this code, the output is now:

	I have been run

And this time, the program automatically completes without user input.

Let's now revisit what we have done here. In line 2, the channel variable **done** is created using the built-in function **make()**. The keyword **chan** tells you that we are creating a channel, and what follows **chan** is the data type of the communicating element, which is **bool** in this case. This channel is then used by both the main thread and the goroutine **runMe**.

After the main thread spawns the goroutine, it attempts to receive a value from the channel **done**. This is achieved using the receive operator, represented using the left-handed arrow "<-". This operator indicates the flow of data in the channel; in this case, data is received from the channel **done**. Since we do not care what the data is, the presence of data suffices to unblock the main thread, there is nothing to the left of the receive operator.

In the spawned goroutine, the function outputs the desired message to the console, and eventually sends **true** to the channel. This is again achieved using the receive operator. But in this case, the channel **done** is on the left hand side, indicating the flow of data.

Previously we mentioned that a goroutine does not have return values. The same effect of a return value can be achieved using a channel. As we demonstrated in this simple example, the goroutine **runMe** actually returns a **bool** value. We can make a slight change in the main thread to catch this return value:

{{<highlight go>}}
func main() {
	done := make(chan bool)
	go runMe(done)
	ret := <-done // wait on channel "done" and assign value to ret
	fmt.Printf("runMe returns %t\n", ret)
}
{{</highlight>}}

Here, we are omitting the definition of **runMe** since it stays the same. With this modification, the boolean value from the goroutine is passed back to the main thread, and the output of this code is:

	I have been run
	runMe returns true

### Unbuffered and Buffered Channel

A channel created without a second argument is called an **unbuffered channel**. That basically means that sending a value is blocked until this value can be received. 

Conceptually you may think of an **unbuffered channel** like passing a baton, where the first runner is unable to let go of the baton until the second runner successfully catches it.

Let's consider the following code using an **unbuffered channel**:

{{<highlight go>}}
func main() {
	c := make(chan bool)
	c <- true
}
{{</highlight>}}

The main thread creates an unbuffered channel and attempts to send a data to it. The result is a deadlock, as the Go compiler will indicate:

	fatal error: all goroutines are asleep - deadlock!

If you think back to the passing the baton analogy, this is like having a lone runner attempting pass the baton to nobody. The result is the poor runner running in circles without anyone to pass to.

Even though channels are designed to communicate between different goroutines, you don't always have to. It is absolutely possible to send a message to itself.

The way to do it is to use a **buffered channel**.

A channel can be created using **make()**, with an optional second argument to indicate the number of elements. The sender only blocks when there are no more space available to send the new element.

Therefore, a **buffered channel** of size 1 will only start blocking when the sender attempts to send the second element. Using a buffered channel, we can then send the message from the same thread as the receiver:

{{<highlight go>}}
func main() {
	c := make(chan bool, 1) // buffered channel of size 1
	c <- true // first element won't block
	v := <-c
	fmt.Printf("v = %t\n", v)
}
{{</highlight>}}

In this case, the first send does not block, and this code will execute without deadlock:

	v = true

### Select

A goroutine may receive messages from multiple channels. Consider a case where 2 channels are used between main thread and a goroutine. Channel **c1** is an integer channel, while channel **c2** contains a string. The goroutine sends an integer, a string, and another integer sequentially to the main thread, where the value gets printed out to the console:

{{<highlight go "linenos=inline">}}
func main() {
	c1 := make(chan int)
	c2 := make(chan string)

	go sender(c1, c2)

	fmt.Printf("%d\n", <-c1)
	fmt.Printf("%s\n", <-c2)
	fmt.Printf("%d\n", <-c1)
} 

func sender(c1 chan int, c2 chan string) {
	c1 <- 1
	c2 <- "test"
	c1 <- 2
}
{{</highlight>}}

Since both **c1** and **c2** are unbuffered channels, each of the expressions from line 7 through 9 are blocking calls. This means that orders between **c1** and **c2** are critical.

If line 14 and 15 are swapped, the application will deadlock:

{{<highlight go>}}
	c1 <- 1
	c1 <- 2 // now we send to c1 before c2
	c2 <- "test"
{{</highlight>}}

In order to propery receive messages on multiple channels without relying on a mutually-enforced order between sender and receiver, which is not always possible, Go supports the **select** construct.

**Select** allows you to block on multiple channels, and return when any of the channels become available. A **select** block looks similar to a **switch** block, except that each **case** statement needs to receive a message from a channel. The previous code can be rewritten using **select**, and there is no need to worry about ordering between channels:

{{<highlight go "linenos=inline">}}
func main() {
	c1 := make(chan int)
	c2 := make(chan string)

	go sender(c1, c2)

	for {
		select {
		case i := <-c1:
			fmt.Printf("%d\n", i)
		case s := <-c2:
			fmt.Printf("%s\n", s)
		}
	}
}

func sender(c1 chan int, c2 chan string) {
	c1 <- 1
	c1 <- 2
	c2 <- "test"
}
{{</highlight>}}

Here, we use a **select** block to check on 2 different channels: **c1** and **c2**, and print out their values as soon as data is received. The **select** block exits on the first time any of the channel unblocks, therefore we wrap that in an infinite loop to continue waiting for new data from any of the channels. With this new change, the order of the channel sends in the **sender** goroutine will no longer cause deadlocks. 

If you run this code, it will print out these values in the right order:

	1
	2
	test

But you will also notice that after these values are printed, the application is again deadlocked. What happens is that after all 3 data have been processed, the main thread then attempts to receive value from either **c1** and **c2**, which blocks forever. Go runtime detects that every thread is in the **sleep** state, and reports the deadlock error.

This can be fixed with a terminating condition. Suppose a value of zero on channel **c1** is used to indicate goroutine **sender** terminates, we can now rewrite the **select** block to detect that:

{{<highlight go>}}
for {
	select {
	case i := <-c1:
		if i == 0 {
			return // exit function entirely
		}
		fmt.Printf("%d\n", i)
	case s := <-c2:
		fmt.Printf("%s\n", s)
	}
}
{{</highlight>}}

The **sender()** function then sends a zero to indicate function completion:

{{<highlight go>}}
func sender(c1 chan int, c2 chan string) {
	c1 <- 1
	c1 <- 2
	c2 <- "test"

	c1 <- 0 // all done, send 0
}
{{</highlight>}}

Now if you run this code again, it prints out these values correctly without deadlock:

	1
	2
	test
