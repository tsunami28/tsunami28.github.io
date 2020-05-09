# Azure Application Gateway Reconfiguration


<!-- Your front matter up here -->

Azure team is doing great job with giving us a chance to change more and more options for different services via gui. Application gateway is one of them. But what when you need to implement any kind of automation, IaC ? Here, I still rely on PowerShell.

<!--more-->

In one of the projects I'm working on we needed to "reinvent" Blue-Green type of deployment for IaaS type of environment. We had 2 or more VMs in one backend pool of the Application Gateway.

In order to deploy code to only one VM without affecting production environment and what we serve to the clients, we decided to do it the way so we remove 1 VM from backend pool and push the code to it.

Easy task if you wanna do it manually each time with Application gateway. For IaC PowerShell to the rescue :)

<details>
    <summary markdown="span">Set-AzAppGw.ps1</summary>

{{< highlight powershell "linenos=false" >}}
function Set-AzAppGw {
    [CmdletBinding()]
    param (
        # Application Gateway name
        [Parameter(Mandatory = $true)]
        [string]
        $Name,
        # Backend IP address
        [Parameter(Mandatory = $true)]
        [string]
        $IPAddress
    )
    begin {
        $gw = Get-AzApplicationGateway -Name $Name
        <# In this case I assume there is only one backend pool. If you have more,
        you need to play with some small changes of this function #>
        $pool = Get-AzApplicationGatewayBackendAddressPool -ApplicationGateway $gw
        $BackendName = $pool.Name
        # Here we grab all the IPs from the backend
        [array]$Backend = ($pool.BackendAddresses).ipaddress
        # Defining new (future) list
        $NewIpList = [system.collections.arraylist]::new()
        foreach ($object in $Backend) {
            [void]$NewIpList.Add($object)
        }
        # Here we actually remove the needed IP
        [void]$NewIpList.Remove($IPAddress)
    }
    process {
        try {
            $ErrorActionPreference = 'Stop'
            # Setting up new BackendAddressPool
            [void](Set-AzApplicationGatewayBackendAddressPool -ApplicationGateway $Gw `
              -backendipaddresses $NewIpList -Name $BackendName)
            # This command actually configures Application Gateway with previously preparedchanges
            [void](Set-AzApplicationGateway -ApplicationGateway $gw)
        }
        catch {
            Write-error "$_" -ErrorAction stop
        }
    }
}
{{< /highlight >}}

</details>
<br/>

