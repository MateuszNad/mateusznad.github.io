---
layout: post
title:  "How to fast catch up on modules that you need in Azure Automation"
date:   2021-09-08 20:00:00 +0200
categories: powershell
---

I don't know how you but if I create a new runbook for Azure Automation, I've made it on my device, and then moved a script to the Azure service.
For me, that approach is easier and faster. I can check the results of working and debug problems in my runtime if ones appear.

Of course, in my local workstation, I've got available a lot of PowerShell modules, and I don't overthink whether a used function will be available on a destinated Azure Automation.
On the one hand, it's no problem. We only need to install a missing module, but we have to know the names of the ones which we used.

<!-- I wouldn't write about it if I didn't have got an idea for it. -->
<!--
This is the reason why I've written a script that recognizes all used functions in a script and returns an object with details, for instance, a name of a module and version. For that script, I used the method Tokenize from a class [System.Management.Automation.PSParser]. -->

My solution comprises 3 scripts. The first script gets statistics about used PowerShell cmdlets. In the one, I used the method Tokenize from a class `[System.Management.Automation.PSParser]`.

```powershell
Get-CommandStatistic '.\Runbook.ps1'

<#
Name                     Count Path
----                     ----- ----
Connect-AzAccount            1 .\Runbook.ps1
ForEach-Object               1 .\Runbook.ps1
Get-AutomationConnection     1 .\Runbook.ps1
Get-AzResourceGroup          1 .\Runbook.ps1
Get-DbaDatabase              1 .\Runbook.ps1
Import-Module                1 .\Runbook.ps1
Remove-AzResourceGroup       1 .\Runbook.ps1
Write-Error                  2 .\Runbook.ps1
Write-Verbose                1 .\Runbook.ps1
#>
```

The second takes the object from Get-CommandStatistic and returns information about the source of those commands.

```
Name        : Connect-AzAccount
DisplayName : Connect-AzAccount
CommandType : Cmdlet
Module      : Az.Accounts
Version     : 2.5.1

Name        : ForEach-Object
DisplayName : ForEach-Object
CommandType : Cmdlet
Module      : Microsoft.PowerShell.Core
Version     : 7.1.3.0

Name        : Get-AzResourceGroup
DisplayName : Get-AzResourceGroup
CommandType : Cmdlet
Module      : Az.Resources
Version     : 4.2.0

Name        : Get-DbaDatabase
DisplayName : Get-DbaDatabase
CommandType : Function
Module      : dbatools
Version     : 1.0.136

Name        : Import-Module
DisplayName : Import-Module
CommandType : Cmdlet
Module      : Microsoft.PowerShell.Core
Version     : 7.1.3.0
```

Last but not least script uses that data to install required modules in Azure Automation. When we put together all these functions, a command looks like that. Additionally, an Add-AzAutomationModule function checks modules dependencies to avoid errors.

> ⚠️ Error importing the module Az.Resources. Import failed with the following error: Orchestrator.Shared.AsyncModuleImport.ModuleImportException: An error occurred when trying to import the module to an internal PowerShell session. Either the module dependencies are not imported correctly or the module is unsupported. Internal Error Message: The specified module 'Az.Accounts' with version '2.5.1' was not loaded because no valid module file was found in any module directory


```powershell
Get-CommandStatistic '.\Runbook.ps1' |  Get-CommandSource |  Add-AzAutomationModule -AutomationAccountName 'aa-shared-prod' -ResourceGroupName 'rg-shared'

<#
ResourceGroupName     : rg-shared
AutomationAccountName : aa-shared-prod
Name                  : Az.Accounts
IsGlobal              : False
Version               : 2.5.2
SizeInBytes           : 7011512
ActivityCount         : 33
CreationTime          : 05.08.2021 10:18:54 +02:00
LastModifiedTime      : 07.09.2021 14:00:28 +02:00
ProvisioningState     : Creating

ResourceGroupName     : rg-shared
AutomationAccountName : aa-shared-prod
Name                  : Az.Resources
IsGlobal              : False
Version               : 4.3.0
SizeInBytes           : 1321918
ActivityCount         : 132
CreationTime          : 05.08.2021 15:39:32 +02:00
LastModifiedTime      : 07.09.2021 14:03:06 +02:00
ProvisioningState     : Creating

ResourceGroupName     : rg-shared
AutomationAccountName : aa-shared-prod
Name                  : dbatools
IsGlobal              : False
Version               : 1.1.15
SizeInBytes           : 30708900
ActivityCount         : 640
CreationTime          : 06.09.2021 17:05:00 +02:00
LastModifiedTime      : 07.09.2021 14:03:09 +02:00
ProvisioningState     : Creating
#>
```

