---
layout: post
title: Storing dotfiles in GitHub
subtitle: Using a bare repository for dotfiles
tags: [howto]
author: Matt Himrod
excerpt: >
  While I don't use Linux as my primary operating system, I do use it a lot both at home and at work. It can be frustrating to try to sync up configurations between different hosts or even just figuring out how to make a new laptop work like an old one. I remembered that some coworkers at my old job had some way of storing all of their dotfiles in Github to make this easier. It took a bit to figure out the best way to make this work.
---

## Using a bare repository for dotfiles

While I don't use Linux as my primary operating system, I do use it a lot both at home and at work. It can be frustrating to try to sync up configurations between different hosts or even just figuring out how to make a new laptop work like an old one. I remembered that some coworkers at my old job had some way of storing all of their dotfiles in Github to make this easier. It took a bit to figure out the best way to make this work.

First, I use a bare git repository. Bare repositories are typically used for sharing rather than working because they don't store a working directory. In this case, we're going to use it because we don't need all of features of a full repo. We're just making it easier for us to store these files and deploy them to other machines.

I'm going to create another post about my Raspberry Pi setup script that I use to set up a new pi. It includes the initial setup of the dotfiles repo on a new machine from an existing repo.

## Creating the Repository

First, we have to create our bare repository. We have to change things around so that the working directly is the home directory and the git files are stored in a hidden folder (normally it's the other way around), and we have to tell git not to show tracked files. We'll do this as follows:

```bash
git init --bare $HOME/.cfg
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
config config --local status.showUntrackedFiles no
```

This also creates an alias config that we'll use to interact with the repo. We'll need to make this permanent by adding it to .bashrc as follows:

```bash
echo "alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'" >> $HOME/.bashrc
```

Once this is done, you can version the files in the home directory with normal git commands, except you use the config alias for this instead of git. For example:

```bash
config add .bashrc
config commit -m "Added config alias to .bashrc"
config push
```

You'll need to push your repo to Github or whatever site you're using. You'll need the URL for your repo from there for the next steps. 

```bash
config remote add origin <remote repository URL>
config push
```

## Installing on a New Computer

Once you have your files in a git repository, you'll need to manually create the alias to the config command that we created above. Note that you can also just add the --git-dir and --work-tree parameters to each command if you have your .bashrc in your repo and you don't want to edit it manually. I have a script for deploying a new Raspberry Pi image that does this that I'll cover in a later post. Either add those parameters or make sure this alias is defined in your current shell scope by running this command at the shell.

```bash
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
```

You'll also need to add a .gitignore file to avoid weird recursion issues:

```bash
echo ".cfg" >> .gitignore
```

Next, clone the repo into a bare repository in a hidden "dot" directory:

```bash
git clone --bare <git-repo-url> $HOME/.cfg
```

Check out the repo. You may run into an error if you have existing files already in your home directory that are also in the repo. Git doesn't like to overwrite files.

Finally, you'll want to set the showUntrackedFiles flag to no for this repository. That way you won't see every file in your home directory when you're working with the repo:

```bash
config config --local status.showUntrackedFiles no
```

## Summary

That's it! I find this particularly helpful when I set up a new Raspberry Pi, but I also use it when I migrate to a new computer. I keep my .ssh/authorized_keys file in my repo so that when I change keys or add a new one, I just have to update the repo on all of my machines by running this:

```bash
config pull
```
