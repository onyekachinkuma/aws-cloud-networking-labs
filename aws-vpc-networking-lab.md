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


---

# AWS Route Tables and Subnet Associations – November 5, 2026

## 1. Practical Overview

This practical exercise focused on understanding AWS route tables and how subnets use them.

The exercise used AWS CLI with the CloudLab read-only profile.

No AWS resources were created, modified or deleted.

Region: us-east-1

VPC ID: vpc-0613c43bac933f8f0

## 2. Understanding Route Tables

A route table contains rules called routes.

These routes determine where network traffic is directed when it leaves a subnet or needs to reach another network destination.

Each route contains:

- Destination: The network or IP address range the traffic is intended for.
- Target: Where AWS sends the matching traffic.
- State: Whether the route is active or inactive.

A subnet must be associated with a route table.

A subnet without an explicit association automatically uses the VPC's main route table.

## 3. Inspecting Route Tables in the VPC

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --filters Name=vpc-id,Values=vpc-0613c43bac933f8f0 \
  --query 'RouteTables[*].[RouteTableId,Associations]' \
  --output json

Finding:

One route table was returned:

Route table ID: rtb-0509333dff5995e2a

Main: true

Association state: associated

No explicitly associated subnet IDs were listed.

Explanation:

The Main value of true identifies this as the VPC's main route table.

The main route table is automatically used by subnets that do not have an explicit association with another route table.

## 4. Inspecting the Routes

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --route-table-ids rtb-0509333dff5995e2a \
  --query 'RouteTables[*].Routes[*].[DestinationCidrBlock,GatewayId,State]' \
  --output table

Findings:

| Destination | Target | State |
|---|---|---|
| 172.31.0.0/16 | local | active |
| 0.0.0.0/0 | igw-08ac9a33ea66555a7 | active |

### Understanding the local route

Destination: 172.31.0.0/16

Target: local

This route allows communication between resources within the VPC using their private IP addresses, subject to applicable network security controls.

### Understanding the default route

Destination: 0.0.0.0/0

Target: igw-08ac9a33ea66555a7

This is the default IPv4 route.

It directs traffic that does not match a more specific route toward the Internet Gateway.

The route is active.

Important:

A route to an Internet Gateway does not automatically guarantee internet connectivity for every resource.

An EC2 instance also needs an appropriate public IP address and suitable network security rules.

## 5. Checking an Individual Subnet Association

Subnet inspected:

subnet-0c6e00b5b3b5c4030

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --filters Name=association.subnet-id,Values=subnet-0c6e00b5b3b5c4030 \
  --query 'RouteTables[*].[RouteTableId,Associations]' \
  --output json

Result:

[]

Explanation:

The empty result means that no explicit route table association was found for this subnet.

It does not mean that the subnet has no route table.

Because the subnet has no explicit association, AWS automatically uses the VPC's main route table.

## 6. Checking All Six Subnets

The following read-only shell loop checked each subnet individually:

for subnet in \
subnet-0c6e00b5b3b5c4030 \
subnet-0e6c61a11aad2ed5d \
subnet-0babfdc409b7b1bbd \
subnet-07bb55d565f9a66b7 \
subnet-0af6aadebd14df396 \
subnet-0c40d3aae49615559
do
  echo "Checking $subnet"
  aws ec2 describe-route-tables \
    --profile CloudLab \
    --region us-east-1 \
    --filters Name=association.subnet-id,Values="$subnet" \
    --query 'RouteTables[*].RouteTableId' \
    --output text
done

Finding:

All six subnet checks returned no explicit route table association.

Therefore, all six subnets use the VPC's main route table:

rtb-0509333dff5995e2a

## 7. Verifying the Main Route Table Association

Command:

aws ec2 describe-route-tables \
  --profile CloudLab \
  --region us-east-1 \
  --filters Name=vpc-id,Values=vpc-0613c43bac933f8f0 \
  --query 'RouteTables[*].[RouteTableId,Associations[*].[Main,SubnetId,AssociationState.State]]' \
  --output json

Finding:

