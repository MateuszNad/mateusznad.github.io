---
layout: post
title:  "PSModuleManager"
date:   2021-05-26 20:00:00 +0200
categories: powershell
---

The number of collected modules and their new versions while working with PowerShell can be surprising. Personally, I use in most of the cases only the up-to-date versions of modules therefore the previous ones only take disk space.

Because the beginning of the year is a good moment to clean up, I would like to recommend you to use a PSModuleManager module. I want to show you 3 functions that will ease managing PowerShell modules

## Why PSModuleManager?

The functions of this module were created to complement the ability of native commands such as ```Get-Module, Update-Module, and Uninstall-Module```. Thanks to them I easily update and remove the old version of the PowerShell modules.

## Fetching list of module

You can get a list of modules using either ```Get-Module``` or ```Get-InstalledModule```. The commands don’t return information which I need.

Hence my idea of writing a new Get-PSInstalledModule command that returns a number of versions for each module and information about space used on disk. You can execute it for CurrentUser and AllUsers area.

```powershell
# default - CurrentUser
Get-PSInstalledModule

# AllUsers
Get-PSInstalledModule -Scope AllUsers
```

> Thanks to this information we can check how much space take currently installed modules.

```powershell
# zakres CurrentUser
Get-PSInstalledModule | Measure-Object -Property SpaceUsedMb -Sum

# zakres AllUsers
Get-PSInstalledModule -Scope AllUsers | Measure-Object -Property SpaceUsedMb -Sum
```
![how much space take currently installed modules](../_site/assets/images/spaceused-powershell-module.png)

Other examples if using the Get-PSInstalledModule command.

```powershell
# Pobranie informacjo o jednym module
Get-PSInstalledModule -Name dbatools

# Pobranie informacjo o jednym module - zakres AllUsers
Get-PSInstalledModule -Name dbatools -Scope AllUsers

# Wyświetlenie pełnego obiektu dla jednego modułu
Get-PSInstalledModule -Name dbatools  | Select-Object *
# Name          : dbatools
# LatestVersion : 1.0.113
# Count         : 1
# Modules       : {@{Name=dbatools; Version=1.0.113; Scope=CurrentUser; ModuleBase=C:\Users\Lenovo\Documents\PowerShell\Modules\dbatools\1.0.113; SpaceUsed=104308229; PowerShellVersion=3.0}}
# Scope         : CurrentUser
# SpaceUsedMb   : 99,48
# Description   : The community module that enables SQL Server Pros to automate database development and server administration
# Author        : the dbatools team
# CompanyName   : dbatools.io

# Wykorzystanie wyrażeń regularnych
Get-PSInstalledModule -Name az*

# Top 5 modułów pod kątem miejsca
Get-PSInstalledModule | Sort-Object -Property SpaceusedMb -Descending | Select -First 5

# Top 5 modułów pod kątem miejsca dla AllUsers
Get-PSInstalledModule -Scope AllUsers | Sort-Object -Property SpaceusedMb -Descending | Select -First 5

# Top 5 modułów pod kątem ilości zainstalowanych wersji
Get-PSInstalledModule | Sort-Object -Property Count -Descending | Select -First 5

# Top 5 modułów pod kątem ilości zainstalowanych wersji dla AllUsers
Get-PSInstalledModule -Scope AllUsers | Sort-Object -Property Count -Descending | Select -First 5
```

The function returns a custom object that is used by the two other commands.

## Updating modules

The command checks the availability of a new version for each module and updates if needed. The Update-PSModule function requires the ModuleManagerList object from Get-PSInstalledModule.

```powershell
Get-PSInstalledModule | Update-PSModule -Verbose -Force
```
![Update PowerShell Module](/_site/assets/images/update-powershell-module.png)

Examples of commands:

```powershell
# polecenie wspiera przełącznik -WhatIf
Get-PSInstalledModule -Name dbatools -Scope AllUsers | Update-PSModule -WhatIf
# What if: Performing the operation "Update-PSModule" on target "Version '1.0.136' of module 'dbatools'".

# aktualizacja jednego modułu dla AllUsers
Get-PSInstalledModule -Name dbatools -Scope AllUsers | Update-PSModule -Verbose -Force

# aktualizacja modułów zgodnych z wyrażenien regularnycm
Get-PSInstalledModule -Name az* | Update-PSModule -Force
```

## Deleting old module versions

The last  function of this module is the most useful for me in the context of deleting old and not used versions of PowerShell modules. Uninstall-PSOlderModule keeps only the newest installed versions.

> The function requires the ModuleManagerList object that is returned by Get-PSInstalledModule.

```powershell
# usunięcie starych wersji dla modułu Az.Accounts
Get-PSInstalledModule -Name Az.Accounts | Uninstall-PSOlderModule -Verbose
```
![Deleting old module versions](/assets/images/remove-old-powershell-module.png)

Examples of commands:

```powershell
# wykorzystaj przełącznik -WhatIf do sprawdzenia efektów działania polecenia
Get-PSInstalledModule -Name dbatools -Scope AllUsers | Uninstall-PSOlderModule -Verbose -WhatIf

# What if: Performing the operation "Uninstall-PSOlderModule" on target "Version '0.9.742' of module 'dbatools'".

# usuniecie starych wersji dla modułów w zakresie CurrentUser
Get-PSInstalledModule | Uninstall-PSOlderModule -WhatIf

# usunięcie starych wersji dla wskazanego modułu
Get-PSInstalledModule -Name dbatools -Scope AllUsers | Uninstall-PSOlderModule -Verbose
```

## Pseudo-GUI

The last thing I want to show you is an option on pseudo-GUI using an Out-GridView command. It isn't closely related to the PSModuleManager module but I think it can be useful for you.


> Out-GridView has a hands-no OutputMode parameter (accept Single, Multiple, and Node values) that passes to a pipeline this data that we will choose in an interactive window.

```powershell
Get-PSInstalledModule -Scope AllUsers | Out-GridView -OutputMode Multiple | Uninstall-PSOlderModule -Verbose -WhatIf
# Get-PSInstalledModule -Scope AllUsers | Out-GridView -OutputMode Multiple | Uninstall-PSOlderModule -Verbose
```

![PSModuleManager](/assets/images/psmodulemanager-powershell.gif)

## Recap (installing the module)

If you've reached here, it seems that the module can be useful for you. PSModuleManager is available on PowerShell Gallery and you can download it using a standard command.

```powershell
# domyślnie instalacja w zakresie CurrentUser
Install-Module -Name PSModuleManager -Force
```

> ⚠️ The module needs a PowerShell console with administrator permission to work properly. In another case it will be only available the Get-PSInstalledModule function.

If you see fields to improve I invite you to cooperate through GitHub
