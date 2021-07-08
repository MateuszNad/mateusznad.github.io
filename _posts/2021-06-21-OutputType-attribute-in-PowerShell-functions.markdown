---
layout: post
title:  "Why and how to use the OutputType attribute in custom PowerShell functions"
date:   2021-05-26 20:00:00 +0200
categories: powershell
---

The Tab Completion feature in the PowerShell console simplifies a life. It automatically completes cmdlets, parameters, file names, and sometimes properties of objects also.
In this post, I want to pay attention to the last thing - properties. Take a look at the below example.

![Suggest to Sort-Object properties of an object returned by Get-Process](/assets/images/tab-completion-properties.gif)

The console was able to suggest to Sort-Object properties of an object returned by Get-Process.

Now I will show you how to achieve the same effect within your own function. First, we have to create a new simple class with properties that will be used in a function.
The class will replace a custom object that we create in most cases in PowerShell functions.

> More information about PowerShell classes do you find [there](https://docs.microsoft.com/en-US/powershell/module/microsoft.powershell.core/about/about_classes?view=powershell-7.1).

```powershell
class NewClass {
    [string]$Name
    [version]$LatestVersion
    [int]$Count
    [string]$CompanyName
}
```
If the class exists, we can move on. Take a look at the below script. There is used the new class and additional OutputType attribute.
The attribute reports the type of object that the function returns.  It allows completing properties correctly by Tab Completion.

> The [OutputType attribute](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_outputtypeattribute?view=powershell-7.1) lists the .NET types of objects that the functions returns.

```powershell
function Get-Info {
    [OutputType('NewClass')]
    param()

    $Object = [NewClass]::new()
    $Object.Name = 'PSReadLine'
    $Object.LatestVersion = [version]'2.2.0'
    $Object.Count = '2'
    $Object.CompanyName = 'Microsoft Corporation'

    Write-Output $Object
}
```

We need only those two elements to achieve a similar effect as with native cmdlets.

```
Get-Info | Sort-Object -Property LatestVersion
```
![A custom PowerShell function](/assets/images/tab-completion-custom-function.gif)
