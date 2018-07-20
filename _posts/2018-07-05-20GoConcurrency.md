---
title:  Concurrency in Go
author: Will Andrews
date: 2018-07-05
order: 20
---

In Go, concurrency allows the application to do one thing while it's waiting for something else to happen. So if one method is sleeping or waiting on a response from a service, then it can carry on and do something else in another function. This is done by using something called "Go Routines" which is Gos clever way of using a single thread and controlling which function or line of code uses this thread. To explain this better, it's probably best to write some examples. 

### Code running in order (no concurrency)

First of all, lets run some code in order and see what happens.

Note: When writing out a function, if you put a pair of () at the end of the closing '}', it will self execute.

```go
func main() {
    fmt.Println("Let's start")
    func() {
        time.Sleep(5 * time.second) // You will need to import the time package
        fmt.Println("Hello World")
    }()

    func() {
        fmt.Println("Go Go Go")
    }()
}
```
The output of this will be:
```
Let's start
** A 5 second delay **
Hello World
Go Go Go
```

As you can see, we have to wait 5 seconds for something to happen. But why can't we print the Go Go Go to the console while we wait? 

### Running concurrently

Now let's get the code to print the Go Go Go code while we wait for the Hello World after the sleep. To do this, we need to use Go routines which is as simple as putting 'go' in front of the function and using a "Wait Group".

The wait group is something that will stop the main function from exiting before all other Go routines have completed. 
```go
func main() {
    fmt.Println("Let's start concurrently")

    // you will nee to import the sync package
    var waitGroup sync.WaitGroup 
    waitGroup.Add(2)

    go func() {
        defer waitGroup.Done()

        time.Sleep(5 * time.second) // You will need to import the time package
        fmt.Println("Hello World")
    }()

    go func() {
        defer waitGroup.Done()

        fmt.Println("Go Go Go")
    }()
     
     waitGroup.Wait()
}
```

This will output the following:
```
Let's start concurrently
Go Go Go
** a less than 5 second delay here **
Hello World
```
So what's going on in this code?

1: We create a wait group which is a collection of Go routines that need to be completed before the code can continue.

2: We say that there are going to be 2 Go routines that we want to add to the wait group.

3: We use 'Go' in front of the function to indicate that it is a Go routine.

4: The defer command is an instruction to the function for when it has completed. In this case we are going to tell the wait group that this function is done, and therefore decrement the wait group count.

5: The waitGroup.Wait() is a blocker that will stop the main function from exiting until all Go routines have completed (the count of go routines has reached 0 by each function using the defer command to say it's done)

So what happens is the first function starts and then when it gets to the sleep command, the Go routine manager realizes that the application is not doing anything, and it could do something else while it waits. It just so happens that it has another Go routine to execute and so it does that and runs the second function while it's waiting. Then when that's finished, it will go back to the first function and continue to wait for the sleep to finish and then run the rest of the code. 

So as we can see this can be very useful for not wasting time. If we have multiple functions that are doing different things and one of them has to wait for an external service to complete, we may as well do some other work. It's a bit like being a waiter in a restaurant. You take the order of a table and give it to the kitchen to prepare and cook. You wouldn't stand there and wait for them to cook it, you would go back out to the restaurant and take other orders or other jobs.