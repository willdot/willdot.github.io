---
title:  Server side streaming gRPC in Go
author: Will Andrews
date: 2021-01-23
order: 33
published: true
---

In the world of gRPC most cases will be a request response type of communication. The client sends the request to the server. The server does something, perhaps get some data from a database, and then returns it in a response to the client. This is known as a Unary RPC. 

However there are 3 other types that can be used which involve streaming.
1. Server side - The server sends multiple messages back to the client, who waits for each response
2. Client side - The client sends multiple messages to the server, which waits for each request before sending the response
3. Bidirectional - Both server and client send messages to each other at the same time.

I'm only going to focus on server side streaming in this post.

## Use cases
One thing I discovered when reading about server side streaming, is that a lot of the time it's not actually necessary. That might seem like an odd statement to make when doing a write up on it, but when it is necessary it's a really powerful tool.

### The response message is too big
Let's say the message we want to send back is 50mb. Well if you try to do that, you'll get an error because the message is bigger than the 4mb default size limit. So how about splitting the message into chunks and streaming it to the client chunk by chunk. Perfect. Well actually, the easiest thing to is to just increase the size limit on the client to allow this. It's a single line of code in most languages, and for 50mb, is probably all that's needed. However, for very large files, you probably want to send them chunked as sending a single file thats 1GB could take a long time, and if the connection times out 60% of the way through, well that's just annoying. But if we chunk it and send the chunk number in the response, then if the connection drops and the client has to retry, it can ask the server to start again starting at a particular chunk number.

### Send lots of items back
Proto messages allows you to send a repeated message, so there's no need to stream lots of items back when they can all be send in a single response. However, if the server is having to make lots of calls to a database to get each item, well sending them back 1 by 1 might be quicker. For example:

A server is querying a database but using paging to do so. If it's an expensive query that takes a while per page, then there's no point holding onto the items already fetched until it has them all. The client might want to start processing them while it waits for the next page of results. This might make the whole process quicker. Another way to do this, is by using paging in the request, but why should the client need to handle paging if the server can do all that and stream the responses back.


## A simple example

I'm going to demonstrate the first use cases of the above; splitting a large file into smaller chunks.

First lets create the proto file. I want a request that is going to provide a filename and the chunk to start at. The second is only used for when retrying after a previous attempt failed. Also defined will be a response that is going to contain the chunk data and the current chunk number.

``` protobuf
syntax = "proto3";

package demo;

service Files {
    rpc GetFile(Request) returns (stream Response) {}
}

message Request {
    // The file name to get
    string name = 1;
    // The chunk number to start at (optional)
    int32 start_at = 3;
}

message Response {
    // The data in this chunk
    bytes chunk = 1;
    // The current chunk number
    int32 chunk_number = 2;
}
```

Once this has been defined, I will use the tooling to generate the client and server interfaces and start to build up a simple gRPC server. 

Here is the code for the basic server implementation. It registers a gRPC server using a `Server` struct which implements the interface defined in the generated proto interfaces. One thing to note though, is that the function signature for the streaming function differs to normal gRPC function signatures.

``` go

const (
	addr = "localhost:50051"
)

func main() {
	log.Println("server started")
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to listen: %v", err)
	}

	s := grpc.NewServer()
	reflection.Register(s)
	pb.RegisterFilesServer(s, &Server{})

	if err := s.Serve(lis); err != nil {
		fmt.Fprintf(os.Stderr, "failed to serve grpc server: %v", err)
	}
}

type Server struct {
}

func (s *Server) GetFile(req *pb.Request, stream pb.Files_GetFileServer) error {
	return nil
}
```


The function signature is different to normal in the following ways:
1. There isn't a context present in the signature. Instead you get it out of the stream parameter.
2. The second parameter is a server, which is what you use to stream the data back. I'll explain how this works shortly.
3. Only an error is returned, because as above, the responses are streamed back.

So how does all of this work? Well here is a really simple example of streaming data. I'm not going to do any of the file handling just yet, instead I'm going to stream back 2 words........ `Hello` and `World`.

``` go
func (s *Server) GetFile(req *pb.Request, stream pb.Files_GetFileServer) error {
	helloWorld := []string{"hello", "world"}

	for _, word := range helloWorld {
		res := &pb.Response{
			Chunk: []byte(word),
		}

		if err := stream.Send(res); err != nil {
			return errors.Wrap(err, "error streaming response")
		}
	}

	return nil
}
```

Here I've created a slice of the words I'm sending back. Then for each word in the slice, I create a response message and then call the send function on the stream which will then stream it to the client. If at any point that fails, I'll return the error. Nice and simple. Now lets create a client to stream that data to!

``` go
const (
	address = "localhost:50051"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewFilesClient(conn)

	// make the request and open the stream
	stream, err := c.GetFile(context.Background(), &pb.Request{Name: "hello.json"})

	for {
		// get a stream response
		resp, err := stream.Recv()

		// if the err is EOF that means there is nothing else to receive so break out and finish
		if err == io.EOF {
			break
		}

		if err != nil {
			fmt.Fprintf(os.Stderr, "error streaming data from server: %v", err)
		}

		fmt.Fprintf(os.Stdout, "%s ", resp.Chunk)
	}

}
```

