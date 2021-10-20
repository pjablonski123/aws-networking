# Observations on Networking in AWS

In all scenarios using a 3rd party appliance provides additional capabilities WAF, IDS/IPS, L7 Firewalling etc.

# 1. Third Party Appliance in a VPC without Ingress Routing Feature

![](1.jpg)
  
## Use Case

- L7/DPI protection using 3rd party security appliances.
- Protection of not only server instances but also load balancers which are running in the backend.
- Multiple of different services behind a single public IP address. 

## How the traffic flows

- Elastic IP (EIP) is associated with eni123. This means the traffic to EIP is translated on IGW to a primary IP address assigned to eni123.
- For the traffic from **server instances (in subnet 1 and 2) to internet**, an SNAT rule on appliance instance is configured. The server instance IP addresses are SNATed to the private IP address of the eni123 interface of the appliance instance. The SNAT rule is stateful so for the reverse traffic direction there is a mirrored DNAT rule.
- For the traffic from **internet to server instances (in subnet 1 and 2)**, DNAT rule(s) are configured on the appliance. The DNAT rule is stateful so for the reverse traffic direction there is a mirrored SNAT rule.
- The bidirectional traffic between internet and server instances (in subnet 1 and 2) always gets double NATed; once by AWS on IGW and once by the appliance instance. 

## Configuration Tips

- SRC/DST Check is disabled on eni123
- SNAT/DNAT rules are configured on the appliance instance for subnet 1 and subnet 2

## Pros/Cons

