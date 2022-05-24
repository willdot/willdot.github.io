---
title:  Making HTTP requests and JSON decoding in Go
author: Will Andrews
date: 2018-08-04
order: 9
---

While creating a small command line app to make use of Azure's translation services, I used the built in HTTP request package and the JSON decoding.

#### Making HTTP requests

There are a few things required to make an HTTP call in Go. First, you need a client.
```go
client := &http.Client{}
```

You also need a url and if making a post request, a body in the form of JSON.

```go
url := "http://localhost:80/something"

body := strings.NewReader("[{\"SomeProperty\" : \"SomeValue\"}]")
```

The next thing required is an http request.
```go
request, err := http.NewRequest("", url, body)
```

After this you can add any headers to the request.
```go
request.Header.Set("Content-Type", "application/json")
request.Header.Add("ApiKey", "some api key here")
```

Now you can make the HTTP request and check for errors and the returned status code. It's also good practice to close the response body as well by deferring it before using. That way, when you have finished with it, and the current function is exited, the response will be closed.

```go
response, err := client.Do(request)

	if err != nil {
		fmt.Printf("Error on request: %v\n", err)
		return nil
	}
	defer response.Body.Close()

	if response.StatusCode == http.StatusOK {
		// DO SOMETHING
	} else {
        // DO SOMETHING ELSE
    }
```

### Decoding the JSON response into a model of your own

Now that you've made your HTTP request, it's time to use the data that has been sent back. If the data is JSON, then you can use the "encoding/json" package that comes with Go.

First things first, you need to create a model struct for the data you've received. This is easy enough and only requires you to create a struct and add in some properties. You don't have to create a property for each JSON property, the decoder will only map the properties you have in your struct. 

So if the JSON was something like this:
```json
{
    "name" : "Will",
    "age" : 28,
    "location" : {
        "street" : "Somewhere",
        "postCode" : "Something"
    }
    }
}
```

Then you could create the following structs to get the name and location. On the end of each property line, theres a "tag" which allows you to say what the name of the property is in the JSON file. Note that the location, is a struct in itself and I haven't mapped the age.

```go
type Person struct {
    Name string `json:"name"`
    Location Address `json:"location"`
}

type Address struct {
    Street string `json:"street"`
    Postcode string `json:"postcode"`
}
```

Now that you have the models you want to map the response to, it's time to decode.

```go
// Create the decoder
decoder := json.NewDecoder(response.Body)

// Create the object in which to map the JSON to
var result = new(Person)

// Do the decoding by passing in the reference to the object you created
err := decoder.Decode(&result)

if err != nil {
    // DO SOMETHING TO WARN USER OF ERROR
} else {
    // Print the persons name
    fmt.Prinln(result.Name)
}
```
This will result in getting "Will" printed to the console.


That's it. We have now made a post HTTP request and converted the response to a model we have created. 

Here is the full code. I have split the decode JSON into it's own function. 

```go

type Person struct {
    Name string `json:"name"`
    Location Address `json:"location"`
}

type Address struct {
    Street string `json:"street"`
    Postcode string `json:"postcode"`
}

func main() {

    client := &http.Client{}

	url := "http://localhost:80/something"

    body := strings.NewReader("[{\"SomeProperty\" : \"SomeValue\"}]")

	request, err := http.NewRequest("POST", url, body)

	if err != nil {
		fmt.Printf("Error on request: %v\n", err)
		return nil
	}

	request.Header.Set("Content-Type", "application/json")

	response, err := client.Do(request)

	if err != nil {
		fmt.Printf("Error on request: %v\n", err)
		return nil
	}
	defer response.Body.Close()

	if response.StatusCode == http.StatusOK {
		convertedJSON := convertResponse(response)
    }
}
    
func convertResponse(response *http.Response) *Person {

    decoder := json.NewDecoder(response.Body)

    var result = new(Person)

    err := decoder.Decode(&result)

    if err != nil {
        fmt.Println(err)
    }

    return result
}
```