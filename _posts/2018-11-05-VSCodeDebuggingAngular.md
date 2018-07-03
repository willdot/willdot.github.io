---
title:  Debugging and Angular App in Visual Studio Code
author: Will Andrews
date: 2018-05-11
order: 8
---

Not being able to debug Angular code in Visual Studio Code always bugged me (pun intended) and then I stumbled across an extension that allows just this!

It's called 'Debugger for Chrome' and once installed, is very easy to configure.

First thing to do is press F5 to start debugging and it will ask you for an environment to use. Select 'Chrome'.

Then it will launch a launch.json file that will have a default configuration populated. Change it to look something like this where the URL is the default URL that your app uses when it's launched.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach",
            "type": "chrome",
            "request": "attach",
            "port": 9222,
            "url": "http://localhost:4200/",
            "webRoot": "${workspaceFolder}"
        }
    ]
}
```

Now this bit is different between OS.

Windows:
    Right click on your Chrome launcher, go to properties and in the 'Target' section, add the following onto the end of what ever is already there.
    
    --remote-debugging-port=9222


MacOS:
    In a terminal, execute.
   
    /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
    

Linux:
    In a terminal, launch.
   
    google-chrome --remote-debugging-port=9222


NOTE: The port in the commands above, is the same port specified in port section of the launch.json above.

Restart Chrome and Visual Studio Code.

Open your app, start / serve your app and open it up in Chrome. Then press F5 in Visual Studio Code and you will be able to use the debug features as well as breakpoints!