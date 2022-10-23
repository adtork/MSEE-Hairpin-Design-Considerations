# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET connected over an expressroute circuit egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to another destination Vnet. This is the default behavior for intra-region (Single circuit with multple Vnets -Single Region) and inter-region (Mutiple circuits with multiple Vnets -Two Regions). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location pops hosting the MSEEs. The old designs to this approach which are **not** recommended today is advertising a summary route for intra-region traffic encompossing the entire Vnet address prefix from on-premise, and doing the tradtitonal bow-tie approach where you connect each on-prem location to the other express-route circuit forming a bow-tie. See the older article which talks about this approach for Option#1 Intra-Region and Option#1 Inter-Region:https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture

Its important to note, in the near future this behavior will be changed and Azure Fabric will likely throw an error if you try the old design apporaches above. We are going to discuss the various design alternatives to MSEE hairpinning. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Option 1: Using an NVA inside Hub Vnets
The first option to get around traffic hairpinning down to the MSEE pops is to simply put a NVA or VM with ipforwarding in the hub Vnets. On each spoke Vnet, you would then need to create a UDR pointing to the remote spoke Vnet prefix with next hop hub NVA. Simmilary for inter-region, you would take the same approach but would also need to add global vnet peering between the hubs for them to commmunicate cross region. If you wanted to reach the remote spokes, you would need to either peer the spokes directly since VNET peering is not transitive, or you could use a VM or NVA in the hub Vnet as a jumpbox to reach the remote hub Vnet and then up to the remote spokes.

![image](https://user-images.githubusercontent.com/55964102/197368592-2ee716d4-80ff-4d7f-bea2-51a7157b7af8.png)


Pros:

Traffic no longer hairpins at the MSEE pop locations

Traffic no longer ingresses to the GWs on remote Vnets

Visability into NVA or VM since its customer managed

Cons:

Cost of running the NVA

Responsibility of managing the NVA

Vnet peering costs

# Option 2: Vnet Peering
The second option and really easiest to deploy is to just simply peer all the spoke Vnets directly. Like above with Option 1, inter region spokes would require global Vnet peering to communicate.

![image](https://user-images.githubusercontent.com/55964102/197368460-279f97af-e60e-4aba-92d8-3ef09af87ea8.png)

Pros:

Easiest to setup and deploy

No administration cost

Routing is done automatically

Lowest possible latency on Azure WAN

Cons:

Subjet to Vnet peering costs
Limit to the number of Vnets that can be peered (500)
https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits

# Option 3: Azure Virtual WAN with HRP(AS-PATH)
The third and final option is to deploy Azure virtual WAN.  The benefit of this approach is vWAN natively provides full mesh connectivity between the vHub and spoke Vnets. Once the spokes are connected to the vHub, there is no need for NVAs, as the router in the vHub provides full mesh connectivity. This same connectivity also applies with two vHubs inter-region. Spokes across regions will be able to communicate directly. There is one additonal thing that is needed for traffic **not** to hairpin using vWAN cross region, and that is to enable Hub Routing Preference (HRP) with AS-PATH. That way, when traffic needs to communicate across regions and hubs, traffic will take hub-to-hub as opposed to hairpinning at the MSEE which is default behavior **without** HRP set to AS-PATH. Since shortest AS-PATH wins, that tells Azure fabric to route the packets hub to hub. More info on HRP can be found here: https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference

![image](https://user-images.githubusercontent.com/55964102/197368188-699c11c8-dcfb-415c-9266-db01b655987e.png)





