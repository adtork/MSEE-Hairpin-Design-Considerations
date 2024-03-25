# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET connected over an expressroute circuit egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to the destination Vnet. This is the default behavior for intra-region (Single circuit with multple Vnets) and inter-region (Mutiple circuits with multiple Vnets). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location pops hosting the MSEEs. This also puts increased load on the express-route gateways for inbound traffic. The old designs to this approach which are **not** recommended today is advertising a summary route for intra-region traffic encompossing the entire Vnet address prefix from on-premise, and doing the tradtitonal bow-tie approach where you connect each vnet to the other express-route circuit forming a "bow-tie". See the older article which talks about these approachs: Option#1 Intra-Region (Summary Route) and Option#1 Inter-Region (Bow-Tie):https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture


# Option 1: Using an NVA inside Hub Vnets
The first option to get around traffic hairpinning down to the MSEE pops is to simply put a NVA or VM with ipforwarding in the hub Vnets. On each spoke Vnet, you would then need to create a UDR pointing to the remote spoke Vnet prefix, or default route (0.0.0.0/0) with next hop hub NVA. Doing this, traffic would be routed to the NVAs for spoke to spoke communication instead of going down to the MSEE POP location. Simmilary for inter-region flows, you would take the same approach but would also need to global vnet peer the hubs for them to commmunicate cross region. You would then point to the remote hub firewall with destination remote spokes. Its important to note, you would need to add a UDR for every spoke you wish to connect to, unless you do a summary route composing a supernet of all the spoke vnets. Another approach to using NVAs in hubs is to BGP peer them using ARS. This takes away the need to manage UDRs. For inter-region topology using ARS, see article: https://learn.microsoft.com/en-us/azure/route-server/multiregion#topology

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/76215072-59e2-41e6-ae3c-5441613c245c)

**Pros:**

 °Traffic no longer hairpins to MSEE pop locations

 °Traffic no longer ingresses threw remote gws, less load

 °Control over NVAs

**Cons:**

 °Cost of running the NVA/additonal VMs

 °Responsibility of managing the NVA/VMs

 °UDR Management 

# Option 2: Vnet Peering
The second option and really the easiest to deploy is to simply peer all the spoke Vnets directly that require connectivity. Like above with Option 1, inter region spokes would require global Vnet peering to communicate. Another recently introduced option to build full Vnet peering meshes is to use Azure Virtual Network Manager (AVNM). You can build intra region meshes and global peering meshes. The goal of AVNM is to manage resources at scale and simplify management overhead. Currently this is in public preview at the time of this article. More information can be found here: https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview 

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/24982b06-5971-4a2b-b1c3-d099b53b3dcc)

**Pros:**

 °Easiest to setup/deploy

 °No administration cost

 °No UDR Management

 °Lowest latency, no VM chokepoint

**Cons:**

Limit to the number of Vnets that can be peered (500)
https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits

# Option 3: Azure Virtual WAN with HRP(AS-PATH)
The third and final option is to deploy Azure virtual WAN. The benefits to using vWAN is that it simplifies routing overall and provides native transit connectivity for everything except ExR to ExR, which would require global reach. For intra region traffic, spokes simply take the hub routers to communciate directly using default route table. Spoke traffic can also be steered using custom routes, but that is beyond the scope of this article. For inter-region traffic, (vhub to vhub), in order to avoid the MSEE hairpin, you would need to set the hub routing preference (HRP) to AS-PATH if using ExR bow-tie. Its important to note, if you're not using bow-tie and each vhub has a unique circuit, it will take hub to hub automatically. You only need to set HRP when doing the bow-tie!. In a normal scneario using bow-tie as shown below, traffic still hairpins at the MSEE for inter-region flows. We need to change HRP from **Expressroute which is default**, to **HRP AS-PATH**. Information on Hub Routing Preference can be found here: https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference

![image](https://github.com/adtork/MSEE-Hairpin-Design-Considerations/assets/55964102/763535cc-e4e0-4ecd-bab4-675464483acd)


**Pros:**

 °Routing is taking care of automatically via vhub routers and default route table

 °No NVAs or UDRs to manage

 °No MSEE hairpin as long as HRP is set to AS-PATH on the vHubs

**Cons:**

 °Would require a redesign if traditonal Hub+Spoke already deployed

 °HRP may affect other routes in the environment, for example VPN and SDWAN tunnels

 °Less visability into vHubs since they are MSFT managed Vnets

# Conclusion
The above are design alternatives to direct traffic for intra and inter region designs using ExpressRoute. The old approaches of doing a "summary route" for intra region and "bow-tie" for inter-region should be discourgaged because traffic still hairpins at the MSEE which adds latency and is discouraged moving forward. 





