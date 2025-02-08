---
layout: post
title: Making PowerShell "Posh"
tags: [howto]
author: Matt Himrod
---

Back in the days when DOS and Windows were separate programs, the default Command Prompt didn't
look exactly the way that it does now. I can't recall if it was DOS 5 or DOS 6 that changed the
default prompt to `$P$G` ("path" followed by "greater than") which looks like the familiar prompt
that we're accustomed to: `C:\Users\matt>` -- that is, the current working directory (path)
followed by a "greater than" sign.

Back then, DOS had a "boot script" called `AUTOEXEC.BAT` that would set up the initial programs
that supported all of the computer's peripherals where you'd define your prompt. When I had
the older version of DOS, one of the first things I'd do is edit the autoexec.bat and add the
`PROMPT $P$G`, which is the familiar path prompt instead of the original default, which was just
the current drive and ghe "greater than" sign: `C>`. I think there were some ways to get really
fancy with the prompt using ANSI cursor movement and color control sequences, but displays at the
time were both small and slow -- fixed at 80 columns and 25 rows in most cases -- there wasn't much
room to get fancy. Now we have terminal windows and high resolution displays that can give us a lot
of real estate to play with.

Enter Oh My Posh! -- Posh both because **PO**wer **SH**ell and because it's fancy.

![](/assets/img/2025-02-08-making-powershell-posh.png)

THe example above is my own Oh My Posh custom prompt. It shows my current Git repository status
followed by the currently logged-in username (which is green because I'm signed in as a regular
user but would be red if I were elevated to superuser). Not shown currently are my currently
selected AWS Profile (fron my AWS_PROFILE environment variable) and my current Python virtual
environment name and version. Finally, I have a portion of the current working directory, which is
the full working directory here because I'm fewer than 3 levels deep.

Oh My Posh works best with a font that supports Power Lines and glyphs such as Cascadia Code PL or
one of [Oh My Posh's Fonts](https://ohmyposh.dev/docs/installation/fonts), but there are profiles
that don't use the special glyphs in these fonts.

The installation instructions are here: [Oh My Posh Windows Installation](https://ohmyposh.dev/docs/installation/windows)

There are [themes](https://ohmyposh.dev/docs/themes) to get you started. My theme is a custom one
based on an older version of the [agnoster](https://ohmyposh.dev/docs/themes#agnoster) theme.
