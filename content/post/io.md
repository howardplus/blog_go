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

In Go, the built-in **copy()** function directly runs on **io.Reader** and **io.Writer** interfaces, so that piping data from standard input to output can be succinctly done as follows:

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

This blocks the main thread until an EOF has been reached. On a unix environment, you can press "ctrl-D" to simulate EOF from the console.

Go's generic buffer struct **bytes.Buffer** conveniently implements the **io.Reader** and **io.Writer** interfaces, so you can use that as a temporary buffer:

{{<highlight go>}}
// example 2: copy from stdin to stdout using a temporary buffer
func main() {
    var b bytes.Buffer
    io.Copy(&b, os.Stdin) // feed stdin to buffer
    io.Copy(os.Stdout, &b) // send from buffer to stdout
}
{{</highlight>}}

Since all the data are written to the temporary buffer, your input will not echo on the screen. It is only when you enter "Ctrl-D", or EOF, that all the inputs will be displayed on the screen all at once.

This is slightly different in the way that input is **buffered** into a temporary space. If you want to echo input to the console, this code won't work.

Go also implements an in-memory pipe: **io.Pipe()**. What comes into the pipe must come out exactly as it was. We can reimplement the copy this way:

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

This is a complicated way to implement a simple task. But it enables us to insert a function that operates on the data between input and output.

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

To encode the object "u" into JSON format, and send the output to an **io.Writer** "w", you can use the following code snippet:

{{<highlight go>}}
u := MyInput{UserInput: "some string", Timestamp: time.Now()}
json.NewEncoder(w).Encode(u)
{{</highlight>}}

The first line simply creates an object of "MyInput", with some arbitrary text. The second line creates a JSON encoder object using the function "NewEncoder()", specify the output stream, and encode the data from the object "u".

Remember our goal is to convert the console input, insert the data into the "UserInput" field of the JSON template. To do so, we need to read from the receiving end of the pipe. To read from the receiving end, which is an **io.Reader**, we can either loop until EOF, or use the **ioutil**'s **ReadAll** function:

{{<highlight go>}}
b, _ := ioutil.ReadAll(r) // "r" is the read end of the pipe
{{</highlight>}}

This utility function reads all the way until EOF, and returns a byte slice with error. Let's ignore the error for now. The output "b" now contains the byte slice of all the data from the pipe.

Notice that a pipe does not automatically send the EOF from the console to the pipe; to send EOF on the pipe, you need to explicitly close the pipe on the write end.

Now we have everything we need, let's rewrite the code to encode the console input, and write the JSON formatted output to the console output:

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

There it is. We used a pipe to send data from standard input to output with extra processing logic.

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

In network applications where bandwidth is a critical resource, you may want to consider using binary protocol. A binary protocol is one where data is not represented in plain-text format, like JSON.

Go supports a native binary protocol package called **encoding/gob**. The GOB Package contains encoding and decoding methods, where encoding prepares for data to be sent to the network, while decoding interprets data received from the network:

    struct -> gob.Encode(r io.Reader) -> to network
    from network -> gob.Decode(w io.Writer) -> struct
