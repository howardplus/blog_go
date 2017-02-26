+++
date = "2017-02-22T23:32:26-08:00"
title = "REST Server" 
+++

Let's look at how to build a network application based on what we have learnt. The application we are building here is a simple REST server. REST, or RESTful API, is the most widely adopted network communication mechanism. For more information on REST, just [google it](http://lmgtfy.com/?q=REST).

### HTTP and Handler

At the core of a REST server is a HTTP server. A traditional HTTP server outputs to a web browser;a REST server returns data rather than web pages. These "data" can be presented in many forms, but typically in JSON format.

Before we start with REST server, let's see how to build a simple HTTP server using built-in "net/http" package. This can be succinctly achieved via a single line of code:

{{<highlight go>}}
log.Fatal(http.ListenAndServe(":8000", http.FileServer(http.Dir("./"))))
{{</highlight>}}

What this does is create a HTTP server listening on port 8000, and serve any file out of the current directory "./". If a special file "index.html" is present on the current path, this file is served, otherwise the server returns the directorying listing. If the file path contains a specific file, that file is served. 

Let's dig into some details here. First, we use the "log" package for debug logging. Using the **log.Fatal()** function prints an error message via **Error** interface and exit the program. Since **ListenAndServe()** does not return on success, there is nothing to print, and program continues.

Now if you try listening on port 80 instead of port 8000:

{{<highlight go>}}
log.Fatal(http.ListenAndServe(":80", http.FileServer(http.Dir("./"))))
{{</highlight>}}

Since only priviledged user can listen on priviledged port in the range 1 to 1023, this typically fails for a normal user:

    2017/02/24 18:44:36 listen tcp :80: bind: permission denied
    exit status 1

The main HTTP server entry function is **ListenAndServe()**. This function creates a listener and starts the HTTP server. The first parameter to **ListenAndServe()** is a string that contains the host name and port. When the host name or IP address is omitted, as it is in our example, **ListenAndServe()** will listen on localhost. This means that the web server is not accessible outside of the same machine it is running on.

The second parameter is **http.Handler**, an interface type that defines the following function:

{{<highlight go>}}
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
{{</highlight>}}

The **Request** is the input, while the **ResponseWriter** is the output. This handler, when given, is the function that gets run every time an HTTP request is received by the server. The **ResponseWriter** implements **io.Writer**, therefore you may use the function **Write()** to output data. In this case, we are using the built-in **http.FileServer()** handler, which turns a directory into a HTTP handler.

If a **nil** handler is given to **http.ListenAndServe()**, the http server uses the default handler defined using **http.HandleFunc()**. For a simple example, let's create 2 handlers that either prints "Hello World\n" or "Hi There\n" to the output, and assign to the the HTTP server via different URL path: "/hello" and "/hi":

{{<highlight go>}}
func HelloWorldHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello World\n))
}

func HiThereHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hi There\n))
}

http.HandleFunc("/hello", HelloWorldHandler)
http.HandleFunc("/hi", HiThereHandler)
log.Fatal(http.ListenAndServe(":8000", nil))
{{</highlight>}}

This way, the HTTP server can serve URL "/hello" and "/hi", which uses **HelloWorldHandler()** and **HiThereHandler()** respectively.

### Gorilla Mux REST Framework

It may seem like we can easily extend the built-in HTTP framework to create a REST server. All you need to do is to properly assign paths to a specific handler.

Well, yes and no.

The HTTP framework does not have explicit support for 2 features required by any serious REST server:

1. Parameter encoded as URL
2. Resource hierarchy 

With REST, the resource you are requesting is typically encoded in the URL. For example, if a REST server is serving student database, and you want query a student by student ID number, the REST API may look like this:

    GET /student/{id}/

The curly bracket is typically how REST defines a variable. This path is no longer static, and typically does not correspond to a file path on the system. And the ID field may only accept numeric values up to 10 characters. We need additional parsing and validation to make it work using the built-in HTTP framework.

Secondly, the URL of a REST server typically has semantic meanings. They are not file path, but rather correspond to the way resource are organized. Again, additional parsing and validation on the URL are required to make it work cleanly.

And that's where Gorilla Mux framework comes in handy. It's a small framework that builds on top of the built-in HTTP server to provide additiona URL parsing and validation, with support for resource hierarchy.

At the heart of the Gorilla Mux framework is the **Router**. The **Router** is a **http.Handler** that can take REST style URL specification. Using the previous student ID example, we can create the following router:

{{<highlight go "linenos=inline">}}
import "github.com/gorilla/mux"

r := mux.NewRouter()
s := r.PathPrefix("/student").Subrouter()
s.HandleFunc("/{id:[0-9]{10}}", StudentsIDHandler)

log.Fatal(http.ListenAndServe(":8000", r))
{{</highlight>}}

