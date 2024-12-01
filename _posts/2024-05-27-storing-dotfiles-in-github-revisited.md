---
layout: post
title: Storing dotfiles in GitHub Revisited
subtitle: Using dotbot to bootstrap a dotfiles repository
tags: [howto]
author: Matt Himrod
---

A while ago, I wrote about using a bare git repository to manage my dotfiles. That worked somewhat well, but it had some shortcomings. Honestly, it's been a while since I switched from that particular setup, and I forget what my main motivating reasons were. I do vaguely recall running `config clean` one day... that removes any files in the profile that aren't in the repository, which would essentially delete my entire user profile... that *might* have had something to do with it.

Regardless of my motivations, I quickly adapted to using an open source application called [dotbot](https://github.com/anishathalye/dotbot) to help with the task becuase it just ended up being a better way to do things. First, instead of making a bare repository out of my entire user profile and ignoring files that are not under version control, all of the files are in a normal repository and symlinked to their original locations. Fortunately dotbot handles all of this more gracefully.* 

(* with a quick local security policy modification in Windows)

Since I already had my dotfiles set up to be shared, I only needed to set up dotbot. However, if you're new this sort of thing, you have to keep in mind the target machines that you use. For me, that includes machines from Raspberry Pi Zeros through my Windows 11 desktop and both work and personal. Keep in mind that these configurations include all of my user preferences. I want my shell key bindings to be the same at work and home, for example, so things have to play nice on both sides of that aisle. Fortunately, that's not too difficult.

## Know Your Environment

The user environment variables can give a lot of hints that can set things for work or home. I want certain things to be the same between work and home, but not everything. However, I want things to be the same between workstations at work and between computers at home. Fortunately, I can look at an environment variable for a hint. In this case, `$ENV:USERDOMAIN` is very helpful:

```
if($IsWindows -and
   ($ENV:COMPUTERNAME -ne $ENV:USERDOMAIN) -and 
   (Test-Path -Path (Join-Path $ENV:USERPROFILE ".dotfiles" "powershell" "profile.$ENV:USERDOMAIN.ps1"))) {
    . $(Join-Path $ENV:USERPROFILE ".dotfiles" "powershell" "profile.$ENV:USERDOMAIN.ps1")
}
```

This excerpt will source in a separate `profile.WORKDOMAIN.ps1` file with some work-specific functions.

## Windows Modification

Everything works out-of-the-box as documented except for one thing -- in Windows, one needs Administrator rights to create symbolic links. Fortunately, this is pretty easy to overcome. 

1. Open Local Security Policy. It's easiest to search for it, but it can be found under Windows Tools in the Windows 11 Programs menu.
2. In the left pane, navigate under Security Settings > Local Policies > User Rights Assignment. On the right pane, find and open "Create symbolic links."
3. In the "Create symbolic links properties" window that opens, click "Add User or Group..."
4. In the "Select Users or Groups" box that opens, type "Authenticated Users" in the text box and click OK.
5. Close Local Security Policy and restart the computer for the changes to take effect. 
