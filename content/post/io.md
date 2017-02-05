+++
date = "2016-12-29T18:15:52-08:00"
title = "I/O"

+++

In this post, we explore how Go handles the important task of I/O.

### io.Reader and io.Writer

The **io** package contains a set of basic interfaces to perform I/O operations. The most important set of interfaces are **io.Reader** and **io.Writer** defined as follows:

{{<highlight go>}}
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
{{</highlight>}}

Both **io.Reader** and **io.Writer** expects a byte slice, and return the number of bytes with corresponding error. The returned error interface is **nil** when the function returns successfully. Data structures that perform I/O operations should implement **Read()** and **Write()** functions.

To see how **io.Reader** and **io.Writer** helps you deal with I/O operations, consider the scenario where you simply want to read from standard input and print it out verbatim. 

In C, you can use **fread()** and **fwrite** to achieve this:

{{<highlight c>}}
#include <stdio.h>

int main()
{
    while (!feof(stdin)) {
        char buf[1];
        size_t bytes = fread(buf, sizeof(char), 1, stdin);
        fwrite(buf, bytes, sizeof(char), stdout);
    }
    return 0;
}
{{</highlight>}}

With C++, the code gets a bit cleaner with the standard template library:

{{<highlight cpp>}}
#include <iterator>
#include <algorithm>
#include <iostream>
#include <string>

using namespace std;

int main()
{
    copy(istream_iterator<string>(cin),
         istream_iterator<string>(),
         ostream_iterator<string>(cout, "\n"));

    return 0;
}
{{</highlight>}}

Go's **io.Reader** and **io.Writer** are similar to C++'s iostream in the sense that they provide standard I/O interface. But they are not containers. This means that you do not wrap standard I/O inside a container structure, such as **istream_iterator**, before they can be used by the **copy()** function. 

In Go, the built-in **copy()** function, which copy from source to destination, takes **io.Writer** and **io.Reader** interfaces. This makes piping data from standard input to output much more succinctly:

{{<highlight go>}}
// example 1: directly copy from stdin to stdout
import (
    "io"
    "os"
)

func main() {
    io.Copy(os.Stdout, os.Stdin)
}
{{</highlight>}}

This code blocks the main thread until an EOF has been reached. In Unix environment, you can press "ctrl-D" to simulate EOF from the console.

Go's generic buffer struct **bytes.Buffer** conveniently implements **io.Reader** and **io.Writer** interfaces, so you can use that as a temporary buffer:

{{<highlight go>}}
// example 2: copy from stdin to stdout using a temporary buffer
func main() {
    var b bytes.Buffer
    io.Copy(&b, os.Stdin) // feed stdin to buffer
    io.Copy(os.Stdout, &b) // send from buffer to stdout
}
{{</highlight>}}

Since all the data are written to the temporary buffer, your inputs do not immediately echo on the screen. It is only when you enter "Ctrl-D", or EOF, that all the inputs will be echoed on the screen all at once.

This is slightly different in the way that input is **buffered** into a temporary space. If your goal is to echo input to the console, this code won't work.

Let's see how we can do that using **io.Pipe()**.

An **io.Pipe()** is an in-memory data pipe. What comes into the pipe must come out exactly as it was. To achieve immediate echo, we can therefore do the folllowing:

{{<highlight go>}}
// example 3: copy from stdin to stdout using a pipe
func main() {
    r, w := io.Pipe()
    go func() {
        io.Copy(os.Stdout, r)
    }()
    io.Copy(w, os.Stdin)
}
{{</highlight>}}

This is a slightly complicated way to implement such a simple task. But it serves as a generic framework that transform data between input and output.

Suppose that we want to format the output using JSON format, with the timestamp of the user input. The output should look like the following:

{{<highlight json>}}
{"UserInput":"12345\n","TimeStamp":"2017-01-01T20:44:02.41548329Z"}
{{</highlight>}}

The first element is the user input; where the second element is the current timestamp. To do this, we will utilize the following packages:

1. **encoding/json**: provides JSON encoding
* **io/ioutil**: I/O utility, specifically ReadAll() that reads until EOF
* **time**: Time stamp