- Route table ID: rtb-0509333dff5995e2a
- Main: true
- SubnetId: null
- Association state: associated

Explanation:

The main route table has a main association rather than a separate explicit association for each subnet.

Subnets without explicit route table associations automatically use this main route table.

## 8. Explicit vs Implicit Route Table Associations

### Explicit association

An explicit association occurs when a subnet is directly associated with a particular route table.

This allows different subnets to use different routing configurations.

For example, a public subnet and a private subnet can use separate route tables.

### Implicit association

An implicit association occurs when a subnet has no explicit route table association.

AWS automatically uses the VPC's main route table for that subnet.

Important:

A subnet using the main route table is still associated with a route table. The association is simply not listed as an individual subnet association.

## 9. Current VPC Route Table Findings

VPC: vpc-0613c43bac933f8f0

Region: us-east-1

- One route table was identified.
- Route table ID: rtb-0509333dff5995e2a.
- The route table is the main route table.
- All six inspected subnets have no explicit route table association.
- All six subnets therefore use the main route table.
- The local route 172.31.0.0/16 is active.
- The default route 0.0.0.0/0 points to the Internet Gateway.
- Both inspected routes are active.

Conclusion:

The inspected VPC uses one main route table for all six subnets.

The local route supports communication within the VPC, while the default route directs other IPv4 traffic toward the Internet Gateway.

## 10. Important Learning Points

1. Route tables control where network traffic is directed.
2. Routes contain a destination and a target.
3. The local route supports communication within a VPC.
4. The default IPv4 route is represented by 0.0.0.0/0.
5. An Internet Gateway can be the target of a default route.
6. A subnet can have an explicit route table association.
7. A subnet without an explicit association automatically uses the main route table.
8. The main route table is identified by Main: true.
9. A subnet using the main route table still has an effective route table association.
10. A route to an Internet Gateway alone does not guarantee internet connectivity.

## 11. Read-Only Safety Record

All commands used in this practical were AWS EC2 describe operations.

The shell loop only performed read-only AWS inspection commands.

No AWS infrastructure was created, modified or deleted.

The exercise was performed using the CloudLab read-only AWS profile in us-east-1.

---

# AWS VPC Internet Connectivity, Endpoints and DNS – November 5, 2026

## 1. Practical Overview

This practical extended the VPC networking exercises by inspecting Internet Gateway connectivity, NAT Gateways, VPC Endpoints, IPv4 and IPv6 configuration, subnet address capacity, automatic IP assignment, DNS settings and DHCP options.

The exercise used the AWS CLI with the CloudLab read-only profile in us-east-1.

No AWS resources were created, modified or deleted.

## 2. Internet Gateway Inspection

Finding:
- Internet Gateway: igw-08ac9a33ea66555a7
- Attachment state: available
- Attached VPC: vpc-0613c43bac933f8f0

An Internet Gateway enables communication between a VPC and the internet when routing and other network requirements are satisfied.

Attaching an Internet Gateway does not automatically make every subnet public. The effective route table must contain a route to the Internet Gateway.

For IPv4 internet communication, an EC2 instance also needs a public IPv4 address and suitable security controls.

## 3. NAT Gateway Inspection

Two read-only inspections were performed:
- A VPC-specific NAT Gateway inspection.
- An account-wide regional NAT Gateway inspection.

Both returned no NAT Gateway entries.

Conclusion:

No NAT Gateway was found in the inspected VPC or in the account-wide results for us-east-1. This does not establish whether NAT Gateways exist in other regions or accounts.

### Internet Gateway vs NAT Gateway

- An Internet Gateway provides a VPC connection to the internet.
- A NAT Gateway allows resources in private subnets to initiate outbound IPv4 internet connections.
- A NAT Gateway is normally deployed in a public subnet.
- A private subnet's default route can point to a NAT Gateway.
- The NAT Gateway's public subnet needs a route to an Internet Gateway.

A NAT Gateway does not make private instances directly reachable through unsolicited inbound internet connections.

## 4. Route Table Target Inspection

The inspected route table was rtb-0509333dff5995e2a.

