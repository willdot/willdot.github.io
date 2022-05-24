---
title:  Some uses of context with gRPC in Go 
author: Will Andrews
date: 2020-09-20
order: 19
published: true
---

In Go there is a package called context, which defines a type that contains deadlines, cancellation signals or some other data and is usually passed from function to function. One of the most common use of the context is the ability to work out if a call should be cancelled. 

For example, when a gRPC request is being handled by a server, the client may have provided a 20 second timeout. If the request takes longer than 20 seconds, the client is going to give up. But if the server is currently getting data from a database
then that call should be aborted if the client has cancelled. Or if there are 2 calls that the server needs to make to the database, if it makes the first one and then the client cancels, then there is no need to make the second. The way this is handled
in Go is by using the context package.

## Context basics

Context is an interface, with 4 functions defined.
``` go
Deadline() (deadline time.Time, ok bool)

Done() <-chan struct{}

Err() error

Value(key interface{}) interface{}
```
The full details of the package can be found [here](https://golang.org/pkg/context/)

These 4 functions allow the context to be a powerful tool. It allows the caller to get a deadline time if it exists, use a channel to work out if the context is "done" and find out if there's been an error in the context. The last one, "value" actually allows the caller to retrieve data from the context. This means that data of any kind can be stored in the context and the retrieved later on. This is useful for storing metadata such as headers in a gRPC request or an ID for logging purposes. It's not intended to store state.

## Deadlines in context

What I'm interested in is the deadline. To create a context with a deadline, you create a new context with a timeout, passing in a parent context (which can also be a new context) and a time, like so:

``` go
ctxWithTimeout, cancel := context.WithTimeout(context.Background(), time.Second*3)
```

It returns a new context, with the timeout set and a cancel func. This cancel func should always be deferred as it releases resources associated with it.

This context can now be passed around and any function that receives it can check if the deadline has passed by making use of the Done channel that the context interface provides. You can also get the Deadline() time and compare with `time.Now()` but that's not as pleasant. 

Something like this can be used:
A constant for loop, with a select to keep checking the Done channel. If no value is sent onto the Done channel, the do some work:

``` go
// run a continuous loop until the context is canceled
for {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Do some work
    }
}
```

## Using headers and context with gRPC

With gRPC the client can create a context and pass it in with the request. Using the "google.golang.org/grpc/metadata" package, headers can be added like so:

``` go
// create the headers
headers := metadata.Pairs(
    "source", "client service")

// add the headers to the context
ctx := metadata.NewOutgoingContext(context.Background(), headers)


// make gRPC request to server
response, err := c.MakeSomeRequest(ctx, &pb.Input{Id: 1})
if err != nil {
    // handle error
}
```

Then on the server, to get the headers out you can do this:

``` go
// get metadata out of context
metadata, _ := metadata.FromIncomingContext(ctx)

// check if the metadata has a key "source"
if len(metadata["source"]) != 0 {
    // get the value of "source"
    source := metadata.Get("source")[0]

    // This will print "client service"
    log.Printf("request source: %s\n", source)
}
```

Being able to pass metadata between services when making gRPC request calls can be really useful. I use it a lot (in a similar way to the example) so that I can do some additional logging to work out which client service made a request that might have failed. 