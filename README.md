# ExpressRoute Hairpin Design Considerations

# Intro
In this topic we are going to talk about hair-pinning, also known as MSEE hair-pinning whereas traffic leaving one VNET over expressroute egresses to the MSEE (Microsoft Enterprise Edge) before ingressing to another destination Vnet. This is the default behavior for intra-region (Single circuit with multple Vnets -Single Region) and inter-region (Mutiple circuits with multiple Vnets -Dual Region). Although this design works today and has been around for many years, its discouraged to use this approach due to increased latency of the traffic hairpinning to the peering location hosting the MSEEs. Its important to note, in the near future this behavior will be changed, but its still current behavior at the time of this article. We are going to discuss the various design alternatives to this behavior and the pluses and minuses of each design. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

In both designs above we are using standard hub and spoke model. The spoke vnets are peered to hub with "Allow GW Transit" and "Use Remote Gateways" enabled on the peerings in order to reach on prem and have BGP routes plummed to the spokes. In the following sections we will discuss options for spoke to spoke communication via intra region and spoke to spoke communuation via inter-region. We will also discuss hub to hub communciation in both deployment models.

# Intra-Region 
# Option 1: Summary Route
Its important to note off the bat, the hubs will be able to reach each other directly in all options below. Traffic from Hub Vnet to Hub Vnet2 would bypass the gateway on egress, hit the MSEE, ingress to Hub Vnet2's GW and reach the VM. Where we run into issues if Spoke Vnet1 wants to talk to Spoke Vnet2 on the Hub Vnet or Spoke Vnet3 to Spoke Vnet4 on Hub Vnet 2. If we try ping, or rdp/ssh these connections will fail. The reason is, peering is not transitive and the spokes have no idea about each others routes. Spoke1 vnet **would** be able reach the other spokes because those are using remote gateways and those routes are plummed to the MSEE which in turn gets plubmed over to Hub Vnet. The MSEE is the common denominator. In order to make that connection work from Spoke Vnet 1 to Spoke Vnet2 (or Spoke Vnet3 to Spoke Vnet4), we would need to advertise a summary route from on-prem encompossing the entire prefix range, so 0/0 or 10.0.0.0/8 to attract the traffic. That would end up being the LPM, and get plumbed up from the Hub Vnet to each spoke. After this, we would be able to reach Spoke Vnet2 from Spoke Vnet1

![image](https://user-images.githubusercontent.com/55964102/195739906-db45a66b-03be-4c9c-8e9a-63252c73cd72.png)


Pros:
<br>
Easily to implement by advertising a summary route
<br>
No addded cost

Cons:
<br>
You still have MSEE hairpin
<br>
Address overlap can be an issue
<br>
Limited by ExR GW Bandwidth for Ingress

# Option 2 : Deploy NVA in the Hub
The second option is to deploy an NVA or Azure VM with IPforwarding enabled in the hub to steer the traffic to the spokes. We would then need to create UDRs on each Spoke Vnet1 and Spoke Vnet2 and apply to destination Vnet with NVA as next Hop in the Hub Vnet. 

![image](https://user-images.githubusercontent.com/55964102/195955930-76d32b2e-26b2-4afe-b9ed-04894d346ac7.png)

Pros:
<br>
Spoke to Spoke communucation would no longer hairpin over the MSEEs
<br>
ExR GW throughput limit would no longer apply for ingress traffic
<br>
Full ownership of NVA inclcuding inspection 

Cons:
<br>
Cost of deploying the NVA's
<br>
Management of UDRs
<br>
Responsible for management/updates
<br>

# Option 3: Vnet Peering
The third option is by far the easiest to implement. We simply peer the spokes (Spoke Vnet1 to Spoke Vnet2) and (Spoke Vnet3 to Spoke Vnet4) directly so there is full mesh connectivity and peering would take priority over ExpressRoute for Spoke to Spoke.

![image](https://user-images.githubusercontent.com/55964102/195743838-67ef3797-eeb1-473a-b3e3-2688e453940b.png)

Pros:
<br>
Fast and easy to implement
<br>
Traffic stays on WAN backbone, no MSEE hairpin
<br>
Good performance, no more ExR GW for Ingress

Cons:
<br>
A Vnet can only be peered up to 500 times

# Option 4: Virtual WAN
The fouth option is to create a vWAN and vHubs which would provide native transit connectivity. This is also ideal for larger environments including inter-region which we cover next and the behavior would be the same. Spoke Vnet1 and Spoke Vnet2 would talk directly due to the router in the vHub. However, for Spoke Vnet1 to talk to Spoke Vnet3 on the remote hub, it would hairpin at the MSEE, same behavior for inter-region. In order to prevent this behavior on each vHub you would change Hub Routing preference from ExpressRoute (which is default) to AS-PATH. As we know in networking, shortest AS-PATH wins. The vHub ASN is 65520, so append that twice 65520-65520. That would be shorter then hairpinning and then ingressing at the GW. So, we are telling the platform if I have two routes to the same destination, prefer AS-PATH as opposed to ExpressRoute. After enabling AS-PATH for HRP, hub to hub traffic would stay on the WAN backbone. More info can be found here:
https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing-preference

![image](https://user-images.githubusercontent.com/55964102/195915372-9d0675bc-1669-4d22-b8e6-5ac0265c7183.png)

Pros:
<br>
Routing is done automatically via any-any connectivity intra/Inter region
<br>
No MSEE hairpin or ExR GW on ingress
<br>
No UDR or NVA Mgmt

Cons:
<br>
Requires a redesign going from Trad. Hub+Spoke
<br>
vHub is a managed Vnet, user does not have full control
<br>
Enabling HRP AS-PATH here may affect other traffic patterns

# Inter-Region

# Option 1: ExpressRoute Bow-Tie
In this scenario for inter-region, each hub Vnet will advertise its address space and adjacent peered spokes to the remote express-route circuit since the spokes are using "Use Remote Gateway" and bow tie. Since the GWs are conneted to each MSEE, the remote circuit will learn the other circuits hub and peered spoke address spaces. This provides redudancy and full adjacency across each circuit. 

![image](https://user-images.githubusercontent.com/55964102/195952342-0384fdf7-2894-4af6-8361-ace3dd346f84.png)

Pros:
<br>
Traffic will ride the native MSFT Wan backbone and no transit fees
<br>
Configuration is simple, just add each ExR GW and connect to the remote circuit

Cons:
<br>
Traffic is still hairpinning at the MSEE adding latency
<br>
Ingress traffic is still bound by ExRGW limits

# Option 2: Using NVAs in each Hub
For this solution, we simply create two NVAs in each Hub Vnet, same as intra-region, and we create UDRs on each spoke Vnet for each circuit pointing to the NVA as next hop for the destination VNET. We would also need to global peer both hub VNETs, so that they would learn each others address-space and have full reachability across both ciruits for hub+Spoke.

![image](https://user-images.githubusercontent.com/55964102/196298801-08eeeae6-fe62-4398-aef4-64ac52715845.png)














 


