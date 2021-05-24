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

![Data from a CSV file](../assets/images/data-from-csv.png)

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

```powershell
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
![Data from JSON file](../assets/images/data-from-json.png)
## 3. Getting data from a database
When we talk about data, I have to mention databases. PowerShell allows us to read, write and process data from a relational database, for instance, SQL Server.

I have to point out that PowerShell enables only a connection. , To determine data required from a database we use a declarative SQL language. We don’t find native commands, that is why I recommend installing the SqlServer or the dbatools module.

Here, for example, I used the Invoke-DbaQuery command from the dbatools module. I’ve selected SystemName, Name, Locked, LoginFailures columns from a dba.User table where a value of the Locked column is equal to 1.

```powershell
$param = @{
    SqlInstance  = 'serwer-sql'
    Database = 'ExampleDB'
    Query =  "SELECT SystemName, Name, Locked, LoginFailures FROM dbo.Users WHERE Locked = 1"
}
Invoke-DbaQuery @param

<#
SystemName    Name Locked LoginFailures
----------    ---- ------ -------------
     86166  161561   True             2
     85067  159501   True             0
    106890  200418   True             2
     82272  154260   True             1
    106365  199435   True             0
     79398  148872   True             0
#>
```

## 4. Getting data from REST API
Most modern systems enable direct connection between applications with an application programming interface knows as API. Loading data from a system through an API is one of the most requent features. Of course, we have to learn documentation before using it because each API has its specificity.

You should use the native Invoke-RestMethod command (availability in PowerShell 5.1 and 7.0) to send a request to a REST API.

Below there is an example with a request to Zabbix API.

```powershell
# # https://www.zabbix.com/documentation/3.0/manual/api

# na początku musimy zalogować się do API w celu wygenerowania i pobrania tokenu
# https://www.zabbix.com/documentation/3.0/manual/api/reference/user/login

$AddressToAPI = 'https://test-zabbix/api_jsonrpc.php'
$RequestUserLogin = '{
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
        "user": "username",
        "password": "passwordZabbix"
    },
    "id": 1
}'

($Token = Invoke-RestMethod -Method POST -Uri $AddressToAPI -Body $RequestUserLogin -ContentType "application/json")

# Metoda pozwala na pobieranie hostów zgodnie z podanymi parametrami.
# https://www.zabbix.com/documentation/3.0/manual/api/reference/host/get
$RequestHostGet = '{
    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {

    },
    "auth": "' + $Token.result+'",
    "id": 1
}'

($Hosts = Invoke-RestMethod -Method POST -Uri $AddressToAPI -Body $RequestHostGet -ContentType "application/json")
($Hosts).result | Select-Object hostid, name, host
```

The next example it’s a request to a public Finance Ministry API.

```powershell
$param = @{
    Method = 'Get'
    Uri = 'https://wl-api.mf.gov.pl/api/search/nip/5270103391?date=2020-07-07'
    ContentType = 'application/json'
}
(Invoke-RestMethod @param).result.subject
```

## 5. Data from websites
We can also use PowerShell to download data from websites, also known as web scraping.

![Data from website](../assets/images/data-from-websites.png)

```powershell
$ContentAP = Invoke-WebRequest 'https://akademiapowershell.pl'
($ContentAP.ParsedHtml).getElementsByTagName('h2') | Select innerText
```

Here you can find an example function to download information from otoDom, that I wrote weeks ago.


> In PowerShell 7 the Invoke-WebRequest command doesn’t parse websites. In this case, my function won’t work correctly. Along with PowerShell 7, I recommend using the PowerHTML module.

## 6. Getting information from Active Directory
In average and large companies we almost always find Active Directory services. It’s an enormous collection of data with objects like users, workstations, and servers.

To get data from Active Directory, we use a dedicated PowerShell module which we use to search, get and modify.

Examples of commands to get data from the Active Directory using the PowerShell.

```powershell
# Pobranie grup domenowych zgodnych filtrem
Get-ADGroup -Filter "Name -like '*DB*'"

# Pobranie użytkowników, których konta są wyłączone
Get-ADUser -Filter "Enabled -eq 'False'"

# Pobranie wszystkich członków grup zgodnych z filtrem
Get-ADGroup -Filter "Name -like 'FL*DB*'" | Foreach { Get-ADGroupMember -Identity $_.Name
```

## The Recap
These were six ways, probably the most popular to get data into PowerShell functions and scripts.
