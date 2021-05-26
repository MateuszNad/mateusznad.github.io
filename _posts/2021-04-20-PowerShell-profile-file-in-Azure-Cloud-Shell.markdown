---
layout: post
title:  "PowerShell profile file in Azure Cloud Shell"
date:   2021-05-20 20:00:39 +0200
categories: powershell, azure
---

I prefer to work on my machine. It’s the place where I have got exactly everything that I need. In the way that I need. For instance, Powershell gives us the ability to personalize by profile files. These files enable us to add commands, scripts that will be executed every time when a new console is started.

But sometimes working on a personal machine is not possible and then online tools are a pretty good solution. **Azure Cloud Shell** is one of them. It enables us to work directly with PowerShell and Azure.

What is important Azure Cloud Shell gives us the possibility to customize a profile file the same way as the PowerShell console.

## What is Azure Cloud Shell?
> Azure Cloud Shell is an interactive, authenticated, browser-accessible shell for managing Azure resources. It provides the flexibility of choosing the shell experience that best suits the way you work, either Bash or PowerShell.
> ###### https://docs.microsoft.com/pl-pl/azure/cloud-shell/overview

## Creating the profile file in Azure Cloud Shell
A few days ago I encountered a [post blog about tracking changes in Microsoft Azure](https://luke.geek.nz/keep-up-to-date-with-latest-changes-on-azure-using-powershell). There, you can find a PowerShell function that gets news from the [Azure Updates blog](https://azure.microsoft.com/en-us/updates/). It’s a simple solution but very hands-on. That’s why I decided to add the one to my Azure Cloud Shell profile.

I will show you how to create and add this function to the profile file. In the first step, you have to create an empty file. Use the below command to do it.

```powershell
New-Item -Path $Profile.CurrentUserAllHost -Force
```

I always create an individual directory for scripts or functions that I use in the PowerShell profiles. Therefore, the next script will create a directory and download the function there.

```powershell
$path = Join-Path $home -ChildPath 'psprofile'
New-Item -Path $path -Force -ItemType Directory
$param = @{
    Uri     = 'https://gist.githubusercontent.com/lukemurraynz/7bf30a1f0f5f12e622e2bbe11ff7033d/raw/85641ed9a2cbbe1ab2a97de6948e45e2b6328930/Get-AzureBlogUpdates.ps1'
    OutFile = (Join-Path $path -ChildPath 'Get-AzureBlogUpdates.ps1')
}
Invoke-WebRequest @param
```

Add a below fragment of scripts to the profile.ps1 file that will be executed every time when you open Azure Cloud Shell.

```powershell
# functions load
Get-ChildItem (Join-Path $home -ChildPath 'psprofile') | ForEach-Object {
    Write-Verbose "Load $($_.BaseName) function" -Verbose
    . $_.FullName
}

$ToMarkdown = @{
    Label      = 'ToMarkdown'
    Expression = { "# $($_.'Published Date') - $($_.Title)
$($_.Description)\
-&gt; [$($_.Link)]($($_.Link))\" }
}

$Result = Get-AzureBlogUpdates | Where-Object 'Published Date' -gt ((Get-Date).AddDays(-2)) | Select-Object -Property $ToMarkdown
$Result.ToMarkdown | Out-String | Show-Markdown -ErrorAction SilentlyContinue
```

You can do it that way.

```powershell
cd ~
code ./.config/PowerShell/profile.ps1
```
![Creating the profile file in Azure Cloud Shell](../assets/images/azure-cloud-shell-profile.png)

> The script returns only new information that was published the day before. Additionally, I modified the result by using Show-Markdown function to make it look better. (of course in my opinion 😉

![Azure blog](../assets/images/Azure-news.png)

By using this simple way I will always be up-to-date.

# The Recap

In this short post, I wanted to show you the example, how you can use the profile file as well in Azure Cloud Shell.
