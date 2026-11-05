# AWS VPC Networking Lab – Beginner's Practical Notes

## 1. Lab Overview

- **Project:** AWS VPC Networking Inspection
- **Cloud Provider:** Amazon Web Services (AWS)
- **Region:** US East (N. Virginia) – `us-east-1`
- **AWS CLI Profile:** `CloudLab`
- **Access Level:** CloudLab-ReadOnly
- **AWS Account:** `919984817385`

### Objective

The purpose of this practical was to understand AWS networking by inspecting an existing VPC and its networking components using the AWS Command Line Interface (AWS CLI).

We used read-only commands. No AWS resources were created, modified, or deleted.

### Skills Practised

- Amazon VPC
- Subnets and Availability Zones
- CIDR and IP addressing
- Route tables and routing
- Internet Gateway
- Security Groups
- Network ACLs
- AWS CLI and JMESPath queries

---

## 2. Virtual Private Cloud (VPC)

### What is a VPC?

A Virtual Private Cloud (VPC) is a logically isolated network inside AWS where cloud resources such as EC2 instances and databases can be deployed.

Think of a VPC as a private network within AWS.

### What does a VPC do?

- Provides an isolated network for AWS resources.
- Defines the IP address range available to the network.
- Allows us to create subnets.
- Supports routing between network destinations.
- Works with security controls to regulate traffic.

### Command used

`aws ec2 describe-vpcs --profile CloudLab --region us-east-1`

### Findings

- **VPC ID:** `vpc-0613c43bac933f8f0`
- **CIDR block:** `172.31.0.0/16`
- **State:** `available`
- **Default VPC:** Yes
- **Instance tenancy:** `default`

### What I learned

The VPC uses the IPv4 CIDR range `172.31.0.0/16`. This provides 65,536 total IPv4 addresses before subnet-level AWS reservations.

---

## 3. Subnets and Availability Zones

### What is a subnet?

A subnet is a smaller network range inside a VPC. It allows us to organise AWS resources into separate network sections.

A subnet belongs to one Availability Zone.

### What is an Availability Zone?

An Availability Zone (AZ) is an isolated location within an AWS Region. AWS provides multiple AZs to support availability and resilience.

### Command used

```bash
aws ec2 describe-subnets \
--filters "Name=vpc-id,Values=vpc-0613c43bac933f8f0" \
--profile CloudLab \
--region us-east-1 \
--query "Subnets[].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,State:State,DefaultForAZ:DefaultForAz,AutoAssignPublicIP:MapPublicIpOnLaunch,AvailableIPs:AvailableIpAddressCount}" \
--output table \
--no-cli-pager
```

### Findings

We found six subnets in the default VPC.

| Availability Zone | Subnet CIDR |
|---|---|
| us-east-1a | 172.31.0.0/20 |
| us-east-1b | 172.31.80.0/20 |
| us-east-1c | 172.31.16.0/20 |
| us-east-1d | 172.31.32.0/20 |
| us-east-1e | 172.31.48.0/20 |
| us-east-1f | 172.31.64.0/20 |

All six subnets were in the `available` state.

Each subnet had:
- `DefaultForAZ: true`
- `MapPublicIpOnLaunch: true`
- `AvailableIpAddressCount: 4091`

### Understanding the CIDR range

Each subnet uses a `/20` CIDR block.

A `/20` contains 4,096 IPv4 addresses. AWS reserves five addresses in each subnet, leaving 4,091 available addresses.

### What I learned

- A VPC can contain multiple subnets.
- Each subnet belongs to one Availability Zone.
- Subnets help organise resources and network traffic.
- Automatic public IP assignment does not, by itself, prove that a subnet is public.

---

## 4. Route Tables and Routing

### What is a route table?

A route table contains rules called routes. These rules determine where network traffic is directed.

### Command used to inspect route tables

```bash
aws ec2 describe-route-tables \
--filters "Name=vpc-id,Values=vpc-0613c43bac933f8f0" \
--profile CloudLab \
--region us-east-1 \
--query 'RouteTables[].{RouteTableId:RouteTableId,VPC:VpcId,Main:Associations[?Main==`true`].Main|[0],AssociatedSubnets:Associations[].SubnetId}' \
--output table \
--no-cli-pager
```

### Findings

