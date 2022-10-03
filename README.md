# Best-Practices

# Intro
In this scneario we are going to talking about hairpinning, also known as MSEE hairpinning whereas traffic leaving one VNET over expressroute egresses to the MSEE (Microsoft Provider Edge) before ingressing to another Vnet. This is the default behavior for intra-region (single express-route circuit) with multiple vnets and inter-regoin (multiple express-route circuits with multple Vnets. Although this design works today and has been around for many years, its discouraged to use this design due to increased latency of the traffic egressing the peering location pops hosting the MSEEs. In this article we are going to discuss the various alternatives to this default behavior and the pluses and minuses of each design. Public documentation on Express-Route hairpinning can be found here: https://learn.microsoft.com/en-us/azure/expressroute/virtual-network-connectivity-guidance

# Topology
![image](https://user-images.githubusercontent.com/55964102/193679955-089ce726-ac9d-422b-92c8-7233fc473436.png)


Lets take the default behavior. If a VM in VnetA wants to talk to a VM in VnetB connected to a single circuit, or in this case two circuits using standard bow-tie, traffic leaves VnetA bypassing the source VnetA gateway, hits the MSEE at the pop location, ingresses to the MSEE in the other region, then finally ingressing VnetB's gateway. We can see how this is not ideal having to hairpin all the way to the provider pop location housing the MSEE. We are going to explore some alternatives topologes and way the pros and cons to each.

# Option 1: Vnet Peering
The simpliest and best peforming option is to simply peer VnetA to VnetB. This approach is by far the easiest to implement and best options in terms of peformance

![image](https://user-images.githubusercontent.com/55964102/193679218-82c2394f-3564-4730-b982-f5b07ab99f1a.png)


Pros:

-Quickest and easiest to implement

-Best peformance because traffic is taking the Microsoft WAN directly to reach the destinatoin VNET and there is no ingress gateway for bottleneck

-Cheapest to maintain as you're not paying for an NVA to route the traffic

Cons:

-Subject to Vnet peering limits that can quickly be approached with large multi-region designs