| Destination | Target | State |
|---|---|---|
| 172.31.0.0/16 | local | active |
| 0.0.0.0/0 | igw-08ac9a33ea66555a7 | active |

The route table has a local route for communication within the VPC and a default IPv4 route to the Internet Gateway.

No NAT Gateway target was shown in this route table.


## 5. VPC Endpoint Inspection

The VPC Endpoint inspection returned an empty list.

No VPC Endpoints were returned for this VPC in us-east-1.

A VPC Endpoint provides private connectivity between a VPC and supported AWS services without requiring internet access for that service traffic.

Two important types:

1. Gateway Endpoint:
   - Supports Amazon S3 and DynamoDB.
   - Uses route table entries.
   - Is associated with route tables.

2. Interface Endpoint:
   - Uses AWS PrivateLink.
   - Creates network interfaces in selected subnets.
   - Uses security groups and DNS configuration.

## 6. IPv4 and IPv6 Inspection

Findings from the VPC inspection:

- VPC IPv4 CIDR: 172.31.0.0/16
- IPv6 CIDR association: none
- VPC state: available

The inspected VPC has an IPv4 CIDR but no associated IPv6 CIDR.

Important concepts:

- IPv4 addresses are 32 bits.
- IPv6 addresses are 128 bits.
- The IPv4 default route is 0.0.0.0/0.
- The IPv6 default route is ::/0.
- IPv6 routing and security configuration must be considered separately from IPv4.


## 7. Subnet CIDR and Available Address Inspection

The six inspected subnets have the following configurations:

| Subnet ID | CIDR | Availability Zone | Available IPv4 addresses |
|---|---|---|---:|
| subnet-0c6e00b5b3b5c4030 | 172.31.80.0/20 | us-east-1b | 4091 |
| subnet-0e6c61a11aad2ed5d | 172.31.16.0/20 | us-east-1c | 4091 |
| subnet-0babfdc409b7b1bbd | 172.31.64.0/20 | us-east-1f | 4091 |
| subnet-07bb55d565f9a66b7 | 172.31.0.0/20 | us-east-1a | 4091 |
| subnet-0af6aadebd14df396 | 172.31.32.0/20 | us-east-1d | 4091 |
| subnet-0c40d3aae49615559 | 172.31.48.0/20 | us-east-1e | 4091 |

### Understanding /20

A /20 IPv4 network contains:

2^(32 - 20) = 4096 total IPv4 addresses.

AWS reserves five IPv4 addresses in each subnet, leaving:

4096 - 5 = 4091 available addresses.

The VPC /16 contains:

2^(32 - 16) = 65,536 IPv4 addresses.

The six /20 subnets account for 24,576 addresses in total. The remaining VPC address space is not automatically assigned to additional subnets.

## 8. Automatic Public IPv4 and IPv6 Assignment

The subnet inspection returned these settings for all six subnets:

- MapPublicIpOnLaunch: True
- AssignIpv6AddressOnCreation: False

Meaning:

- MapPublicIpOnLaunch controls whether instances launched in a subnet automatically receive a public IPv4 address, subject to launch settings.
- AssignIpv6AddressOnCreation controls automatic IPv6 address assignment at instance launch.
- All six inspected subnets have automatic public IPv4 assignment enabled.
- Automatic IPv6 address assignment is disabled.

Important:

Automatic public IP assignment alone does not make a subnet public. The effective route table must provide a route to an Internet Gateway for the subnet to be public.


## 9. VPC DNS Configuration

### DNS Support

Finding:

EnableDnsSupport = true

This setting enables DNS resolution within the VPC using the Amazon-provided DNS resolver.

### DNS Hostnames

Finding:

EnableDnsHostnames = true

This enables DNS hostname assignment for eligible instances in the VPC. Instances with public IPv4 addresses can receive public DNS hostnames, subject to AWS requirements.

### Difference between DNS Support and DNS Hostnames

- DNS Support: enables DNS resolution.
- DNS Hostnames: enables DNS hostname assignment for eligible instances.

