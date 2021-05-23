---
layout: post
title:  "6 ways to get data into PowerShell"
date:   2021-05-23 22:04:39 +0200
categories: powershell
---

Most automations, functions, and Powershell scripts are based on input data. That data we process in a few ways, we check, clean, compare, or rich in order to take suitable actions. Most frequently we input data to functions by parameters. But sometimes we need information from several sources in bulk quantity. Therefore below, you can find 6 ways to get data into PowerShell.

## 1. Data from a CSV file
A CSV is a file with comma-separated values. A lot of systems and applications enable exporting data to this format. Data from Microsoft Excel or Google Sheet you save to a file with CSV extension.

A value of each column will be separated by a comma and each new row will be in a new line. The simple structure allows for creating your file like that. You can use a simple notepad to prepare the file. See for instance using a native PowerShell command to get data from CSV.

CSV files very often are used as a batch for massive operation, for instance, scanning a specific list of virtual machines or processing Active Directory’s objects.

See for instance using a native PowerShell command to get data from CSV.

```powershell
<#
ComputerName, Environment
serv1, test
db-server, prod
app-test, test
#>

Import-Csv -Path C:\Temp\file.csv
# or
Get-Content -Path C:\Temp\file.csv | ConvertFrom-Csv
```

## 2. Data from JSON file
A JSON file is the next simple and very popular data exchange format. Either writing and reading data in the format is easy to do.

The JSON files are worth using as configuration parameters for more complex scripts. Especially when the native commands were added since PowerShell 3.0 to work with files in the JSON format.

Use the ConvertFrom-Json command to convert content into an object.

```
# json file
'{
    "TEST": {
        "Environment": "TEST",
        "ComputerName": ["server-test-1", "server-test-2"],
        "SqlInstance": "server-test-db-1",
        "httpPort": 8081,
        "httpsPort": 443,
        "logsPath": "D:\\log"
    },
    "PROD": {
        "Environment": "SUP",
        "ComputerName": ["server-sup-1", "server-sup-2"],
        "SqlInstance": "server-sup-db-1",
        "httpPort": 8082,
        "httpsPort": 443,
        "logsPath": "D:\\log"
    }
}' | Out-File -FilePath C:\Temp\settings.json

$Settings = Get-Content -Path C:\Temp\settings.json | ConvertFrom-Json
$Settings.TEST
```
