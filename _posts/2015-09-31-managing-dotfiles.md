---
layout: post
title: "Managing dotfiles like a boss"
date: 2015-09-30
author: Virendra Singh Bhalothia
tags: systems management automation dotfiles likeaboss best-practices
category: tutorial
excerpt: "Manage the most important files on your system, like a boss."
---

:metal:

Hello world! This post is all about managing your dotfiles like a boss.

![likeaboss][10]

I'm gonna talk about the importance of [dotfiles][6] and how to manage them effortlessly with version control systems. Due to whatever reasons, I changed 3 laptops in last few months and it was really hard for me to keep track of all my personal configuration files.

For example: `.bashrc`, `.gitconfig`,`.gitigignore`,`.vimrc` etcetera.

![dotfiles][11]

**I believe in putting absolutely everything in version control** and have introduced and enforced the same in every project I have worked for.  

Then **why not** my dotfiles? ~~I was doing something wrong.~~

I wanted to fix this. Badly.

> **Note**: If you spend any amount of time on terminal and want to automate the way you manage your various configuration settings, then this is a must have tutorial for you.

The problem is not versioning these files, but distributing across the workstations in use. Even if you make an interesting tweak in one of the git config files, it would either require you to copy that file or manually adding the same change in other locations, **which is a pain**.

I looked around and found that many people are using version control systems as a centralized source to backup, restore, and sync their dotfiles, just like anything else. I also found that a lot of **[github repositories][6]** with tons of useful dotfiles are available publicly. That is the power of sharing.

> **However, if you are just starting off, so much amount of information could be confusing as well. Go slow.**

Many people believe that dotfiles are meant to be forked, which is really good but you should know exactly what you are using.

### Playtime

I'm going to explain in detail how to manage your dotfiles in an automated and effortless fashion:

- Fork a publicly available dotfiles repository
- Customize it as per use
- Install it by creating symlinks
- Sync with the latest updates
- Push back changes as well

## Fork a publicly available dotfiles repository

We are going to put all your dotfiles into git,  which means you’ll be able to use them on any OS X or Linux machine with Internet access. Misconfiguring your setup would not be your worst fear then and if you know basic git, you can setup a new workstation in less than 2 minutes.

> I believe you already have some dotfiles in your home directory and would want to backup those before knowing and installing new dotfiles from github. Don't worry, it will be taken care of.

The typical location for dotfiles on an OS X or Linux machine would be in a users home directory, e.g. `/home/bhalothia/.vimrc`. Let's create a dotfiles folder in the home directory itself for better organization: `mkdir dotfiles`

Fork out a publicly available [repository][7], like I did. Clone it.

``git clone git@github.com:bhalothia/dotfiles.git ~/dotfiles``

## Customizing dotfiles

Fine-tune your prompt, carefully plan your aliases, and write some pretty time-saving functions. Modify the forked repo and customize the dotfiles accordingly.


## Installation

So, we will be putting all our dotfiles inside the `dotfiles` folder and we’ll create symlinks to them from our home directory.

```
cd ~/dotfiles
./makesymlinks.sh
```
This script would backup all your old dotfiles to a backup folder and create symlinks to the `dotfiles` folder. This is how it would look like if you do a `ls -alrt` in your home directory

```
.zshrc -> /Users/bhalothia/dotfiles/zshrc
.vimrc -> /Users/bhalothia/dotfiles/vimrc
.vim -> /Users/bhalothia/dotfiles/vim
```

## Syncing with latest updates

Due to improved github support for triangular workflows, it has become very easy to collaborate.

In open-source world, this is how it works:

- You fetch from a canonical "upstream" repository to keep your local repository up-to-date.
- When you want to share your own modifications with other people, you push them to your own fork and open a pull request.
- If your changes are accepted, the project maintainer merges them into the upstream repository.

Here's how I'm syncing with my upstream repo:

- Fork the [repo][7] under my namespace
- Clone it

```
git clone git@github.com:bhalothia/dotfiles.git ~/dotfiles
cd dotfiles
git config remote.pushdefault origin
git config push.default current
```

After this step, the remote called `origin` refers to my fork of dotfiles. It also sets the default remote for pushes to be `origin` and the default push behavior to `current`. Together this means that if I just type `git push`, the current branch is pushed to the origin `remote`.

- Create a second remote called `upstream` that points at the main `dotfiles` repository and fetch from it:

```
git remote add upstream git@github.com:michaeljsmalley/dotfiles.git
git fetch upstream
```

Sharing my `dotfiles` remote repositories:

```
  Virendras-MacBook-Pro: git remote -v
  origin  git@github.com:bhalothia/dotfiles.git (fetch)
  origin  git@github.com:bhalothia/dotfiles.git (push)
  upstream  git@github.com:michaeljsmalley/dotfiles.git (fetch)
  upstream  git@github.com:michaeljsmalley/dotfiles.git (push)
```

## Pushing back changes to upstream


- Make sure that your local repository is up-to-date with the upstream repository:

``git fetch upstream``

- Create a branch `new_feature` off of upstream master to work on a new feature, and check out the branch:

``git checkout -b new_feature upstream/master``

- This automatically sets up your local `new_feature` branch to track the upstream `master` branch. This means that if more commits are added to `master` upstream, you can merge those commits into your `new_feature` branch by typing ``git pull``.

- Push your `new_feature` branch to your fork: ``git push`` and create a pull request if you think the change should be pushed to the upstream repository.

 **Hope this helps! Keep forking.**


Credits: [John Glovier][8] for this beautiful [dotfiles logo][9]

  [1]: http://theremotelab.com
  [2]: https://www.linkedin.com/company/the-remote-lab
  [3]: https://www.facebook.com/TheRemoteLab
  [4]: https://github.com/TheRemoteLab
  [5]: https://twitter.com/TheRemoteLab
  [6]: https://dotfiles.github.io/
  [7]: https://github.com/michaeljsmalley/dotfiles
  [8]: https://twitter.com/jglovier
  [9]: https://dribbble.com/shots/1466768-dotfiles-logo
  [10]: http://38.media.tumblr.com/tumblr_lzpoecnb781rolf0zo1_500.gif
  [11]: /assets/img/blog/dotfiles/dotfiles_logo.png