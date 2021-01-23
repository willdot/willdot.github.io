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
Let's say the message we want to send back is 50mb. Well if you try to do that, you'll get an error because the message is bigger than the 4mb default size limit. So how about splitting the message into chunks and streaming it to the client chunk by chunk. Perfect. Well actually, the easiest thing to is to just increase the size limit on the client to allow this. It's a single line of code in most languages, and for 50mb, is probably all that's needed. However, for very large files, you probably want to send them chunked as sending a single file thats 1GB could take a long time, and if the connection times out 60% of the way through, well that's just annoying. But if we chunk it and send the chunk number and total number of chunks in the response, then if the connection drops and the client has to retry, it can ask the server to chunk it up by ```x``` amount and start sending me the results starting at ```y```.

### Send lots of items back
Proto messages allows you to send a repeated message, so there's no need to stream lots of items back when they can all be send in a single response. However, if the server is having to make lots of calls to a database to get each item, well sending them back 1 by 1 might be quicker. For example:

A server is querying a database but using paging to do so. If it's an expensive query that takes a while per page, then there's no point holding onto the items already fetched until it has them all. The client might want to start processing them while it waits for the next page of results. This might make the whole process quicker. Another way to do this, is by using paging in the request, but why should the client need to handle paging if the server can do all that and stream the responses back.


## A simple example

I'm going to demonstrate the fiest use cases of the above; splitting a large file into smaller chunks.

First lets create the proto file. I want a request that is going to provide a filename, the number of chunks to split by and the chunk to start at. The last 2 are only used for when retrying after a previous attempt failed. Also defined will be a response that is going to contain the chunk data, the number of chunks the file was split into and the current chunk number.

``` protobuf
syntax = "proto3";

package demo;

service Files {
    rpc GetFile(Request) returns (stream Response) {}
}

message Request {
    // The file name to get
    string name = 1;
    // The how many chunks to split it by (optional)
    int32 total_chunks = 2;
    // The chunk number to start at (optional)
    int32 start_at = 3;
}

message Response {
    // The data in this chunk
    bytes chunk = 1;
    // How many chunks the file was split into
    int32 total_chunks = 2;
    // The current chunk number
    int32 chunk_number = 3;
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

func (f *FileChunks) GetFileAndChunk(chunkFunc ReturnChunk, filename string) error {

	file, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	fi, err := file.Stat()
	if err != nil {
		return err
	}

	numberOfChunksToSplitBy := int64(4)
	size := fi.Size()

	chunkSize := size / numberOfChunksToSplitBy

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

		fmt.Println(string(b[0:n]))
	}

	return nil
}

```

Here I have created a struct with a function that takes a chunkFunc (I called it this because it sounded funny to me) and a file name. The chunkFunc is a function declared as a type. This is the same function signature as our wrapped streaming function, so it can be passed in here.

I then have the logic that opens the file, works out how to split the file into chunks (hardcoded 4 chunks, but will make configurable later). Then using the bufio reader type I am able to create a slice of bytes to put my data in (to the size of the chunks I've calculated) ***THIS NEEDS REWORDING*** and then in a loop, read the file and 