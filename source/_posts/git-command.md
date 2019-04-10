---
title: git_command
date: 2019-04-10 21:56:46
tags: ForRemember
---
### git config
```
git config --local  
git config --global  
git config --system  
```
`git config --global user.name 'your_name'`  
`git config --global user.email 'your_email'`  
`git config --list --local`  
`git config --local user.name`  
### git init
`git init`  
`git init your_project && cd your_project`  
`git add`  
`git commit -m'your_comment'`  
`git commit -am'your_comment'`  
modify last commit info
`git commit --amend`  
`git status`  
`git log`
### git rename
```
mv file1 file2
git add file2
git rm file1
```
or  
`git mv file1 file2`  
### git discard staging
`git reset --hard`  
### git log
`git log --oneline`  
`git log -n2`  
`git log --all`  
`git log --all --graph`
### git rebase  
`git rebase -i commit_hash`    
### git branch  
create new branch and go into new branch  
`git checkout -b new_branch_name commit_hash`  
`git branch -v`  
`git branch -av`  
remove branch  
 `git branch -d branch_name`  
 `git branch -D branch_name`  

### git help
`git help --web log`  

`git diff HEAD HEAD^`  
`git diff HEAD HEAD~1`

### git advance
there are 3 type (commit|tree|blog)
`git cat file -t hash`  
`git cat file -p hash`

### detached HEAD
happens when you checkout a commit without make it a branch, git will not persist any change without a branch  
`git checkout commit_hash`

### git GUI
`gitk -all`