Below you find all functions to try my solution. Try it and let me know if you see something to improve.

```powershell
<#
.SYNOPSIS
    The function returns statistics about used functions in a script.
.DESCRIPTION
    The function returns statistics about used functions in a script.
.EXAMPLE
    PS C:\> Get-CommandStatistic '.\Runbook.ps1'
#>
function Get-CommandStatistic
{
    [cmdletbinding()]
    param (
        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        [Alias('Path')]
        [string]$FullName
    )
    process
    {
        [array]$Command = $null
        if (Test-Path -Path $FullName)
        {
            $ElapsedTime = [System.Diagnostics.Stopwatch]::StartNew()
            Write-Verbose ("{0};Get-ChildItem" -f $ElapsedTime.Elapsed)
            $ElapsedTime.Restart()

            try
            {
                $ContentScript = Get-Content -Path $FullName -ErrorAction 'Stop'
            }
            catch
            {
                $ContentScript = $null
            }

            if ($ContentScript)
            {
                $Command += [System.Management.Automation.PSParser]::Tokenize($ContentScript, [ref]$null)
            }

            Write-Verbose ("{0};System.Management.Automation.PSParser" -f $ElapsedTime.Elapsed)

            $ElapsedTime.Restart()

            # searching, selecting and grouping commands
            $GroupedCommand = ($Command | Where-Object Type -EQ 'Command' | Group-Object -Property Content |
                Select-Object -Property @{L = 'Name'; E = { $_.Name } }, Count, @{L = 'Path'; E = { $FullName } })

            Write-Verbose ("{0};Group-Object" -f $ElapsedTime.Elapsed)
            Write-Output $GroupedCommand

            $ElapsedTime.Stop()
        }
        else
        {
            Write-Warning "The relayed path doesn't exist."
        }
    }
}

<#
.SYNOPSIS
    The function returns a name of a module for used functions in a script.
.DESCRIPTION
    The function returns a name of a module for used functions in a script.
.EXAMPLE
    PS C:\> Get-CommandStatistic '.\Runbook.ps1' |  Get-SourceCommand
#>
function Get-CommandSource
{
    [OutputType([object])]
    [cmdletbinding()]
    param(
        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        $Name
    )
    process
    {
        $Result = Get-Command $Name -ErrorAction SilentlyContinue

        if ($null -ne $Result)
        {
            $DisplayName = if ($null -eq $Result.DisplayName)
            {
                $Result.Name
            }
            else
            {
                $Result.DisplayName
            }

            $Object = [PSCustomObject][ordered]@{
                Name        = $Result.Name
                DisplayName = $DisplayName
                CommandType = $Result.CommandType
                Module      = $Result.Source
                Version     = $Result.Version
            }

            if ($Result.CommandType -eq 'Alias' -and $Result.DisplayName -match '->')
            {
                $Result = Get-Command $Result.ResolvedCommandName
                $Object.Name = $Result.Name
                $Object.Module = $Result.Source
                $Object.Version = $Result.Version
            }
            Write-Output $Object
        }
        else
        {
            Write-Warning "The source of the $Name command can be found."
        }
    }
}

<#
.SYNOPSIS
    The function adds modules to Azure Automation
.DESCRIPTION
    The function adds modules to Azure Automation
.EXAMPLE
    PS C:\> Add-AzAutomationModule -Verbose -AutomationAccountName 'aa-name' -ResourceGroupName 'rg-name' -Module "Az.Resources"
    Add the Az.Resources module to Azure Automation

.EXAMPLE
    PS C:\> Add-AzAutomationModule -Verbose -AutomationAccountName 'aa-name' -ResourceGroupName 'rg-name' -Module "Az.Resources", "dbatools"
    Add the Az.Resources and dbatools module to Azure Automation

.EXAMPLE
    PS C:\> Get-CommandStatistic '.\Runbook.ps1' |  Get-SourceCommand |  Add-AzAutomationModule -AutomationAccountName 'aa-name' -ResourceGroupName 'rg-name'
    Add all modules used in a runbook.ps1 script to Azure Automation
#>
function Add-AzAutomationModule
{
    [cmdletbinding()]
    param(
        [Parameter(Mandatory, Position = 0)]
        $AutomationAccountName,
        [Parameter(Mandatory, Position = 1)]
        $ResourceGroupName,
        [Parameter(Mandatory, ValueFromPipelineByPropertyName)]
        [string[]]$Module
    )
    begin
    {
        $Modules = @()
        $ToInstallModules = @()
        $DependenciesModules = @()
        $InstalledModules = @()
    }
    process
    {
        try
        {
            foreach ($Item in $Module)
            {
                if ($Item -notlike 'Microsoft.PowerShell*')
                {
                    $FoundModule = Find-Module $Item
                    $Modules += [PSCustomObject]@{
                        Name                     = $FoundModule.Name
                        RepositorySourceLocation = $FoundModule.RepositorySourceLocation
                        Version                  = $FoundModule.Version
                        DependenciesName         = $FoundModule.Dependencies.Name
                    }
                }
            }
        }
        catch
        {
            Write-Warning $_
        }
    }
    end
    {
        try
        {
            $DependenciesModules = ($Modules).DependenciesName | Sort-Object -Unique
            Write-Verbose (ConvertTo-Json $DependenciesModules)

            $ToInstallModules = $Modules | Sort-Object -Unique -Property Name
            Write-Verbose (ConvertTo-Json $ToInstallModules)

            #"Sleeping for 180 sec in order to wait the installation of the Az.Accounts module"
            foreach ($Item in $DependenciesModules)
            {
                $DependenciesModule = Find-Module $Item
                $Linkuri = $DependenciesModule.RepositorySourceLocation + "/package/" + $DependenciesModule.Name + "/" + $DependenciesModule.Version
                Write-Verbose "Link: $LinkUri"

                $paramAzAutomationModule = @{
                    AutomationAccountName = $AutomationAccountName
                    Name                  = $DependenciesModule.Name
                    ContentLinkUri        = $LinkUri
                    ResourceGroupName     = $ResourceGroupName
                    OutVariable           = 'Result'
                }

                New-AzAutomationModule @paramAzAutomationModule

                do
                {
                    Write-Verbose "Installing the $($Result.Name) module..."
                    Start-Sleep 30
                } until ((Get-AzAutomationModule -ResourceGroupName $ResourceGroupName -AutomationAccountName $AutomationAccountName  -Name $Result.Name).ProvisioningState -eq 'Succeeded')

                $InstalledModules += $Result.Name
            }

            foreach ($ToInstallItem in $ToInstallModules)
            {
                if ($InstalledModules -notcontains $ToInstallItem.Name)
                {
                    $LinkUri = $ToInstallItem.RepositorySourceLocation + "/package/" + $ToInstallItem.Name + "/" + $ToInstallItem.Version
                    Write-Verbose "Link: $LinkUri"

                    $paramAzAutomationModule2 = @{
                        AutomationAccountName = $AutomationAccountName
                        Name                  = $ToInstallItem.Name
                        ContentLinkUri        = $Linkuri
                        ResourceGroupName     = $ResourceGroupName
                    }
                    New-AzAutomationModule @paramAzAutomationModule2
                }
            }
        }
        catch
        {
            Write-Warning $_
        }
    }
}
```
