---
title:  Channels and Go routines
author: Will Andrews
date: 2020-05-03
order: 31
published: true
---

Recently I've been extracting data from a MongoDB collection, but I've had a requirement to keep the ordering of some of the data. For example, some of the data I've pulled out is stored as a ```primitive.Binary```, and in this binary data is stack trace information. Part of the stack trace is a segment of code, which contains elements line number and code value. So something like this:

```json
{
    "code": {
             "27": "func main() {",
             "28": "    err := errors.New("BANG")",
             "29": "    Bugsnag.notify(err)",
             "30": "}"
           }
}
```

Part of the Go language is the type ```map[string]interface``` which is really handy for this type of decoding, because it means you can store pretty much any JSON into this type, and then iterate over it really easily. However, the downside to this is that JSON does not support ordering, and neither do Maps in Go. This proves to be a problem in my case because the order of the code in a stack trace is very important.


### ExtendedJSON to the rescue

The ```primitive.Binary``` type has a field called data, which is a slice of bytes. This is where my data is stored. To get it out, I simply need to use the standard way of querying Mongo and then decode into a struct with a ```primitive.Binary``` type, like this.

``` go
// StackTraceDoc describes the stack trace document stored in mongo
 type StackTraceDoc struct {
 	Frames *primitive.Binary `bson:"frames,omitempty"`
 }
```

Once I have my stack trace doc, I need to extract that binary data to something useable.

MongoDB has the ability to marshal and unmarshal using extendedJSON, which along with some other neat tricks, allows the maintaining of order which is exactly what I want. To do this, it's very similar to unmarshaling with the standard library. Note: The false parameter into the UnmarshalExtJSON function is for canonical. That is useful for some other clever stuff that extended JSON allows, which I won't go into here.

``` go
// StackFrameDoc represents the one element of a StackTrace
 type StackFrameDoc struct {
 	File         string
 	LineNumber   int32
 	ColumnNumber int32
 	Method       string
 	Code         *bson.D
 }

var stackTrace []*StackFrameDoc

err = bson.UnmarshalExtJSON(stackFrameData, false, &stackTrace)
if err != nil {
    return nil, errors.Wrap(err, "error unmarshalling stack trace")
}
```

Here I have defined the type StackFrameDoc which contains the fields I am expecting. Note that the Code field is of type ```Bson.D```. This is key to maintaining order when we use the the unmarshaled data (more info on that later).

I know that the stack trace can be more than one stack frame, so the type that I pass into the unmarshal function is a slice of my StackFrameDoc. I pass in the stack trace docs binary data field (remember the binary type has a field that is a slice of bytes), and the bson library will decode that into my slice of stack frame doc. (Error checking because....well this is Go and that's how we do things).



Now I have decoded the binary data into a useable type, I want to get that data out in order!

### bson.D

This type is the key to this working. What Mongo have done with this type, is basically an ordered map. The docs state:
```
type D []DocElem

D represents a BSON document containing ordered elements. For example:

bson.D{{"a", 1}, {"b", true}}
```

```
type DocElem struct {
    Name  string
    Value interface{}
}
```

If you think about it, what they've done is created a ```map[string]interface{}``` wrapper using a slice of a key value pair. This means that I can iterate over the DocElem slice, and then extract the key value data. Like this:

``` go
for _, stackFrame := range stackTrace {
    for _, code := range *stackFrame.Code {
        fmt.Println("line number: " + x.Key)
        fmt.Printf("code: %s\n", x.Value)
        fmt.Println("---")
    }
}
```

Using our test data from earlier, this will output:

```
line number: 27
code: func main() {
---
line number: 28
code:     err := errors.New("BANG")
---
line number: 28
code:     Bugsnag.notify(err)
---
line number: 29
code:  }
---
```

And you can see it has maintained order!