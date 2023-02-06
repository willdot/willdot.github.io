---
title:  Don't let ignore files get the better of you
author: Will Andrews
date: 2023-02-06
order: 22
published: true
---

Ignore files are great. They allow you to ignore (of course) files/directories in different scenarios. For example a `.gitignore` file can be used to prevent files from being committed to Git such as local secrets, built binaries or any temp files. Or a `.dockerignore` file can be used to only copy files into a Docker container that are actually needed for a build (there's no point copying test code / resources if they aren't needed).

But what happens when you start ignoring files / directories that you didn't know you were ignoring?

### Story time
Let me tell you a story about how I learned the hard way.

About a year ago I was working on a Go service and updated the dependencies. (One thing to note is that we were vendoring). I ran local tests and linters and everything was good. However when I committed the changes, I was getting build errors in CI because of a missing file. Interesting. So I ran CI locally in a Docker container and it worked fine. Very interesting. What happened next was a morning of banging my head against my desk trying to work out why a file was "missing" in CI. 

Eventually while digging around in the file explorer with VS Code, I noticed that the directory that contained the "missing file" was shaded grey. I hadn't noticed that before on directories or files and so after a Google it turns out that it means a directory or file is being ignored because of an ignore file. So I opened up `.gitignore` for this project and there it was. We were ignoring a directory within our repository because we didn't want to commit the contents. Unfortunately the directory name also matched the directory for the dependency that got updated within the vendor directory. The fix was simple. Instead of having `folder-to-ignore` I put `\folder-to-ignore` indicating I only want the directory in the root of the project to be ignored. Once committed, all was right and CI passed.

Fast forward a year and I'm in the third week of a new job and I'm on a Zoom call with my team listening to them trying to debug a build problem in CI and I think you can see where this is going. They had a similar issue that had me frustrated a year ago. They had updated dependencies and the path of an updated dependency was also listed inside the `.gitignore` file. I let them know what I think the issue was, and I saved them from any more frustrating time. Fast forward a couple of days and in another Zoom meeting with different members of the team and they had the exact same issue. What are the chances?!?!?!

### Let's take a look with an example

Lets say we have a project structure like this:

```
main.go
tests/
client # binary called client
vendor/
    github.com/
        somelibary
            client/
                client.go
            server/
                server.go

```

Ideally we don't want to commit the built binaries and so we have a `.gitignore` file like so.

```
# ignore the built binary
client
```

If we commit this, the vendor directory `/vendor/github.com/somelibrary/client` and it's contents will be ignored. 

However updating the `.gitignore` file to be 
```
/client
```

we will only ignore the binary at the root directory and not the directory nested inside the vendors directory.

We could improve this a bit more by ensuring we never accidentally come across this again by explicitly making sure we always commit the `/vendor` directory and all it's contents by doing this:

```
!/vendor/**
client
```
Note: A gotcha is that if you put `client` first it will still ignore it inside the `vendor` directory. So always put your "nots" at the top.

This doesn't just apply for vendoring. Using the example above where we were ignoring `client` but you had something like `pkg/database/client`, that too would be ignored. 

The same thing is true for `.dockerignore` files, so always check your ignore files and be as explicit as you can about files and directories you wish to ignore. Otherwise you'll spend hours banging your head against your desk working out why things aren't working.

### Takeaways

* Be as explicit as you can when using ignore files.
* If you're going to use "not" to ensure something isn't matched by something that is to be ignored, make sure it's at the top of the ignore file.

I hope that this saves someone from wasting a morning like I did a year ago.