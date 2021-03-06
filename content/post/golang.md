---
date: "2019-09-01"
title: "Golang"
author : "Nanik Tolaram (nanikjava@gmail.com)" 
---
Following are some of the thing that need to be remembered when programming in Golang. This information is taken from variety of sources.

--------
Variables
---------

*  For variable naming, omit words that are obvious given a variable's type declaration or otherwise clear from the surrounding context. Following is not a good example
{{< highlight go >}}
returnSlice := make([]string, len(src))
{{< /highlight >}}
* Variable names in Go should be short rather than long. This is especially true for local variables with limited scope. Prefer c to lineCount. Prefer i to sliceIndex.
* The basic rule: the further from its declaration that a name is used, the more descriptive the name must be. For a method receiver, one or two letters is sufficient. Common variables such as loop indices and readers can be a single letter (i, r). More unusual things and global variables need more descriptive names.
* For example, "URL" should appear as "URL" or "url" (as in "urlPony", or "URLPony"), never as "Url". ServeHTTP not ServeHttp. 
* For identifiers with multiple initialized "words", use for example "xmlHTTPRequest" or "XMLHTTPRequest".
* Have a reasonable number of parameters and reasonably short variable names.
* Having one doubled up field in the struct makes the test data difficult to scan. Following is not a good example : 
{{< highlight go >}}
...
prefix, postfix string
{{< /highlight >}}
* Declare empty slices as
{{< highlight go >}}
var t []string (declares a nil slice value) 
{{< /highlight >}}
and not
{{< highlight go >}}
t := []string{} (is non-nil but zero-length)            
{{< /highlight >}}


--------
Interfaces & Receivers
--------
* The following snippets shows how to do 'force checking' of contact between **type interface** and **type struct**. 
{{< highlight go >}}
type ValueConverter interface {
ConvertValue(v interface{}) (Value, error)
}

type boolType struct{}

var Bool boolType

var _ ValueConverter = boolType{}  --> (a)

func (boolType) String() string { return "Bool" }

func (boolType) ConvertValue(src interface{}) (Value, error) {....}
{{< /highlight >}}

the above provides a static (compile time) check that boolType satisfies the ValueConverter interface. The _ used as a name of the variable tells the compiler to effectively discard the RHS value, but to type-check it and evaluate it if it has any side effects, but the anonymous variable per se doesn't take any process space.

It is a handy construct when developing and the method set of an interface and/or the methods implemented by a type are frequently changed. The construct serves as a guard against forgetting to match the method sets of a type and of an interface where the intent is to have them compatible. It effectively prevents to go install a broken (intermediate) version with such omission.

