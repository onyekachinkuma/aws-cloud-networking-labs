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

