---
title:  Setting up a Go dev environment
author: Will Andrews
date: 2018-07-02
order: 15
---

Today I decided to take a step away from the usual stuff I work with (.net, Angular) and learn something new. I have heard a few people tell me that Go is a fun and easy to pick up language and after a bit of Googling, I soon discovered how popular it was. So the next few blog posts will be Go related, starting with this post on setting up a dev environment.

### Installing Go
From the Go website, download the latest version for the OS required and run through the simple installer.

To test that it's installed, open up a terminal and run the following:
``` 
go version
```

You should see something similar to this, with the version and the OS type.

```
go version go1.10.3 darwin/amd64
```

### Setting up environment variables
Go requires that all projects are stored in the same place on your machine and within that folder a few sub folders. The root folder is 'Go' and some sub folders include 'src' and 'bin'. 

The 'Go' folder can be placed anywhere on your machine, but you do need to set an environment variable to point the Go tools at this folder. Create these folders where ever you wish.

#### Windows
Open up Environment Variables and add in (or change if already there) the 'GOPATH' variable and set the value to be where ever you created your Go folder. 
#### MacOs
Open up terminal and edit your bash profile (or Zsh if using that) by running the following:
```
vim ~/.bash_profile
or
vim ~/.zshrc
```

At the bottom of this, add the following and adding your path if different (note that $HOME is the user home):
```
export GOPATH=$HOME/go
```

Restart your terminal and then run the following:
```
cd $GOPATH
```
You should now be in the directory you created. Make sure you have the src folder in there, this is where you need to create your project folders (or modules as they are kinda known as in Go)

### Create a quick Hello World app to test all is working.
The best way to test everything is working, as a developer, is to create a Hello World app. So inside your src folder, create a new folder HelloWorld and inside there, create a file HelloWorld.go Inside this file, write the following:

``` Go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello World")
}
```
I will go into the syntax in another blog post.
Save this, and then head back to your terminal and run: 

```
go build
```
Now check the contents of the HelloWorld folder and there should now be an executable file called HelloWorld. Run this file and you should see in your terminal the classic Hello World text.