Another example (taken from __https://github.com/Kong/kuma/tree/master/pkg/core/discovery__) where the _sink.go_ enforce the contract inside the _interfaces.go_

**sink.go**
{{< highlight go >}}
    package discovery

    import (
        mesh_core "github.com/Kong/kuma/pkg/core/resources/apis/mesh"
        core_model "github.com/Kong/kuma/pkg/core/resources/model"
    )

    var _ DiscoverySource = &DiscoverySink{}
    var _ DiscoveryConsumer = &DiscoverySink{}

    // DiscoverySink is both a source and a consumer of discovery information.
    type DiscoverySink struct {
        DataplaneConsumer DataplaneDiscoveryConsumer
    }

    func (s *DiscoverySink) AddConsumer(consumer DiscoveryConsumer) {
        s.DataplaneConsumer = consumer
    }

    func (s *DiscoverySink) OnDataplaneUpdate(dataplane *mesh_core.DataplaneResource) error {
        if s.DataplaneConsumer != nil {
            return s.DataplaneConsumer.OnDataplaneUpdate(dataplane)
        }
        return nil
    }
    func (s *DiscoverySink) OnDataplaneDelete(key core_model.ResourceKey) error {
        if s.DataplaneConsumer != nil {
            return s.DataplaneConsumer.OnDataplaneDelete(key)
        }
        return nil
    }
{{< /highlight >}}


**interfaces.go**
{{< highlight go >}}
    package discovery

    import (
        discovery_proto "github.com/Kong/kuma/api/discovery/v1alpha1"
        mesh_core "github.com/Kong/kuma/pkg/core/resources/apis/mesh"
        "github.com/Kong/kuma/pkg/core/resources/model"
    )

    // DiscoverySource is a source of discovery information, i.e. Services and Workloads.
    type DiscoverySource interface {
        AddConsumer(DiscoveryConsumer)
    }

    // ServiceInfo combines original proto object with auxiliary information.
    type ServiceInfo struct {
        Service *discovery_proto.Service
    }

    // WorkloadInfo combines original proto object with auxiliary information.
    type WorkloadInfo struct {
        Workload *discovery_proto.Workload
        Desc     *WorkloadDescription
    }

    // WorkloadDescription represents auxiliary information about a Workload.
    type WorkloadDescription struct {
        Version   string
        Endpoints []WorkloadEndpoint
    }

    type WorkloadEndpoint struct {
        Address string
        Port    uint32
    }

    // DiscoveryConsumer is a consumer of discovery information, i.e. Services and Workloads.
    type DiscoveryConsumer interface {
        DataplaneDiscoveryConsumer
    }

    type DataplaneDiscoveryConsumer interface {
        OnDataplaneUpdate(*mesh_core.DataplaneResource) error
        OnDataplaneDelete(model.ResourceKey) error
    }
{{< /highlight >}}

* Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring.
* Design the API so that it can be tested using the public API of the real implementation.
* Do not define interfaces before they are used: without a realistic example of usage, it is too difficult to see whether an interface is even necessary, let alone what methods it ought to contain.
* Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. 
{{< highlight go >}}
    package consumer  // consumer.go

    type Thinger interface { Thing() bool }

    func Foo(t Thinger) string { … }
{{< /highlight >}}
{{< highlight go >}}
    package consumer // consumer_test.go

    type fakeThinger struct{ … }
    func (t fakeThinger) Thing() bool { … }
    …
    if Foo(fakeThinger{…}) == "x" { … }
{{< /highlight >}}
    Instead of doing this
{{< highlight go >}}
    // DO NOT DO IT!!!
    package producer

    type Thinger interface { Thing() bool }

    type defaultThinger struct{ … }
    func (t defaultThinger) Thing() bool { … }

    func NewThinger() Thinger { return defaultThinger{ … } }
{{< /highlight >}}
	...better do this
{{< highlight go >}}  
    package producer

    type Thinger struct{ … }
    func (t Thinger) Thing() bool { … }

    func NewThinger() Thinger { return Thinger{ … } }
{{< /highlight >}}

* In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.
    * If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines:
        * _**NO POINTER**_ for the following:
            - If the receiver is a map, func or chan, don't use a pointer to them. If the receiver is a slice and the method doesn't reslice or reallocate the slice, don't use a pointer to it.
            - If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense. A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can't always succeed.) Don't choose a value receiver type for this reason without profiling first.            
        * _**USE POINTER**_ for the following:
            * If the method needs to mutate the receiver, the receiver must be a pointer.
            * If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying.
            * If the receiver is a large struct or array, a pointer receiver is more efficient. How large is large? Assume it's equivalent to passing all its elements as arguments to the method. If that feels too large, it's also too large for the receiver.
            * Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer.
            * If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention more clear to the reader.
            * Finally, when in doubt, use a pointer receiver.


* Compile-time checking
{{< highlight go >}}  
// Compile-time check to assert this config matches requirements.
var _ setup.KeyManagerProvider = (*Config)(nil)
var _ setup.BlobStorageConfigProvider = (*Config)(nil)
var _ setup.DBConfigProvider = (*Config)(nil)
{{< /highlight >}}
the above is to check whether the Config object implement the function defined in KeyManagerProvider, BlobStorageConfigProvider and DBConfigProvider.

	The (*Config)(nil) is to do the verification between the interface and struct WITHOUT allocating any memory. See below for more examples

* Compile-time checking examples
{{< highlight go >}}  
package main

type I interface{ M() }
type T struct{}

func (T) M() {}

//func (*T) M() {} //var _ I = T{}: T does not implement I (M method has pointer receiver)

func main() {
  //avoids allocation of memory
  var _ I = T{}       // Verify that T implements I.
  var _ I = (*T)(nil) // Verify that *T implements I.
  //allocation of memory
  var _ I = &T{}      // Verify that &T implements I.
  var _ I = new(T)    // Verify that new(T) implements I.
}
{{< /highlight >}}

* Compile-time checking examples
{{< highlight go >}}  
var (
        _ blobref.StreamingFetcher = (*CachingFetcher)(nil)
        _ blobref.SeekFetcher      = (*CachingFetcher)(nil)
        _ blobref.StreamingFetcher = (*DiskCache)(nil)
        _ blobref.SeekFetcher      = (*DiskCache)(nil)
)
{{< /highlight >}}
Ensure compiler checks:
	* That CachingFether implements public functions of StreamingFetcher and SeekFetcher. 
	* RHS portion uses a pointer constructor syntax with a nil parameter. What does this syntax mean in Go language ?

* Type Casting for checking. The following is based on the example from **github.com/google/exposure-notifications-server**. The **KeyManagerProvider** is an interface as follows (found inside **setup.go**)
{{< highlight go >}}  
	type KeyManagerProvider interface {
		KeyManager() bool
	}
	{{< /highlight >}}
**Config** object is defined as follows (found inside **config.go**)
{{< highlight go >}}  
	type Config struct {
		Database          *database.Config
		...
		...
		WorkerTimeout     time.Duration `envconfig:"WORKER_TIMEOUT" default:"5m"`
		...
	}

	func (c *Config) KeyManager() bool {
		return false
	}
{{< /highlight >}}
the struct implement the **KeyManager()** function. The **setup.Setup(..)** function is defined as follows
{{< highlight go >}}  
	func Setup(ctx context.Context, config DBConfigProvider) (*serverenv.ServerEnv, Defer, error) {
		...
		...
	}
{{< /highlight >}}
it is called as follows from export/main.go
{{< highlight go >}}  
	var config export.Config
	env, closer, err := setup.Setup(ctx, &config)
{{< /highlight >}}
the following code inside Setup(..) will be analysed (found inside setup.go)
{{< highlight go >}}  
	...
	if _, ok := config.(KeyManagerProvider); ok {
		...
	}
	...
{{< /highlight >}}
is actually a way to check whether the passed config parameter implemented the function **'KeyManager()'**. 

-------
Testing
-------

* Tests should fail with helpful messages saying what was wrong, with what inputs, what was actually got, and what was expected.
{{< highlight go >}}  
if got != tt.want {
    t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
{{< /highlight >}}

--------
Comments
--------

* For comments use // instead of /* */
* Start comment with the name of the function 
* All top-level, exported names should have doc comments, as should non-trivial unexported type or function declarations.
* For readability, move the comment to appear on it's own line before the variable. Following is NOT a good example
{{< highlight go >}} 
...
OutPrefix   = "> " // OutPrefix notes output
{{< /highlight >}}
* Package comments, like all comments to be presented by godoc, must appear adjacent to the package clause, with no blank line.
{{< highlight go >}} 
// Package math provides basic constants and mathematical functions.
package math
{{< /highlight >}}
{{< highlight go >}} 
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
{{< /highlight >}}

* For "package main" comments, other styles of comment are fine after the binary name. For example, for a package main in the directory seedgen you could write:
{{< highlight go >}} 
// Binary seedgen ...
package main

// Command seedgen ...
package main

// Program seedgen ...
package main

// The seedgen command ...
package main

// The seedgen program ...
package main

// Seedgen ..
package main
{{< /highlight >}}
    
---------
Functions
---------
* If a function returns two or three parameters of the same type, or if the meaning of a result isn't clear from context, adding names may be useful in some contexts. 
* Don't name result parameters just to avoid declaring a var inside the function; that trades off a minor implementation brevity at the cost of unnecessary API verbosity.
* Naked returns are okay if the function is a handful of lines. Once it's a medium sized function, be explicit with your return values. Corollary: it's not worth it to name result parameters just because it enables you to use naked returns. Clarity of docs is always more important than saving a line or two in your function.
* Don't pass pointers as function arguments just to save a few bytes, however This advice does not apply to large structs, or even small structs that might grow.
    
       
--------------         
Error Handling
--------------

* Don't use panic for normal error handling. Use error and multiple return values.
* Error strings should not be capitalized (unless beginning with proper nouns or acronyms) or end with punctuation
* Do not discard errors using _ variables. Handle the error, return it, or, in truly exceptional situations, panic.
* Do not use return value such as in C using -1 to flag error. Function can return multiple result so use one of the returned result to flag if there is an error. Eg:
{{< highlight go >}} 
value, ok := Lookup(key)
if !ok  {
    return fmt.Errorf("no value for %q", key)
}
return Parse(value)
{{< /highlight >}}

* This is recommended
{{< highlight go >}} 
if err != nil {
    // error handling
    return // or continue, etc.
}
// normal code
{{< /highlight >}}
     instead of
{{< highlight go >}} 
if err != nil {
    // error handling
} else {
    // normal code
}
{{< /highlight >}}

--------
Packages
--------

* Avoid renaming imports except to avoid a name collision; good package names should not require renaming. In the event of collision, prefer to rename the most local or project-specific import.
* Packages that are imported only for their side effects (using the syntax import _ "pkg") should only be imported in the main package of a program, or in tests that require them.
* In this case, the test file cannot be in package foo because it uses bar/testutil, which imports foo. So we use the 'import .' form to let the file pretend to be part of package foo even though it is not. Except for this one case, do not use import . in your programs. It makes the programs much harder to read because it is unclear whether a name like Quux is a top-level identifier in the current package or in an imported package.
* Avoid meaningless package names like util, common, misc, api, types, and interfaces

-----------
Concurrency
-----------

* [This article](https://blog.gopheracademy.com/advent-2019/directional-channels/) explain in simple ways on how the directional nature of the channel works. The article details the different direction that a channel can have.

    {{< highlight go >}}
    // SliceIterChan returns each element of a slice on a channel for concurrent
    // consumption, closing the channel on completion
    func SliceIterChan(s []int) <-chan int {
        outChan := make(chan int)
        go func() {
            for i := range s {
                outChan <- s[i]
            }
            close(outChan)
        }()
        return outChan
    }
    {{< /highlight >}}

    The article explains

    > Diving into the implementation, the function creates a bidirectional channel for its own use, and then all it needs to do to ensure that it has full control over writing to and closing the channel is to return it, whereupon it will be converted into a read-only channel automatically.




* Following are excerpts from [this](https://github.com/golang/go/wiki/LearnConcurrency) and [this](https://golang.org/ref/spec#Send_statements)

    * Both the channel and the value expression are evaluated before communication begins. 
    * Communication blocks until the send can proceed. 
    * A send on an unbuffered channel can proceed if a receiver is ready. 
    * A send on a buffered channel can proceed if there is room in the buffer. 
    * A send on a closed channel proceeds by causing a run-time panic. 
    * Communication on nil channels can never proceed, a select with only nil channels and no default case blocks forever. 
    
{{< highlight go >}}
repeatFn := func(
    done <-chan interface{},
    fn func() interface{},
) <-chan interface{} {
    valueStream := make(chan interface{})

    go func() {
        defer close(valueStream)

        for {
            select {
            case <-done:
                return
            case valueStream <- fn():
            }
        }
    }()

    return valueStream
}
{{< /highlight >}}

One of repeatFn(..) argument is to receive 'done' variable that the repeatFn(..) will use as a flag to exit from the function
The value of the receive operation <-ch is the value received from the channel ch. The expression blocks until a value is available. Receiving from a nil channel blocks forever. A receive operation on a closed channel can always proceed immediately, yielding the element type's zero value after any previously sent values have been received.


* Following are questions to answer in order to have a better understanding how the whole multi-threading channel based works in Go. These are not an exhaustive list but a starting point that will help to stir in the right direction.
    * How do we call goroutine inside a function ?
    * When using goroutine inside a function how will the code flow ?
    * How does the **for(..) range** loops works with channels ? how does it differ than normal for(..) counter loop
    * How to use the **select{ case : }** operator with channels ? 
    * When closing channel is it closed AFTER operation is done ?
* Send the channel that your function want to receive the data from    
{{< highlight go >}}
    func First(query string,  c chan Result, replicas ...Search){
        collating := make(chan Result)

        searchReplica := func(i int, collating chan Result) { 
            c <- replicas[i](query) 
        }
        for i := range replicas {
            go searchReplica(i,collating)
        }
    }
{{< /highlight >}}
in the above example the First(..) method's param is a channel parameter 'c', it uses this channel to send data that will be read by the caller function.
    
* Use timeout as much as possible to avoid the app from 'hang' mode. Having timeout enable troubleshooting as function will automatically terminate after certain amount of time.
{{< highlight go >}}
    timeout := time.After(800 * time.Millisecond)

    for {
        select {
        ...
        ...
        case <-timeout:
            fmt.Println("timed out")
            return
        }
    }
{{< /highlight >}}

 
* If we close the channel and we try to read data (even AFTER reading the data) no exception thrown
{{< highlight go >}}
func main(){
    // size chosen to ensure it never gets filled up completely.
    threadChannel := make(chan string,1) 

    go func() {
        threadChannel <- "TESTING from function 1"
        defer close(threadChannel)
    }()

    fmt.Println("Obtained  1" , <-threadChannel)
    time.Sleep(1000 * time.Millisecond)
    fmt.Println("----------------")
    fmt.Println("Obtained  2" , <-threadChannel)
}
{{< /highlight >}}


* If we DO NOT close the channel and try to read data (AFTER reading the data) deadlock exception will be thrown
{{< highlight go >}}
func main(){
    threadChannel := make(chan string,1) // size chosen to ensure it never gets filled up completely.


    go func() {
        threadChannel <- "TESTING from function 1"
        //defer close(threadChannel)
    }()

    fmt.Println("Obtained  1" , <-threadChannel)
    time.Sleep(1000 * time.Millisecond)
    fmt.Println("----------------")
    fmt.Println("Obtained  2" , <-threadChannel)
}
{{< /highlight >}}
* ALWAYS close(..) the channel to stop the go func(..) running and resides in memory. Without closing the channel possibility of memory leak and/or deadlock is high
* Mechanism should always be in place to make sure that the parent can instruct the child goroutine to cancel it's operation. 
 
    
<h2>Goroutine</h2>

* When you spawn goroutines, make it clear when or whether - they exit.
* Goroutines can leak by blocking on channel sends or receives: the garbage collector will not terminate a goroutine even if the channels it is blocked on are unreachable.
* Even when goroutines do not leak, leaving them in-flight when they are no longer needed can cause other subtle and hard-to-diagnose problems. 
* Try to keep concurrent code simple enough that goroutine lifetimes are obvious. If that just isn't feasible, document when and why the goroutines exit.
* _DO NOT_ use package **math/rand** to generate keys, instead, use **crypto/rand's Reader**.
* Every time running a goroutine(..) we must think how it will finish. What will be the trigger to return or exit from the gorotine
* At any time we want to to send data into a channel make sure the channel variable is send as parameter to the called method

<h2>Select packages</h2>

Following is an example to use **SelectCase** in situation where there are multiple channels to process

{{< highlight go >}}
package main

import (
	"fmt"
	"reflect"
)

func produce(ch chan<- string, i int) {
	for j := 0; j < 5; j++ {
		ch <- fmt.Sprint(i*10 + j)
	}
	close(ch)
}

func main() {
	numChans := 4

	var chans = []chan string{}

	for i := 0; i < numChans; i++ {
		ch := make(chan string)
		chans = append(chans, ch)
		go produce(ch, i+1)
	}

	cases := make([]reflect.SelectCase, len(chans))
	for i, ch := range chans {
		cases[i] = reflect.SelectCase{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch)}
	}

	remaining := len(cases)
	for remaining > 0 {
		chosen, value, ok := reflect.Select(cases)
		if !ok {
			// The chosen channel has been closed, so zero out the channel to disable the case
			cases[chosen].Chan = reflect.ValueOf(nil)
			remaining -= 1
			continue
		}

		fmt.Printf("Read from channel %#v and received %s\n", chans[chosen], value.String())
	}
}
{{< /highlight >}}

**SelectCase** can be used to receive or send data. 

To use it to receive data use the SelectRecv direction

{{< highlight go >}}
cases[i] = reflect.SelectCase{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch)}
{{< /highlight >}}

the _Chan_ will point to the channel that will be used to receive the data.

To read the data use the following

{{< highlight go >}}
chosen, value, ok := reflect.Select(cases)
{{< /highlight >}}

the _value_ variable will contain the data from the channel


<h1>References</h1>
* [Best practises in Golang](http://peter.bourgon.org/go-in-production/#testing-and-validation)
* [Lesson learned from contributing minikube](https://github.com/kubernetes/minikube/pull/5718#discussion_r339308904)
* [Code Review Best Practises Golang](https://github.com/golang/go/wiki/CodeReviewComments)
* [Go Advices](https://github.com/cristaloleg/go-advices)
