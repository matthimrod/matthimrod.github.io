---
layout: post
title: FOSS (Free and Open Source Software) Trivia of the Day
subtitle: a.k.a. Why I Don't Use Nano
tags: [texteditor, trivia]
author: Matt Himrod
excerpt: a.k.a. Why I Don't Use Nano
---

## a.k.a. Why I Don't Use Nano

I've been on a "kids these days don't use command-line editors... get off my lawn..." kick lately...

Here's some trivia. Many tutorials and how-to documents are written to use the really user-friendly `nano` text editor. Back when I was learning Linux, `nano` was really new, not as widely ported as it is today, and even if available, it wasn't always in default install bundles. Back then, the equivalent (almost literally) was an editor called `pico` from which from which `nano` was cloned. `Pico` was created as the editor component of the [Pine email client](https://en.wikipedia.org/wiki/Pine_(email_client)).

Pine was created by the University of Washington and published under the BSD license until version 3.91. The BSD license is a free software license whose only requirement is that the license text be included with any distributions of the source code or binaries of the software. It does not require that the source code be distributed, so it's not *technically* an open-source license. However, it is a license and a classification for licenses that are similar to- or based on- the BSD license.

In the mid-90s, the University of Washington registered a trademark for the name Pine and included the trademark in their license along with language that no longer allowed modifications to the code. They also retroactively claimed that because of the trademark, even though the prior distributions were under the BSD license, any modifications or distributions that were not from UW had never been allowed.

In response, open-source developers created a fork that was adopted by the GNU project called MANA (Mail and News Agent). However, UW threatened to sue the Free Software Foundation, so the project was never released.

Amid all of the confusion about whether to use `nano` or `pico` on the command line, I ended up learning `emacs`, which was not ideal because it's a behemoth, also not installed by default (hence the old saying that `emacs` stands for Eight Megs And Constantly Swapping... back when 8 MB was a lot of RAM), and also very tricky to exit if you don't know what you're doing. I ended up getting myself stuck in `vim` often enough to give in and learn how to get into and out of insert mode, save, exit, and exit without saving. I still have to look up how to yank and paste, but I know 3 different ways to get into insert mode!
