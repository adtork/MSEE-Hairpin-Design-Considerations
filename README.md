# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this article, we will explore the concept of hair-pinning, also referred to as MSEE hair-pinning. This involves a process where traffic from a VNET, connected via an ExpressRoute circuit, exits to the Microsoft Enterprise Edge (MSEE) pop location prior to entering the destination Vnet. This default behavior is observed both within a single region (a single circuit with multiple Vnets) and across multiple regions (multiple circuits with multiple Vnets).

> [!Important]
>While this design methodology has been functional for years, it is no longer recommended due to the increased latency caused by traffic hair-pinning to the peering location points of presence (POPs) hosting the MSEEs. This approach also places a significant load on the ExpressRoute gateways for ingress traffic to the destination Vnet.

> [!NOTE]
>Historically, solutions to this approach, which are currently deemed outdated, involved advertising a summary route for intra-region traffic, encompassing the entire Vnet address prefix from on-premise. Additionally, the traditional 'bow-tie' approach was utilized, where each Vnet is connected to the other ExpressRoute circuit, thus forming a 'bow-tie'. For more details on these outdated approaches, please refer to the previous article titled: [Common design principles for a hub and spoke](https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture). 

# Option 1: Using an NVA inside Hub Vnets
The initial strategy for circumventing traffic hairpinning to the MSEE points of presence (POPs) involves the incorporation of a Network Virtual Appliance (NVA) or a Virtual Machine (VM) with IP forwarding within the hub Vnets. Subsequently, each spoke Vnet necessitates the creation of a User-Defined Route (UDR) that points either to the remote spoke Vnet prefix or to the default route (0.0.0.0/0) with the next hop being the hub NVA. This setup ensures that traffic is routed via the NVAs for spoke-to-spoke communication, instead of being sent to the MSEE POP location.

For inter-region flows, the same approach is applicable, with the addition of global Vnet peering for the hubs to facilitate cross-region communication. The next step would be to point to the remote hub NVA with the destination being the remote spokes. It is critical to note that for every spoke you wish to connect to, a UDR would need to be added, unless a summary route is used that composes a supernet of all the remote spoke vnets.

An alternative strategy for utilizing NVAs in hubs is to establish BGP peering using Azure Route Server (ARS), which eliminates the need for managing UDRs. For a detailed understanding of inter-region topology using ARS, refer to the article titled: [Multi region ARS Design](https://learn.microsoft.com/en-us/azure/route-server/multiregion#topology).

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/74a9f81e-72cb-4fb5-996f-554182cae8c3)

**Pros:**
 - Traffic no longer hairpins to MSEE edge pop locations
 - Traffic no longer ingresses threw remote gateways. This takes the load off the gateways. 
 - Granular control over NVAs

**Cons:**
 - Cost of running the NVAs
 - Responsibility of managing the NVAs
 - UDR Management overheard

# Option 2: Vnet Peering
The second method, which is notably the most straightforward to deploy, involves direct peering of all the spoke Vnets that necessitate connectivity. Similar to Option 1, inter-region spokes would require global Vnet peering for communication. An additional method, recently introduced, for constructing comprehensive Vnet peering meshes is the utilization of Azure Virtual Network Manager (AVNM). This tool facilitates the creation of both intra-region meshes and global peering meshes. The primary objective of AVNM is to manage resources on a large scale and reduce management overhead. Please note that at the time of this writing, AVNM is currently in public preview. For further details, please refer to: [AVNM Design Considerations](https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview). 

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
A third option entails deploying Azure Virtual WAN (vWAN). The advantages of employing vWAN include a significant simplification of routing as a whole and the provision of native transit connectivity for all aspects, excluding ExR to ExR, which necessitates the use of global reach. For intra-region traffic, spokes utilize the hub routers to communicate directly by employing the default route table.

Regarding inter-region traffic (vhub to vhub), to bypass the MSEE hairpin, it is necessary to set the Hub Routing Preference (HRP) to AS-PATH if the ExR bow-tie is being used. It is crucial to note that if you are not employing the bow-tie and each vhub possesses a unique circuit, it will automatically take hub-to-hub. The HRP only needs to be set when adopting the bow-tie! In a typical scenario using the bow-tie, as demonstrated below, traffic continues to hairpin at the MSEE for inter-region flows. To rectify this, it is necessary to change the vhubs routing preference to Hub Routing Preference (HRP) from ExpressRoute (which is the default) to HRP AS-PATH. For more information on Hub Routing Preference, please consult the following resource: [vWAN HRP](https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference).

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
The aforementioned outlines various design alternatives for directing traffic in both intra and inter-region designs utilizing ExpressRoute. Traditional methodologies such as implementing a 'summary route' for intra-region configurations and a 'bow-tie' for inter-region configurations should be discouraged. These approaches result in traffic continuing to hairpin at the MSEE, and increase load on the express-route gateway. As a result, these methods are not recommended for future deployments, and are discouraged moving forward! 





