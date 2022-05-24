---
title:  I wrote my first Go app and what I learnt
author: Will Andrews
date: 2019-06-11
order: 25
---

When I'm working with multiple branches in Agile projects, I found I was typing git checkout, git commit, git push far too much. And then there the remembering the branch name that I was wanting to checkout (some of them were so long as well......). I decided that I didn't want to use alias, but instead put my knowledge of Go that I had learnt and put it to use. So I wrote a command line app that allows quick shortcuts to be used to do the commands I did the most. And that's how uGit was born.

## What it does

Instead of typing "git checkout feature/someFeatureHere", after possibly doing "git branch" to remember the name of the branch, I wanted to be able to type something shorter. So instead, I now type "ugit cko". So ugit as the application name, and then cko as a parameter to say I want to checkout. But instead of typing a branch name, I wanted to be able to see a list of all my branches, and then use arrows to select the branch I wanted. I then got greedy and wanted to be able to filter by name.

## What did I use?

I started writing an app in Go, and found out that there was a library that had been written called Cobra, that allowed me to create my own CLI commands. That was pretty neat. Once I had played around with it for a while, I had a simple working app that allowed me to type a command, and it did the git checkout feature/BranchHere that I wanted. 

Then I stumbled across this package called Suvery, which someone had written which allowed me to prompt the user for input. This was handy for listing all branches that I could select, and then using the arrow keys to select and checkout the branch I wanted. BONUS, it also had filtering!! So I plugged that in, and had a fiddle with some config and got it working. 

## Testing?

From the get Go (pun intended), I decided that I was going to write this application with TDD. I had recently done a course in TDD, so was keen to try it out. I found it so much easier to produce readable and simplistic code, that made sense and wasn't bloated. And as a bonus, I came across far fewer bugs than I was expecting.

## It works!

I use this application every day at work and it's a life saver. I even added in the ability to commit with a shortcut, and be presented with a list of files that were not staged for commit, and I could select which ones I wanted to stage. Then I could enter a commit message and bish, bash, bosh it was done. Simples.

## What did I learn?

* The importance of TDD.

* Use open source libraries if available. If they don't do what you want, of there's a problem, fork it, fix it, and pull request.

* I may have used the package feature of Go a bit too much, and probably didn't need to split everything out into as many packages as I did.

* Go is such a fun language to write in.



Source code found here [uGit project](https://github.com/willdot/uGit) 

Feel free to use it, raise any issues or contribute to the project! Next I want to add in the ability to do merges...that sounds like a fun challenge!