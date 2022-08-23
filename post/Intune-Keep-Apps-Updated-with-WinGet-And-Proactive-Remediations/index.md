+++
title = "Keep Applications Updated with WinGet and Proactive Remediations"
date = "2022-08-22"
author = "David Just"
tags = ['O365','Intune','WinGet']
useRelativeCover = true
cover = "WingetTools.png"
+++

# The Why

In a previous article, I demonstrated [how to deploy applications to Intune using WinGet](https://davidjust.com/post/intune-install-software-with-winget/). I recieved a request to demonstrate how to use WinGet to update applications, and more importantly, how to run this on a schedule to keep applications updated. Since then, I found a really handy PowerShell wrapper module for WinGet called [WinGetTools by Jeffrey Hicks](https://github.com/jdhitsolutions/WingetTools). I made a small contribution to this module to allow it to work running under SYSTEM context. 

# The How

First, we need to install the above mentioned module WingetTools. This module has commands for listing out, installing, upgrading, and uninstalling packages, including listing which packages have updates available!

Head over to my Github repository and grab with [Install-WingetTools.intunewin](https://github.com/djust270/Intune-Scripts/blob/master/Install-WingetTools.intunewin) file. You can find the raw scipt [here](https://github.com/djust270/Intune-Scripts/blob/master/Install-WingetTools.ps1). Upload to Intune as a Win32 app set to install as System. For the installation command use:
```PowerShell
powershell.exe -executionpolicy bypass -file Install-WingetTools.ps1
```
For Uninstall:
```PowerShell
powershell.exe -command "Uninstall-Module -Name WingetTools"
```

For detection, use the detection script [here](https://github.com/djust270/Intune-Scripts/blob/master/Winget-InstallDetection.ps1)
As best practice always test new deployments with a pilot group. Go ahead and assign to this group. This can be user or devices, it does not matter. 

The app deployment should look like this:

 {{< image src="./Install-WingetTools.png" position="center" style="border-radius: 8px;" >}}

Now that we have the pre-requisites out of the way, lets move onto the proactive remediation. If you are not familiar with proactive remediations, [read up on it here](https://docs.microsoft.com/en-us/mem/analytics/proactive-remediations).

We need a detection script and a remediation script. The detection script will detect any applications installed with updates available in the WinGet repository (minus any apps we blacklisted). [Here is the script:](https://github.com/djust270/Intune-Scripts/blob/master/WingetUpgrade-ProactiveDetection.ps1)

```PowerShell
<#	
	.NOTES
	===========================================================================
	 Created with: 	SAPIEN Technologies, Inc., PowerShell Studio 2021 v5.8.195
	 Created on:   	8/22/2022 3:08 PM
	 Created by:   	David Just
	 Website: 	davidjust.com
	 Filename: WingetUpgrade-ProactiveDetection.ps1    	
	===========================================================================
	.DESCRIPTION
		Proactive Remediation detection script for applications with updates
#>

# Add any apps you do not wish updated here. These are apps which auto-update or which you know will cause a deployment issue. 
$Blacklisted = @(
	'Microsoft.Teams'
	'Microsoft.Office'
	'Microsoft.VC++2015-2022Redist-x64'
)
Import-Module -Name WingetTools
$AvailableUpdates = Get-WGInstalled | where-object { $_.id -notin $Blacklisted -and $_.update }

if ($AvailableUpdates.count -gt 0)
{
	"There are applications with Updates available"
	$AvailableUpdates | Select-Object -Property Name, ID, InstalledVersion, OnlineVersion
	exit 1
}
else
{
	"There are no apps to update"
	Exit 0
}
```

Now for the [remediation script](https://github.com/djust270/Intune-Scripts/blob/master/WingetUpgrade-Remediation.ps1):

```PowerShell
<#	
	.NOTES
	===========================================================================
	 Created with: 	SAPIEN Technologies, Inc., PowerShell Studio 2021 v5.8.195
	 Created on:   	8/22/2022 10:46 AM
	 Created by:   	David Just
	 Website: david.just.com
	 Filename: WingetUpgrade-Remediation.ps1
	===========================================================================
	.DESCRIPTION
		A description of the file.
#>
function Write-Log($message) #Log script messages to temp directory
{
	$LogMessage = ((Get-Date -Format "MM-dd-yy HH:MM:ss ") + $message)
	Out-File -InputObject $LogMessage -FilePath "$LogPath\$Log" -Append -Encoding utf8
}

$LogName = 'WinGetPackageUpgrade'
$LogDate = Get-Date -Format dd-MM-yy_HH-mm # go with the EU format day / month / year
$Log = "$LogName-$LogDate.log"
$LogPath = "$env:ProgramData\Microsoft\IntuneManagementExtension\Logs"
Import-Module WingetTools
$WingetPath = Get-WingetPath

# Add any apps you do not wish to get updated here (for instance apps that auto-update). Use the Winget ID
$Blacklisted = @(
	'Microsoft.Teams'
	'Microsoft.Office'
	'Microsoft.VC++2015-2022Redist-x64'
)
$AvailableUpdates = Get-WGInstalled | where-object { $_.id -notin $Blacklisted -and $_.update }
Write-Log -message "Packages with Updates Available:"
$AvailableUpdates | select Name, Version | Out-File -FilePath "$logpath\$Log" -Append -Encoding utf8


foreach ($App in $AvailableUpdates) # Invoke upgrade for each updatable app and log results
{
	[void](Get-Process | Where-Object { $_.name -Like "*$App.Name*" } | Stop-Process -Force)
	$UpgradeRun = & $WingetPath upgrade --id $App.id -h --accept-package-agreements --accept-source-agreements
	$UpgradeRun | Out-File -FilePath "$logpath\$log" -Append -Encoding utf8
	$Status = [bool]($UpgradeRun | select-string -SimpleMatch "Successfully installed")
	if ($Status -eq $true)
	{
		$Success += $App
	}
	else
	{
		$Failed += $App
	}
	
}


if ($Success.count -gt 0)
{
	Write-Log -message "Successful Upgraded the following:"
	$Success | Out-File -FilePath "$logpath\$log" -Append -Encoding utf8
	"Sucessfully Updated the following apps:`n{0}" -f $($Success | Select-Object -Property name)
}

if ($Failed.count -gt 0)
{
	Write-Log -message "Failed to Upgrade the following:"
	$Failed | Out-File -FilePath "$logpath\$log" -Append -Encoding utf8
}
```

Create a new Proactive Remediation deployment by going to Reports < Endpoint Analytics < Proactive Remediations [[direct link]](https://endpoint.microsoft.com/#view/Microsoft_Intune_Enrollment/UXAnalyticsMenu/~/proactiveRemediations). 

* Give the script package a descriptive name, such as "Winget Update Apps". 
* Upload the detection and remediations scripts. 
* Run in 64-bit and do not run with user credentials

{{< image src="./ProRem2.png" position="center" style="border-radius: 8px;" >}}

* Select your desired schedule. To test, I set this to run hourly, but daily or weekly would probably be better. Assign to your pilot group. 

This may take awhile to feed back results. When looking at the proactive remediation results, add the columns "Pre-remediation detection output" and "Post-remediation detection output" to see the output from the scripts. 

And thats it. Now you can keep applications updated with Intune and Proactive remediations. 
