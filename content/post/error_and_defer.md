+++
date = "2016-10-03T06:42:50Z"
title = "Error Handling and Defer"

+++

With C, it has always been a struggle when it comes to error handling. Most C API follows the rule of thumb that a **0** value represent success, and a negative value represents failure. However, this rule is usually not true with a pointer return type, where a **NULL** pointer can be used to represent failure. The result is an ad-hoc use of return values.

Although you are still free to use any return value to indicate failure or success, Go recommends using **error interface** to deal with errors.

### Error Interface

The built-in interface type **error** is the convention used in Go library to represent error value. The interface is defined as followed:

{{<highlight go>}}
type error interface {
	Error() string
}
{{</highlight>}}

Being an interface type, an error value can be **nil** to indicate success. A typical Go API looks like this:

{{<highlight go>}}
ln, err := net.Listen("tcp", ":8080")
if err != nil {
	fmt.Printf("listen failed: %s\n", err.Error())
	return
}
{{</highlight>}}

This code uses the **net** package that provides networking capabilities. The function **net.Listen** create a server socket that listens on TCP port 8080. The error handling checks if the API returns any non-nil value, prints out the error string and returns.

### Defer

Consider that you want to open a file and write some data. The bulk of the code will be on the part that writes data. The file needs to be closed at the end, otherwise the file descriptor is leaked.

The corresponding C code looks something like the following:

{{<highlight c>}}
/* C example to write buffer to file */
int write_to_file(const char *filename, char *buf, int len)
{
	FILE *fd = NULL;

	fd = fopen(filename, "w+");
	if (fd == NULL) {
		return -1;
	}

	if (len != fwrite(buf, 1, len, fd)) {
		fclose(fd);
		return -1;
	}

	fclose(fd);
	return 0;
}
{{</highlight>}}

This code has redundant **fclose()** in 2 spots: when the function is successfully executed, and when **fwrite()** failed. One way to reduce this redundancy is to use a **goto** block and check for **fd** being NULL:

{{<highlight c>}}
/* C example to write buffer to file 
 * error handling with "goto"
 */
int write_to_file(const char *filename, char *buf, int len)
{
	FILE *fd = NULL;
	int rc = 0;

	fd = fopen(filename, "w+");
	if (fd == NULL) {
		rc = -1;
		goto error;
	}
	
	if (len != fwrite(buf, 1, len, fd)) {
		rc = -2;
		goto error:
	}

error:
	if (fd != NULL)
		fclose(fd);
	return rc;
}
{{</highlight>}}

This gets messy when there are more than one thing to clean up. With C++, this can be done using exceptions. In this case, explicit **try** block is used to indicate the main logic, and **catch** blocks to catch exceptions.

Go takes a different approach called **defer**, which feels more like a smart pointer. The keyword **defer** pushes the execution of the expression to the end of the current function. Go's way to handle closing a file descriptor is the following:

{{<highlight go>}}
func WriteToFile(filename string, s string) (written int, err error) {
	file, err := os.OpenFile(filename, os.O_WRONLY|os.O_CREATE, 0777)
	if err != nil {
		return 0, err
	}
	defer file.Close() // Close() is called upon exiting

	n, err := file.WriteString(s)
	if err != nil {
		return 0, err	// no need to Close() here
	}

	return n, nil
}
{{</highlight>}}

Notice that **file.Close()** is only required once, and at a location immediately following the call to **os.OpenFile()**, a location that is more intuitive and convenient than further down in the function block.

There may be multiple **defer** calls within a single function. These **defer** calls are put on a local stack, and popped when function exits. All the defer calls are executed **first-in-last-out**:

{{<highlight go>}}
func DeferTest() {
	defer fmt.Printf("1\n")
	defer fmt.Printf("2\n")
	defer fmt.Printf("3\n")
}
{{</highlight>}}

Running the **DeferTest()** function will then result in the following output:

	3
	2
	1

The variables are evaluated when the deferred expression is pushed to the stack, not when they are executed when function exits. In the following code, the variable **i** is 0 when the expression is deferred, and incremented before function exits:

{{<highlight go>}}
func DeferTest() {
	i := 0
	defer fmt.Printf("deferred i = %d\n", i) // i is evaluted here, which is 0
	i++
	fmt.Printf("i = %d\n", i)
}
{{</highlight>}}

The result is that the second **fmt.Printf**, which executes first, will have a higher **i** value than the first one, which executes when **DeferTest()** function exits. This code results in the following output:

	i = 1
	deferred i = 0
