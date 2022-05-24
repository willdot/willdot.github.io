---
title:  Interfaces in Go
author: Will Andrews
date: 2018-08-12
order: 10
---

Interfaces are a very useful part of software development as they allow the code to be decoupled. In Go, a language where there aren't classes, we can use the structs and kind of use them as a class. That way we can create interfaces and then create structs that implement these interfaces.

### Creating an interface in Go

Creating an interface is quite easy. 

```go
type SomeService interface {
    DoSomething(string) string
    DoSomethingElse(int, string) string
}
```

### Creating a struct and then making it implement the interface

Next we create a struct, but instead of putting the functions from the interface inside the struct (because structs can only contain fields), we create the functions and then tell the functions what the receiver is. This essentially assigns the function to the struct, and so it can only be called from a structs instance.

So first, we create a struct
```go
type aService struct{}
```

Then we create the functions and just before the function name, we tell it what the receiver is. In this case, the receiver will be the struct 'aService'. So for both functions from within the interface, we will have an implementation of them both.
```go
func (aService) DoSomething(s string) string {
    return s + "!"
}

func (aService) DoSomethingElse(i int, s string) string {
    return strconv.Itoa(i) + s
}
```

Now that we have both functions that the interface requires, assigned to a struct, we can now use them.

### Using the functions 

Create a new instance of the aService struct and call the functions. Simple.

```go
x := aService{}

response := x.DoSomething("Hello")

fmt.Println(response)
```

We should then get "Hello!" as the response.

### Create a function that takes an interface as a parameter

So now we can create structs that implement an interface, we can now create a function that takes the interface as a parameter, meaning that any struct that implements this interface can be passed into the function.

```go
func printSomething(x SomeService) {
    fmt.Println(x.DoSomething("Hello again"))
}

func main() {
    serv := aService{}

    printSomething(serv)
}
```

Now when we call this function, and pass in the service as a parameter, we will get "Hello!" printed to the console.

### Using generic interfaces

Sometimes a function needs to receive a parameter of just about anything. This can be done by using 'interface{}' as the parameter. Then you can send in anything to the function and use it. In this example, I am going to create a function that accepts a generic interface, and then if it is of my 'aService' type, then I'll call one of the functions on this struct.

```go
func genInterface(i interface{}) {

	v, ok := i.(aService)

	if ok == true {
		fmt.Println(v.DoSomething("This was generic"))
	}
}

```

### Summary

Try removing the doSomethingElse function from your code, and seeing if it will run. It won't because the struct won't implement the interface. Even though we aren't using that function at the moment, it is something that is required to meet the interfaces requirements.

Full code:

```go
type SomeService interface {
    DoSomething(string) string
    DoSomethingElse(int, string) string
}

type aService struct{}

func (aService) DoSomething(s string) string {
    return s + "!"
}

func (aService) DoSomethingElse(i int, s string) string {
    return strconv.Itoa(i) + s
}

func printSomething(x SomeService) {
    fmt.Println(x.DoSomething("Hello again"))
}

func genInterface(i interface{}) {

	v, ok := i.(aService)

	if ok == true {
		fmt.Println(v.DoSomething("This was generic"))
	}
}

func main() {
    serv := aService{}

    response := serv.DoSomething("Go Go Go")

    fmt.Println(response)

    response = serv.DoSomethingElse(3, " is the magic number")

    fmt.Println(response)

    printSomething(serv)

    genInterface(serv)
}
```
This code should return the following:
```
Go Go Go!
3 is the magic number
Hello again!
This was generic!
```

Now that you understand interfaces, the next thing to investigate is using these interfaces as proper services, dependency injection and testing.