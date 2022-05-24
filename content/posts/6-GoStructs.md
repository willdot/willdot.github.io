---
title:  Go structs
author: Will Andrews
date: 2018-07-04
order: 6
---

This post will go through the use of Structs in Go and give an understanding of what they are used for.

### What's the point of Go structs?
Let's say the data types provided to us aren't enough, and we want to create our own. Ones that have different properties of various types. 

Imagine we have a person. This person has a name, age, occupation and height. Well we can create a struct that's a person and then use this in our code.

### Creating a struct

It's easy to create:
```go
type person struct {
    name string
    occupation string
    age int32
    height float32
}
```

### Using a struct

There are a few ways of using a struct you've created, like there are a few ways of creating normal variables.

1: Create a variable and then populate the properties after.
```go
func main() {
    var aPerson person 
    aPerson.name = "Will"
    aPerson.occupation = "Software developer"
    aPerson.age = 28
    aPerson.height = 180
    fmt.Println(aPerson)
}
```

2: Create a reference to a person variable. Note that you have to use the * to print the value of the person. If you don't you print the address of the reference.
```go
func main() {
    aPerson := new(person)
    aPerson.name = "Will"
    aPerson.occupation = "Software developer"
    aPerson.age = 28
    aPerson.height = 180
    fmt.Println(*aPerson)
}
```

3: Create a new variable and initialize it at the same time.
```go
func main() {
    aPerson := person{
        name: "Will",
        occupation: "Software developer",
        age: 28,
        height : 180,
    }
    fmt.Println(aPerson)
}
```

### Models package
Sometimes it's useful to create a package with multiple structs in it that can be used in different packages. Remember if you want something to be exportable, when giving it a name, it needs to start with a capital letter!!

Let's create a models package and give it some structs:
```go
package models

type Person struct {
	Name       string
	Occupation string
	Age        int32
	Height     float32
}

type House struct {
	Number int32
	Street string
	Town   string
}
```

Now we can import this package into our main package and use it.
```go
package main

import (
	"Playground/Models"
	"fmt"
)

func main() {

	aPerson := models.Person{
		Name:       "Will",
		Occupation: "Software developer",
		Age:        28,
		Height:     180,
	}
	fmt.Println(aPerson)

	aHouse := models.House{
		Number: 22,
		Street: "This street",
		Town:   "Some town",
	}
	fmt.Println(aHouse)
}
```

This will output the following:
```
{Will Software developer 28 180}
{22 This street Some town}
```

So that's how to create your own structs and how to use them.