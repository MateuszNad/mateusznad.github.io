---
layout: post
title:  "PowerShell function to create a service principal with a certificate"
date:   2021-06-03 20:00:00 +0200
categories: powershell azure
---

Somebody has asked me how he could pass in safe way authorization data to execute a Connect-AzureAD command. He needed to do some unattended tasks in Windows Task Scheduler.

In a case like that the best solution is creating a service principal (SP) with a certificate (self-signed). Everything is very well described [here](https://docs.microsoft.com/en-US/azure/active-directory/develop/howto-authenticate-service-principal-powershell).

If you search for a ready-to-go PowerShell script to create a service principal and add a directory role from Azure Active Directory, you are at a good place.

Of course, you must log into an account that has got a permission to create a new service principal.

> ... your account must have Microsoft.Authorization/*/Write access to assign a role to an AD app. This action is granted through the Owner role or User Access Administrator role
> ###### https://docs.microsoft.com/en-US/azure/active-directory/develop/howto-create-service-principal-portal#check-azure-subscription-permissions


```powershell
Connect-AzureAD
```

Next, you can create a new SPN using a Create-AzureAdServicePrincipal function. The Service Principal Name, DnsName, name of role member, etc. are depended on your requirements.
Additionally, you will be asked for a password of the created certificate.

```powershell
$paramSPN = @{
    ServicePrincipalName = 'ServiceScriptAccount' # nazwa konta serwisowego
    DnsName              = 'clouddb.pl'
    RoleName             = 'User Administrator' # uprawnienia - 'Global Administrator', 'Reports Reader', 'Global Reader', 'User Administrator', 'Directory Writers', 'Directory Readers'
    OutPath              = 'C:\01_Temp\sp_cert' # lokalizacja gdzie zostanie dodatkowo zapisany certyfikat
}
$Result = New-AzureADServicePrincipalWithCertificate @paramSPN

"Connect-AzureAD -TenantId '{0}' -ApplicationId '{1}' -CertificateThumbprint '{2}'" -f $Result.TenantId, $Result.ApplicationId, $Result.CertificateThumbprint
```

The script will prepare an account which you would use to automate other scripts without keeping a password on the plaintext.

> The certificate will be saved to a personal certificate store (Cert:\CurrentUser\My) and to a location from an OutPath parameter.

If you need the certificate to be in another place, import it into a new host. Don't forget about the password :)

The last line of the script returns a complete command with parameters that you can use in your scheduled task.

```powershell
Connect-AzureAD -TenantId '0c000b33-0000-46ea-9df2-1d0000efb01' -ApplicationId '9000327b-0000-000-000-7b26589ec7d0' -CertificateThumbprint 'EDAA0ABBCE98C035223DE5A971BB74820656457F'
```


```powershell
<#
    .EXAMPLE

    Connect-AzureAD
    $paramSPN = @{
        ServicePrincipalName = 'ServiceScriptAccount' # nazwa konta serwisowego
        DnsName              = 'clouddb.pl'
        RoleName             = 'User Administrator' # uprawnienia - 'Global Administrator', 'Reports Reader', 'Global Reader', 'User Administrator', 'Directory Writers', 'Directory Readers'
        OutPath              = 'C:\01_Temp\sp_cert' # lokalizacja gdzie zostanie dodatkowo zapisany certyfikat
    }
    $Result = New-AzureADServicePrincipalWithCertificate @paramSPN

#>
function New-AzureADServicePrincipalWithCertificate
{
    [cmdletbinding()]
    param(
        [Parameter(Mandatory)]
        $ServicePrincipalName,
        [Parameter(Mandatory)]
        [string]$DnsName,
        [ValidateSet('Global Administrator', 'Reports Reader', 'Global Reader', 'User Administrator', 'Directory Writers', 'Directory Readers')]
        $RoleName = 'Global Reader',
        [Parameter(Mandatory)]
        [securestring]$Password,
        [Parameter(Mandatory)]
        [string]$OutPath,
        [switch]$PassThru
    )

    try
    {
        if (Get-AzureADServicePrincipal | Where-Object ServicePrincipalNames -EQ "https://$ServicePrincipalName") {
            Write-Warning "Another object with the same value for property ServicePrincipalNames already exists."
            exit 1
        }

        $currentDate = Get-Date
        $endDate = $currentDate.AddYears(1)
        $notAfter = $endDate.AddYears(1)

        $param = @{
            CertStoreLocation = 'Cert:\CurrentUser\My'
            DnsName           = "$($ServicePrincipalName.ToLower())`.$DnsName"
            KeyExportPolicy   = 'Exportable'
            Provider          = "Microsoft Enhanced RSA and AES Cryptographic Provider"
            NotAfter          = $notAfter
        }
        $Thumbprint = (New-SelfSignedCertificate @param).Thumbprint

        $paramExport = @{
            Cert     = (Join-Path $param.CertStoreLocation -ChildPath $Thumbprint)
            FilePath = (Join-Path $OutPath -ChildPath "$ServicePrincipalName-cert.pfx")
            Password = $Password
        }
        Export-PfxCertificate @paramExport | Out-Null

        $Certificate = New-Object System.Security.Cryptography.X509Certificates.X509Certificate($paramExport.FilePath, $Password)
        $KeyValue = [System.Convert]::ToBase64String($Certificate.GetRawCertData())

        # Create the Azure Active Directory Application
        $Application = New-AzureADApplication -DisplayName $ServicePrincipalName -IdentifierUris "https://$ServicePrincipalName" -ErrorAction Stop

        $paramApplicationKey = @{
            ObjectId            = $Application.ObjectId
            CustomKeyIdentifier = $ServicePrincipalName
            StartDate           = $currentDate
            EndDate             = $endDate
            Type                = 'AsymmetricX509Cert'
            Usage               = 'Verify'
            Value               = $KeyValue
        }
        New-AzureADApplicationKeyCredential @paramApplicationKey -OutVariable ApplicationKeyCredential | Out-Null
        $ServicePrincipal = New-AzureADServicePrincipal -AppId $application.AppId

        $ADDirectoryRole = Get-AzureADDirectoryRole | Where-Object DisplayName -EQ $RoleName
        Add-AzureADDirectoryRoleMember -ObjectId $ADDirectoryRole.ObjectId -RefObjectId $ServicePrincipal.ObjectId -OutVariable AdRoleMember

        # Get Tenant Detail
        $Tenant = Get-AzureADTenantDetail

        $Object = [PSCustomObject]@{
            RoleAssigned          = $RoleName
            ServicePrincipalName  = $ServicePrincipalName
            CertificateThumbprint = $Thumbprint
            ApplicationId         = $ServicePrincipal.AppId
            TenantId              = $Tenant.ObjectId
        }

        Write-Output $Object
        $Object | ConvertTo-Json -Depth 1 | Out-File (Join-Path $OutPath -ChildPath "$ServicePrincipalName-Details.json")
    }
    catch
    {
        Write-Warning $_.Exception.Message
    }
}
```
