# ExpressRoute Hairpin Design Considerations

# Intro
In this scneario we are going to talking about hairpinning, also known as MSEE hairpinning whereas traffic leaving one VNET over expressroute egresses to the MSEE (Microsoft Provider Edge) before ingressing to another Vnet. This is the default behavior for intra-region (single express-route circuit) with multiple vnets and inter-regoin (multiple express-route circuits with multple Vnets. Although this design works today and has been around for many years, its discouraged to use this design due to increased latency of the traffic egressing the peering location pops hosting the MSEEs. Its important to note, in the near future this behavior will be changed, but its still current at the time of this article. We are going to discuss the various alternatives to this design behavior and the pluses and minuses of each design. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Topology

# Inter-Region
![image](https://user-images.githubusercontent.com/55964102/193679955-089ce726-ac9d-422b-92c8-7233fc473436.png)

# Intra-Region
![image](https://user-images.githubusercontent.com/55964102/193708856-64d9f123-c898-40b7-a093-8f066ec3eda7.png)

Lets take the default behavior. If a VM in VnetA wants to talk to a VM in VnetB connected to a single circuit (Intra-Region) Traffic leaves VnetA bypassing the gateway, hits the MSEE and then ingress the gateway on VnetB. The behavior is the same on two circuits (Inter-Region) using standard bow-tie, traffic leaves VnetA bypassing the source VnetA gateway, hits the MSEE at the pop location, ingresses to the MSEE via the other circuit, then finally ingressing through VnetB's gateway. We can see how this is not ideal because either Intra or Inter region, traffic always hits the MSEE at the peering location before ingressing to the other Vnet adding latency.

# Option 1: Vnet Peering
The simpliest and best peforming option is to simply peer VnetA to VnetB. This approach is by far the easiest to implement and best in terms of performance. This also holds true via global VNET peering for inter-region if applicable. 

![image](https://user-images.githubusercontent.com/55964102/193679218-82c2394f-3564-4730-b982-f5b07ab99f1a.png)


Pros:

-Quick and easiest to implement the peering

-Best peformance because traffic is taking the Microsoft WAN directly to reach the destinatoin VNET and there is no ingress gateway for bottleneck

-Cheapest to maintain as you're not paying for an NVA to route the traffic or manage UDRs

Cons:

-Subject to Vnet peering limits that can quickly be approached with large multi-region designs (500 limit per Vnet)
https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits

# Option 2: Connectivity or Transit Vnet hosting NVAs
For this option we create a new spoke VnetC and peer that to each of our hub Vnets (VnetA and VnetB). In the spoke Vnet we deploy an NVA capable of doing the ipforwarding. For this scenario we could simply do Windows, Linux and enable the forwarding on the NIC and inside the GuestOS. The customer could also choose a third party NVA if they wanted inspection as well. From each Hub (VnetA and VnetB), we would create a UDR pointing to the NVA in VnetC as next hop in order to reach the destination VNET. We could also take a simmilar approach for inter-region. You could deploy NVAs in each hub (VnetA and VnetB) and then assuming each of those have spokes, the spokes could use those hub NVAs to transit to other Vnets on the second circuit. We could also connect this VNET (VnetC) to the existing circuit as well and still use UDRs and the NVA to overide the gateway. With no gateway (diagram below), if you wanted to reach on-premise from VnetC, you would need to use "Allow Gateway Transit" and "Use Remote Gateway" on the VNET peering properties. 

![image](https://user-images.githubusercontent.com/55964102/193691974-85ad8188-52c9-48f9-94f9-b879b4d94afe.png)


Pros:

-Traffic will not hairpin to the MSEE pop location, reducing latency

-Full management of NVA, no black box for inspection and routing. 

Cons:

-Cost of the NVA and bandwdith limits on the NVA/VM with ipforwarding

-Complexities of managing UDRs and Route Tables (Its posible to do a summary route to attract the traffic)

# Option 3: Virtual Wan
The main advatnage of virtual WAN is by default it offers any to any connectivity across hubs, spokes and branches via the default route table. The hub routers inside the vHub(s) facilitate this routing. It greatly simplies routing by taking away the need to manually create UDRs unless you need Vnet or branch isolation via custom routes. In terms of ExpressRoute, in order to avoid MSEE hairpain, customer would need to enable the hub-hub routing preference. At the time of this artcile if you have two circuits connected to two differet vhubs, traffic will still hairpin to the MSEE to reach the other vhub. In order to avoid this behavior, you would need to enable the gated preview of hub-hub routing, see here: https://learn.microsoft.com/en-us/azure/virtual-wan/whats-new#preview

![image](https://user-images.githubusercontent.com/55964102/193703052-df6c92fb-eeb3-40d5-ad90-9de852426ab4.png)


Pros:

-Simmilar to Vnet peering in option 1, this option is the easiest to implent overall. If you already have a multi hub vWAN setup with Express-Route, you simply need to enable hub to hub feature in order to make the traffic flow work and not hairpn

-Peformance is also just as good as VNET peering as traffic stays on the Microsoft WAN backbone

-There is no additonal cost at the time of this writing to use this feature.

Cons:

-Virtual WAN visability is limited since its a managed Vnet

-No easy way to diagnose if the traffic flow breaks

# Conclusion
We can see the various topologies above they are alternative approaches to hairpinning to the MSEE when it comes to ExpressRoute Intra and Inter region. Each option has its own pluses and minuses. Depending on the customers environment and workload requirements, they should choose the option that makes the most sense for their environment. As we can see cost, peformance and constraints all play a factor into the decesion. 



