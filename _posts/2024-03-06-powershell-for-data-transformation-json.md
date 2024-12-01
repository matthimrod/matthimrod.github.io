---
layout: post
title: 'PowerShell for Data Transformation: JSON'
subtitle: PowerShell makes quick data transformations easy
tags: [powershell, code]
author: Matt Himrod
---

#### PowerShell makes quick data transformations easy

My team at work has a custom FHIR resource that we work with often called an Operation Outcome. We generate one of these for nearly every record we process, and we also generate versions of these with statistics after a batch is processed. The version of the resource for a record operation includes both the raw incoming record and the transformed outbound record. If we're uploading a document, these outcomes can get large. Recently, one of my teammates was trying to get diagnostic information and reconstruct a document that failed to transmit. He was having trouble loading the file into his editor to pull out the pieces he needed because of the size of the file. 

## PowerShell to the Rescue!

Because it's a FHIR resource, it's essentially a JSON document, and it can be easily manipulated with anything that can work with JSON. My teammate was considering writing a few lines of Python to try to manipulate the document. Instead, I showed him how easily this can be done in PowerShell.

Here's a sample of what one looks like. I've removed some of the extra metadata, but you can see how this can become large and complicated to edit by hand, specifically the large base64 blobs of text.

```json
{
    "resourceType": "OperationOutcome",
    "meta": {
        "profile": [
            "http://upmc.com/fhir/integration/logging/StructureDefinition/integeration-operation-outcome"
        ]
    },
    "extension": [
        {
            "url": "http://upmc.com/fhir/integration/logging/StructureDefinition/integration-record",
            "valueAttachment": {
                "data": "[ HUGE BLOB OF BASE64-ENCODED TEXT ]"
            }
        },
        {
            "url": "http://upmc.com/fhir/integration/logging/StructureDefinition/integration-outbound-payload",
            "valueAttachment": {
                "data": "[ HUGE BLOB OF BASE64-ENCODED TEXT ]"
            }
        }
    ],
    "issue": [
        {
            "severity": "information",
            "code": "informational",
            "details": {
                "coding": [
                    {
                        "code": "SENT",
                    }
                ],
            }
        }
    ]
}
```

The objective at hand is to decode both of the base64 blobs into text. The incoming record is pipe-delimited text, and the outbound payload is JSON... but that JSON also has large blobs of base64 text that need to be removed and replaced with a link using data from the tab-delimited text.

Did I mention that the files my teammate was working with were around 150 MB?

We could write a quick Python script or even use the Python command line, but I've found that PowerShell can really easily load and transform this type of data into a PowerShell object, and from there, it's easily manipulated with combinations of `Select-Object` and `Where-Object`. 

First, since we're dealing with Base64 strings, we'll need the help of a .NET function to transform between base64 and plain text. Add these functions to your PowerShell Profile. If you have VSCode installed, you can easily bring that up by typing `code $PROFILE` at a PowerShell prompt.

```
function ConvertFrom-Base64 {
    param (
      [Parameter(Mandatory, ValueFromPipeline, ValueFromRemainingArguments)]
      [string]$Base64Text
    )
    begin {}
    process {
        [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($Base64Text))
    }
    end {}
}

function ConvertTo-Base64 {
    param (
      [Parameter(Mandatory, ValueFromPipeline, ValueFromRemainingArguments)]
      [string]$PlanText
    )
    begin {}
    process {
        [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetString($PlanText))
    }
    end {}
}
```

The important thing about the way that these functions are written is that **ValueFromPipeline** in the parameter declaration. That will allow us to use this function as part of a pipeline. This allows us to do the following (note: I've added backticks for line continuation - this would normally go all on one long line):

```
Get-Content <filename> `
   | ConvertFrom-Json `
   | Select-Object -ExpandProperty extension `
   | Where-Object url -eq 'http://upmc.com/fhir/integration/logging/StructureDefinition/integration-record' `
   | Select-Object -ExpandProperty valueAttachment `
   | Select-Object -ExpandProperty data `
   | ConvertFrom-Base64 `
   | ConvertFrom-Csv -Delimiter '|'
```

In no time at all, this gives us our delimited text as a PowerShell Object. Depending on the terminal settings, the field names will be colored, dates will be formatted (and I think localized, but don't quote me on that), and you can further refine the query with another `Select-Object` if you needed.

For the other component, we have another JSON document. We'll need a couple fields stripped from the JSON document, but fortunately for us, PowerShell is good at that too:

```
Get-Content <filename> `
   | ConvertFrom-Json `
   | Select-Object -ExpandProperty extension `
   | Where-Object url -eq 'http://upmc.com/fhir/integration/logging/StructureDefinition/integration-outbound-payload' `
   | Select-Object -ExpandProperty valueAttachment `
   | Select-Object -ExpandProperty data `
   | ConvertFrom-Base64 `
   | ConvertFrom-Json `
   | Select-Object -ExcludeProperty base64_data, contained_patient_documents `
   | ConvertTo-Json -Depth 10
```

This command runs in fractions of a second even on the gigantic documents that we were manipulating. He still needed to do some hand-editing on the final result before he could upload it, but PowerShell did the really heavy part in seconds.

I should note that I didn't write these pipelines all at once. I couldn't remember the exact schema of our document, so I started with the first steps (`Get-Content <file> | ConvertFrom-Json`) and added the next cmdlet each time. Strategic use of tab-completion also makes this go quickly. 

Another thing that helps out in PowerShell is the `help` command. It's an easy way to quickly get to the documentation, and if you add `-Online` either before or after the command, the documentation will open in your default browser instead of the terminal window.