On line 3, we create the root router. On line 4, a "/student" subsection is created to handle everything under this category. In this case, we accept a student ID number, which contains exactly 10 digits. The variable name is "id"; what comes after the colon is a regular expression to validate that URL. Finally, we pass the router as a handler to the HTTP server.

### REST Message Server

Now we have armed ourselves with the basic knowledge of HTTP server and Gorilla Mux, we are ready to build a REST server.

The first step toward building a REST application is to define what it does. From the previous post, we have used GOB to receive an user input string, and present it with timestamp on the screen. In this post, we will make our application take inputs from the user via REST API, and allows user to retrieve the last N messages from the server.

Let's define 2 REST methods:

**POST** /msg/
: submit a user input to the server

**GET** /msg/{n}/
: get the last n user inputs stored on the server

First, we create a router. The number of messages to be presented to the user is encoded in the "GET" method right after "/msg/". The following code does just that:

{{<highlight go>}}
type Message struct {
    UserInput string
    Timestamp time.Time
}

var Messages []*Message

func main() {

    // initialize the message array
    Messages = make([]*Message, 0)

    // set up router
    r := mux.NewRouter()
    s := r.PathPrefix("/msg").Subrouter()
    s.HandleFunc("/", MsgPostHandler).Methods("POST")
    s.HandleFunc("/{n:[0-9]+}/", MsgGetHandler).Methods("GET")

    // starts server
    log.Fatal(http.ListenAndServe(":8000", r))
}
{{</highlight>}}

The **Message** structure is same as the previous post, which encapsulates the user input string and the time stamp. We use a global **Messages** to store any message received from a user. Notice that this memory will grow indefinitely, which is not a good thing to allow happening in a real server. But for a simple example, let's ignore that problem for now.

The second part sets up the router so that we can process a "POST" method that takes a message as input, and a "GET" method to return number of messages specified by the user. All we need to do is to create the handlers. First, let's look the **MsgPostHandler()**:

{{<highlight go>}}
func MsgPostHandler(w http.ResponseWriter, r *http.Request) {
    b, _ := ioutil.ReadAll(r.Body)
    Messages = append(Messages, &Message{UserInput: string(b), Timestamp: time.Now()})
    log.Println("Post message: ", string(b))
}
{{</highlight>}}

When a "POST" is received, the HTTP body is taken verbatim as the user input. This input is stored in the **Messages** slice, and we print out an informational log.

The second handler **MsgGetHandler()** outputs messages in JSON format:

{{<highlight go>}}
func MsgGetHandler(w http.ResponseWriter, r *http.Request) {
    n, err := strconv.Atoi(mux.Vars(r)["n"])
    if err != nil {
        log.Fatal("unexpected value")
    }

    log.Println("Get message: ", n)

    // get last n message, or as many as possible
    m := make([]*Message, 0)
    if n <= len(Messages) {
        m = Messages[len(Messages)-n:]
    } else {
        m = Messages
    }
    json.NewEncoder(w).Encode(m)
}
{{</highlight>}}

First, the parameter **n** is retrieved and converted from **string** to **int** using the "strconv" package. The error check is a fatal error because it should not happen. Gorilla Mux only calls the handler when the parameter matches the regular expression. The only case **err** is non-nil is when the regular expression is wrong, in which case the the fatal error is appropriate.

Secondly, we get a copy of all the messages and retrieve up to "n" messages. These message, represented as a slice of **Message**, can be encoded using the "encoding/json" package. Since **http.ResponseWriter** implements **io.Writer**, it can be passed to **json.NewEncoder()** function.

That's all there is. You may test your REST server using "curl", a command line HTTP client available on Linux compatible machines:

    $curl -X POST http://127.0.0.1:8000/msg/ -d "This is the first message"
    $curl -X POST http://127.0.0.1:8000/msg/ -d "This is the second message"
    $curl -X POST http://127.0.0.1:8000/msg/ -d "This is the third message"

Using "curl" with "POST" method stores the messages into the server. If you now issue the "GET" method, you get a JSON array of all the messages with their respective time stamps:

    $curl -X GET http://127.0.0.1:8000/msg/10/
    [{"UserInput":"This is the first message","Timestamp":"2017-02-26T07:23:36.252603569Z"},
     {"UserInput":"This is the second message","Timestamp":"2017-02-26T07:23:41.085334108Z"},
     {"UserInput":"This is the third message","Timestamp":"2017-02-26T07:23:47.431041519Z"}]

Since there are only 3 messages to show, our server returns 3 items instead of 10 being requested. On the other hand, if we request just 1 message, only the last (third) message will be returned:
   
    $curl -X GET http://127.0.0.1:8000/msg/1/
    [{"UserInput":"This is the third message","Timestamp":"2017-02-26T07:23:47.431041519Z"}] 
