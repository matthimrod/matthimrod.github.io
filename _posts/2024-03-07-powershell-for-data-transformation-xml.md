---
layout: post
title: 'PowerShell for Data Transformation: XML'
subtitle: PowerShell makes more quick data transformations easy
tags: [powershell, code]
author: Matt Himrod
---

#### PowerShell makes more quick data transformations easy

This one bugged me for a few days. I don't think this script is the best solution to the problem at hand, but it was still fun to figure out.

We use an Interface Engine for a lot of our data feeds because they originate as HL7, and Interface Engines are the easiest way to deal with streams of HL7 messages. Under the hood, the interface engine uses a SQL engine, and some of the underlying data tables have fields that contain XML data. Most of it is reporting or metrics data - in this case, the process that prunes successfully processed records after a day.

In other words, given a SQL query with XML in one of the fields, convert the XML to fields that can be written to CSV, Excel, or a database row.

I plan to talk more about `Invoke-Sqlcmd` in the future, so for now, the cmdlet runs the SQL command and returns the results as a PowerShell Object. I guess that's what it says on the label.

```
Invoke-Sqlcmd -TrustServerCertificate -ServerInstance <Server> -Database MirthDB -Query "SELECT ATTRIBUTES FROM EVENT WITH (NOLOCK) WHERE NAME = 'Data Pruner'"
```

This returns rows that look something like this:

```
<map>
  <entry>
    <string>Message Date Threshold</string>
    <string>Sun Mar 03 01:00:00 EST 2024</string>
  </entry>
  <entry>
    <string>Messages Pruned</string>
    <string>1338</string>
  </entry>
  <entry>
    <string>Content Rows Pruned</string>
    <string>14718</string>
  </entry>
  <entry>
    <string>Channel ID</string>
    <string>1feb8cdf-4220-497a-9a50-1780232b64a9</string>
  </entry>
  <entry>
    <string>Channel Name</string>
    <string>MY_HL7_CHANNEL</string>
  </entry>
  <entry>
    <string>Time Elapsed</string>
    <string>0 minutes, 7 seconds</string>
  </entry>
</map>
```

Unfortunately, XML isn't as straightforward as CSV or JSON. XML has features like XPath, XQuery, XSLT, and such. There is probably a better way to do this... but here goes... In order to keep each entry together and not get the whole thing gobbled up, I piped my set of fields from the database through `Select-Xml`, which lets me apply an XPath to the record and parse out a Node. I could have probably just given an XPath of `/` to do a basic parse on this document, but if I give it `/map`, then my node starts with the entries. Each record from `Select-Xml` has a Node with the actual data and some parting information that I don't care about, so I can just expand the Nodes.

```
Invoke-Sqlcmd -TrustServerCertificate -ServerInstance <Server> -Database MirthDB `
              -Query "SELECT ATTRIBUTES FROM EVENT WITH (NOLOCK) WHERE NAME = 'Data Pruner'" `
| Select-Object -ExpandProperty Attributes `
| Select-Xml -XPath "/map" `
| Select-Object -ExpandProperty Node 
```

The output here is still not very useful. It's a bunch of entries with entries, and everything is truncated, but it's all there. Each entry contains an array of entry entries each having a list of strings, which are key-value pairs. The object is a little weird in structure. I had to do a lot of examination in half-written `ForEach-Object` blocks or using `Select-Object -First 1`, which doesn't sound fun, but actually was.

There are a few summary rows in this "report" in the database that mess up the parsing, so I'm going to skip some trial and error and just say, I needed to filter out the rows with two or fewer fields... Basically the first row has to have all of the fields or else the objects won't roll up together correctly, so each row had to go through a `Where-Object` to check its count before expanding the entry list into another `ForEach-Object` where we create a new PSObject and add each field, otherwise, my idea of going through each entry's entries and adding each as a field/value pair to the PSObject was spot-on (and possibly from StackOverflow). 

```
Invoke-Sqlcmd -TrustServerCertificate -ServerInstance <Server> -Database MirthDB `
              -Query "SELECT ATTRIBUTES FROM EVENT WITH (NOLOCK) WHERE NAME = 'Data Pruner'" `
| Select-Object -ExpandProperty Attributes `
| Select-Xml -XPath "/map" `
| Select-Object -ExpandProperty Node `
| ForEach-Object { $_ | Where-Object { $_.entry.Count -GT 2 } `
                      | Select-Object -ExpandProperty entry `
                      | ForEach-Object -Begin {$x = New-Object PSObject } `
                                       -Process { $x | Add-Member NoteProperty $($_.string[0]) $_.string[1] } `
                                       -End { $x } `
                  } `
| Select-Object 'Channel Name','Messages Pruned','Content Rows Pruned','Time Elapsed' `
| Format-Table
```

If each of the rows of XML had the same fields, then I could have skipped the `ForEach-Object` on line 6 and its closure on line 11. Fortunately, that workaround slotted in with minimal revision.

It does feel like there should be a better way to build a PowerShell Object, but this does the job!
