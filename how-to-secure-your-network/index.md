# How-to-Secure-Your-Network

Lately I came accross a lot of unsecurity and even knowledge lack about how to secure your Azure networking communication. Based on the example I will showcase you differences between no security and using Service or Private Endpoints.
<!--more-->
Let's start with building the initial services we need for this POC:
1. Resource group for our POC (consider everything below to be created inside this RG)
2. Virtual network with 2 subnets (one will be used for VM and the other one later on for Private Endpoint)
3. Network Security Group - connected to our VM subnet allowing RDP from only mine/yours Public IP
4. Virtual machine (I used Win10 with public IP)
5. Storage Account - during creation choose "Public access from all networks"

After creating all these, we need our Storage account Blob service URL (You will find it under Properties).

Next step is to connect to our VM and check how it resolves Blob Service URL. You should get something like this (notice that it has Public IP!):

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_215.png" title="Blob Public IP" >}}

Now if we use Network Watcher to check Next hop we get this:
 
{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_216.png" title="Next Hop" >}}

You will notice that we are accessing our Blob over its public point via Internet. If you haven't enabled only https access, we already have a big security issue potentialy sending data unencrypted over the Internet!

Let us now use first option to secure our access - [Service Endpoint.](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_217.png" title="Enabling Service endpoint on subnet in VNet" >}}

As stated in docs it will allow us to reach our Azure service without needing a public IP address on the VNet. The thing is that resolution is still the same, so the final access to the service is still over its public IP.

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_218.png" title="Change in the Next Hop" >}}

Latest implemented security option for most of the services in Azure is now [Private Endpoint.](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview)

In order to use it there are some prerequisites:
1. We need VNet subnet dedicated for it (best pactise if you ask me)
2. Private DNS zone - It will host our Private endpoints A records

Private endpoint is created per service, so we need to initiate its creation from our Storage account settings. During the creation we will be asked about DNS zone (it can create it for us if we don't have it yet) which is highly recomended.

Now, after creating it, let's see how we resolve our Blob from VM:

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_219.png" title="Blob Private IP" >}}

We can confirm the same looking at the DNS zone:

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_220.png" title="DNS record" >}}

As you can see, our Blob now has private IP assigned from our VNet subnet.

And the final step is to test Next hop with Network Watcher:

{{< figure src="/images/How-To-Secure-Your-Network/Screenshot_221.png" title="Fully secured from the Network flow perspective" >}}

Don't forget to additionally set, in this case, Storage Account Firewall settings. Only then you will be fully secured.

And to conclude this, I strongly believe that there is no reason NOT to use Private endpoints. They will allow you to use your internal IP address spacing and thus integrate more security while accessing Azure services. You will also use full potential of the Azure backbone network since network traffic won't "leave" it.

P.S. Don't forget to delete previously created RG in order to save money :smile:

