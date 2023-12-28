---
layout: post
title: >
  Boost Your Workflow: Exploring Git Worktrees
image:
  path:    /assets/img/posts/git_worktree/worktree.png
  srcset:
    960w: /assets/img/posts/git_worktree/worktree.png
    480w: /assets/img/posts/git_worktree/worktree.png@0,5x.png
    240w: /assets/img/posts/git_worktree/worktree.png@0,25x.png
---

# Boost Your Workflow: Exploring Git Worktrees

Last week I bought a Christmas tree. It's not the tallest, nor the bushiest, the tip is kinda crooked, but it gets the work done. While carrying my new acquisition through the snowy streets, the comforting smell of freshly cut pine reminded me of how much I like trees, specially git worktrees.

# But, what is even a git worktree?

Funny that you ask, you have been using git worktrees the whole time!

If we clone a repo and inspect its contents:

```
$ git clone git@github.com:pabloariasal/my_repo.git
$ ls -a my_repo
.git
LICENSE
README.md
```

We can see two different things contained within:

1. A folder called `.git`
2. Files like `LICENSE` and `README.md`

The `.git` folder contains the git metadata. Every git repo has one and it contains all the information needed by git to work: branches, commits, config, remotes, etc. This directory is hidden to the user and should never be manipulated directly (unless you screwed up really bad).

Let's see what's inside `.git`:

```
$ ls -a .git
branches  config  description  HEAD  hooks  index  info  logs  objects  packed-refs  refs
```

Besides the `.git` folder, the git repo also contains some files like `LICENSE` and `README.md`: the actual contents of the repo. These are the files and directories tracked by git: the files we *work on*, this is the git worktree!

Per default `git init` or `git clone` initializes a worktree in the repo's directory, along with metadata under the `.git` subfolder. Hence each repo has one worktree per default.

<img src="/assets/img/posts/git_worktree/always_has_been.jpg" width="700" height="auto">

# Work~~tree~~forest

We can list the worktrees in a repo using the `git worktree list` command:

```
$ cd my_repo
$ git worktree list
/tmp/my_repo  84424a2 [master]
```

Our repo has a single worktree located in `/tmp/my_repo` has the contents of the branch `master` checked out.

But can we have more than a worktree?

Yes! Even though most of the time we work on repos containing of a single worktree, a repo can have any number of worktrees, each one with a different branch checked out.

## Bare Repositories

When I say that a git repo can have any number of worktrees, I mean it.
In fact, a git repo can have zero worktrees, this is known as a *bare repo*.

We can clone a bare repo by passing the `--bare` flag to `git clone`:

```
$ git clone --bare git@github.com:pabloariasal/my_repo.git
```

Let's inspect the contents of the bare repo we just cloned:

```
$ ls my_repo.git
branches  config  description  HEAD  hooks  info  objects  packed-refs  refs
```

This are the same contents of the `.git` folder above!

A bare repo is a repo that contains only metadata, but no files to work on, it has no worktree! You are essentially cloning the `.git` folder without any tracked files.

A bare repo behaves to some extend like a normal git repo, one can perform some usual git commands:

```
$ cd my_repo.git
$ git log --oneline
84424a2 (HEAD -> master) Initial commit

$ git branch my_branch
$ git branch --all
  my_branch
* master
```

One can created and list branches, now let's try to checkout the branch `my_branch` in our bare repo:

```
$ git checkout my_branch
fatal: this operation must be run in a work tree

$ git add --all
fatal: this operation must be run in a work tree
```

We can not really checkout a branch, nor stage any files, we need a worktree for that.

Let's fix that, let's add a worktree to our bare repo!

# Creating Worktrees

