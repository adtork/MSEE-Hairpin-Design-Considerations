# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this article, we will explore the concept of hair-pinning, also referred to as MSEE hair-pinning. This involves a process where traffic from a VNET, connected via an ExpressRoute circuit, exits to the Microsoft Enterprise Edge (MSEE) pop location prior to entering the destination Vnet. This default behavior is observed both within a single region (a single circuit with multiple Vnets) and across multiple regions (multiple circuits with multiple Vnets).

> [!Important]
>While this design methodology has been functional for years, it is no longer recommended due to the increased latency caused by traffic hair-pinning to the peering location points of presence (POPs) hosting the MSEEs. This approach also places a significant load on the ExpressRoute gateways for ingress traffic to the destination Vnet.

> [!NOTE]
>Historically, solutions to this approach, which are currently deemed outdated, involved advertising a summary route for intra-region traffic, encompassing the entire Vnet address prefix from on-premise. Additionally, the traditional 'bow-tie' approach was utilized, where each Vnet is connected to the other ExpressRoute circuit, thus forming a 'bow-tie'. For more details on these outdated approaches, please refer to the previous article titled [Common design principles for a hub and spoke](https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture). 

# Option 1: Using an NVA inside Hub Vnets
The initial strategy for circumventing traffic hairpinning to the MSEE points of presence (POPs) involves the incorporation of a Network Virtual Appliance (NVA) or a Virtual Machine (VM) with IP forwarding within the hub Vnets. Subsequently, each spoke Vnet necessitates the creation of a User-Defined Route (UDR) that points either to the remote spoke Vnet prefix or to the default route (0.0.0.0/0) with the next hop being the hub NVA. This setup ensures that traffic is routed via the NVAs for spoke-to-spoke communication, instead of being sent to the MSEE POP location.

For inter-region flows, the same approach is applicable, with the addition of global Vnet peering for the hubs to facilitate cross-region communication. The next step would be to point to the remote hub NVA with the destination being the remote spokes. It is critical to note that for every spoke you wish to connect to, a UDR would need to be added, unless a summary route is used that composes a supernet of all the remote spoke vnets.

An alternative strategy for utilizing NVAs in hubs is to establish BGP peering using Azure Route Server (ARS), which eliminates the need for managing UDRs. For a detailed understanding of inter-region topology using ARS, refer to the article titled [Multi region ARS Design](https://learn.microsoft.com/en-us/azure/route-server/multiregion#topology).

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
The second method, which is notably the most straightforward to deploy, involves direct peering of all the spoke Vnets that necessitate connectivity. Similar to Option 1, inter-region spokes would require global Vnet peering for communication. An additional method, recently introduced, for constructing comprehensive Vnet peering meshes is the utilization of Azure Virtual Network Manager (AVNM). This tool facilitates the creation of both intra-region meshes and global peering meshes. The primary objective of AVNM is to manage resources on a large scale and reduce management overhead. Please note that at the time of this writing, AVNM is currently in public preview. For further details, please refer to [AVNM Design Considerations](https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview). 

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





