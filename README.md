# ExpressRoute MSEE Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET connected over an expressroute circuit egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to another destination Vnet. This is the default behavior for intra-region (Single circuit with multple Vnets -One Region) and inter-region (Mutiple circuits with multiple Vnets -Two Regions). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location pops hosting the MSEEs. The old designs to this approach which are **not** recommended today is advertising a summary route for intra-region traffic encompossing the entire Vnet address prefix from on-premise, and doing the tradtitonal bow-tie approach where you connect each on-prem location to the other express-route circuit forming a bow-tie. See the older article which talks about these approachs: Option#1 Intra-Region (Summary Route) and Option#1 Inter-Region (Bow-Tie):https://github.com/narayankumargupta/Common-Design-Principles-for-a-Hub-and-Spoke-VNET-Archiecture

Its important to note, in the near future this behavior will be changed and Azure control plane will likely throw an error if you try the old design apporaches per above. We are going to discuss the various alternatives to avoid MSEE hairpinning with ExpressRoute. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Option 1: Using an NVA inside Hub Vnets
The first option to get around traffic hairpinning down to the MSEE pops is to simply put a NVA or VM with ipforwarding in the hub Vnets. On each spoke Vnet, you would then need to create a UDR pointing to the remote spoke Vnet prefix with next hop hub NVA. Doing this traffic would be routed to the NVAs for spoke to spoke communication instead of going down to the MSEEs. Simmilary for inter-region, you would take the same approach but would also need to add global vnet peer the hubs for them to commmunicate across regions. If you wanted to reach the remote spokes, you would need to either peer the spokes directly since VNET peering is not transitive, or you could use the VM or NVA in the hub Vnet as a jumpbox to reach the remote hub Vnet and then up to the remote spokes from there.

![image](https://user-images.githubusercontent.com/55964102/220209806-21254dbe-987f-4237-a33e-abb32f4fb66b.png)

**Pros:**

 °Traffic no longer hairpins at the MSEE pop locations

 °Traffic no longer ingresses threw the GWs

 °Full NVA control and management

**Cons:**

 °Cost of running the NVA

 °Responsibility of managing the NVA

 °UDR Management 

# Option 2: Vnet Peering
The second option and really the easiest to deploy is to simply peer all the spoke Vnets directly that require connectivity. Like above with Option 1, inter region spokes would require global Vnet peering to communicate. Another recently introduced option to build full Vnet peering meshes is to use Azure Virtual Network Manager (AVNM). You can build intra region meshes and global peering meshes. The goal of AVNM is to manage resources at scale and simplify management overhead. Currently this is in public preview at the time of this article. More information can be found here: https://learn.microsoft.com/en-us/azure/virtual-network-manager/overview 

![image](https://user-images.githubusercontent.com/55964102/220211126-6a29401e-5121-4ba8-a862-cfb1eaf895b5.png)

**Pros:**

 °Easiest to setup/deploy

 °No administration cost

 °Routing is seamless

 °Lowest possible latency on as no GW or MSEE in path

**Cons:**

Vnet peering limits

Limit to the number of Vnets that can be peered (500)
https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits

# Option 3: Azure Virtual WAN with HRP(AS-PATH)
The third and final option is to deploy Azure virtual WAN. The benefits to using vWAN is that it simplifies routing overall and provides native transit connectivity for everything except ExR to ExR, which would require global reach. For intra region traffic, spokes simply take the routers in the vHub to communciate directly using default route table. Spoke traffic can also be steered using custom routes, but that is beyond the scope of this article. For inter-region traffic, (hub to hub), in order to avoid the MSEE hairpin, you would need to set the hub routing preference (HRP) to AS-PATH. Public docs still show this as a seperate feature, but Hub to Hub using ExR has been rolled into Hub Routing Preference. In a normal scneario using bow-tie as shown below, traffic still hairpins at the MSEE for inter-region flows. We need to change HRP from **Expressroute which is default**, to **HRP AS-PATH**. Shortest AS-PATH wins, so we are telling fabric to prefer the shorter route for inter region flows. Information on Hub Routing Preference can be found here: https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference

![image](https://user-images.githubusercontent.com/55964102/220211769-2de461ca-5ec6-4bfd-97c7-125e54c541fa.png)


**Pros:**

 °Routing is taking care of automatically via vhub routers and default route table

 °No NVAs or UDRs to manage

 °No MSEE hairpin as long as HRP is set to AS-PATH on the vHubs

**Cons:**

 °Would require a redesign if traditonal Hub+Spoke already deployed

 °HRP may affect other routes in the environment

 °Less visability into vHubs since they are MSFT managed Vnets

# Conclusion
The above are design alternatives are additonal ways to direct traffic for intra and inter region designs using ExpressRoute. The old approaches of doing a "summary route" for intra region and "bow-tie" for inter-region should be discourgaged because traffic still hairpins at the MSEE which adds latency and is discouraged moving forward. Down the road, Azure will likely prevent users from using this approach if it detects this being configured.





