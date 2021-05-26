---
layout: post
title:  "Panes in Windows Terminal"
date:   2020-07-13 20:00:39 +0200
categories: powershell azure
---
Windows Terminal is great and split panes are one of the best functions of this tool. We can split our work area how we want and avoid the need for switching between tabs.

> We can split panes thanks to keyboard shortcuts: alt+shift+plus or alt+shift+-. For more details click the [link](https://docs.microsoft.com/en-us/windows/terminal/panes).

Another option is using command-line arguments which gives us more flexibility. The syntax isn’t difficult but for instance, I've never remembered the names of my profiles. That's why I created a Start-WindowsTerminal (alias – start-wt) function. The function helps me open Windows Terminal with shells that  I need in one window.

See examples:
![Start-WindowsTerminal.gif](/assets/images/start-windowsterminal.gif)

```powershell
Start-WindowsTerminal
# The function starts wt.exe (Windows Terminal) process with the default profile.

Start-WindowsTerminal -Tab PowerShell
# The function starts wt.exe (Windows Terminal) process with PowerShell in the tab.

Start-WindowsTerminal -Tab PowerShell, 'PowerShell 7'
# The function starts wt.exe (Windows Terminal) process with two tabs.

Start-WindowsTerminal -Tab CommandLine -Vertical 'PowerShell 7'
# The function starts wt.exe (Windows Terminal) process with CommandLine and additional with one vertical pane.

Start-WindowsTerminal -Tab CommandLine -Horizontal 'PowerShell 7'
# The function starts wt.exe (Windows Terminal) process with CommandLine and additional with one horizontal pane.

Start-WindowsTerminal -Tab CommandLine -Horizontal 'PowerShell 7' -Vertical 'Remote Docker', Ubuntu
```

If it's helpful for you, the function is below or on [GitHub](https://github.com/MateuszNad/PowerShell_Scripts/blob/master/Start-WindowsTerminal.ps1).

```powershell
function Start-WindowsTerminal
{
    #Requires -Version 7.0
    [cmdletbinding()]
    [Alias('start-wt')]
    param()
    DynamicParam
    {
        # Get array for ValidateSet
        $SettingsPath = Join-Path $HOME -ChildPath 'AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json'

        if (Test-Path $SettingsPath)
        {

            $ParameterValidateSet = ((Get-Content $SettingsPath | ConvertFrom-Json).profiles | Where-Object { $_.hidden -notmatch 'True' } | Select-Object -Unique Name).Name
            # Create the dictionary
            $RuntimeParameterDictionary = New-Object System.Management.Automation.RuntimeDefinedParameterDictionary

            # Set the dynamic parameters' name
            $paramDefaultProfile = 'Tab'
            # Create the collection of attributes
            $AttributeCollection = New-Object System.Collections.ObjectModel.Collection[System.Attribute]
            # Create and set the parameters' attributes
            $ParameterAttribute = New-Object System.Management.Automation.ParameterAttribute
            $ParameterAttribute.Mandatory = $false
            $ParameterAttribute.Position = 0
            # Add the attributes to the attributes collection
            $AttributeCollection.Add($ParameterAttribute)
            $ValidateSetAttribute = New-Object System.Management.Automation.ValidateSetAttribute($ParameterValidateSet)
            # Add the ValidateSet to the attributes collection
            $AttributeCollection.Add($ValidateSetAttribute)
            # Create and return the dynamic parameter
            $RuntimeParameter = New-Object System.Management.Automation.RuntimeDefinedParameter($paramDefaultProfile, [string[]], $AttributeCollection)
            $RuntimeParameterDictionary.Add($paramDefaultProfile, $RuntimeParameter)


            # Set the dynamic parameters' name
            $paramVertical = 'Vertical'
            $AttributeCollection = New-Object System.Collections.ObjectModel.Collection[System.Attribute]
            $ParameterAttribute = New-Object System.Management.Automation.ParameterAttribute
            $ParameterAttribute.Mandatory = $false
            $ParameterAttribute.Position = 1
            $AttributeCollection.Add($ParameterAttribute)

            $ValidateSetAttribute = New-Object System.Management.Automation.ValidateSetAttribute($ParameterValidateSet)
            $AttributeCollection.Add($ValidateSetAttribute)
            $RuntimeParameter = New-Object System.Management.Automation.RuntimeDefinedParameter($paramVertical, [string[]], $AttributeCollection)
            $RuntimeParameterDictionary.Add($paramVertical, $RuntimeParameter)

            $paramHorizontal = 'Horizontal'
            $AttributeCollection = New-Object System.Collections.ObjectModel.Collection[System.Attribute]
            $ParameterAttribute = New-Object System.Management.Automation.ParameterAttribute
            $ParameterAttribute.Mandatory = $false
            $ParameterAttribute.Position = 2
            $AttributeCollection.Add($ParameterAttribute)

            $ValidateSetAttribute = New-Object System.Management.Automation.ValidateSetAttribute($ParameterValidateSet)
            $AttributeCollection.Add($ValidateSetAttribute)
            $RuntimeParameter = New-Object System.Management.Automation.RuntimeDefinedParameter($paramHorizontal, [string[]], $AttributeCollection)
            $RuntimeParameterDictionary.Add($paramHorizontal, $RuntimeParameter)

            $CreatedParam = $true
            return $RuntimeParameterDictionary
        }
        else
        {
            $CreatedParam = $false
        }
    }
    process
    {
        if ($CreatedParam)
        {
            $Tabs = $PSBoundParameters[$paramDefaultProfile]
            $Vertical = $PSBoundParameters[$paramVertical]
            $Horizontal = $PSBoundParameters[$paramHorizontal]

            if ($Tabs.Count -eq 1)
            {
                $defaultTab = 'new-tab --profile "' + $Tabs[0] + '"'
            }
            elseif ($Tabs.Count -gt 1)
            {
                for ($i = 1; $i -lt $Tabs.Count; $i++)
                {
                    [string]$additionalTabs += '`;new-tab --profile "' + $Tabs[$i] + '"'
                }
            }

            # $DefaultProfile = 'new-tab --profile "' + $Default + '"'

            foreach ($Item in $Vertical)
            {
                [string]$splitPaneVertical += '`; split-pane --vertical --profile "' + $Item + '"'
            }

            foreach ($Item in $Horizontal)
            {
                [string]$splitPaneHorizontal += '`; split-pane --horizontal --profile "' + $Item + '"'
            }

            Start-Process wt.exe -ArgumentList $defaultTab, $splitPaneHorizontal, $splitPaneVertical, $additionalTabs
            #start wt 'new-tab "cmd" ; split-pane -p "Windows PowerShell" ; split-pane -H wsl.exe'
        }
        else
        {
            Write-Warning "The settings file not exists or Windows Terminal isn't installed."
        }
    }
}
```
