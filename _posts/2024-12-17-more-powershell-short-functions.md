---
layout: post
title: More PowerShell Short Scripts
tags: [howto]
author: Matt Himrod
---

Sometimes it’s the little things…

PowerShell functions are prefect for little repetitive things that we do every few days. One of the things I find myself doing very frequently is creating a directory with the current date while processing daily errors or doing other daily tasks. Sure, this doesn’t take long to do by hand, but it’s also a very quick timesaving function that’s evolved to have a few neat features:

```PowerShell
function New-DateDirectory {
    [CmdletBinding(PositionalBinding=$false)]
    param (
        [Parameter()]
        [string]$Format = 'yyyy-MM-dd',
        [Parameter(ValueFromRemainingArguments)]
        [string]$Suffix
      )
    New-Item -ItemType Directory $(@($(Get-Date -Format $Format), $Suffix) -join " ")
}
```

The default date format can be overridden by specifying a valid parameter to `-Format`. Any other text provided on the command line will be added to the directory name.

Another timesaver that I’ve adopted more quickly than I expected is a script that finds the URL for the git repository in the current directory and opens it in the default browser. This one might require a bit more explanation:

```PowerShell
function Open-GitRepository { 
    git remote -v `
        | Select-String -Pattern '(?:https?://|git@)(.*)\.git \(fetch\)' `
        | ForEach-Object { 
            $url = $_.Matches.Groups[1].Value -replace ':', '/'
            Start-Process "https://$url" 
        } 
}
```

If the current directory is a Git repository, we can get the URL from which we cloned the repository by typing `git remote -v`. This command gives us the URLs where we fetch and any that we use to push; a given repository will have one fetch URL (the source of truth) and at least one push URL but can occasionally have multiple. Using this information, we can use `Select-String` with a handy Regular Expression to match the Git URL (http, https, and SSH URLs in this case) that has `(fetch)` after it. The result is a single-item list that we can pick apart and turn into a https URL.

So now, after I push a commit, I can easily run `Open-GitRepository` from the same PowerShell prompt to create my merge request!