We found the following main route table:

- **Route Table ID:** `rtb-0509333dff5995e2a`
- **VPC:** `vpc-0613c43bac933f8f0`
- **Main route table:** Yes

No explicit subnet IDs were displayed in the association output.

Subnets without an explicit route table association use the VPC's main route table.

### Routes found

The main route table contained two routes:

| Destination | Target | State |
|---|---|---|
| 172.31.0.0/16 | local | active |
| 0.0.0.0/0 | igw-08ac9a33ea66555a7 | active |

### What these routes mean

- `172.31.0.0/16` with target `local` allows communication within the VPC's local network.
- `0.0.0.0/0` represents all IPv4 destinations not matched by a more specific route.
- The internet gateway route provides a path towards the internet, subject to other networking requirements.

---

## 5. Internet Gateway

### What is an Internet Gateway?

An Internet Gateway (IGW) is a VPC component that allows communication between resources in a VPC and the internet when routing and other required conditions are correctly configured.

### Command used

```bash
aws ec2 describe-internet-gateways \
--filters "Name=attachment.vpc-id,Values=vpc-0613c43bac933f8f0" \
--profile CloudLab \
--region us-east-1 \
--query 'InternetGateways[].{InternetGatewayId:InternetGatewayId,State:Attachments[0].State,VPC:Attachments[0].VpcId}' \
--output table \
--no-cli-pager
```

### Findings

We found one Internet Gateway attached to the VPC:

- **Internet Gateway ID:** `igw-08ac9a33ea66555a7`
- **State:** `available`
- **Attached VPC:** `vpc-0613c43bac933f8f0`

### What I learned

A route to an Internet Gateway is one requirement for internet connectivity. A resource also needs appropriate public IP addressing, subnet routing, and security rules.

---

## 6. Security Groups

### What is a Security Group?

A Security Group acts as a virtual firewall for AWS resources such as EC2 instances.

It controls inbound and outbound network traffic using rules.

Security Groups are stateful. This means return traffic for an allowed connection is automatically permitted.

### Command used to list Security Groups

```bash
aws ec2 describe-security-groups \
--filters "Name=vpc-id,Values=vpc-0613c43bac933f8f0" \
--profile CloudLab \
--region us-east-1 \
--query 'SecurityGroups[].{GroupName:GroupName,GroupId:GroupId,Description:Description}' \
--output table \
--no-cli-pager
```

### Findings

We found one default Security Group:

- **Group Name:** `default`
- **Group ID:** `sg-0697b37bce19effab`
- **Description:** default VPC security group

### Command used to inspect its rules

```bash
aws ec2 describe-security-groups \
--group-ids sg-0697b37bce19effab \
--profile CloudLab \
--region us-east-1 \
--query 'SecurityGroups[].{GroupName:GroupName,Inbound:IpPermissions,Outbound:IpPermissionsEgress}' \
--output json \
--no-cli-pager
```

### Findings

**Inbound rules:**
- Protocol: All protocols (`-1`)
- Source: The same Security Group
- No public IPv4 CIDR source was listed.

This means inbound traffic is allowed from network interfaces associated with this same Security Group.

**Outbound rules:**
- Protocol: All protocols (`-1`)
- Destination: `0.0.0.0/0`

This allows all IPv4 outbound destinations at the Security Group level.

### What I learned

- Security Groups control traffic at the resource level.
- They are stateful.
- Inbound and outbound rules determine which traffic is allowed.
- The default Security Group does not automatically allow inbound connections from everyone on the internet.

---

## 7. Network Access Control Lists (Network ACLs)

### What is a Network ACL?

A Network Access Control List (Network ACL or NACL) is a security layer that controls inbound and outbound traffic at the subnet level.

Unlike Security Groups, Network ACLs are stateless. This means inbound and outbound traffic must be allowed separately.

### Command used

```bash
aws ec2 describe-network-acls \
--filters "Name=vpc-id,Values=vpc-0613c43bac933f8f0" \
--profile CloudLab \
--region us-east-1 \
--query 'NetworkAcls[].{NetworkAclId:NetworkAclId,Default:IsDefault,Entries:Entries[].{RuleNumber:RuleNumber,Protocol:Protocol,Action:RuleAction,Egress:Egress,CIDR:CidrBlock}}' \
--output json \
--no-cli-pager
```

