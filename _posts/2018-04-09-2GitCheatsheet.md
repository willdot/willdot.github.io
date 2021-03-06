---
title:  Git cheatsheet
author: Will Andrews
date: 2018-04-10
order: 2
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
<br/>
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
<br/>
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
```

Get latest branch list from Git
```
git remote update origin --prune
```

Delete local branch
```
git branch -d <Branch Name> 
```
(use -D to force)
<br/>
# Checking out branches

```
git checkout <Branch name here>
```

# Undoing commits

You've made some changes and then committed, but want to undo that commit and keep working with the changes, use:

```
git reset --soft HEAD~1
```

You've made some changes and committed, but you want to discard tha commit and the changes, use:

```
git reset --hard HEAD~1
```

In each of the commands, the number at the end can be replaced with the amount of commits you wish to undo.
