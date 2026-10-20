# Enterprise Network Lab (Branch + HQ) in EVE-NG

So i wanted to build a proper enterprise network and not just a couple routers pinging each other. I found a campus design diagram with two sites, a bunch of core routers, distribution switches and access switches, and i thought ok lets try to build my own version of that in EVE-NG and actually get it working end to end.

The end result is two sites, Branch-1 and HQ, with a fake ISP sitting in the middle connecting them. Each site has an edge router, two core routers, two layer 3 distribution switches, and three access switches with PCs (Branch-1) or servers (HQ) plugged in. All the configs are in the configs folder if you wanna look.

## What i used

IOSv for the routers and the ISP, vIOS-L2 for the distribution switches (these can route when you turn on ip routing), and IOL L2 for the access switches because its way lighter on RAM. I started everything at once at the start and my EVE-NG server almost froze, so i ended up starting the nodes in batches.

## How i set it up

I started with the addressing because i wanted it to look like a real network. Every link between routers, or router and switch, is a /30. Every VLAN is a /24. Branch-1 uses 10.1.x.x and HQ uses 172.16.x.x. There are three floor VLANs (10, 20, 30) and a management VLAN 99. The full list of addresses is in docs/ip-plan.csv and the cabling is in docs/cabling.csv.

At the access layer i kept it simple. Each access switch has two trunks, one to each distribution switch, and the PC ports have portfast, BPDU guard and port security turned on. Unused ports are shut down.

For the distribution layer i used HSRP so the PCs always have a gateway even if one switch dies. I split it up so DSW1 is the main gateway for VLAN 10 and 20, and DSW2 is the main one for VLAN 30. HQ is the same idea but flipped. Then i made the spanning tree root match whoever the HSRP gateway is, so traffic doesnt take a weird path. The two distribution switches are also joined with a LACP bundle. I also put ACLs on the VLAN 10 and VLAN 30 interfaces so those two cant talk to each other, just to show how policy works.

The core and edge are where it connects together. Inside each site its OSPF (area 0, point to point links). The edge routers talk BGP to the ISP, Branch-1 is AS 65001, HQ is AS 65002 and the ISP is AS 65000. The ISP gives each site a default route and has a loopback at 8.8.8.8 that pretends to be the internet. The edge routers also do NAT so the PCs can reach 8.8.8.8, but i made sure traffic going between the two sites doesnt get NATed.

One thing thats a bit weird, the Branch-1 PCs get their IPs from DHCP running on EDGE-R1, and the distribution switches relay the requests to it. In a real network you would use a proper DHCP server for this, i just didnt have one in the lab so the edge router did it. The HQ servers just use static IPs.

## Does it actually work

Yeah it does. Here are some screenshots from the lab.

HSRP and spanning tree on DSW1, you can see it's the active gateway for VLAN 10, 20 and 99 and standby for 30:

![hsrp and stp](images/01-hsrp-and-stp-dsw1.png)

The LACP bundle and the trunks on DSW1:

![etherchannel and trunks](images/02-etherchannel-and-trunks-dsw1.png)

OSPF neighbors on EDGE-R1 and the default route it got from the ISP over BGP:

![ospf and default route](images/03-ospf-edge-r1.png)

The ISP with both BGP sessions up:

![bgp on the isp](images/04-bgp-isp.png)

NAT on EDGE-R1, only the internet traffic gets translated:

![nat](images/05-nat-edge-r1.png)

And this is PC1 in Branch-1 pinging the internet and an HQ server, then a traceroute showing it going through the whole path (DSW1, core, edge, ISP, then HQ edge, core, DSW3):

![end to end from pc1](images/06-end-to-end-pc1.png)

I also tested failover. I shut down the VLAN 10 interface on DSW1 while PC1 was pinging an HQ server, and DSW2 took over the gateway and i only lost 2 pings out of 45. Then i shut a link on EDGE-R1 while pinging 8.8.8.8 and OSPF moved the traffic to the other core router, lost 4 pings out of 35.

## Stuff i would still do or fix

The IOL L2 image doesnt support SSH, so the access switches arent as locked down as the rest. I didnt do any QoS or OSPF authentication. I also left the unused ports on the distribution switches up and the HTTP server is on because thats the default on that image, i would turn both off in a real network. And like i said, DHCP should be on a real server.

The passwords in the configs are removed and it's just a lab, nothing real in here.
