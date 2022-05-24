---
title:  How the race flag caught me out
author: Will Andrews
date: 2021-02-18
order: 21
published: true
---

One of the many strengths of Go is its concurrency. But using concurrency comes with caution. Sometimes it's not necessary and a lot of Go developers jump to using it too soon because they think it'll solve a problem. I'm very guilty of having done that many times. However sometimes concurrency comes "for free" in the sense of http servers. 

Let me explain that a little better. When you create an http server and run it, you can make 1000s of requests to that server and it will handle them concurrently, and you didn't even have to do a thing, other than write ~10 lines of code. 

For example, a simple http server that just returns hello as a response.

``` go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", Hello)
	http.ListenAndServe(":5050", nil)
}

func Hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "hello there!")
}
```

Run that, and head to `http://localhost:5050` in your browser and it will say hello. Happy days.

## So what's the problem?

Well in Go, there is a race flag that can be used to let you know if your code will result in a data race. 

A data race in a nutshell is when you have more than 1 Go routine trying to access the same variable. It indicates that you should probably rethink your approach. Sometimes it's not a problem but still not a good idea. One case where this needs to be handled properly is when using maps. 

If you try to write / read a map at the same time from 2 different Go routines, you'll get a panic. Not ideal. Thankfully though, as I mentioned, there is a race flag that you can use and Go will let you know that you have racey code. To use it when running tests you can do:
 
```
go test --race
```

So as an example, lets create a struct that has a map of some values. In one pointer receiver we will add a value and in another we will read a value.

```go
type Thing struct {
	someMap map[string]string
}

func newThing() Thing {
	return Thing{
		someMap: make(map[string]string),
	}
}

func (t *Thing) add(key, val string) {
	t.someMap[key] = val
}

func (t *Thing) fetch(key string) string {
	return t.someMap[key]
}
```

And now lets write a test that adds something and then tries to get it again and checks the result
``` go
func TestSet(t *testing.T) {
	x := newThing()

	x.add("hello", "there")

	res := x.fetch("hello")

	if res != "there" {
		t.Fatalf("expected 'there' but got %s", res)
	}
}
```

This will pass. But what if there are 2 requests made at the same time. That will panic right? Correct. Ok so, lets run the test with the race flag.....

It passes. Huh?

Well at the moment we aren't trying to concurrently access the map. So lets do that in the test.

``` go
func TestSet(t *testing.T) {

	var wg sync.WaitGroup
	x := newThing()

	for i := 0; i < 2; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()

			x.add("hello", "there")

			res := x.fetch("hello")

			if res != "there" {
				t.Fatalf("expected 'there' but got %s", res)
			}
		}()
	}

	wg.Wait()
}
```

Here I'm creating a new thing. Then in running 2 Go routines that will access the map at the same time. Running this with the race flag will fail. 

We can fix this by protecting the map with a mutex.

``` go

type Thing struct {
	someMap map[string]string
	mu      sync.Mutex
}

func newThing() Thing {
	return Thing{
		someMap: make(map[string]string),
	}
}

func (t *Thing) add(key, val string) {
	t.mu.Lock()
	defer t.mu.Unlock()
	t.someMap[key] = val
}

func (t *Thing) fetch(key string) string {
	t.mu.Lock()
	defer t.mu.Unlock()
	return t.someMap[key]
}
```

Run the test with the race flag again and it will pass. 

## So how did I get caught out?

Well. I had something a little bit more complex but along the same lines. A map on a struct, that was being written to in 1 function and read in another. This struct was being used from within an http server inside some business logic. However I made the mistake of forgetting the mutex. OPPS! I had some tests that were calling the http server and validating the response and the tests passed. I also had CI tests that ran with the race flag, and they passed. 

What caught me out was that my tests weren't making concurrent calls to the http server and therefore the race flag didn't pick anything up. It made me realise that I rarely create tests that make concurrent calls to the server, which is as I said at the beginning of this post, is what happens when the server is being used for real.

I only discovered that something was wrong when I ran my code and was using the server for real in my dev environment. I discovered the panic and instantly knew what I had done wrong. But this could have easily gone un noticed because I was under the assumption that the race flag would work it out for me, without having to write concurrent tests.

So I wrote a simple test that did that (without the mutex) and tried it out.

Here's some really minimal server code that adds something to the map on each request.
``` go
func main() {

	thing := newThing()

	http.HandleFunc("/add", thing.add)
	http.ListenAndServe(":5050", nil)
}

func Hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "hello there!")
}

type Thing struct {
	someMap map[string]string
}

func newThing() *Thing {
	return &Thing{
		someMap: make(map[string]string),
	}
}

func (t *Thing) add(w http.ResponseWriter, req *http.Request) {
	t.someMap["hello"] = "there"
}
```

And then here's a test that makes calls the add function at the same time.

``` go
req, err := http.NewRequest("GET", "/", nil)
	if err != nil {
		t.Fatal(err)
	}

	thing := newThing()

	addHandler := http.HandlerFunc(thing.add)

	var wg sync.WaitGroup

	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()

			rr := httptest.NewRecorder()

			addHandler.ServeHTTP(rr, req)

			if status := rr.Code; status != http.StatusOK {
				t.Errorf("handler returned wrong status code: got %v want %v",
					status, http.StatusOK)
			}

			expected := ``
			if rr.Body.String() != expected {
				t.Errorf("handler returned unexpected body: got %v want %v",
					rr.Body.String(), expected)
			}
		}()
	}

	wg.Wait()
```

If I run this test with the race flag, it will fail and tell me there's a data race. If I then add a mutex around the setting of the map, it will pass!

## What did I learn

First thing I learnt was that I had totally misunderstood how the race flag worked. I didn't know that for it to work, the tests needed to be concurrent. Secondly, it made me realise that all this time I've been testing my http servers by making a single response, which is not what happens in the real world. From now on I will make at least 1 test where 2 requests are made at the same time. 

#### Notes:
The code in this blog post was written using Go 1.15