### Findings

We found one default Network ACL:

- **Network ACL ID:** `acl-094f8c990545effd2`
- **Default:** Yes

The IPv4 rules included:

| Rule number | Direction | Action | Destination/Source |
|---|---|---|---|
| 100 | Outbound | Allow | 0.0.0.0/0 |
| 32767 | Outbound | Deny | 0.0.0.0/0 |
| 100 | Inbound | Allow | 0.0.0.0/0 |
| 32767 | Inbound | Deny | 0.0.0.0/0 |

### How NACL rules work

- Rules are evaluated in ascending rule-number order.
- The first matching rule determines whether traffic is allowed or denied.
- The default deny rule is evaluated if no earlier rule matches.
- NACLs are stateless, so return traffic must be allowed by a separate rule.

### What I learned

Network ACLs provide subnet-level traffic filtering, while Security Groups provide traffic filtering at the resource level.

---

## 8. Security Groups vs Network ACLs

Both Security Groups and Network ACLs protect AWS networks, but they work differently.

| Feature | Security Group | Network ACL |
|---|---|---|
| Level | Resource level | Subnet level |
| Behaviour | Stateful | Stateless |
| Rules | Allow rules only | Allow and deny rules |
| Rule evaluation | All applicable rules | Lowest numbered matching rule first |
| Return traffic | Automatically allowed | Must be allowed separately |
| Association | Network interface/resource | Subnet |

### Practical lesson

Security Groups and Network ACLs work together to provide network protection.

For example:
- A Network ACL can control traffic entering or leaving a subnet.
- A Security Group can control traffic reaching an EC2 instance.

Understanding both is important when troubleshooting AWS connectivity problems.

---

## 9. AWS CLI Concepts Practised

During this practical, I used several AWS CLI options.

| Option | Purpose |
|---|---|
| `--profile CloudLab` | Uses my configured AWS CLI profile |
| `--region us-east-1` | Specifies the AWS region |
| `--query` | Filters and selects information from AWS responses using JMESPath |
| `--output table` | Displays results in a readable table |
| `--output json` | Displays results in JSON format |
| `--no-cli-pager` | Prevents output from opening in a terminal pager |

### Why use read-only commands?

The practical used AWS `describe` commands to inspect existing resources without changing the AWS environment.

This helped me practise AWS networking while following a read-only access approach.

---

## 10. Practical Lessons and Skills Gained

By completing this practical, I gained experience with:

1. Inspecting an existing AWS VPC.
2. Understanding VPC CIDR ranges and subnet addressing.
3. Identifying subnets across Availability Zones.
4. Inspecting route tables and their routes.
5. Understanding the role of an Internet Gateway.
6. Examining Security Group inbound and outbound rules.
7. Understanding Network ACL rules.
8. Comparing stateful and stateless network security.
9. Using AWS CLI queries to retrieve specific resource information.

### Important networking lesson

A subnet is not automatically public simply because it has automatic public IP assignment enabled.

To determine whether a subnet is public, we must also inspect its effective route table and check for a route to an Internet Gateway. Public IP addressing and security rules are also relevant to internet connectivity.

---

## 11. Lab Findings Summary

| AWS Component | Result |
|---|---|
| VPC | 1 default VPC |
| VPC CIDR | 172.31.0.0/16 |
| Subnets | 6 |
| Availability Zones | 6 |
| Main Route Table | 1 |
| Internet Gateway | 1 attached |
| Security Groups | 1 default Security Group |
| Network ACLs | 1 default Network ACL |

### AWS Environment

- **Region:** `us-east-1`
- **Profile:** `CloudLab`
- **Access:** Read-only
- **Lab type:** AWS networking inspection

---

## 12. Next Learning Stage

The next practical will focus on EC2 networking.

Topics to explore:

- How EC2 instances connect to a VPC.
- How subnet selection affects an instance.
- How public and private IP addresses work.
- How Security Groups affect EC2 connectivity.
- How to troubleshoot basic network connectivity.

---

## Conclusion

This practical helped me understand how the main components of AWS VPC networking work together.

I used AWS CLI read-only commands to inspect a real AWS environment and identify its VPC, subnets, routing, Internet Gateway, Security Groups, and Network ACLs.

