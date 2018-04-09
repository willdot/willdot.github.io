---
title:  Git cheatsheet
author: Will Andrews
date: 2018-04-09
---

I'm fairly new to using git as a form of source control, so I'm always having to look up how to do things using the command line. So here are a few of the commands I use the most.

# Commit and pushing to Git 

Commit to local repo
```
git commit -a -m "Comments here"
```

Push to Git
```
git push
```

# Adding files

Add all files

```
git add *
```

Add a folder and it's contents
```
git add foldername/
```

Add a single files
```
git add filename.txt
```

# Branching

Create new branch locally by branching off another branch (or master)
```
git checkout <Branch to branch off of>

git checkout -b <New branch name>
```

Pushing new branch to Git
```
git push --set-upstream origin <New branch name>
```

Get new branches from Git
```
git fetch

git branch -add
```

# Checking out branches

```
git checkout <Branch name here>
```