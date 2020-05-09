# Mighty Azure portal vol1


<!-- Your front matter up here -->
How many times did you find yourself in need to do some repetitive tasks on Windows Servers? Since 2006. when PowerShell first appeared, a lot has changed. We've been writing scripts, functions, and all the code that could help us in order to ease some Admin pain and speed up things.
With PowerShell 2 came PS remoting, so we were able to execute commands on other servers too.

How do you manage certain things today in the era of Cloud? Let me guide you through one simple how-to.
<!--more-->

After spinning a certain infrastructure in Azure, based on IaaS, I was asked to create a couple of local admin users on each VM. It would be ok if there weren't 10+ VMs at the start. Without even logging in to the first, I decided to try to automate the task as much as possible and again to use the security guidelines that we agreed upon.

When you go to the VM, under "Operations" there is a "Run command" which, among other things, gives you the option to execute a PowerShell script directly from Portal. Easy and powerful. But this is just a start.

I had to use Secrets stored in KeyVault. No problem, there is a straight PS command that can do that. But how will my VM access the protected KeyVault (Networking set to use Private endpoint and selected networks, Access policies hardening who/what can even read data)? Well, it needs to authenticate somehow. If we look at our VM again, under "Settings" there is an "Identity" option. Here you can activate System assigned managed identity which will enable our VM to authenticate to e.g KeyVault.
After enabling this option, you will have an object that you can add in KeyVault Access Policies with desired permissions. Great, now our VM can grab the data it needs.

Now, in already mentioned VM Run Command we should be able to execute the script.

<details>
    <summary markdown="span">Add-AzureVMLocalUser.ps1</summary>

{{< highlight powershell "linenos=false" >}}
<# If you haven't previously logged to VM and install any module, this is mandatory,else
our script will hang forever because "Run Command" doesn't ask for confirmation when needed #>
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted

<#This command is needed on Windows Server 2016 - You will first have to install Nuget package
which is refusing default Tls1.0 #>
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

Install-Module Az

<#Here we use previously enabled Managed identity to login to our Azure subscription #>
Connect-AzAccount -Identity

$vaultName = "ENTER VAULT NAME HERE"
$user = "AdminUsername" # change with your KeyVault data
$pass = "AdminPassword"

$username = (Get-AzKeyVaultSecret -VaultName $vaultName -Name $user).SecretValueText

$password = (Get-AzKeyVaultSecret -VaultName $vaultName -Name $pass).SecretValueText
$secpasswd = ConvertTo-SecureString $password -AsPlainText -Force

<# This can be skipped if not needed #>
$date = get-date
$dateExpire = $date.AddDays(35)

New-LocalUser -Name $username -Password $secpasswd -AccountExpires $dateExpire

Add-LocalGroupMember -Group "Administrators" -Member $username
{{< /highlight >}}

</details>

If you have a certain naming convention for Username and Password data in KeyVault, you can grab multiple secrets and with a simple "foreach" push n users to a VM.
Smooth, isn't it :)

Listening to some great talks on numerous Azure meetups and online conferences in the past 2 months of WFH, I came to an additional idea - I will try to use Azure Functions to trigger user creation/update on certain changes in KeyVault. Interested if that is possible? Me to :)

Stay tuned and stay healthy!

