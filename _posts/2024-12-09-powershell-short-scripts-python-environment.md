---
layout: post
title: PowerShell Short Scripts
subtitle: Python Environment Inspection
tags: [howto]
author: Matt Himrod
---

When dealing with Python dependencies, it’s sometimes necessary to determine if a given project is using a given version of a library. Since our team is using Pipenv to manage our Python dependencies, it’s not very difficult to determine which version of a given library is in a given project, but what can be tricky is actually searching through the Pipfile.lock file to find what you’re looking for, and if you need to look at multiple packages, well, this could take a while.

Fortunately, we know that PowerShell can easily chew through JSON files and pick out values. Here’s how we can easily print out the versions of all of our default and development dependencies:

```PowerShell
function Get-PythonEnvironment {
    $pipfile = Get-content .\Pipfile.lock | ConvertFrom-Json
    $default = $pipfile | Select-Object -ExpandProperty default
    $develop = $pipfile | Select-Object -ExpandProperty develop
    @($default,  $develop) | ForEach-Object {$_.PSObject.Properties `
        | ForEach-Object {$_ | Select-Object Name -ExpandProperty Value | Where-Object index -ne $null | Select-Object name, version}}
}
```

The output to this is similar to what you’d get from pip freeze except that only items appearing in the Pipfile.lock file will appear in this list. This can be helpful to quickly figure out if a new library was pulled in after locking a pipfile, for example.
