---
title:  Intro to Microservices and gRPC - part 1
author: Will Andrews
date: 2019-06-13
order: 14
published: false
---

After I made my first CLI Go application, I fancied having a go at microservices, as I had read that Go was perfect for it. I found a really good guide online, walking me through the creation of a couple of Microservices, and once I had finished that I worked on moving my Daily Do project into a Microservice structure. I also found another user for Docker containers....whoop whoop!

## gRPC

I had seen this before on some sites when reading about Go, and assumed the 'g' stood for Go. But I was wrong. gRPC can be used by many languages, which is pretty cool because as a .NET engineer as my main job, I can transfer the skills I learnt over. 

It's actually a Remote Procedure Call system that uses HTTP/2 for transporting Protocol Buffers. The protocol buffers are a written in a 'proto' language, and specify interfaces. Then there are tools that turn these proto files into implementation code of your language choice, so that you can use the interfaces with your applications. 

For example this would be a 'HelloWorld.proto' file:

``` 
syntax = "proto3"; 

package HelloWorld; 

message Request {
    string firstName = 1;
    string lastName = 2;
}

message Response {
    string result = 1;
}

service TaskService {
    rpc Get(Request) returns (Response) {}
}
```

In the above, we are specifying that a message called Request (some data we want to send) and a message called Response. In each of the messages, we are specifying some fields that can be set and sent in the message. The number at the end are there for identifying the fields.

We are then saying that a TaskService has a function called Get that takes a Request message and returns a Response message. For an application to use this service, it must implement the TaskService interface. 


## Turn gRPC code into Go code

Next we run the protocol bugger complier against our proto file. Protoc and how to use it can be found [Here](https://github.com/golang/protobuf)

This will create a package called HelloWorld that can be then imported into other Go packages and used, so lets do that.

## Using the generated package

 