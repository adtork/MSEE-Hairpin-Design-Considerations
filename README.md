# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this article, we will explore the concept of hair-pinning, also referred to as MSEE hair-pinning. This involves a process where traffic from a VNET, connected via an ExpressRoute circuit, exits to the Microsoft Enterprise Edge (MSEE) pop location prior to entering the destination Vnet. This default behavior is observed both within a single region (a single circuit with multiple Vnets) and across multiple regions (multiple circuits with multiple Vnets).

> [!Important]
>While this design methodology has been functional for years, it is no longer recommended due to the increased latency caused by traffic hair-pinning to the peering location points of presence (POPs) hosting the MSEEs. This approach also places a significant load on the ExpressRoute gateways for ingress traffic to the destination Vnet.

> [!NOTE]
>Historically, solutions to this approach, which are currently deemed outdated, involved advertising a summary route for intra-region traffic, encompassing the entire Vnet address prefix from on-premise. Additionally, the traditional 'bow-tie' approach was utilized, where each Vnet is connected to the other ExpressRoute circuit, thus forming a 'bow-tie'. For more details on these outdated approaches, please refer to the previous article titled [Common design principles for a hub and spoke](https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture). 

# Option 1: Using an NVA inside Hub Vnets
The first option to get around traffic hairpinning down to the MSEE pops is to simply put a NVA or VM with ipforwarding in the hub Vnets. On each spoke Vnet, you would then need to create a UDR pointing to the remote spoke Vnet prefix, or default route (0.0.0.0/0) with next hop hub NVA. Doing this, traffic would be routed to the NVAs for spoke to spoke communication instead of going down to the MSEE POP location. Simmilary for inter-region flows, you would take the same approach but would also need to global vnet peer the hubs for them to commmunicate cross region. You would then point to the remote hub firewall with destination remote spokes. Its important to note, you would need to add a UDR for every spoke you wish to connect to, unless you do a summary route composing a supernet of all the spoke vnets. Another approach to using NVAs in hubs is to BGP peer them using ARS. This takes away the need to manage UDRs. For inter-region topology using ARS, see article: [Multi region ARS Design](https://learn.microsoft.com/en-us/azure/route-server/multiregion#topology).

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/20cbbdf6-6dcc-4302-a31a-076de029f3c9)

**Pros:**
 - Traffic no longer hairpins to MSEE edge pop locations
 - Traffic no longer ingresses threw remote gateways. This takes the load off the gateways. 
 - Granular control over NVAs

**Cons:**
 - Cost of running the NVAs
 - Responsibility of managing the NVAs
 - UDR Management overheard

# Option 2: Vnet Peering
The second option and really the easiest to deploy is to simply peer all the spoke Vnets directly that require connectivity. Like above with Option 1, inter region spokes would require global Vnet peering to communicate. Another recently introduced option to build full Vnet peering meshes is to use Azure Virtual Network Manager (AVNM). You can build intra region meshes and global peering meshes. The goal of AVNM is to manage resources at scale and simplify management overhead. Currently this is in public preview at the time of this article. [AVNM Design Considerations](https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview). 

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/8ec123ce-5361-40d4-b6cf-78377ec2f8d9)

**Pros:**
 - Easy to setup/deploy
 - No administration cost
 - No UDR Management
 - Lowest possibly latency. There are no VM or gateway choke points. 

**Cons:**

- Limit to the number of Vnets that can be peered (500)
[Vnet Peering Limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits).

# Option 3: Azure Virtual WAN with HRP(AS-PATH)
The third and final option is to deploy Azure virtual WAN. The benefits to using vWAN is that it simplifies routing overall and provides native transit connectivity for everything except ExR to ExR, which would require global reach. For intra region traffic, spokes simply take the hub routers to communciate directly using default route table. Spoke traffic can also be steered using custom routes, but that is beyond the scope of this article. For inter-region traffic, (vhub to vhub), in order to avoid the MSEE hairpin, you would need to set the hub routing preference (HRP) to AS-PATH if using ExR bow-tie. Its important to note, if you're not using bow-tie and each vhub has a unique circuit, it will take hub to hub automatically. You only need to set HRP when doing the bow-tie!. In a normal scneario using bow-tie as shown below, traffic still hairpins at the MSEE for inter-region flows. We need to change HRP from **Expressroute which is default**, to **HRP AS-PATH**. Information on Hub Routing Preference can be found here: [vWAN HRP](https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference).

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/e9eb5596-82a1-4721-9029-3c393f862727)


**Pros:**

 - Routing is taking care of automatically by the Azure platform via hub routing instances.
 - No NVAs or UDR mangement
 - No hairpin to the MSEE edge pop if HRP is set to AS-PATH

**Cons:**

 - Would require a redesign if traditonal Hub+Spoke already deployed
 - HRP may affect other routes in the environment, for example VPN and SDWAN tunnels
 - Less visability into vHubs since they are MSFT managed Vnets

# Conclusion
The above are design alternatives to direct traffic for intra and inter region designs using ExpressRoute. The old approaches of doing a "summary route" for intra region and "bow-tie" for inter-region should be discourgaged because traffic still hairpins at the MSEE which adds latency and is discouraged moving forward. 





