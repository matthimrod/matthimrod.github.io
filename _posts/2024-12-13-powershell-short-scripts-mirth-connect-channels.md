---
layout: post
title: PowerShell Short Scripts
subtitle: Mirth Connect Channel Inspection
tags: [howto]
author: Matt Himrod
---

As a Data Engineer using Mirth Connect, it’s often tricky to review my team's merge requests because the merge request document contains the XML representation of the channel. In order to understand what we’re looking at without loading the Mirth Connect Console, we have to be able to sight-inspect XML, which is not easy. Fortunately, there are a few things we can do in PowerShell to make this easier to read through, but it does require a copy of the file being reviewed.

First, we can check out the list of transformer steps in each of the destinations with some Select-Xml and Select-Object manipulation:

```PowerShell
Select-Xml  .\CHANNEL.xml -XPath '//channel/destinationConnectors/connector' `
     | Select-Object -Property @{Name='Destination'; Expression={$_.node.name}}, `
                               @{Name='TransformerStep'; Expression={$_.node.transformer.elements.ForEach({$_.ChildNodes.name.Where({$_ -ne '#whitespace'})})}} `
     | ConvertTo-Json
```

Note: that there’s a ConvertTo-Json at the end only to make the output more readable.

If we want to take a closer look at the transformer steps, we can do that as well:

```PowerShell
$connector="DATA_EXTRACTION"; 
Select-Xml .\CHANNEL.xml -XPath "//channel/destinationConnectors/connector[name='$connector']/transformer/elements/*" `
     | Select-Object -ExpandProperty Node
```

We can also take a look at the names of the filter steps:

```PowerShell
Get-Childitem .\CHANNEL.xml | ForEach-Object { $ChannelFile = $_.Name; Select-Xml $_ -XPath '//channel/destinationConnectors/connector' `
     | Select-Object -Property @{Name='Channel'; Expression={$ChannelFile}}, `
                               @{Name='Destination'; Expression={$_.node.name}}, `
                               @{Name='FilterStep'; Expression={$_.node.filter.elements.ForEach({$_.ChildNodes.name.Where({$_ -ne '#whitespace'})})}}} `
     | ConvertTo-Json
```

This obviously doesn’t show absolutely all of the details so it’s not a substitute for viewing the channel in the Mirth Connect Editor, but it can give us a way to take a quick look when we’re in a pinch.
