+++
title = "Git Fundamentals"
date = "2025-05-13T09:34:58-04:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Abhinav Chavali"
authorTwitter = "" #do not include @
cover = ""
tags = ["git", "freebsd"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

Simple cheatsheet for me to get started tracking freebsd and making changes to the codebase. Requirements: `git`, `git arc`

## Tracking the Source Tree

### Copy local repository
```sh
git clone git@github.com:dumrich/freebsd-src.git
```

### Set Upstream
Upstream is the original codebase, usually set as a remote repository
```sh
git remote add upstream https://git.freebsd.org/src.git
```

### Show recent commits
Show a list of recent commits with some other information

```sh
git log
```

### Checkout a branch
You want to work on a feature locally. Create a branch off main and work on feature.

```sh
git checkout -b <feature_name>
```
### Sync local repository with upstream (main)
Upstream made X commits since last merge

```sh
git fetch upstream # Fetch all new upstream tree
git merge upstream/main # Only merge if it is a fast forward
```

or

```sh
git pull upstream main
```

These pull into the current branch

#### Rebase vs Merge
Merge preserves the original commit history, making a nonlinear history with merge commits.

Rebase rewrites your branch history by moving commits of a branch onto a new base. This means that if upstream is ahead by 10 commits, rebase will let you move your commits to the top of the main branch, creating a simple linear history.

### Push commits to local origin
Synchronize your changes on your remote fork
```sh
git push origin vmm-mods
git push --force origin vmm-mods  # After rebase
```

### Interactive Rebase
An interactive rebase allows you to modify your branches commit history by editing commit messages, combining (squash), reordering, or deleting commits.

It's a very versatile command that rewrites the history of the branch.

```sh
git rebase -i # Interactive rebase on branch
```

### Cherry Pick
*TODO*

### Bisect
*TODO*

## Contributing Changes to Phabricator
You must create a diff and upload it to phabricator. Make sure commit history is linear (rebase) and you've installed certificate.

```sh
git-arc stage
git-arc diff
```


## Sources
FreeBSD git p1: https://www.youtube.com/watch?v=BRACcRqgnWQ
FreeBSD git p2: https://www.youtube.com/watch?v=Fe-dJrDMK_0