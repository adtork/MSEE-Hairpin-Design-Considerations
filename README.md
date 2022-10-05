# ExpressRoute Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET over expressroute egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to another destination Vnet. This is the default behavior for intra-region (Single Circuit with multple Vnets -Single Region) and inter-region (Mutiple circuits with multiple Vnets -Dual Region). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location hosting the MSEEs. Its important to note, in the near future this behavior will be changed, but its still current behavior at the time of this article. We are going to discuss the various alternatives to this behavior and the pluses and minuses of each design. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Topology

# Intra-Region (One Region)
![image](https://user-images.githubusercontent.com/55964102/194118074-255c79b9-5b85-40a2-a1c0-6ac747496537.png)

# Inter-Region (Two Regions)
![image](https://user-images.githubusercontent.com/55964102/194113252-44ddf2db-1033-476f-8809-3a5280c8293a.png)

 


