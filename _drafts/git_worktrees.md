---
layout: post
title: Understanding git worktree
---

# Understanding git worktree

Even though extremely useful, git worktree remains an obscure and poorly understood git feature. In this post we will dive into git worktree and showcase how it can improve your git workflow.

# What is a git worktree?

We must begin by answering the most fundamental question: what is even a git worktree? 

Funny that you ask, you have been using git worktrees the whole time!
If we clone a repo and inspect its contents:

```sh
> git clone git@github.com:pabloariasal/my_repo.git
> ls -a my_repo
.git
LICENSE
README.md
```

we can see two different things contained within:

1. a hidden folder called `.git`
2. multiple files like `LICENSE` and `README.md`

The `.git` folder contains the git metadata. Every git repo has one and it contains all the information used by git: branches, commits, config, remotes, etc. This directory is transparent to the user and should never be manipulated directly (or you screwed up really bad).

```sh
> ls -a .git
branches  config  description  HEAD  hooks  index  info  logs  objects  packed-refs  refs
```

Besides the `.git` folder, the git repo also contains some files like `LICENSE` and `README.md`: the data itself. These are the files tracked by the repo: the files we *work on*, this is the git worktree!

Per default a freshly clone or inited git repo consists of two parts: a `.git` folder and a worktree consisting of the files and directory we work on.

<img src="/assets/img/posts/git_worktree/always_has_been.jpg" width="600" height="auto">

# Bare repos

We can list the worktrees in a repo using the `git worktree` command:



