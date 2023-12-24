---
layout: post
title: Understanding git worktree
---

# Understanding git worktree

Even though extremely useful, git worktree remains an obscure and poorly understood git feature. In this post we will explore git worktrees and showcase how they can improve your git workflow, for example for managing your dotfiles.

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

Per default `git init` or `git clone` initializes a worktree in the repo's directory, along with metadata under the `.git` subfolder.

<img src="/assets/img/posts/git_worktree/always_has_been.jpg" width="600" height="auto">

# Multiple Worktrees

We can list the worktrees in a repo using the `git worktree list` command:

```sh
> cd my_repo
> git worktree list
/tmp/my_repo  84424a2 [master]
```

Our repo has a single worktree located in the folder `/tmp/my_repo` with the branch `master` checked out.

But can we have more than a worktree?

yes! even though most of the time we work on repos containing of a single worktree, a repo can have any number of worktrees, each one with a different branch checked out.

## Bare Repos

When I say that a git repo can have any number of worktrees, I mean it.
In fact, a git repo can have even _zero_ worktrees, this is known as a *bare repo*.

We can clone a bare repo by passing the `--bare` flag to `git clone`:

```sh
> git clone --bare git@github.com:pabloariasal/my_repo.git
```

Let's inspect the contents of the bare repo we just cloned:

```sh
> ls my_repo.git
branches  config  description  HEAD  hooks  info  objects  packed-refs  refs
```

This are the same contents of the `.git` folder above!

A bare repo is a repo that contains only metadata, but no files to work on, it has no worktree!

A bare repo behaves to some extend like a normal git repo, one can perform some usual git commands:

```sh
> cd my_repo.git
> git log --oneline
84424a2 (HEAD -> master) Initial commit

> git branch my_branch
> git branch --all
  my_branch
* master
```

One can created and list branches, now let's try to clone `my_branch` in our bare repo:

```sh
> git checkout my_branch
fatal: this operation must be run in a work tree

> git add --all
fatal: this operation must be run in a work tree
```

We can not really checkout a branch, nor stage any files, we need a worktree for that.

Let's fix that, let's add a worktree to our bare repo!

# Adding Worktrees

`git worktree add` can be used to add a worktree to an existing git repo (it doesn't need to be a bare repo).

```sh
> cd my_repo.git
> git worktree add ../my_worktree master
Preparing worktree (checking out 'master')
HEAD is now at 84424a2 Initial commit
```

`git worktree add` initializes a worktree and adds it to the repo. It receives two arguments: the path of the worktree (this is the folder where the worktree will be initialized) and the branch or commit that defines the contents of the worktree.

```sh
> git worktree list
/tmp/my_repo.git                   (bare)
/tmp/my_worktree                   84424a2 [master]
```

We see that the repo has a worktree located in `/tmp/my_worktree` and has the branch `master` checked out.
We can also see that the metadata of the repo is located at `/tmp/my_repo.git` (listed as `(bare)`).

Now we can add more worktrees to our repo:

```sh
> git worktree add ../my_second_worktree master
Preparing worktree (checking out 'master')
fatal: 'master' is already checked out at '/tmp/my_worktree'
```

Oops, the master branch is already checked out in an existing worktree!
You can only have the same branch checked out once in a given repo.

Let's try again, but this time we create a new branch to be checked out in the new worktree.
This can be done by passing the `-b` to `git worktree add`  flag:

```sh
> git worktree ../my_second_worktree -b some_branch
Preparing worktree (new branch 'some_branch')
HEAD is now at 84424a2 Initial commit
```

Now our repo contains two worktrees:

```sh
> git worktree list
/tmp/my_repo.git                         (bare)
/tmp/my_worktree                         84424a2 [master]
/tmp/my_second_worktree                  84424a2 [some_branch]
```

We have two branches checked out in two different worktrees: `master` in `/tmp/my_worktree` and `some_branch` in `/tmp/my_second_worktree`. Both branches belong to the same repo.

# Removing Worktrees

worktrees can be removed with `git worktree remove`:

```sh
> cd my_repo.git
> git worktree remove ../my_second_worktree
```

```sh
> git worktree list
/tmp/my_repo.git                      (bare)
/tmp/my_worktree                      84424a2 [master]
```

# Worktrees in Practice

Now that we have understood what git worktrees are and how to use them, which problems do they solve? Next I'll present two usages of worktrees that have greatly improved my daily git workflow:

## The hotfix problem

git worktrees solve one of the most annoying limitations of git: not being able to checkout multiple branches a at the same time.

We all know this situation:

You are working on a feature branch and then your teammate calls you saying that something you committed in the past has broken the build.
You have to check out the master branch, reproduce the problem locally and create a hotfix solving the problem.

Here you have are two options:

1. commit the all of yuour change to the feature branch, and checkout master
1. stash your changes, switch to `master` and when you are done apply them again.

Both options are annoying, specially if you work on many different branches at once daily.

With git worktrees this problem is solved: just create a new worktree for the hotfix branch in a different folder and do all the fixes there, your current work will remain untouched.

After you are done with the fixes just remove the worktree.

## Detached Worktree - Managing dotfiles

By using bare repos and git worktrees you can separate the contents of the repo from the metadata itself. This means that the files can be in a folder, but the metadata in another one. This can be very useful for managing dotfiles.

If you are like me, you probably store all your configuration files in a git repo (we spent too many hours configuring our systems for these files to be lost).

The problem is that the dotfiles must be placed inside the home folder, something like `~/.config/my_config.conf`. So how can we solve this? One can for example initialize a git repo inside of the home folder, but this causes countless issues. Many people use [GNU Stow]() to symlink files from the dotfiles repo to the home folder.

There is another way to manage your dotfiles using git worktrees:

1. clone your dotfiles repo as bare somewhere in the filesystem
2. Checkout the worktree in your home folder.

A detailed explanation of how to achieve this can be found [here](https://www.atlassian.com/git/tutorials/dotfiles).