This is part of my ongoing Cloud and DevOps engineering learning journey.


---

# AWS VPC Security Inspection – November 2026

## 1. Network ACL Inspection

Network ACL ID: acl-094f8c990545effd2

Findings:
- Default Network ACL: True
- Associated subnets: 6
- Inbound rule 100: Allow all IPv4 traffic
- Outbound rule 100: Allow all IPv4 traffic
- Rule 32767: Deny remaining unmatched IPv4 traffic

## 2. Security Group Inspection

Security Group ID: sg-0697b37bce19effab

Name: default

Inbound rules:
- Allows all protocols from network interfaces associated with the same Security Group.
- Does not allow unrestricted inbound internet traffic.

Outbound rules:
- Allows all IPv4 outbound traffic to 0.0.0.0/0.

## 3. Security Group vs Network ACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Protection level | Resource level | Subnet level |
| Stateful | Yes | No |
| Rules | Allow rules | Allow and deny rules |
| Rule evaluation | All applicable rules | Lowest numbered first |
| Default behaviour | Inbound restricted | Default NACL allows traffic |

## 4. Important Learning Points

1. Security Groups are stateful.
2. Network ACLs are stateless.
3. Security Groups protect individual resources.
4. Network ACLs protect subnets.
5. The default Security Group allows inbound traffic from resources associated with the same Security Group.
6. The default Network ACL allows inbound and outbound IPv4 traffic through its rule 100.

## 5. Practical Lab Summary

I used AWS CLI with the CloudLab read-only profile to inspect:

- VPC configuration
- Route tables
- Route-table associations
- Subnets across six Availability Zones
- Internet Gateway routes
- Network ACLs
- Security Groups

No AWS resources were created, modified, or deleted during this inspection.


---

# AWS VPC Internet Traffic Flow Exercise – November 2026

## 1. EC2 Instance Inspection

Used AWS CLI to inspect EC2 instances in the default VPC.

Result:
- EC2 instances found: 0
- No existing EC2 instance was available for live traffic testing.
- No resources were created or modified.

## 2. Hypothetical Internet Traffic Flow

For learning purposes, an example EC2 instance was considered.

Traffic flow:
1. Internet user sends an HTTP request.
2. Internet Gateway connects the VPC to the internet.
3. Route table directs internet-bound traffic to the Internet Gateway.
4. Traffic reaches the public subnet.
5. Network ACL evaluates inbound traffic.
6. Security Group checks whether HTTP traffic is permitted.
7. EC2 web server receives the request.
8. Response traffic returns to the internet.

## 3. Important Requirements

For an internet-facing EC2 web server:
- The subnet needs a route to an Internet Gateway.
- The EC2 instance needs a public IPv4 address or Elastic IP.
- The Security Group must permit the required inbound traffic.
- Network ACL inbound and outbound rules must permit the traffic.
- The web server must be running and listening on the required port.

## 4. Important Learning Points

- HTTP normally uses TCP port 80.
- HTTPS normally uses TCP port 443.
- SSH normally uses TCP port 22.
- Security Groups are stateful.
- Network ACLs are stateless.
- A public IP alone does not make a subnet public.

## 5. Exercise Limitation

This traffic-flow exercise is a conceptual walkthrough based on the inspected VPC configuration.

No live EC2 traffic test was performed because no EC2 instances were found in the VPC.


---

# AWS Public vs Private Subnets – November 2026

## 1. Practical Overview

This practical exercise focused on understanding public and private subnets, subnet-to-route-table associations, implicit main route-table associations, Internet Gateways and NAT Gateways.

The exercise used AWS CLI with the CloudLab read-only profile.

No AWS resources were created, modified or deleted.

Region: us-east-1

VPC ID: vpc-0613c43bac933f8f0

## 2. Inspecting VPC Subnets

Command:

aws ec2 describe-subnets \
  --profile CloudLab \
  --region us-east-1 \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' \
  --output table

Purpose:
- Identify existing subnets.
- Inspect subnet CIDR blocks.
- Identify Availability Zones.
- Check automatic public IPv4 assignment.

Findings:

Six subnets were identified across six Availability Zones:

