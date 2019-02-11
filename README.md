# Run-a-Custom-Script-on-a-VM-when-Log-Analytics-Alert-is-triggered
Running a custom script on a specific machine when an alert is triggered in Log Analytics

Sometimes you want to trigger a specific action when something is detected by one of your alert rules inside of Azure. If you want to immediately remediate the specific issue you are facing normally you would have to login to the machine once you get the message, but by using an Automation Account and you don’t have to do anything just create it and leave it to run whenever the alert is triggered. As simple as that.

This works perfectly when you need to resolve a common issue and to fix this you have trusty PowerShell script that you always run to resolve this. Using this method will save you time and effort, you can rest assured that the issue is being taken cared of with the help of a Custom Script Extension.

Running a custom script on a specific machine when an alert is triggered in Log Analytics is quite easy, here are the following steps you need to follow to achieve this:

1.	Upload your script file in a Storage Account
2.	Create the Automation Account and a new Runbook
3.	Link your runbook to the Action Group in Log Analytics Alerts

1.	Upload your script file in a Storage Account
The first step is to upload the script file that will run whenever your defined alert is raised. Go to your Storage Account and click on Blobs. 

Click on Add a Container, select the Public access level to a Container Level. Give a name and click Ok, set access to Public Access for container and blobs.

Next is to upload the script file, click on Upload. Save your file. Now, copy the blob file URL, you will need this later.

2.	Create the Automation Account and a new Runbook
Let’s create a new Automation Account, click Yes to create a new Run As Account.

Once created, go to Runbooks. In here we will add the PowerShell Script that will be used to trigger the action. Click on Create a Runbook. In my case this Runbook will retrieve a PowerShell script from a Storage Account and will execute it into the machine I define inside of the Runbook.

Add the following code inside of the Runbook:

###################################################################
$InitialExecutionDatetime = (Get-Date)
Write-Output "Script start time $($InitialExecutionDatetime)"
$CurrentDateTime=$InitialExecutionDatetime.ToUniversalTime()
$connectionName = "AzureRunAsConnection"
$Conn = Get-AutomationConnection -Name $connectionName 
#We are login using a Service Principal Account
[int]$NoOfDays=1
Write-output "Login to Azure"
$RetunrObjItm = new-object PSObject
[int]$counter=0
$AzureLoginResults = Login-AzureRmAccount `
-ServicePrincipal `
-TenantId $Conn.TenantId `
-ApplicationId $Conn.ApplicationId `
-CertificateThumbprint $Conn.CertificateThumbprint -ErrorAction SilentlyContinue -ErrorVariable LoginError `

# We set the Subscription we are going to use
Set-AzureRmContext -SubscriptionId "<SubscriptionID>"

# Credential Defined in the Automation Account, this account will be used to execute the script
$Cred = Get-AutomationPSCredential -Name 'TestVM Cred'

#Specify the VM onto which you want to act upon
$VMResourceGroup = "GMtestingVaultRG"
$ResourceGroupLocation = "Central US"
$VMName = "GMTest01"

#Provide the script file URL
$ScriptURI = "<URL of the blob file>"

#Run Script on VM
Write-output "Running script on the VM"
Set-AzureRmVMCustomScriptExtension -ResourceGroupName $VMResourceGroup `
-VMName $VMName `
-Location $ResourceGroupLocation `
-FileUri $ScriptURI `
-Run "DiskCleanup.ps1" `
-Name "CustomScriptExtension" `
-ForceRerun $(New-Guid).Guid `

#Script Results
Write-output "Completed running script on the VM"                
###################################################################

Let’s take a closer look at how to get the values for each key component of the script:

# Credential Defined in the Automation Account, this account will be used to execute the script
$Cred = Get-AutomationPSCredential -Name 'TestVM Cred'

To create this, we need to go to our Automation Account and click on Credentials:

 

Add a new credential, this account must have appropriate permissions to be able to run a script inside of the VM.

#Specify the VM onto which you want to act upon
$VMResourceGroup = "GMtestingVaultRG"
$ResourceGroupLocation = "Central US"
$VMName = "GMTest01"

This one is fairly obvious, you just need the Name, Resource Location and Resource Group.


#Provide the script file URL
$ScriptURI = "<URL of the blob file>"

You can find the URL of the blob file here:

  

#Run Script on VM
Write-output "Running script on the VM"
Set-AzureRmVMCustomScriptExtension -ResourceGroupName $VMResourceGroup `
-VMName $VMName `
-Location $ResourceGroupLocation `
-FileUri $ScriptURI `
-Run "DiskCleanup.ps1" `
-Name "CustomScriptExtension" `
-ForceRerun $(New-Guid).Guid `

This part is where everything comes together at the end, we will use the Custom Script Extension to run this specific script on the target machine. We must use all of the variables we gathered and also put the name of the script we want to trigger as well as the friendly name of the extension we will create on the VM. 

-ForceRerun $(New-Guid).Guid `

This line is very important because it will help us to rerun the script on the machine, if you don’t put that the script will try to install again the Extension on the machine, we just need it to run whenever the alert is triggered.

3.	Link your Runbook to the Action Group in Log Analytics Alerts

Last step is to go to the Alerts section of Log Analytics and create a new Management Action Group. On Actions lets select as the Action Type Automation Runbooks. In here we can select the Runbook we previously created.

Click Ok, now let’s link this action group to the Alert itself, on the Action Groups click on Select existing and we can add the action group we just created:

Click on Save and whenever the alert is raised you will run a custom script on the target machine you specified on the Runbook.