Both settings are enabled in the inspected VPC.

## 10. DHCP Options Inspection

The VPC is associated with DHCP options set:

dopt-05041fe7760e52364

The inspected DHCP configuration returned:

| DHCP configuration | Value |
|---|---|
| domain-name | ec2.internal |
| domain-name-servers | AmazonProvidedDNS |

Meaning:

- domain-name specifies the DNS domain suffix used by instances.
- domain-name-servers specifies the DNS servers supplied through DHCP.
- AmazonProvidedDNS means instances use the DNS resolver provided by AWS.

DHCP options can also define other network settings, such as NTP servers and NetBIOS settings.


## 11. Revision Questions and Answers

### Question 1
What is the main purpose of an Internet Gateway?

Answer:

It enables communication between a VPC and the internet when routes and other network requirements are correctly configured.

### Question 2
What makes a subnet public?

Answer:

Its effective route table has a route to an Internet Gateway, commonly 0.0.0.0/0 for IPv4.

### Question 3
Does enabling automatic public IPv4 assignment make a subnet public?

Answer:

No. Public subnet classification depends on the effective route table's route to an Internet Gateway.

### Question 4
What is the purpose of a NAT Gateway?

Answer:

It allows resources in private subnets to initiate outbound IPv4 internet connections without being directly reachable through unsolicited inbound internet connections.

### Question 5
Where is a NAT Gateway normally deployed?

Answer:

In a public subnet with a route to an Internet Gateway.

### Question 6
What is a VPC Endpoint?

Answer:

It provides private connectivity between a VPC and supported AWS services without requiring internet access for that service traffic.

### Question 7
Which AWS services support Gateway Endpoints?

Answer:

Amazon S3 and DynamoDB.

### Question 8
What is the difference between IPv4 and IPv6?

Answer:

IPv4 uses 32-bit addresses, while IPv6 uses 128-bit addresses.

### Question 9
How many total IPv4 addresses are in a /20 subnet?

Answer:

4096 addresses.

### Question 10
How many IPv4 addresses does AWS reserve in each subnet?

Answer:

Five addresses.

### Question 11
How many available IPv4 addresses remain in an otherwise unused /20 subnet?

Answer:

4091 addresses.

### Question 12
What does MapPublicIpOnLaunch control?

Answer:

Whether instances launched in the subnet automatically receive public IPv4 addresses, subject to launch settings.

### Question 13
What does AssignIpv6AddressOnCreation control?

Answer:

Whether IPv6 addresses are automatically assigned at instance launch.

### Question 14
What does EnableDnsSupport do?

Answer:

It enables DNS resolution within the VPC.

### Question 15
What does EnableDnsHostnames do?

Answer:

It enables DNS hostname assignment for eligible instances in the VPC.

### Question 16
What does AmazonProvidedDNS mean in DHCP options?

Answer:

Instances use the DNS resolver provided by AWS.

### Question 17
What is the purpose of DHCP options in a VPC?

Answer:

They provide network configuration information to instances, including DNS domain and DNS server settings.

### Question 18
What happens when a subnet has no explicit route table association?

Answer:

It uses the VPC's main route table.

### Question 19
What is the difference between a Security Group and a Network ACL?

Answer:

A Security Group is stateful and operates at the network interface/resource level. A Network ACL is stateless and operates at the subnet level.

### Question 20
What must be considered before concluding that an EC2 instance can communicate with the internet?

Answer:

The effective route table, Internet Gateway attachment, instance public IP configuration, Security Group rules and Network ACL rules.

## 12. Read-Only Safety Record

All AWS inspection commands in these practical exercises used the CloudLab read-only profile in us-east-1.

No AWS resources were created, modified or deleted.

The exercises focused on understanding existing VPC networking, internet connectivity, IP addressing, DNS, DHCP and network security configuration.

---

# 13. AWS VPC Network Connectivity and Traffic Flow – November 6, 2026

## 1. Lesson Overview

This practical exercise explored how network traffic moves through an AWS VPC, from an EC2 instance to the internet.

