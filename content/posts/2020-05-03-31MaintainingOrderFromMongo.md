---
title:  Maintaining order of data from Mongo
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
             "28": "    err := someBadFunc()",
             "29": "    if err != nil {",
             "30": "        Bugsnag.notify(err)",
             "31": "    }",
             "32": "}"
           }
}
```

Part of the Go language is the type ```map[string]interface``` which is really handy for this type of decoding, because it means you can store pretty much any JSON / BSON into this type, and then iterate over it really easily. However, the downside to this is that JSON does not support ordering, and neither do Maps in Go. This proves to be a problem in my case because the order of the code in a stack trace is very important.


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

bson.D{ {"a", 1}, {"b", true} }
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
code:     err := someBadFunc()
---
line number: 29
code:     if err != nil {
---
line number: 30
code:         Bugsnag.notify(err)
---
line number: 31
code:     }
---
line number: 32
code: }
---
```

And you can see it has maintained order!

### What about the other way

I also had a requirement where I needed to take some of the data I had got out of mongo, and turn it into a JSON string. This was things such as metadata, that can't easily be returned over gRPC because it doesn't always have a defined structure. With extended JSON and the mongo library, I was able to do this easily.

Here is my document structure that I decoded out of mongo
``` go
type Thing struct {
   ID       *primitive.ObjectID `bson:"_id,omitempty"`
   MetaData *bson.D             `bson:"metaData,omitempty"`
}
```

Here is the actual document in mongo

```
{
    "_id" : ObjectId("5e8cadfd0000474af4010000"),
    "metaData": {
        "thing": {
            "ID": 1,
            "Name": "thing1"
        },
        "otherThing": {
            "ID": 2,
            "Name": "thing2"
        }
  }
}
```

As you can see metadata is made up of 2 different objects.

What I want to do is get that and create a JSON string. If I were to use a JSON library like this: https://github.com/json-iterator/go I would get a JSON string, but I would not retain the order of the data. This means I may end up with otherThing first. Not ideal for my use case.

Instead, I can use the ```bson.MarshalExtJSON``` function and get back an extended JSON string that will retain ordering.

``` go
func convertToJSONString(input interface{}) (string, error) {
	if input != nil {
		inputByteArray, err := bson.MarshalExtJSON(input, false, false)

		if err != nil {
			return "", errors.Wrap(err, "error converting metadata to JSON string")
		}

		return string(inputByteArray), nil
	}
	return "", nil
}
```

First I check if the input is nil and return an empty string if it is. Then I use the ```bson.MarshalExtJSON``` function to get the byte array of my struct. The two false parameters are for canonical and for escaping HTML. Once I've done that I check for errors and then convert the byte array to a string. It's that simple.

I'm almost starting to thing that this bson library is handy to use, even when not dealing with mongo 👀