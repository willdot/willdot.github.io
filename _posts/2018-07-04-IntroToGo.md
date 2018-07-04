---
title:  Introduction to Go
author: Will Andrews
date: 2018-07-04
order: 16
---

If you read my previous post, you'll know that I've decided to take a dive into the world of Go and how to set up a dev environment to start programming in Go. Next I'm going to go into some of the basics of Go.

### Basics

So to start off, I'll go through how a go file is constructed. Go is done in "Packages" or "Modules". An application will have a "Main" package which will be what is run when the application is started. It will then run code from within a main function and once that code has been executed, the application will exit.

However, another package can be created that is a "library" package, that the main package can use and run. The main package will import other packages to use, and these other packages can also import other packages and so on and so forth.

A folder that contains go code files becomes a package and all packages have a path relevant to your %GOPATH% (more on that later). If you try to create 2 different go files in the same folder and attempt to name them different packages in the code, you will get an error. Each package needs it's own folder. 

Here's an example of a simple HelloWorld application.

```Go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello World")
}
```

Nice and easy. So, what does it all mean?


This line at the top is telling the compiler, what this package is called. The main package is used to tell the compiler that this package is an application.
``` Go
package main
```

This section is where you can import other packages to use in your code. "fmt" is a format package that implements formatted I/O, such as writing a line to the console.
```Go
import (
    "fmt"
)
```

As we are making this a main package, it needs to have a main function. This is the entry point of the application and will be run on start. In it we are using the "fmt" package to print a line to the console.
``` Go
func main() {
    fmt.Println("Hello World")
}
```

That's it. Nice and simple.

### Functions

Now let's say we wanted to have some code that takes an input string, and prints it to the console and then returns a true value. We could either create a new function in the main package, and call it from the main function, like so:

```Go
package main

import (
    "fmt"
)

func main() {
    result := printSomething("Hello World")
    fmt.Println(result)
}

func printSomething(input string) (bool) {
    fmt.Println(input)
    return true
}
```

The "printSomething" function takes a string parameter and then prints it to the console and then returns true.

The signature of a function is as follows:

* func - tells the compiler we are about to create a function
* printSomething - this is the name of the function (if your function is part of a package the will be imported, use a capital letter at the start to make it exportable)
* (input string) - this is the parameters (separated by a comma) where the parameter name is input and the type is string
* (bool) - this is the return value type. (Note that there can also be multiple return values)

### Creating a package and using a function from within it

However, let's say we wanted this function to be available and used by other packages. This means we need to split it out of the main package and put it into its own package. That's easy to do.

Now create a new folder either in the main folder, or at the same level, as long as it's a folder. Inside the folder create a new go file and put the following code in:

```Go
package printsomething

import (
	"fmt"
)

func PrintIt(input string) bool {
	fmt.Println(input)
	return true
}
```

As you can see, we've give the package a name and added a function in. Note that the function has a capital letter at the start to allow it to be exportable and used from within other packages.

Now change your main code to look like this:

```Go
package main

import (
	"Playground/printsomething"
	"fmt"
)

func main() {

	result := printsomething.PrintIt("HelloWorld")
	fmt.Println(result)
}
```

We are importing the printsomething package. My folder is at the same level as the main folder, which is inside a playground folder, which is in the src folder. (Remember all packages are relevant from the src folder in your %GOPATH% folder.)

If you're wondering what "result := " means, that's declaring a variable and assigning the value to the result of the method. I will go into this in more detail in the next post.

Now in the main function, I call the PrintIt function from within the printsomething package. 

### Summary

So that's the introduction to Go. Packages are like modules and can be imported into other packages so that they can be used. Something I didn't mention earlier, is that packages are kind of static. You can't create a new instance of them, as they aren't classes.

My next post will go into variables and how Go uses them. 