The exercise used the existing default VPC in us-east-1.

All AWS inspections were performed using the CloudLab read-only profile.

## 2. EC2 Instance Inspection

Command:

```bash
aws ec2 describe-instances --profile CloudLab --region us-east-1

Result:

No EC2 instances were returned in the selected region using the CloudLab read-only profile.

This means the VPC networking configuration could be inspected, but an actual EC2-to-internet traffic test could not be performed.


## 3. Default VPC Subnet Inspection

The default VPC contained six subnets across six Availability Zones in us-east-1.

| Availability Zone | Subnet ID | IPv4 CIDR |
|---|---|---|
| us-east-1a | subnet-07bb55d565f9a66b7 | 172.31.0.0/20 |
| us-east-1b | subnet-0c6e00b5b3b5c4030 | 172.31.80.0/20 |
| us-east-1c | subnet-0e6c61a11aad2ed5d | 172.31.16.0/20 |
| us-east-1d | subnet-0af6aadebd14df396 | 172.31.32.0/20 |
| us-east-1e | subnet-0c40d3aae49615559 | 172.31.48.0/20 |
| us-east-1f | subnet-0babfdc409b7b1bbd | 172.31.64.0/20 |

Observations:

- All six subnets had automatic public IPv4 address assignment enabled.
- Each subnet reported 4,091 available IPv4 addresses at inspection time.
- Each subnet was marked as a default subnet for its Availability Zone.

Important:

Automatic public IPv4 assignment alone does not make a subnet public. The effective route table must also provide a route to an Internet Gateway for internet routing.


## 4. Route Table Inspection

Route Table ID:

`rtb-0509333dff5995e2a`

Observed routes:

| Destination | Target | State |
|---|---|---|
| 172.31.0.0/16 | local | active |
| 0.0.0.0/0 | igw-08ac9a33ea66555a7 | active |

Explanation:

- The local route allows communication between resources within the VPC.
- The default route `0.0.0.0/0` sends traffic destined outside the VPC through the Internet Gateway.
- The route table was identified as the VPC's main route table.

## 5. Internet Gateway Inspection

Internet Gateway ID:

`igw-08ac9a33ea66555a7`

Observed state:

- State: available
- Attached VPC: `vpc-0613c43bac933f8f0`

An Internet Gateway provides a connection between a VPC and the internet.

However, an attached Internet Gateway alone does not guarantee that an EC2 instance can access the internet. The route table, public IP configuration, Security Group and Network ACL must also permit the required traffic.


## 6. Security Group Inspection

Default Security Group ID:

`sg-0697b37bce19effab`

Observed rules:

**Inbound**
- Allows all protocols from network interfaces associated with the same Security Group.
- Does not allow unrestricted inbound traffic from the internet.

**Outbound**
- Allows all protocols to `0.0.0.0/0`.

Important concepts:

- Security Groups are stateful.
- If an outbound connection is allowed, response traffic is automatically permitted.
- Security Groups operate at the instance/network-interface level.

## 7. Network ACL Inspection

Default Network ACL ID:

`acl-094f8c990545effd2`

Observed rules:

| Direction | Rule Number | Action |
|---|---:|---|
| Inbound | 100 | Allow all IPv4 traffic |
| Inbound | 32767 | Deny remaining traffic |
| Outbound | 100 | Allow all IPv4 traffic |
| Outbound | 32767 | Deny remaining traffic |

Important concepts:

- Network ACLs are stateless.
- Inbound and outbound traffic must be evaluated separately.
- Rules are evaluated in ascending numerical order.
- The first matching rule determines the action.


## 8. Subnet-to-Route-Table Associations

The inspected route table was identified as the main route table.

No explicit subnet associations were returned for the route table.

This means the six default subnets use the VPC's main route table because they do not have explicit associations with another route table.

Important:

- A subnet can have an explicit association with a route table.
- If it has no explicit association, it uses the VPC's main route table.
- The effective route table determines where subnet traffic is directed.

## 9. NAT Gateway Inspection

No NAT Gateways were returned for the inspected VPC.

A NAT Gateway allows resources in a private subnet to initiate outbound internet connections without requiring public IPv4 addresses on those resources.

A NAT Gateway does not provide unsolicited inbound internet access to private instances.

## 10. VPC DNS Configuration

Observed settings:

- DNS support: enabled.
- DNS hostnames: enabled.

These settings support DNS resolution and DNS hostnames for resources in the VPC.



## 11. Local DNS, TCP and HTTPS Connectivity Tests

These tests were performed from the local Ubuntu WSL environment, not from an EC2 instance inside the AWS VPC.

### Test 1: DNS Resolution

Command:

`python3 -c "import socket; print(socket.gethostbyname_ex('google.com'))"`

Result:

The command successfully resolved google.com to an IP address.

### Test 2: TCP Connection

Command:

`python3 -c "import socket; s=socket.create_connection(('google.com', 443), timeout=10); print('HTTPS TCP connection successful:', s.getpeername()); s.close()"`

Result:

The TCP connection to Google on port 443 was successful.

### Test 3: HTTPS Request

Command:

`python3 -c "import urllib.request; r=urllib.request.urlopen('https://www.google.com', timeout=10); print('HTTP Status:', r.status); print('HTTPS request successful')"`

Result:

- HTTP Status: 200
- HTTPS request successful.

These tests confirmed that the local Ubuntu environment could resolve DNS, establish a TCP connection and complete an HTTPS request.

Important distinction:

These tests verified connectivity from WSL only. They did not verify actual internet connectivity from an EC2 instance or prove the AWS VPC's end-to-end traffic flow.


## 12. Understanding AWS VPC Internet Traffic Flow

For an EC2 instance in a public subnet, outbound internet traffic generally follows this path:

1. DNS resolves the destination domain name to an IP address.
2. The Security Group evaluates outbound traffic.
3. The Network ACL evaluates outbound traffic.
4. The route table checks the destination IP address.
5. Traffic matching the default route `0.0.0.0/0` is sent to the Internet Gateway.
6. The Internet Gateway provides connectivity to the internet.

Response traffic follows the return path:

1. The response returns through the Internet Gateway.
2. The Network ACL evaluates inbound traffic.
3. The Security Group automatically permits response traffic for an allowed stateful connection.
4. The response reaches the EC2 instance.

Important:

For direct internet communication, an EC2 instance generally needs a public IPv4 address or Elastic IP, a route to an Internet Gateway, and appropriate security rules.

This traffic flow is a conceptual explanation based on the inspected VPC configuration. It was not observed from an actual EC2 instance because no EC2 instances were returned during inspection.


## 13. Key Lessons Learned

1. A public subnet needs an effective route to an Internet Gateway.
2. Automatic public IPv4 assignment does not independently make a subnet public.
3. An EC2 instance needs an appropriate public IP configuration for direct internet access.
4. Security Groups are stateful, while Network ACLs are stateless.
5. Network ACL rules are evaluated in ascending numerical order.
6. A subnet without an explicit route table association uses the VPC's main route table.
7. A NAT Gateway allows private subnet resources to initiate outbound internet connections.
8. DNS resolution, TCP connectivity and HTTPS requests are separate stages of network communication.
9. An Internet Gateway must be attached to the VPC and referenced by an appropriate route.
10. Successful local WSL connectivity tests do not prove AWS EC2 connectivity.

## 14. Revision Questions

### Question 1
What makes an AWS subnet a public subnet?

### Question 2
Does enabling automatic public IPv4 assignment automatically make a subnet public?

### Question 3
What is the purpose of the `0.0.0.0/0` route?

### Question 4
What is the difference between a Security Group and a Network ACL?

### Question 5
Why does a private subnet use a NAT Gateway?

### Question 6
What three things did the local Ubuntu connectivity tests verify?

## 15. Read-Only Safety Record

All AWS inspection commands used the CloudLab read-only profile in us-east-1.

No AWS resources were created, modified or deleted.

The exercise focused on understanding existing VPC networking, internet connectivity, IP addressing, DNS, route tables and network security.