| Subnet ID | CIDR Block | Availability Zone | Auto Public IP |
|---|---|---|---|
| subnet-0c6e00b5b3b5c4030 | 172.31.80.0/20 | us-east-1b | True |
| subnet-0e6c61a11aad2ed5d | 172.31.16.0/20 | us-east-1c | True |
| subnet-0babfdc409b7b1bbd | 172.31.64.0/20 | us-east-1f | True |
| subnet-07bb55d565f9a66b7 | 172.31.0.0/20 | us-east-1a | True |
| subnet-0af6aadebd14df396 | 172.31.32.0/20 | us-east-1d | True |
| subnet-0c40d3aae49615559 | 172.31.48.0/20 | us-east-1e | True |

Important lesson:

MapPublicIpOnLaunch=True means instances launched in the subnet can automatically receive public IPv4 addresses.

However, this setting alone does not determine whether a subnet is public or private.

The effective route table must also be inspected.

## 3. Inspecting Route-Table Associations

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --query 'RouteTables[*].[RouteTableId,Associations[*].SubnetId]' \
  --output json

Finding:

Route table: rtb-0509333dff5995e2a

Explicit subnet associations: None ([])

Explanation:

An empty association list does not mean that the subnets have no route table.

Subnets without an explicit route-table association automatically use their VPC's main route table.

## 4. Verifying the Main Route Table

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --query 'RouteTables[*].[RouteTableId,Associations[*].Main]' \
  --output table

Finding:

rtb-0509333dff5995e2a

Main: True

This confirmed that the identified route table is the VPC's main route table.

Therefore, all six subnets without explicit associations use this main route table.

## 5. Inspecting Route-Table Routes

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --query 'RouteTables[*].Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId,State]' \
  --output table

Findings:

| Destination | Target | State |
|---|---|---|
| 172.31.0.0/16 | local | active |
| 0.0.0.0/0 | igw-08ac9a33ea66555a | active |

Explanation:

The local route allows communication within the VPC, subject to network security controls.

The 0.0.0.0/0 route sends traffic destined outside the VPC to the Internet Gateway.

Because all six subnets use this main route table, all six have a route to the Internet Gateway.

This means they are configured as public subnets.

An instance still requires appropriate public IPv4 addressing, security group rules, network ACL rules and other necessary configuration for internet connectivity.

## 6. Inspecting NAT Gateways

Command:

aws ec2 describe-nat-gateways \
  --profile CloudLab \
  --region us-east-1 \
  --query 'NatGateways[*].[NatGatewayId,SubnetId,State,VpcId]' \
  --output table

Finding:

No NAT Gateways were returned.

This indicates that no NAT Gateway was found by this query in the selected region.

Explanation:

A NAT Gateway allows instances in private subnets to initiate outbound internet connections without requiring public IP addresses on those instances.

A typical private subnet uses a default route:

0.0.0.0/0 -> NAT Gateway

The NAT Gateway itself is placed in a public subnet and uses an Internet Gateway for internet connectivity.

## 7. Confirming Subnet VPC Membership

Command:

aws ec2 describe-subnets \
  --profile CloudLab \
  --region us-east-1 \
  --query 'Subnets[*].[SubnetId,VpcId,MapPublicIpOnLaunch]' \
  --output table

Finding:

All six subnets belong to:

vpc-0613c43bac933f8f0

All six have automatic public IP assignment enabled.

## 8. Public vs Private Subnets: Key Concepts

Public subnet:

- Has an effective route to an Internet Gateway.
- Can host instances configured for direct internet connectivity.
- Instances generally need public IPv4 addresses for direct IPv4 internet communication.

Private subnet:

- Has no direct route to an Internet Gateway.
- Can use a NAT Gateway for outbound internet access.
- Instances do not need public IP addresses for outbound access through NAT.

Important distinction:

Automatic public IP assignment does not independently make a subnet public.

The effective route table determines whether the subnet has a direct route to an Internet Gateway.

## 9. Current VPC Architecture Findings

VPC: vpc-0613c43bac933f8f0

Region: us-east-1

- Six subnets across six Availability Zones.
- One route table returned by the inspection.
- The route table is the main route table.
- No explicit subnet associations were returned.
- All six subnets use the main route table by default.
- The main route table contains an active Internet Gateway route.
- No NAT Gateways were returned by the inspection.
- All six subnets have automatic public IP assignment enabled.

