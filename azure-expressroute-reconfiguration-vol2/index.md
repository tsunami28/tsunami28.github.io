# Azure-ExpressRoute-Reconfiguration-Vol2

In the previous [post](https://doctorsolution.xyz/azure-expressroute-reconfiguration-vol1/) I described the approach to remove specific ExpressRoute VirtualNetworkGateway connections, solely using the PowerShell basic commands. As mentioned, that did not serve our purpose.
<!--more-->

{{< figure src="/images/Azure-ExpressRoute-Reconfiguration-vol2/ERCircuit.jpg" width="40%">}}

I had to find a different way of getting the connections that we needed to remove. Thankfully, the team that I recently joined, was already using one great Azure service in many other situations - [Azure Resource Graph](https://docs.microsoft.com/en-us/azure/governance/resource-graph/).

As someone who likes to work with PowerShell, I was happy to see that there is a module for Resource Graph. The only catch is that it does not come with "basic" AZ module installation, but you must add it individually.

```PowerShell
# Install the Resource Graph module from PowerShell Gallery
Install-Module -Name Az.ResourceGraph

# Get a list of commands for the imported Az.ResourceGraph module
Get-Command -Module 'Az.ResourceGraph' -CommandType 'Cmdlet'
```

After a bit of playing around with queries within the Azure Resource graph, I managed to create one that would return exactly what we needed :smiley:

Let's see how I built it. The first command will list all types of connections (based on our deployed resources).

```PowerShell
Resources | where type =~ 'Microsoft.Network/connections'
```

Looking into the properties of the result, I found two that will give me the result I need.

```PowerShell
Resources
    | where type =~ 'Microsoft.Network/connections'
    | where (properties.connectionType) == 'ExpressRoute'
    | where (properties.peer.id) contains '$ExpressRouteCircuitName'
```

Now, I was ready to build a function around the Resource Graph query. I believe that the code is relativly simple, so I won't deep dive into explanations. Read and enjoy :wink:

```PowerShell
function Remove-ERConnection {
    [CmdletBinding()]
    param (
        # Name of the ER Circuit
        [Parameter(Mandatory = $true)]
        [string]
        $ExpressRouteCircuitName,
        # Name of the Subscription
        [Parameter(Mandatory = $false)]
        [String]
        $SubscriptionName,
        # Name of the Connection
        [Parameter(Mandatory = $false)]
        [string]
        $ConnectionName
    )
}

$query = "Resources
        | where type =~ 'Microsoft.Network/connections'
        | where (properties.connectionType) == 'ExpressRoute'
        | where (properties.peer.id) contains '$ExpressRouteCircuitName'"

$connectionsData = Search-AzGraph -Query $query
if ($ConnectionsData.Count -eq 0) {
    Write-Error "There are no connections found for the ExpressRoute Circuit named $ExpressRouteCircuitName"
}
$RemovedConnections = @()
# This gives us an option to remove single connection we defined
if ($ConnectionName.Length -gt 0) {
    $SingleConnection = $ConnectionsData | Where-Object -FilterScript { $_.Name -eq $ConnectionName }
    $ConnectionId = $SingleConnection.id
    $ResourceGroupName = $SingleConnection.resourceGroup
    $ConnectionObject = Get-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName
    Write-Output "Connection to be deleted: $ConnectionName with ID: $ConnectionId"
    Remove-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName -Force
    $RemovedConnections += $ConnectionName
    # This implements wait time until connection is removed. It was important for us not to proceed with next task within stage before connection is removed.
    do {
        Write-Output "Waiting for connection $ConnectionName to be deleted"
        Start-Sleep -Seconds 15
    } until (!($ConnectionObject = Get-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName -ErrorAction SilentlyContinue) -eq $true)
}
# This will remove each connection
else {
    foreach ($Connection in $connectionsData) {
        $ConnectionName = $Connection.name
        $ConnectionId = $Connection.id
        $ResourceGroupName = $Connection.resourceGroup
        $ConnectionObject = Get-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName
        Write-Output "Connection to be deleted: $ConnectionName with ID: $ConnectionId"
        Remove-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName -Force
        $RemovedConnections += $ConnectionName
        do {
            Write-Output "Waiting for connection $ConnectionName to be deleted"
            Start-Sleep -Seconds 15
        } until (!($ConnectionObject = Get-AzVirtualNetworkGatewayConnection -Name $ConnectionName -ResourceGroupName $ResourceGroupName -ErrorAction SilentlyContinue) -eq $true)
    }
}
Return $RemovedConnections
```

As you can see, we achieved our goal - having only ExpressRoute name as an input. I also decided to expand the possibilities of the function a bit, so with the connection name, we can choose to delete just one.

This is a solution for a specific situation, but hopefully, it will give you an idea of how to get to other data that will help you in solving other challenges.

