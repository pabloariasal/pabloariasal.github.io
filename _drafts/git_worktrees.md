---
layout: post
title: Understanding git worktree
---

# Understanding git worktree

Even though extremely useful, git worktree remains an obscure and poorly understood git feature. In this post we will dive into git worktree and showcase how it can improve your git workflow.

# What is a git worktree?

But what is even a git worktree?

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

Per default a freshly clone (or inited) git repo consists of two parts: a `.git` folder containing metadata and a worktree with the files and directories tracked by the repo.

<img src="/assets/img/posts/git_worktree/always_has_been.jpg" width="600" height="auto">

# Multiple Worktrees and Bare Repos

We can list the worktrees in a repo using the `git worktree list` command:

```sh
> cd my_repo
> git worktree list
/tmp/my_repo  84424a2 [master]
```

Our repo has a single worktree located in `/tmp/my_repo` with the branch `master` checked out.

But can we have more than a worktree?

yes! even though most of the time we work on repos containing of a single worktree, a repo can have any number of worktrees (each one with a different branch checked out).

## Bare Repos

When I say that a git repo can have any number of worktrees I mean it,
a repo can have even _zero_ worktrees, this is known as a *bare repo*.

We can clone a bare git repo with the `--bare` flag:

```sh
> git clone --bare git@github.com:pabloariasal/my_repo.git
```

Let's inspect the contents of the bare repo we just cloned:

```sh
> ls my_repo.git
branches  config  description  HEAD  hooks  info  objects  packed-refs  refs
```

This are the same contents of the `.git` folder above!
This is a repo that contains only the metadata, but no files to work on. It contains no worktree!

```sh
> cd my_repo
> git worktree list
/tmp/my_repo.git  (bare)
```

A bare repo behaves to some extend like a normal git repo:

```sh
> git log --oneline
84424a2 (HEAD -> master) Initial commit

> git branch
* master
```

However some commands will output an error:

```sh
> cd my_repo.git
> git status
fatal: this operation must be run in a work tree
```

A bare repo is a git repo that contains only metadata, it has no data we can actually work with... so why is it useful? what can I do with it?

We can add worktrees to it!

# Adding worktrees

# Advantages

## Detached worktree (dotfiles)
## Multiple branches
