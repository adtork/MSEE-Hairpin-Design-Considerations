# ExpressRoute Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET over expressroute egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to another destination Vnet. This is the default behavior for intra-region (Single circuit with multple Vnets -Single Region) and inter-region (Mutiple circuits with multiple Vnets -Dual Region). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location hosting the MSEEs. Its important to note, in the near future this behavior will be changed, but its still current behavior at the time of this article. We are going to discuss the various design alternatives to this behavior and the pluses and minuses of each design. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Topology

# Intra-Region (One Region)
![image](https://user-images.githubusercontent.com/55964102/195736520-332ac15c-9781-4517-9115-cb7e7d83a837.png)

# Inter-Region (Two Regions)
![image](https://user-images.githubusercontent.com/55964102/195735758-5fd89852-c687-470f-82da-de868afe8aaf.png)

In both designs above we are using standard hub and spoke model. The spoke vnets are peered to hub with "Allow GW Transit" and "Use Remote Gateways" enabled on the peerings in order to reach on prem and have BGP routes plummed to the spokes. In the following sections we will discuss options for spoke to spoke communication via intra region and spoke to spoke communuation via inter-region. We will also discuss hub to hub communciation in both deployment models.

# Option 1: BGP Summary Route




 