We will talk about the encoding package in a later post; for now, it suffices to know that an encoder can convert a **struct** and writes the JSON-encoded output to a **io.Writer**.

Let's first define the struct called "MyInput" with 2 elements: "UserInput" and "TimeStamp":

{{<highlight go>}}
type MyInput struct {
    UserInput string
    Timestamp time.Time
}
{{</highlight>}}

Now let's create an object of type **MyInput**, and feed some bogus data to **UserInput**. To encode this object into JSON format, and send the output to any **io.Writer** interface, you do the following:

{{<highlight go>}}
u := MyInput{UserInput: "some string", Timestamp: time.Now()}
json.NewEncoder(w).Encode(u)
{{</highlight>}}

The first line simply creates an object of "MyInput", with some arbitrary text. The second line creates a JSON encoder object "NewEncoder()" with underlying **io.Writer** where the encoded data will be written to.

Remember our goal is to convert the console input into JSON format with timestamp specifying the input time. To do so, we need to read from the receiving end of the pipe. The receiving end is an **io.Reader**. We can either read until EOF, or simply use the **ioutil**'s **ReadAll** function:

{{<highlight go>}}
b, _ := ioutil.ReadAll(r) // "r" is the read end of the pipe
{{</highlight>}}

This utility function reads all the data until EOF, and returns a byte slice with error. Let's ignore the error for now. The output "b" now contains the byte slice of all the data from the pipe.

Notice that a pipe does not automatically send the EOF from the console to the pipe; to send EOF on the pipe, you need to explicitly close the writer end of the pipe.

Now we have everything we need, let's rewrite the code to encode the console input, and write the JSON-formatted output to the console output:

{{<highlight go>}}
type MyInput struct {
    UserInput string
    Timestamp time.Time
}

func main() {
    var wg sync.WaitGroup

    r, w := io.Pipe()

    go func(w io.Writer) {
        wg.Add(1)
        defer wg.Done()
        defer r.Close()

        b, _ := ioutil.ReadAll(r)

        u := MyInput{UserInput: string(b), Timestamp: time.Now()}
        json.NewEncoder(w).Encode(u)
    }(os.Stdout)

    io.Copy(w, os.Stdin)
    w.Close()   // send EOF to pipe

    wg.Wait()
}
{{</highlight>}}

Running this code with a single line input "12345", and press "Ctrl-D" to end the standard input with EOF, you should get the following result:

    {"UserInput":"12345\n","Timestamp":"2016-11-02T04:39:42.887672059Z"}

That's it. We can now use a pipe to transform data from standard input to output.

### File I/O

File manipulation are done through the **os** package, where **os.File** represents a file data structure. **os.File** implements **io.Reader** and **io.Writer** so that everything we have said about standard I/O previously holds true on file I/O.

Let's tweak the previous example to read from the standard input, and write to a file "/tmp/output.txt". To achieve that, we only need to change where the anonymous function takes in as a parameter:

{{<highlight go>}}
if f, err := os.Create("/tmp/output.txt"); err == nil {
    go func(w io.Writer) {
        wg.Add(1)
        defer wg.Done()
        defer r.Close()

        b, _ := ioutil.ReadAll(r)

        u := MyInput{UserInput: string(b), Timestamp: time.Now()}
        json.NewEncoder(w).Encode(u)
    }(f)
}
{{</highlight>}}

In this case, we wrap the function inside the **if** statement, creating a temporary file and pass the **os.File** pointer "f" to the function as **io.Writer**. Everything inside the function stays exactly the same as before.

Now run this code again with "12345" as input, you would have the output written to the file found at "/tmp/output.txt".

### Network I/O

Another breed of I/O is network I/O. While networking may be too big a topic to cover here, we attempt to shed some lights on how to perform basic network operations using Go.

Althought web programming has been getting a lot of attentions these days, it is not the only form of network programming. Network programming fundamentally involves transfer of bits and bytes over the network. Although it is possible to use the web protocol, or HTTP, for all network communications, most performance critical network applications use "binary protocol".

A binary protocol is one where data is not represented in plain-text format, like JSON. A binary protocol does not provide privacy; it simply isn't represented as human readable string.