- You can have a multiple of different services behind a single public IP address. 
- A drawback is the cost of the additional appliance instance.
- SNAT and DNAT is not limited to the only/single private IP address on the eni123. A secondary private IP can be configured on the appliance instance’ s eni123 and an elastic IP can be associated with it. But charges will apply in that case. A public IP cannot be associated with a secondary private IP. (Mentioned [here](https://aws.amazon.com/premiumsupport/knowledge-center/secondary-private-ip-address/)) It has to be an elastic IP association.
- If there is 1:1 NAT on the appliance then the traffic can be initialized from Internet to server instances. If there is 1:N NAT/PAT then the traffic can initialized from server instances only.


# 2. Third Party Appliance in a VPC with Ingress Routing Feature

![](2.jpg)

## Use Case

- L7/DPI protection using 3rd party security appliances.
- Protection of not only server instances but also load balancers which are running in the backend.
- Multiple of different services behind a single public IP address. 

## How the traffic flows

- When instance in subnet 1 talks to instance in subnet 2 the traffic can be implemented through public IPs, that way the traffic would always go through appliance instance.
- Server instances in Subnet#1 and Subnet#2 have public IPs assigned by AWS. Meaning that their traffic to the internet is SNATed by AWS. Both server instances are reachable from the internet through those public IPs. For instance, client -> server instance#1 public IP traffic is DNATed by AWS on IGW and then the traffic is routed by application instance and VPC implicit router onwards to the server instance#1. Obviously, the appliance instance can be leveraged to secure this internet -> server instance traffic.

## Configuration Tips

- SRC/DST Check is disabled on eni123
- Public IP assignment is enabled for the server instances (instead, an alternative option is to configure the appliance instance with SNAT and DNAT as in the previous scenario)
- Since because you cannot configure a more specific route entry in the subnet route tables, Server#1 <-> Server#2 traffic still flows through VPC Implicit Router. More specific route entries are allowed only in the IGW route table. AWS Console generates an error when you try to associate a route table with a more specific entry to a subnet.

## Pros/Cons

- You can have a multiple of different services behind a single public IP address. A drawback is the cost of an additional instance.
- If/when a server instance gets provisioned in subnet 0, the server’s traffic would be sent towards internet but return traffic would always hit appliance instance (due to the routing entry with appliance instance’ s eni as the target in the IGW route table) 

Resource : https://aws.amazon.com/blogs/aws/new-vpc-ingress-routing-simplifying-integration-of-third-party-appliances/ 

# 3. Third Party Appliance in a VPC (2 x ENIs) with Ingress Routing Feature

![](3.jpg)

## Use Case

- Various 3rd party appliance limitations around firewalling and routing on the stick

- Clustering/synchronization of firewalls over a seperate interface

## How the traffic flows

xxxxxxxxxxxxxxxx _**FILL THIS PART IN**_ **************

## Configuration Tips

- SRC/DST Check is disabled on eni123
- Public IP assignment is enabled for the server instances (instead, an alternative option is to configure the appliance instance with SNAT and DNAT as in the previous scenario)

## Pros/Cons

- Better seperation with interface based zoning 
- 

Reference : In the following urls, third party firewall vendors how their appliances using two interfaces.

https://www.fortinet.com/blog/business-and-technology/network-security-use-cases-amazon-vpc-ingress-routing  

https://live.paloaltonetworks.com/t5/blogs/amazon-web-services-aws-ingress-routing/ba-p/300885


# 4. Third Party Appliance in a VPC and Inter Subnet Routing

![](4.jpg)

## Use Case

- East west next generation firewalling for the individual subnets in the same VPC
- East west load balancing for the individual subnets in the same VPC

## How the traffic flows

## Configuration Tips

- SRC/DST Check is disabled on eni123, eni456 and eni789
- The default gateway of the server instances (in subnet 1 and 2) has to be changed to force the subnet 1 <-> subnet 2 flows through the appliance
- VPC <-> Internet traffic (with or without ingress routing) is not relevant here, any of the methods mentioned in the previous slides can be used.

## Pros/Cons

- No need for default route in subnet#1 or subnet#2 route tables since the server instances are configured with a default gateway IP of the appliance instance interface (eni456 and eni789 respectively)


**Note :** Verification of packet flow between server instance#1 and server instance#2 can be verified by using “tcpdump -i eth1 host 10.0.12.200” on appliance instance shell. Eth1 corresponds to eni456 in this case.

# 5. NAT Gateway

![](5.jpg)

## Use Case

## How the traffic flows

## Configuration Tips

The IP entry with 18.135.143.66 /32 in Subnet 0 route table and Subnet 1 route table did NOT break the “instances in subnet 1 - > internet” communication.

## Pros/Cons 

AWS NAT Gateway SNATs the traffic to the Elastic IP and sends the traffic to IGW with the Source IP = Elastic IP. 

Reference: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html

# Findings

- Public/Private subnet versus Public/Elastic IP assignment are two seperate things. Subnet type is just a high level term to define whether if that subnet will be publicly accessible from the internet (through route entries using igw as the target). IP assignment on the other hand is about giving the actual application instance its own public/elastic routable IP address.
- There is a "Set Main Route Table" button in "Actions" for each route table
- If an ICMP traffic has already been initiated towards an instance then changing the inbound rules for the security group that the destination instance is part of does not block that ICMP traffic. For new ICMP traffic the rules take affect.
- You can assign a public IP to an instance only during the instance launch. Once the instance is launched then you can only assign an elastic IP to the instance.
- The primary private IP of an interface always stays the same and bound to that interface during the lifetime of an insance. However secondary private IPs assigned to the same interface can be REASSIGNED to other instances.
- Public IP stays the same across instance reboots
- Auto Public IP setting gets disabled when you add multiple interfaces to an instance during instance launch
- When you delete the NAT Gateway, the Elastic IP that is associated with it does NOT get deleted
- It is the ENI Elastic Network Interface which actually has all the subnet and connectivity characteristics and it is a fully abstracted and portable construct
- EC2 to EC2 traffic' s protection by NGFW capabilities are pretty much bound to agent/host based solutions

# Additional References

https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html#route-tables-appliance-routing  

https://aws.amazon.com/blogs/networking-and-content-delivery/how-to-integrate-third-party-firewall-appliances-into-an-aws-environment/ 

# To Do In the Future

- Failover Scenarios of the firewall (HA)
- How to view all the IPs being used in a VPC or subnet ?
- How to view MAC address of an ENI ? 