Conclusion:

The inspected default VPC has six subnets configured as public subnets through their shared main route table.

No private subnet or NAT Gateway was identified in this inspection.

## 10. Important Learning Points

1. A public subnet has a route to an Internet Gateway.
2. A private subnet has no direct route to an Internet Gateway.
3. Every subnet uses a route table.
4. A subnet without an explicit route-table association uses the VPC's main route table.
5. MapPublicIpOnLaunch controls automatic public IP assignment, not subnet classification.
6. NAT Gateways allow private instances to initiate outbound internet connections.
7. NAT Gateways are deployed in public subnets.

## 11. Read-Only Safety Record

All commands used in this practical were AWS EC2 describe operations.

No AWS infrastructure was created, modified or deleted.

The exercise was performed using the CloudLab read-only AWS profile in us-east-1.


---

# AWS Security Groups vs Network ACLs – November 5, 2026

## 1. Practical Overview

This practical exercise focused on understanding AWS Security Groups and Network Access Control Lists (NACLs).

The exercise used AWS CLI with the CloudLab read-only profile.

No AWS resources were created, modified or deleted.

Region: us-east-1

VPC ID: vpc-0613c43bac933f8f0

## 2. Inspecting the Default Security Group

Command:

aws ec2 describe-security-groups \
  --profile CloudLab \
  --region us-east-1 \
  --group-ids sg-0697b37bce19effab \
  --query 'SecurityGroups[*].[GroupId,GroupName,IpPermissions,IpPermissionsEgress]' \
  --output json

Security Group ID:

sg-0697b37bce19effab

Security Group name: default

### Inbound rule findings

- IpProtocol: -1
- The rule allows all IP protocols.
- The source is the same Security Group.

Explanation:

Resources associated with this Security Group can communicate with one another over all protocols, subject to other applicable network controls.

This does not mean that all internet traffic is allowed inbound.

### Outbound rule findings

- IpProtocol: -1
- Destination: 0.0.0.0/0
- All outbound IPv4 destinations are allowed.

Explanation:

The default Security Group allows outbound traffic to any IPv4 address.

## 3. Understanding Security Group Behaviour

Security Groups are stateful firewalls.

This means that response traffic for an allowed connection is automatically permitted.

For example, when an inbound connection is allowed, the response traffic does not require a separate outbound rule.

Important characteristics:

- Applied to network interfaces associated with resources such as EC2 instances.
- Support allow rules.
- Do not have explicit deny rules.
- Are stateful.
- All applicable rules are evaluated together.

## 4. Inspecting the Default Network ACL

Command:

aws ec2 describe-network-acls \
  --profile CloudLab \
  --region us-east-1 \
  --network-acl-ids acl-094f8c990545effd2 \
  --query 'NetworkAcls[*].[NetworkAclId,IsDefault,Entries]' \
  --output json

Network ACL ID:

acl-094f8c990545effd2

IsDefault: true

### Inbound rules

| Rule number | Action | Protocol | IPv4 CIDR |
|---|---|---|---|
| 100 | Allow | All | 0.0.0.0/0 |
| 32767 | Deny | All | 0.0.0.0/0 |

### Outbound rules

| Rule number | Action | Protocol | IPv4 CIDR |
|---|---|---|---|
| 100 | Allow | All | 0.0.0.0/0 |
| 32767 | Deny | All | 0.0.0.0/0 |

Explanation:

- Egress true means outbound traffic.
- Egress false means inbound traffic.
- Protocol -1 means all IP protocols.
- 0.0.0.0/0 represents all IPv4 addresses.
- Allow permits matching traffic.
- Deny blocks matching traffic.

### Understanding rule numbers

Network ACL rules are evaluated in ascending numerical order.

The lowest numbered matching rule is applied first.

In this default NACL:

- Rule 100 allows all IPv4 traffic.
- Rule 32767 denies traffic that has not matched an earlier rule.

Because rule 100 already matches all IPv4 traffic, the final deny rule acts as a fallback.

## 5. Understanding Network ACL Behaviour

Network ACLs are stateless firewalls.

This means inbound and outbound traffic are evaluated independently.

For example:

If an inbound HTTP request is allowed, the NACL must also have an appropriate outbound rule to permit the response traffic.

Important characteristics:

