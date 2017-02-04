+++
date = "2016-07-12T05:41:35Z"
draft = false
title = "Testing Go"

+++

Every good code requires testing, lots of it. Go handily provides a **testing** package that provisions most of the testing needs. To use the **testing** framework, simply import **testing** in your source code:

{{<highlight go>}}
import(
	"testing"
)
{{</highlight>}}

The **testing** framework consists of 2 parts: **testing** and **benchmarking**. Testing validates the functional correctness of your code; while benchmarking measures the performance.

A test suite is automatically included with any file that ends with **_test.go**. These files are run with **go test** command, and are not built when **go build** or **go run** are used. Each test function is prefixed by the word **Test**, and accept a single parameter "*testing.T"; a benchmark function is prefixed by **Benchmark** and accept the parameter "*testing.B".

For example, we have written a function **foo()** which needs to be validated both functionally and performance-wise. This function is defined in package **foo**, where file **foo.go** defines it, and **foo_test.go** tests it. The code are shown here:

**foo.go**:
{{<highlight go >}}
package foo

func foo() bool {
	return true
}
{{</highlight>}}

**foo_test.go**:
{{<highlight go >}}
package foo

func TestFoo(t *testing.T) {
	if b := foo(); b != true {
		t.Error("test", "foo",
			"expect", true,
			"got", b)	
	}
}

func BenchmarkFoo(b *testing.B) {
	for i := 0; i < b.N; i++ {
		foo()
	}
}
{{</highlight>}}

The test suite contains 2 functions: a test function **TestFoo()**, and a benchmark function **BenchmarkFoo()**. To invoke the test function **TestFoo()**, you may use the **go test** command:

	# go test
	PASS
	ok  	_/vagrant/src/test	0.004s

Invoking testing framework without any argument runs all the tests in the current directory, which in this case is just **TestFoo()**. 

To get more detailed information about the tests, use the "-v" argument:

	# go test -v
	=== RUN   TestFoo
	--- PASS: TestFoo (0.00s)
	PASS
	ok  	_/vagrant/src/test	0.004s

Perhaps it is more interesting to show a case where a test fails. If the condition is modified to test for **b != false**, which always fails, we can see that the test will print out the error, by concatenating each items in the **t.Error()** function arguments:

{{<highlight go >}}
func TestFoo(t *testing.T) {
	if b := foo(); b != false {
		t.Error("test", "foo",
			"expect", false,
			"got", b)
		}
	}
}
{{</highlight>}}

In this case, since function **foo()** always return **true**, this test case will fail, resulting in the following output:

	# go test -v
	--- FAIL: TestFoo (0.00s)
	foo_test.go:22: testing foo expect false got true

With more complicated tests, you may want to give meaningful names to certain tests. Consider the following test function:

{{<highlight go>}}
func TestFoo(t *testing.T) {
	if b := foo(); b != true {
		t.Error("test", "foo",
			"expect", true,
			"got", b)
		}
	}
	
	t.Run("FailMe",
		func(t *testing.T) {
			if b:= foo(); b != false {
				t.Error("test", "foo",
					"expect", false,
					"got", b)
			}
		})
}
{{</highlight>}}

The first part of the test is an anonymous test; while the second test has a name **FailMe**. This is called a **subtest**, whose full name is **TestFoo/FailMe**, the concatenation of its parent and itself, joined using a slash. If we invoke this test now, it will show the following output:

	# go test -v
	=== RUN   TestFoo/FailMe
	--- FAIL: TestFoo (0.00s)
    --- FAIL: TestFoo/FailMe (0.00s)
    	foo_test.go:19: testing foo expect false got true

The subtest can also be selectively run using the "-run" argument:

	# go test -v -run TestFoo/FailMe

or with regular expression:

	# go test -v -run TestFoo/*

### Benchmark ###

We have talked about the testing package supports both testing and benchmarking. Benchmarking a function requires running the function N times and take the average time taken to run that function. The **testing.B** benchmark data structure contains a big N, which is automatically adjusted so that the benchmark can produce meaningful result. That's why the benchmarked function is wrapper inside a for loop like this:

{{<highlight go>}}
func BenchmarkFoo(b *testing.B) {
	for i := 0; i < b.N; i++ {
		foo()
	}
}
{{</highlight>}} 

To run the benchmark function, use the "-bench" flag:

	# go test -bench BenchmarkFoo
	BenchmarkFoo   	2000000000	         0.31 ns/op

Benchmarks are only run after all the tests have been successfully passed. Therefore, this benchmark will not run even with the "-bench" flag with the **FailMe** test in place.

The benchmark is basically a stop watch whose timer starts at the function execution. An expensive operation inside the benchmark function will screw up the timer so bad that the benchmark tool will generate insane results like this:

{{<highlight go>}}
func BenchmarkFoo(b *testing.B) {
	time.Sleep(1000 * time.Millisecond)
	for i := 0; i < b.N; i++ {
		foo()
	}
}
{{</highlight>}}

Here we sleep for 1 second and run function **foo()** N times. As the function **foo()** is significantly shorter than 1 second, this messes up the timer pretty bad, and the result is way off:

	# go test -bench BenchmarkFoo
	BenchmarkFoo-2   	       1	1000224631 ns/op

To get around this problem, we can reset the timer right before the for loop:

{{<highlight go>}}
func BenchmarkFoo(b *testing.B) {
	time.Sleep(1000 * time.Millisecond)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		foo()
	}
}
{{</highlight>}}

This way, the benchmark no longer includes the time spent in sleep, and therefore accurately measures the performance of function **foo()**.

Like testing, we can create named sub-benchmark using **b.Run()**. The syntax is identical to the testing counterpart **t.Run()**, and is therefore omitted.

In multi-core programming, it is often desirable to measure performance of parallel execution in the context of number of CPU cores. With Go's testing framework, this can be achieved with the built-in **RunParallel()** benchmark facility:

{{<highlight go>}}
func BenchmarkFoo(b *testing.B) {
	b.RunParallel(
		func(pb *testing.PB) {
			for pb.Next() {
				foo()
			}
		})
}
{{</highlight>}}

Note that with parallel benchmark, the **b.N** loop is replaced by **pb.Next()**, which return the number of iterations left.

Using the **testing.PB**, or parallel benchmark framework, and by passing the "-cpu" flag, we can set it up to run benchmark of parallel execution of function **foo()** in the environment where 1 or 2 CPU cores are active:

	# go test --bench Foo -cpu 1,2
	BenchmarkFoo     	1000000000	         2.21 ns/op
	BenchmarkFoo-2   	2000000000	         1.06 ns/op

This runs the benchmark twice with 1 and 2 CPU's, indicative by the "-2" postfix on the benchmark name. In this case, since **foo()** does not have any lock contention, we expect that performance should double when number of CPU's is doubled. This is verified using the parallel benchmark, where 2 CPU runs 1.06 ns/op vs 1 CPU runs 2.21 ns/op.
