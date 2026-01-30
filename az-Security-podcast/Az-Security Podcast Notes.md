
# Az-Security Podcast

## Intro Myself:

### Matthew Levy
I'm a Solutions Architect and Microsoft Security MVP, specialising in identity and access with Microsoft Entra ID. I spend most of my time helping organisations secure, and modernise their identity environments using Zero Trust principles. I’m a big believer in practical security over theory, and I like sharing real world lessons from the field through blogs, talks, and community work. If it touches identity, access, or security, that's where I can talk for hours.

## Global Secure Access
Today we’re talking about Microsoft Global Secure Access, specifically Private Access, and why it matters in a Zero Trust world. For years we’ve relied on VPNs and network trust to get users to private apps, but that model does not scale and it does not align with modern security threats. Private Access flips that on its head by putting identity at the centre, using Entra ID, Conditional Access, and continuous verification to control access to internal apps wherever they live, in Azure or on premises. This is really about moving away from network based trust and towards identity driven, least privilege access.

First let me introduce what Global Secure Access is and how Identity can fit in your Zero Trust Network Access model:
[Microsoft's SSE Architecture Overview](./Microsoft’s%20SSE%20architecture.pptx)

## Prerequisites
Before you can utilize GSA, you need the following prerequisites:

* A Microsoft Entra ID tenant
* A user with the Global Secure Access administrator, Application Administrator and Conditional Access Administrator roles.
* At least one test user
* At least one client device you'll use for user testing. To simulate remote access, this device should only have Internet access.
  * Windows 10/11 devices need to be either Microsoft Entra ID Joined or Hybrid Joined.
* Paid or trial licenses
  * Microsoft Entra Suite trial licenses https://aka.ms/EntraSuiteTrial
  * Get Microsoft Entra Private Access trial licenses https://aka.ms/PrivateAccessTrial

## Environment Setup

- [1. Activate Global Secure Access in Entra](#1-activate-global-secure-access-in-entra)
- [2. Get Token for Private Connector - PowerShell Script](#2-get-token-for-private-connector) 
- [3. Deploy Private Connector in Azure](#3-deploy-private-connector-in-azure)
- [4. Enable Private Access Traffic Profile in Entra](#4-enable-private-access-traffic-profile-in-entra)
- [5. Create Connector Group](#5-create-connector-group)
- [6. Enable Private Connectors in Entra](#6-enable-private-connectors-in-entra)
- [7. Get DNS suffix for Private DNS](#7-get-dns-suffix-for-private-dns)
- [8. Configure Private DNS in Quick Access](#8-configure-private-dns-in-quick-access)
- [9. Create Quick Access Enterprise App](#9-create-quick-access-enterprise-app)
- [10. Add Entire Network App Segment to Quick Access](#10-add-entire-network-app-segment-to-quick-access)
- [11. Add User(s)](#11-add-users)
- [12. Install GSA Client](#12-install-gsa-client)
- [13. Create Conditional Access Policy](#13-create-conditional-access-policy)
- [14. Test](#14-test)


## Links to resources:
#### Links

- Get the Auth Token for registering your Microsoft Entra private network connector through Azure, AWS, or GCP Marketplaces. - [Get Auth Token for Private Network Connector - PowerShell sample](https://learn.microsoft.com/en-us/entra/global-secure-access/scripts/powershell-get-token)
- [Microsoft Entra Private Network Connector (Preview) on Azure Marketplace](https://marketplace.microsoft.com/en-us/product/microsoftcorporation1687208452115.entraprivatenetworkconnector?tab=Overview)
- [AWS Marketplace: Entra Private Network Connector](https://aws.amazon.com/marketplace/pp/prodview-cgpbjiaphamuc)
- [GCP Marketplace: Entra Private Network Connector](https://console.cloud.google.com/marketplace/product/ciem-entra/entraprivatenetworkconnector)
- [Video: Deploy the connector in Azure - Azure Marketplace](https://www.youtube.com/watch?v=T4QaUSNsn-s)
- [Known limitations of Private Access](https://learn.microsoft.com/en-us/entra/global-secure-access/reference-current-known-limitations)  
- [My LinkedIn Profile](https://www.linkedin.com/in/mattlevy42/)
- [My Blog](https://mattchatt.co.za/blog)
- [Twitter/X](https://x.com/MattChatt42)
- [BlueSky](https://bsky.app/profile/mattchatt.co.za)




### 1. Activate Global Secure Access in Entra

See [40s video](./Presentation2.mp4) of this step 

### 2. Get Token for Private Connector
In order to deploy the Private Connector from the Marketplace in Azure, one first needs to obtain an Authentication Token from Entra which will be used to automatically register the Private Connector to the correct Entra Tenant.
This is done on another Windows machine that does not already have the connector installed. I use PowerShell ISE on a Windows Server.

#### Token Script
```powershell
# This sample script lets you obtain the Auth Token that you can use for registering the Entra private network connector through Marketplace.
#
# Version 1.0
#
# This script requires following 
#    - PowerShell 5.1 (x64) or beyond
#    - Module: MicrosoftEntraPrivateNetworkConnectorPSModule 
#
# The script will get the module as result of Entra Private Network Connector Installation and quiet Registration (/q flag). A quiet installation doesn't prompt you to accept the End-User License Agreement.
# This script will uninstall the Entra Private Network Connector once the required modules are downloaded. 
#
# Before you begin:
#    
# - Make sure you are running PowerShell as an Administrator
# - You are on Windows Machine which is not running the Entra Private Network Connector already. If you already have a connector installed, quite registration step below will fail. 
# - Make sure there in no C:\temp folder on the machine. If you have some files stored, please move those before running the script 
# Make sure ExecutionPolicy is set to Unrestricted
Set-ExecutionPolicy UnRestricted -Force
# The script will use a temp folder on C Drive. First it will remove the folder and create a new folder to ensure its empty.
$tempPath = "C:\temp"
# Check if the folder exists
if (Test-Path -Path $tempPath) {
Write-Host "Your C Drive has existing temp folder that is being deleted"
Remove-Item -Path C:\temp -Recurse
} 
# Creating C:\temp folder
New-Item -ItemType Directory c:\temp
New-Item -ItemType File -Path C:\token.txt -Force

# Copy Required Dlls 
Invoke-WebRequest https://download.msappproxy.net/Subscription/d3c8b69d-6bf7-42be-a529-3fe9c2e70c90/Connector/DownloadConnectorInstaller -OutFile c:\temp\MicrosoftEntraPrivateNetworkConnectorInstaller.exe

# Set the prompt path to C:\temp
cd "C:\temp"

# Quiet Registration of the Connector. This step will provide the required Module for acquiring the token. 
# At the end of this step, you should see 2 folders under C:\Program Files. 1) Microsoft Entra private network connector 2) Microsoft Entra private network connector updater
# These folders contains the required modules needed for getting the token. 
.\MicrosoftEntraPrivateNetworkConnectorInstaller.exe REGISTERCONNECTOR="false" /q

#Wait 60 seconds
Start-Sleep -Seconds 60

$folderPath = "C:\Program Files\Microsoft Entra private network connector\Modules\MicrosoftEntraPrivateNetworkConnectorPSModule"

# Check if the Module exists
if (Test-Path -Path $folderPath) {
    Write-Host "The Module is successfully made available at path: $folderPath"
}
   
# Set the prompt path to C:\Program Files\Microsoft Entra private network connector\Modules\MicrosoftEntraPrivateNetworkConnectorPSModule
cd "C:\Program Files\Microsoft Entra private network connector\Modules\MicrosoftEntraPrivateNetworkConnectorPSModule"

# Import Module 
Import-Module ..\MicrosoftEntraPrivateNetworkConnectorPSModule -ErrorAction Stop

# Load MSAL  
Add-Type -Path .\Microsoft.Identity.Client.dll

# The AAD authentication endpoint uri

$authority = "https://login.microsoftonline.com/common/oauth2/v2.0/authorize"

#The application ID of the connector in AAD. Use the Connector AppId below

$connectorAppId = "55747057-9b5d-4bd4-b387-abf52a8bd489"

#The AppIdUri of the registration service in AAD
$registrationServiceAppIdUri = "https://proxy.cloudwebappproxy.net/registerapp/user_impersonation"

# Define the resources and scopes you want to call

$scopes = New-Object System.Collections.ObjectModel.Collection["string"]

$scopes.Add($registrationServiceAppIdUri)

$app = [Microsoft.Identity.Client.PublicClientApplicationBuilder]::Create($connectorAppId).WithAuthority($authority).WithDefaultRedirectUri().Build()

[Microsoft.Identity.Client.IAccount] $account = $null

# Acquiring the token

$authResult = $null
$authResult = $app.AcquireTokenInteractive($scopes).WithAccount($account).ExecuteAsync().ConfigureAwait($false).GetAwaiter().GetResult()

# Check AuthN result
If (($authResult) -and ($authResult.AccessToken) -and ($authResult.TenantId)) {
$token = $authResult.AccessToken
   $tenantId = $authResult.TenantId
}
else {
Write-Output "Error: Authentication result, token or tenant id returned with null."
}

$accessToken = $token

Set-Content -Path C:\token.txt -Value "$accessToken"

# Set the prompt path to C: 

cd "C:\"

# Uninstall the Connector from your machine.
# You can do so programmatically (below) or manually by double clicking C:\temp\MicrosoftEntraPrivateNetworkConnectorInstaller.exe and choose Uninstall. 
# Note that if the Connector service is not uninstalled properly, next iteration can fail on this machine.

C:\temp\MicrosoftEntraPrivateNetworkConnectorInstaller.exe /uninstall /quiet

#Wait 60 seconds
Start-Sleep -Seconds 60

# Delete the related files. Note that if you need to get the token again from

Remove-Item -Path "C:\temp" -Recurse
Remove-Item -Path "C:\Program Files\Microsoft Entra private network connector" -Recurse
Remove-Item -Path "C:\Program Files\Microsoft Entra private network connector updater" -Recurse

Write-Output "Access Token that you acquired is available in C:\token.txt. "
Write-Output "Please ensure no additional spaces are introduced when copying token to marketplace input form. Introducing spaces can change the token and can cause failures"

else {
    Write-Host "The required module is not made available at path: $folderPath"
	Write-Host "This could be related to left over state from previous installation of connector on this machine."
	Write-Host "You can try to go to c:\temp\ and double click the MicrosoftEntraPrivateNetworkConnectorInstaller.exe file. Click Uninstall if visible. This can clean the state. "
    Write-Host "If you don't have .exe file, you can download it from https://download.msappproxy.net/Subscription/d3c8b69d-6bf7-42be-a529-3fe9c2e70c90/Connector/DownloadConnectorInstaller and double click it to Uninstall"
	Write-Host "Try Again after the state is clean"
    return
}
```

### 3. Deploy Private Connector in Azure

### 4. Enable Private Access Traffic Profile in Entra

### 5. Create Connector Group

### 6. Enable Private Connectors in Entra

### 7. Get DNS suffix for Private DNS

### 8. Configure Private DNS in Quick Access

### 9. Create Quick Access Enterprise App

### 10. Add Entire Network App Segment to Quick Access

```text
10.0.0.0/16 
Ports 1-52,54-65535 
TCP & UDP
```

### 11. Add User(s)

`GSA-Internal-QuickAccess-Users`


### 12. Install GSA Client

### 13. Create Conditional Access Policy

![ConditionalAccess](GSA%20Conditional%20Access.png)

### 14. Test

`mstsc 10.0.1.4`
`mstsc 10.0.1.5`

`\\files.myignite.biz\atlas`

