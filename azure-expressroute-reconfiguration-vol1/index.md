# Azure ExpressRoute Reconfiguration Vol1

Recently I was working on an assignment, that at the start seemed like an easy one. We had to redeploy ExpressRoute Circuit which required that we first remove all the connections towards that Circuit. An important thing to mention is that all is done via Azure DevOps release pipelines and that the parameter we had to use is the circuit name.
<!--more-->

{{< figure src="/images/Azure-ExpressRoute-Reconfiguration-vol1/ERCircuit.jpg" width="40%">}}

As I like to use it, and I have some experience with it, my first weapon of choice was PowerShell and Az module.
So, I started with `Get-AzExpressRouteCircuit` command. When I looked at the [properties](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.commands.network.models.psexpressroutecircuit?view=azurerm-ps#properties) for the output object I realized that there is nothing about the connections in it (at least not the ones we needed).

OK, let us look at the `Get-AzExpressRouteConnection`. It requires info about ExpressRouteGateway. Wait. What? But we are using Virtual Network Gateways. Is that the same?! You're already guessing - it's not. ExpressRouteGateway gets created within Virtual Hub, which is part of the great Azure service - [Virtual WAN](https://docs.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about). But we are still not using that one, but plain and simple Hub and Spoke topology in combination with Virtual Network Gateways.

Since ExpressRoute commands were not helping me, I decided to approach this from the other side - the connection one. What I know about them:

* They are created as part of the Virtual Network Gateway
* I could find their names (multiple connections)
* I could get the VNGW name

Remember at the beginning I mentioned that all is run from the pipeline where I want to pass only ER Circuit name. Now I had to find a correlation between connections and circuits that I could filter by (just because I have multiple circuits and I cannot delete just any connection).

Now I decided to try these two commands: `Get-AzVirtualNetworkGateway` and `Get-AzVirtualNetworkGatewayConnection`.
If we look at the [properties](Get-AzVirtualNetworkGatewayConnection) of the output object for the first command, we may notice that there is no mention of the ExpressRoute Circuit(s) that it's used for. Browsing the [properties](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.commands.network.models.psvirtualnetworkgatewayconnection?view=azurerm-ps#properties) for the second one I had more luck. The "PeerText" property is caring the ID of the circuit that the connection is used for.

So, comparing this one to the ID property of the `Get-AzExpressRouteCircuit` command would give me the desired connection we need to delete. Bingo! Let's write some code.

```PowerShell
function Remove-ERConnection {
    [CmdletBinding()]
    param (
        # ExpressRoute Circuit Name
        [Parameter(Mandatory = $true)]
        [string]
        $ExpressRouteCircuitName
    )
    begin {
        $ERCircuit = Get-AzExpressRouteCircuit -Name $ExpressRouteCircuitName
        $ERCircuitId = $ERCircuit.Id
        $RemovedConnections = @()
    }
    process {
        $Connections = Get-AzVirtualNetworkGatewayConnection -Name * -ResourceGroupName 'RGName'
        foreach ($connection in $Connections) {
            if ($connection.PeerText -match $ERCircuitId) {
                Write-Output "Ready to delete connection: " $connection.Name
                Remove-AzVirtualNetworkGatewayConnection -Name $connection.Name -ResourceGroupName 'RGName' -Force -WhatIf
                $RemovedConnections += $connection.Name
            }
        }
    }
    end {
        Return $RemovedConnections
    }
}
```

There it is. We deleted our connection.

Unfortunately, this 'resolution' brought more questions. You might notice that we had to pass the Resource Group name for both Get and Remove-AzVirtualNetworkGatewayConnection commands. That is not something you want to hard-code and we are not passing that in our pipeline. Also, what if we had multiple connections to the circuit within different Virtual Network Gateways (that are in different RGs)?

The only way we could solve it using PowerShell was to put all Resource Groups in one variable and then iterate through each one of them searching for connections.

This was unacceptable because it would be time-consuming and just not good practice.

Back to the drawing board. Would you like to know how I solved it at the end? Stay tuned for the next post :wink:

