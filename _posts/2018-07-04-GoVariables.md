---
title:  Go variables
author: Will Andrews
date: 2018-07-04
order: 17
---

### Creating a variable


```Go
name := "will"
```
This has created a variable called 'name' and has the value "will". Go knows that this is a string, so you don't need to give it a type.

### Changing the value of a variable

If we want to change the value of a variable we do:
```Go
name = "someone else"
```

### Creating variables globally for the package
If you want to create a few variables that are used in multiple functions within your package, you can declare them in a special way. 

```Go
var (
    firstName, surName string
    age int32
    height = 3.2

)
```
There's a few things going on here.

* If you have multiple strings, you can create them on the same line, separated by a comma. You can't give them a value doing this, so they will be initialized with a nil value.
* We have created an age variable and said it is of type int32.
* We created the height variable and gave it a value.

Now these variables can be used anywhere in the package. 

### Pointers
So far, we've created variables and assigned values to them. However when passing variables to a function, we are passing values and not references. So if we pass the name variable we created earlier into a function, it creates a copy of that variable that is only in scope of the function. So if we alter that value of the variable in the function, once the function has completed, the variable won't have changed in the calling function. For example:

```Go
func main() {
    name := "will"
    fmt.Println(name)
    changeName(name)
    fmt.Println(name)
}

func changeName(input string) {
    fmt.Println(input)
    input = "someone else"
    fmt.Println(input)
}
```
This code will output the following:
```
will
will
someone else
will
```

If we want to change the value of the variable that has been passed in, we need to reference it with a pointer. This allows us to get the memory address of the variable and pass that into a function and then we can use this address to change the value from within that function.

The first thing we need to do is assign the address to a pointer variable:
```Go
namePointer := &name
```

Once we have that, we need to change the function so that we allow a pointer to be sent as a parameter.
```Go
func changeName(input *string) {
}
```

Next we can change the value at the address sent into the method as a parameter.
```Go
func changeNameRef(input *string) {
	*input = "someone else"
}
```
That's it. Let's see what the following code does:
```Go
func main() {
    name := "will"
    fmt.Println(name)
    namePointer := &name
    fmt.Println(namePointer)
    changeName(namePointer)
    fmt.Println(name)
}

func changeName(input *string) {
    fmt.Println(input)
    fmt.Println(*input)
    *input = "someone else"
    fmt.Println(*input)
}
```
This will output the following
```
will
*the memory address will be here*
*the memory address will be here again*
will
someone else
someone else
```

Note that we have printed the memory address out as well just so we can see that we are using a reference in the function.