Go natively supports a binary protocol package called **encoding/gob**. The GOB Package contains encoding and decoding methods, where encoding prepares for data to be sent to the network, while decoding interprets data received from the network:

    struct -> gob.Encode -> to network
    from network -> gob.Decoder -> struct

With GOB, encoding and decoding refers to the data in the network, or the "on-the-wire" data. Therefore, encoding converts an object into data ready for transmission; while decoding converts the data read from the newtork to a data structure.

Like JSON, the GOB encoder/decoder supports utility functions **NewEncoder(io.Writer)** and **NewDecoder(io.Reader)**. This hides the details of how bits and bytes appear on the wire.

To demonstrate network I/O, we need to first introduce the "net" package. This package contains the socket API in Go.

A simple server application can be created using the following code snippet:

{{<highlight go>}}
func main() {
    port := "9999"
    ln, err := net.Listen("tcp", "127.0.0.1:"+port)
    if err != nil {
        return
    }
    defer ln.Close()
    for {
        conn, err := ln.Accept()
        if err != nil {
            go handleConnection(conn)
        }
    }
}
{{</highlight>}}

There are a few things to note here:

1. The single **net.Listen()** function call encapsulates the typical C's **socket()**, **bind()** and **listen()** functions.
* **net.Listen()** always use a default backlog size equal to system maximum.
* Socket type (TCP or UDP), IP address and ports are all passed in as **string**.

This simplifies things quite a bit, but it also raises a few concerns about lack of fine-tuning necessary on more complex network applications.

Fortunately for the more advanced network programmers, the native socket API's are still available via **syscall** package. This package contains a lot of low level API's, including the socket API's. Some of the examples in the **syscall** package are **syscall.Socket()**, **syscall.Bind()**, **syscall.Setsockopt**, etc. This obviously gives you complete control of the socket, but at the same time make things a bit more complicated.

Back to our server application. The application listens on the localhost port 9999, accept the connection and starts a goroutine to handle the connection. Since the data comes from the network, we will need to **decode** it using GOB decoder. This is what the function **handleConnection()** looks like:

{{<highlight go>}}
func handleConnection(conn net.Conn) {
    defer conn.Close()
    var m MyInput
    if err := gob.NewDecoder(conn).Decode(&m); err != nil {
        if err.Error() != "EOF" {
            fmt.Printf("error decoding from %s, err=%s\n",
                conn.RemoteAddr().String(), err.Error())
            return
        }
    }
    fmt.Printf("input = %s\ntime = %s\n", m.UserInput, m.Timestampe.String())
}
{{</highlight>}}

Here we are reusing the data struct **MyInput**, which contains a **UserInput** and **Timestamp**. To decode from the wire, we pass a **net.Conn** to a GOB decoder, and decode the data into the object **m**.

Now let's take a look at how to create the client application. The client takes input from the console, connect to the server and transfer the input using GOB encoder. The client code is shown below:

{{<highlight go>}}
func main() {
    var wg sync.WaitGroup
    r, w := io.Pipe()

    port := "9999"
    conn, err := net.Dial("tcp", "127.0.0.1:"+port)
    if err != nil {
        return
    }
    defer conn.Close()

    go func(w io.Writer) {
        wg.Add(1)
        defer wg.Done()
        defer r.Close()

        b, _ := ioutil.ReadAll(r)
    
        m := &MyInput{UserInput: string(b), Timestamp: time.Now()}
        gob.NewEncoder(w).Encode(m)
    }(conn)

    io.Copy(w, os.Stdin)
    w.Close()

    wg.Wait()
}
{{</highlight>}}

The **net.Dial** function replaces the **socket()** and **connect()** function in C, and just like **net.Listen()**, the socket type, destination address and port are all passed in as string. Once the connection is established, the rest of the code looks strangely similar to the code above when we were encoding the input into JSON format. Whether you are working on console I/O, file I/O, or network I/O, once the endpoint is established, they all support **io.Reader** and **io.Writer** interfaces, which makes the rest of the code around it consistent.

Now if you run the client application with a single line input:

    test string<ctrl-d>

You should be able to see the input string reflected on the server:

    input = test string
    time = 2016-12-15 00:09:54.727447929 +0000 UTC