- Applied at the subnet level.
- Support both allow and deny rules.
- Evaluate rules in numerical order.
- Are stateless.
- Require appropriate rules for both directions of communication.

## 6. Inspecting Network ACL Subnet Associations

Command:

aws ec2 describe-network-acls \
  --profile CloudLab \
  --region us-east-1 \
  --network-acl-ids acl-094f8c990545effd2 \
  --query 'NetworkAcls[*].Associations[*].[SubnetId,NetworkAclAssociationId]' \
  --output table

Finding:

The default NACL is associated with all six subnets:

| Subnet ID | Network ACL Association ID |
|---|---|
| subnet-0c6e00b5b3b5c4030 | aclassoc-0bba62bc7a15bfd10 |
| subnet-0e6c61a11aad2ed5d | aclassoc-0c7adc747c032ff01 |
| subnet-0babfdc409b7b1bbd | aclassoc-05966d3571b4a3786 |
| subnet-0c40d3aae49615559 | aclassoc-081f7731af8950d85 |
| subnet-0af6aadebd14df396 | aclassoc-02951a77d4d5b8a95 |
| subnet-07bb55d565f9a66b7 | aclassoc-09d40e79616a2fb35 |

Explanation:

Each subnet has its own association ID, but all six associations point to the same Network ACL.

## 7. Checking for Additional Network ACLs

Command:

aws ec2 describe-network-acls \
  --profile CloudLab \
  --region us-east-1 \
  --filters Name=vpc-id,Values=vpc-0613c43bac933f8f0 \
  --query 'NetworkAcls[*].[NetworkAclId,IsDefault,Associations[*].SubnetId]' \
  --output json

Finding:

Only one Network ACL was returned:

acl-094f8c990545effd2

IsDefault: true

All six subnets are associated with this default NACL.

No additional custom Network ACLs were identified in this VPC inspection.

Note:

The inspected entries were IPv4 rules. IPv6 rules were not evaluated in this exercise.

## 8. Security Groups vs Network ACLs

| Feature | Security Group | Network ACL |
|---|---|---|
| Applies to | Network interfaces | Subnets |
| Firewall type | Stateful | Stateless |
| Allow rules | Yes | Yes |
| Explicit deny rules | No | Yes |
| Rule evaluation | All applicable rules | Lowest numbered matching rule |
| Return traffic | Automatically permitted | Must be explicitly permitted |

### Practical example: EC2 HTTP traffic

Imagine an EC2 instance receives an HTTP request on port 80.

For the connection to work:

1. The Security Group must allow inbound HTTP traffic.
2. The subnet's Network ACL must allow inbound HTTP traffic.
3. The Network ACL must also allow the appropriate outbound response traffic.
4. Other applicable network controls must permit the connection.

Both the Security Group and Network ACL must permit the relevant traffic.

## 9. Current VPC Security Findings

VPC: vpc-0613c43bac933f8f0

Region: us-east-1

- Default Security Group inspected.
- Inbound Security Group rule allows traffic from the same Security Group.
- Outbound Security Group rule allows all IPv4 destinations.
- One default Network ACL identified.
- Default NACL allows all IPv4 inbound traffic.
- Default NACL allows all IPv4 outbound traffic.
- All six subnets are associated with the default NACL.
- No additional custom NACLs were identified.

Conclusion:

The inspected VPC uses its default Security Group and one default Network ACL.

The Security Group provides stateful traffic filtering at the network-interface level, while the Network ACL provides stateless traffic filtering at the subnet level.

## 10. Important Learning Points

1. Security Groups protect resources at the network-interface level.
2. Network ACLs protect traffic at the subnet level.
3. Security Groups are stateful.
4. Network ACLs are stateless.
5. Security Groups support allow rules but not explicit deny rules.
6. Network ACLs support both allow and deny rules.
7. Network ACL rules are evaluated in ascending numerical order.
8. Both Security Groups and Network ACLs must permit traffic for communication to succeed.
9. Network ACLs require appropriate inbound and outbound rules.
10. A subnet is associated with a Network ACL.

## 11. Read-Only Safety Record

All commands used in this practical were AWS EC2 describe operations.

No AWS infrastructure was created, modified or deleted.

The exercise was performed using the CloudLab read-only AWS profile in us-east-1.

