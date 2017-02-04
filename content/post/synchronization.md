+++
title = "Synchronization"
date = "2016-11-22T22:28:44-08:00"
+++

As noted in the previous post, synchronization in Go is best handled using **channel**. This, however, does not preclude the use of synchronization primitives such as mutex and condition variable. Go provides the **sync** package for these primitives.

### Mutex

Mutex, or mutual exclusion, is a simple lock that prevents concurrent use of a common resource. Go's mutex is aptly named **sync.Mutex**, which implements the **Locker interface**.

**Locker interface** declares 2 methods: **Lock()** and **Unlock()**. Notice that there is no initialization method as in C's POSIX mutex **pthread_mutex_t**. This is because, like every other object in Go, a structure's initial state should be all-zero; and it suffices to initialize a **sync.Mutex** using **new()**, or simply **&sync.Mutex{}**.

The **Locker interface** is declared as the following:

{{<highlight go>}}
type Locker interface {
    Lock()
    Unlock()
}
{{</highlight>}}

Consider the following example where you have 2 goroutines concurrently increment a global variable up to a million, the result with proper resource protection should be 2 million:

{{<highlight go>}}
const limit = 1000000

var m sync.Mutex
var count int32 = 0 // increment until limit

func main() {
    go incr()
    go incr()
    time.Sleep(5000 * time.Millisecond) 
    fmt.Printf("count = %d\n", count)
}

func incr() {
    for i := 0; i < limit; i++ {
        m.Lock()
        count++
        m.Unlock()
    }
}
{{</highlight>}}

For the sake of demonstration, we use a **time.Sleep()** to wait for 5 seconds to allow ample time for both goroutines to complete.

Using a mutex to protect a single global integer can be an overkill. It will be easier, and cleaner, using **atomic** operations, provided by **sync/atomic** package:

{{<highlight go>}}
var count int32 = 0
func incr() {
    for i : = 0; i < limit; i++ {
        atomic.AddInt32(&count, 1) // atomically increment "count"
    }
}
{{</highlight>}}

This is the preferred way of incrementing a variable. It is cleaner, and runs more efficiently than the mutex version. 

### WaitGroup

Our example above uses a **time.Sleep()** to wait for both goroutines to finish before the main thread exits. In practice, this is bad. For one, we don't always know how long the main thread should wait for; for two, the goroutine may change which changes the timing.

The better way is allow the goroutines to tell us when it's finished. One way to do it is using a **channel**, as seen in the previous post. Another way to do it is using a **sync.WaitGroup**.

A **sync.WaitGroup** contains an internal counter. A goroutine may increment this counter by calling **Add()**, and decrement by calling **Done()**. Every goroutine that calls **Wait()** is blocked until the **WaitGroup**'s internal counter reaches 0.

For example, since we want 2 goroutines to run to completion, each goroutine, upon execution, will increment the **WaitGroup** counter, and only decrement when the goroutine is finished. The main thread then **Wait()** on the **WaitGroup**, blocked until the counter reaches 0.

Let's modify the previous code to use **WaitGroup**:

{{<highlight go>}}
const limit = 1000000

var count int32 = 0       // increment until limit
var wg sync.WaitGroup   // synchronize goroutine completion

func main() {
    // invokes 2 goroutines
    for i := 0; i < 2; i++ {
        wg.Add(1) 
        go incr()
    }

    wg.Wait() // blocked until both goroutines are done
    fmt.Printf("count = %d\n", count)
}

func incr() {
    defer wg.Done()  // release the waitgroup when function exits

    for i := 0; i < limit; i++ {
        atomic.AddInt32(&count, 1) // atomically increment "count"
    }
}
{{</highlight>}}

Notice that the **WaitGroup** is incremented by 1 before invoking a goroutine, and decremented with **Done()** when each goroutine exits. The call to **Wait()** blocks until every goroutine runs to completion, and then the main thread exits.

Could we have put the **wg.Add(1)** into the function itself like this?

{{<highlight go>}}
func main() {
    // invokes 2 goroutines
    for i := 0; i < 2; i++ {
        go incr()
    }

    wg.Wait() // blocked until both goroutines are done
    fmt.Printf("count = %d\n", count)
}

func incr() {
    wg.Add(1)   // increment as soon as incr() starts
    defer wg.Done()  // release the waitgroup when function exits

    for i := 0; i < limit; i++ {
        atomic.AddInt32(&count, 1) // atomically increment "count"
    }
}
{{</highlight>}}

The answer is **No**. Most likely the result of **count** is 0 instead of 2000000.

Why is that? The invokation of a goroutine does not guarantee to start before the next line of code executes. In this case, the for loop invokes 2 goroutines, then it may go ahead to executes **wg.Wait()** before each goroutine has a chance to run the first line of code **wg.Add(1)**. Since a **WaitGroup**'s initial state is 0, **wg.Wait()** simply returns. The main thread prints out the **count** value of 0 and application exits.

### Detecting Race with "-race"

The problem above is called a **Race Condition**. Go comes with a native tool to help detect race condition using the command line argument "-race".

To see how it works, run the code above with the following command, assuming the filename is "sync_wg.go":

    go run -race sync_wg.go

You will notice a message similar to the following:

    ==================
    WARNING: DATA RACE
    Write at 0x000000584ebc by main goroutine:
        internal/race.Write()
            /opt/go/src/internal/race/race.go:41 +0x38
        sync.(*WaitGroup).Wait()
            /opt/go/src/sync/waitgroup.go:129 +0x151
        main.main()
            /home/go/src/test/sync_wg.go:19 +0x77

    Previous read at 0x000000584ebc by goroutine 6:
        internal/race.Read()
            /opt/go/src/internal/race/race.go:37 +0x38
        sync.(*WaitGroup).Add()
            /opt/go/src/sync/waitgroup.go:71 +0x292
        main.incr()
            /home/go/src/test/sync_wg.go:24 +0x47

    Goroutine 6 (running) created at:
        main.main()
            /home/go/src/test/sync_wg.go:16 +0x53
    ==================
    count = 2000000
    Found 1 data race(s)
    exit status 66

What this tells us is that there is a potential race between the 3 lines of code:

* sync_wg.go:19
* sync_wg.go:24
* sync_wg.go:16

These lines correspond to:

* wg.Wait()
* wg.Add(1)
* go incr()

This is hint to tell you that a race is happening between these 3 calls.

Notice that the result of the **count** is actually correct with 2000000 with the extra argument "-race". Don't trust that outcome, it is the side effect of Go inserting instrumentation code, essentially slowing down the program to produce the race condition analysis results.

Now re-run the code using "-race" with the correct version, with **wg.Add(1)** outside the goroutine invokation. You will notice that there is no warning. Congratulations, your code is race free.