As you can see in the comments, we create an infinite loop that will keep iterating until it gets an error. If the error is EOF then it's completed successfully, otherwise error out and return. If no error, then print the response.

Now if you start run the server and then run the client, you should see the hello world response. And that's all it takes to stream data from the server to the client. However, we still need to do what we intended to do, and that's chunk up a file. At this point, the biggest question I had was "how the hell do I work this so that my server package, calls an interface which handles the business logic of finding, opening and closing a file".

### Passing the streamer around your codebase

In all of the apps I write, I tend to split business logic into a separate package and use interfaces so that I can test each package independently. In our case we want to create a package that will open a file, read the data out of it, split it into chunks and return each chunk individually. But I don't want to pass my streamer to the business logic. So how do we do this?

Wrapper functions!

If we wrap the code that streams the data to the client, in it's own function, we can then pass that function around our app. So that will look like this. Note that I've changed it slightly as we will be dealing with file bytes, and not words any more.

``` go

func (s *Server) GetFile(req *pb.Request, stream pb.Files_GetFileServer) error {
	helloWorld := []string{"hello", "world"}

	streamFunc := func(chunk []byte) error {
		res := &pb.Response{
			Chunk: chunk,
		}

		if err := stream.Send(res); err != nil {
			return errors.Wrap(err, "error streaming response")
		}
		return nil
    }
    
    err := streamFunc(byte("hello"))
    if err != nil {
        return err
    }

	return nil
}
```

Now that I have the logic to stream the data back to the client wrapped in a variable, I can pass that around as I please. So now lets create the business logic to do so. 

``` go

type FileChunks struct{}

type ReturnChunk func(chunk []byte) error

const chunkSize = 1024 * 32 // 32kb

func (f *FileChunks) GetFileAndChunk(chunkFunc ReturnChunk, filename string) error {
file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	r := bufio.NewReader(file)
	b := make([]byte, chunkSize)
	for {
		n, err := r.Read(b)
		if err != nil {
			if err == io.EOF {
				break
			}

			fmt.Fprintf(os.Stderr, "Error reading file: %v", err)
			return err
		}


		err = chunkFunc(b[0:n])
		if err != nil {
			return err
		}
	}

	return nil
}

```

Here I have created a struct with a function that takes a chunkFunc (I called it this because it sounded funny to me) and a file name. The chunkFunc is a function declared as a type. This is the same function signature as our wrapped streaming function, so it can be passed in here.

I then have the logic that opens the file and defering the closing of the file. Then using the bufio reader type I am able to create a slice of bytes in the size of 32kb to put my data in. Then on each iteration of a loop, it will call Read() on the reader passing in the bytes buffer to write to. Then checking for errors, because if EOF is received then there's nothing else to read from the file. If there is data then we send only the number of bytes read into the byte slice, because we don't want to send empty data. TODO: word this better and perhaps investigate more

This file handling code is in a different package, and because I will want to want to mock the file handling out when testing my gRPC code, I will use an interface in the gRPC server like so.

``` go
func main() {
	// omitted set up code
	file := files.FileChunks{}
	pb.RegisterFilesServer(s, &Server{fileChunker: &file})
	// more omitted code
}

type Chunker interface {
	GetFileAndChunk(chunkFunc files.ReturnChunk, filename string) error
}

type Server struct {
	fileChunker Chunker
}

unc (s *Server) GetFile(req *pb.Request, stream pb.Files_GetFileServer) error {
	streamFunc := func(chunk []byte) error {
		res := &pb.Response{
			Chunk: chunk,
		}

		if err := stream.Send(res); err != nil {
			return errors.Wrap(err, "error streaming response")
		}
		return nil
	}

	s.fileChunker.GetFileAndChunk(streamFunc, req.Name)

	return nil
}
```

Now if I want to test my gRPC code but not do any file handling in the file package, I can mock out that interface. // TODO: make that setup more testable

If I were to create large test file called "hello.txt" and run the client, it will stream the entire contents. Success!

Finally it's time to use that retry mechanism. To do this we will need to change the  `GetFileAndChunk` function signature to take a start at parameter and the use that to "ignore" that number of chunks before sending the rest to the client. It will look like this.

``` go

func (f *FileChunks) GetFileAndChunk(chunkFunc ReturnChunk, filename string, startAt int32) error {
	file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	r := bufio.NewReader(file)
	b := make([]byte, chunkSize)

	currentChunk := int32(-1)
	for {
		n, err := r.Read(b)
		if err != nil {
			if err == io.EOF {
				break
			}

			fmt.Fprintf(os.Stderr, "Error reading file: %v", err)
			return err
		}
		currentChunk++
		// ignore this chunk if it's less than the startAt
		if currentChunk < startAt {
			continue
		}

		err = chunkFunc(b[0:n])
		if err != nil {
			return err
		}
	}

	return nil
}
```

And then the interface in the server will need to be updated and the parameter from the gRPC request can be passed in. Then the client code can then pass in a number in the request, and when the file package is iterating over the file, it will ignore the first `x` number of chunks.