The command `git worktree add` can be used to add a worktree to an existing git repo (it doesn't need to be a bare repo).

```
$ cd my_repo.git
$ git worktree add ../my_worktree master
Preparing worktree (checking out 'master')
HEAD is now at 84424a2 Initial commit
```

`git worktree add` initializes a worktree and adds it to the repo. It receives two arguments: the path of the worktree (this is the folder where the worktree will be initialized) and the branch or commit that defines the contents of the worktree.

```
$ git worktree list
/tmp/my_repo.git                   (bare)
/tmp/my_worktree                   84424a2 [master]
```

We see that the repo has a worktree located in `/tmp/my_worktree` and has the branch `master` checked out.
We can also see that the metadata of the repo is located at `/tmp/my_repo.git` , listed as "`(bare)`".


## Adding More Trees

Now we can add more worktrees to our repo:

```
$ git worktree add ../my_second_worktree master
Preparing worktree (checking out 'master')
fatal: 'master' is already checked out at '/tmp/my_worktree'
```

Oops, the master branch is already checked out in an existing worktree!
You can only have the same branch checked out once in a given repo.

Let's try again, but this time we create a new branch to be checked out in the new worktree.
This can be done by passing the `-b` to `git worktree add`  flag:

```
$ git worktree ../my_second_worktree -b some_branch
Preparing worktree (new branch 'some_branch')
HEAD is now at 84424a2 Initial commit
```

Now we have a small forest, our repo contains two worktrees:

```
$ git worktree list
/tmp/my_repo.git                         (bare)
/tmp/my_worktree                         84424a2 [master]
/tmp/my_second_worktree                  84424a2 [some_branch]
```

We have two branches checked out in two different worktrees: `master` in the worktree at `/tmp/my_worktree` and `some_branch` in the one at `/tmp/my_second_worktree`.

# Removing a Worktree

Worktrees can be removed with `git worktree remove`:

```
$ cd my_repo.git
$ git worktree remove ../my_second_worktree
```

```
$ git worktree list
/tmp/my_repo.git                      (bare)
/tmp/my_worktree                      84424a2 [master]
```

# Worktrees in Practice

Now that we have understood what git worktrees are and how to use them, which problems do they solve? Next I'll present two usages of worktrees that have greatly improved my daily git workflow:

## The Hotfix Problem

Git worktrees solve one of the most annoying limitations of git: not being able to checkout multiple branches a at the same time.

We all know this situation:

You are working on a feature branch and then your teammate calls you: something you committed in the past has broken the build.
You have to check out the master branch, reproduce the problem locally and create a hotfix in a new branch.

Now you have two options:

1. Commit all the (unfinished) current changes to the feature branch and switch to the `master` branch
1. Stash your changes, switch to `master` and then apply the changes again when done.

Both options are annoying, specially if you work on many different branches at the same time daily.

With git worktrees this becomes a lot easier: just create a new worktree for the hotfix branch! All the fixes are done in a new folder and the current work remains untouched.

After you are done with the hotfix, remove the worktree and continue working right were you left off, the work you were doing will be intact.

## Detached Worktree - Managing dotfiles

Bare repos and worktrees separate the contents of the repo from its metadata. This can be very useful for managing dotfiles.

If you are like me, you probably store all your configuration files in a git repo (we spent too many hours configuring our systems for these files to get lost). The problem is that the dotfiles must be placed inside the home folder, which kinda forces you to make you home folder a git repo, which something you probably don't want. 

So how can we solve this?

Many people use [GNU Stow](https://www.gnu.org/software/stow/) to symlink files from the dotfiles repo to the home folder, but there is another way:

1. Clone your dotfiles repo as bare somewhere in the filesystem
2. Checkout the worktree in your home folder

A detailed explanation of how to achieve this can be found [here](https://www.atlassian.com/git/tutorials/dotfiles).

# The End

- A git worktree consists of the files we work on
- A repo can have any number of worktrees (even zero)
- Two different branches can be checked out in two different worktrees (in different folders)
- A branch can only be checked out once for every repo

Thank you for reading!

# Discussion

[![](/assets/img/reddit_large.png)](https://www.reddit.com/r/programming/comments/18sp10i/boost_your_workflow_exploring_git_worktrees/)
