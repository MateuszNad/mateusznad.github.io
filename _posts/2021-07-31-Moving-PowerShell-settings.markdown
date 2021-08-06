---
layout: post
title:  "How to transfer your PowerShell goods to a new device?"
date:   2021-07-31 20:00:00 +0200
categories: powershell
---

I've recently had to move to a new laptop. Unfortunately, there always comes a time when we have to do it. For instance, if you think about upgrading your Windows 10 to 11 and your device is older than 5 years, you probably go through this process.

Returning to the main topic. I don't like to do it because it means I need to personalize (or move) everything again modules, PowerShell profile files, settings additional tools such as Visual Studio Code, Windows Terminal, and Management Studio.

I've struggled with a challenge like that, so I decided to collect some tips to deal with it. I hope you will take advantage of something from here.

> The post hasn't been finished yet, and additional content will appear here soon.

## PowerShell modules

Firstly I'm going to show you how I’ve used [PSDepend](https://github.com/RamblingCookieMonster/PSDepend) to install all used PowerShell modules on the new device.

Look at the below script that prepares on the old computer a batch file for PSDepend with information about the name of the module and the accurate version.

```powershell
function New-PSDependFile
{
    param (
        [Parameter(Mandatory)]
        [string]$Path,
        [Parameter(Mandatory = $false)]
        [string[]]$Exclude,
        [Parameter(Mandatory = $false)]
        [string[]]$Include #do poprawienia
    )

    begin
    {
        function Search-PSRepository
        {
            param(
                [Parameter(Mandatory, ValueFromPipeline)]
                [System.Uri]$RepositorySourceLocation
            )

            begin
            {
                $Repositories = Get-PSRepository
            }
            process
            {
                foreach ($Repository in $Repositories)
                {
                    if (($Repository.SourceLocation) -eq $RepositorySourceLocation.OriginalString )
                    {
                        #$RepositorySourceLocation
                        $Repository.Name
                    }
                }
            }
        }
    }
    process
    {
        $Modules = Get-Module -ListAvailable | Where-Object { $_.RepositorySourceLocation -and $_.Name -notlike $Exclude } | Group-Object -Property Name
        Foreach ($Module in $Modules)
        {
            # get the latest module
            $LatestModule = ($Modules | Where-Object Name -eq $Module.Name).Group[0]
            $PSRepository = $LatestModule.RepositorySourceLocation | Search-PSRepository

            $ModuleJson += @"
`t'$($LatestModule.Name)_$($LatestModule.Version)' = @{
    `tName = '$($LatestModule.Name)'
    `tParameters = @{
        `tRepository         = '$PSRepository'
        `tSkipPublisherCheck = `$true
    `t}
    `tVersion = '$($LatestModule.Version)'
`t}`n
"@
        }
    }
    end
    {
        $FullJson = @"
@{
$ModuleJson
}
"@
        $FullJson | Out-File -FilePath $Path
        # Clear $FullJson
        $FullJson = ''
    }
}
```
```powershell
New-PSDependFile -Path 'C:\source\module.depend.psd1'
# Example with exclude
New-PSDependFile -Path 'C:\source\module.depend.psd1' -Exclude '*Az*'
```

Before we can use the prepared file, we have to install the PSDepend module manually on the destined computer.

```powershell
Install-Module -Name PSDepend -Force
Import-Module -Name PSDepend
```

The last step is making all the favorite PowerShell modules appear in the new place. We have to run the below command that will install each module one by one.

```powershell
Invoke-PSDepend -Path 'C:\module.depend.psd1' -Import -Force
# Invoke-PSDepend -Path 'C:\module.depend.psd1' -Install -WhatIf
```
![Invoke-PSDepend -Import](/assets/images/invoke-psdepends-install.png)
