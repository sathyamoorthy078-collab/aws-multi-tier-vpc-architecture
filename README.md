# AWS VPC Architecture with Public and Private Subnets

> **TL;DR:** A custom AWS VPC built with public and private subnets distributed across two Availability Zones. A Bastion Host in a public subnet provides controlled SSH access to an EC2 instance in a private subnet with no public IP address. The project focuses on VPC networking, subnet isolation, routing, security groups, and private instance administration.

## Overview

This project implements a custom AWS VPC architecture designed to separate publicly accessible infrastructure from private workloads.

This project was built using manual AWS Console configuration to focus specifically on understanding VPC networking fundamentals — subnet design, routing, and security groups — before automating the same architecture pattern with Terraform in a separate project.

The VPC uses the CIDR block `10.0.0.0/16` and contains two public subnets and two private subnets distributed across two Availability Zones.

A Bastion Host is deployed in a public subnet and acts as the administrative entry point for SSH access. The private EC2 instance does not have a public IP address and accepts SSH connections only through the Bastion Host access path.

## Architecture

![Architecture](architecture.png)

The network access flow is:

1. The VPC provides the isolated network boundary for the infrastructure.
2. Public and private subnets separate internet-facing resources from private workloads.
3. An Internet Gateway is attached to the VPC.
4. Public subnets use a route table with a route to the Internet Gateway.
5. The Bastion Host is deployed in a public subnet and is reachable through SSH from an allowed source IP.
6. The private EC2 instance is deployed without a public IP address.
7. SSH access to the private instance is allowed through the Bastion Host access path.
8. Administration of the private instance is performed through the Bastion Host using the instance's private IP address.

The architecture provides controlled administrative access to a private workload without exposing the private EC2 instance directly to the internet.

## Infrastructure Summary

| Component | Configuration |
|---|---|
| VPC CIDR | `10.0.0.0/16` |
| Availability Zones | 2 |
| Public Subnets | 2 |
| Private Subnets | 2 |
| Internet Gateway | 1 |
| Public Route Table | 1 |
| Private Route Table | 1 |
| Bastion Host | Amazon Linux 2023 EC2 instance |
| Private Workload | Amazon Linux 2023 EC2 instance |
| Security Groups | Bastion Host SG and Private EC2 SG |

## AWS Components Used

| Component | Role in the Project |
|---|---|
| Amazon VPC | Provides the isolated virtual network |
| Amazon EC2 | Hosts the Bastion Host and private workload |
| Public Subnets | Host resources that require direct internet connectivity |
| Private Subnets | Host workloads without direct public exposure |
| Internet Gateway | Provides internet connectivity for resources in public subnets |
| Route Tables | Control network routing for public and private subnets |
| Security Groups | Restrict inbound and outbound instance traffic |
| Amazon Linux 2023 | Operating system used by the EC2 instances |
| SSH | Provides remote administrative access |

## Network Design

The infrastructure uses a custom VPC with the following layout:

- VPC CIDR: `10.0.0.0/16`
- Two public subnets
- Two private subnets
- Subnets distributed across two Availability Zones
- Internet Gateway attached to the VPC
- Separate route tables for public and private subnets
- Bastion Host deployed in a public subnet
- Private EC2 instance deployed without a public IP address

The public route table provides a route to the Internet Gateway and is associated with the public subnets.

The private subnets use a separate route table without a direct route to the Internet Gateway. A NAT Gateway was not required for the scope of this project, so the private instance does not have general outbound IPv4 internet access through the VPC design.

## Security Design

The Bastion Host provides the administrative path to the private EC2 instance.

### Bastion Host Security Group

The Bastion Host allows:

```text
SSH (TCP 22) from the administrator's allowed public IP
```

Restricting SSH to a specific source IP reduces exposure compared with allowing SSH access from all internet addresses.

### Private EC2 Security Group

The private EC2 instance allows:

```text
SSH (TCP 22) from the Bastion Host Security Group
```

The private instance does not expose SSH directly to an internet CIDR range.

Using the Bastion Host Security Group as the source rule ties private-instance SSH access to the Bastion Host access path rather than allowing a broad IP-based inbound rule.

The resulting administrative path is:

```text
Administrator
    |
    | SSH
    v
Bastion Host
    |
    | SSH over private VPC networking
    v
Private EC2 Instance
```

## Technical Decisions and Considerations

### Public and Private Subnet Separation

Public and private subnets were used to separate resources based on their connectivity requirements.

The Bastion Host requires external SSH access and is therefore placed in a public subnet.

The private EC2 instance does not require direct inbound access from the internet, so it is placed in a private subnet without a public IP address.

This reduces the network exposure of the private workload while still allowing controlled administrative access.

### Why Use a Bastion Host?

The Bastion Host provides a controlled entry point for accessing the private EC2 instance.

Instead of assigning a public IP address to the private instance and exposing its SSH port externally, administrative access is performed through the Bastion Host.

This project uses the Bastion pattern specifically to demonstrate controlled access to private infrastructure. In a production AWS environment, alternatives such as AWS Systems Manager Session Manager could also be considered to reduce or eliminate the need for direct SSH access and a dedicated bastion instance.

### Security Group Referencing

The private EC2 Security Group allows SSH from the Bastion Host Security Group rather than from a broad external CIDR range.

This ensures that the private instance's SSH access is associated with the Bastion Host access path.

### Separate Route Tables

Public and private subnets use separate route tables because their connectivity requirements are different.

The public route table includes a route to the Internet Gateway, allowing appropriately configured resources in public subnets to communicate with the internet.

The private route table does not provide a direct route to the Internet Gateway.

This keeps the routing behavior of public and private network segments separate and explicit.

### Multi-AZ Network Layout

The VPC includes public and private subnets across two Availability Zones.

This creates a network layout that can support workloads distributed across Availability Zones.

However, the project does not claim workload-level high availability. Deploying subnets across multiple Availability Zones alone does not provide high availability unless application resources are also deployed redundantly across those Availability Zones.

## Implementation

The infrastructure was configured in the following sequence:

1. Created a custom VPC with CIDR block `10.0.0.0/16`.
2. Created two public subnets and two private subnets across two Availability Zones.
3. Attached an Internet Gateway to the VPC.
4. Created separate route tables for public and private subnets.
5. Associated the public route table with the public subnets.
6. Associated the private route table with the private subnets.
7. Configured the public route table with internet routing through the Internet Gateway.
8. Created Security Groups for the Bastion Host and private EC2 instance.
9. Deployed the Bastion Host in a public subnet.
10. Deployed the private EC2 instance without a public IP address.
11. Restricted Bastion Host SSH access to the administrator's allowed source IP.
12. Allowed private-instance SSH access through the Bastion Host Security Group.
13. Connected from the local machine to the Bastion Host.
14. Connected from the Bastion Host to the private EC2 instance using its private IP address.

## Verification

The implementation was verified by testing both the network configuration and SSH access path.

The following were validated:

- The Bastion Host was reachable through SSH from the configured administrator IP.
- The private EC2 instance did not have a public IP address.
- The private EC2 instance was not configured for direct SSH access from the internet.
- SSH connectivity from the Bastion Host to the private EC2 instance was successful.
- The Bastion Host and private instance communicated using private VPC networking.
- Public subnets were associated with the public route table.
- The public route table contained a route through the Internet Gateway.
- Private subnets were associated with the separate private route table.
- Security Group rules restricted the SSH access path to the intended sources.

## Troubleshooting

### SSH Authentication from Bastion Host to Private EC2

During connectivity testing, I was able to SSH from my local machine to the Bastion Host, but the next SSH connection from the Bastion Host to the private EC2 instance did not initially work as expected.

As part of the investigation, I verified the VPC configuration, subnet placement, route tables, Security Group rules, private instance IP configuration, and EC2 key pair used for SSH authentication.

The network configuration and Security Group access path were correctly configured. The remaining issue was SSH authentication for the second connection from the Bastion Host to the private instance.

The private key required to authenticate with the EC2 instance was available on my local machine but was not initially available on the Bastion Host.

For this project environment, I copied the key from my local machine to the Bastion Host:

```bash
scp -i vpc-key.pem vpc-key.pem ec2-user@<BASTION_PUBLIC_IP>:~
```

I then connected to the Bastion Host:

```bash
ssh -i vpc-key.pem ec2-user@<BASTION_PUBLIC_IP>
```

On the Bastion Host, I restricted the key file permissions:

```bash
chmod 400 vpc-key.pem
```

I then connected from the Bastion Host to the private EC2 instance using its private IP address:

```bash
ssh -i vpc-key.pem ec2-user@<PRIVATE_EC2_IP>
```

The successful access path was:

```text
Local Machine
    |
    | SSH
    v
Bastion Host
    |
    | SSH using private VPC IP
    v
Private EC2 Instance
```

This confirmed that the VPC routing and Security Group configuration were functioning correctly and that the issue was related to SSH authentication for the second connection.

Copying a private SSH key onto a Bastion Host was used here for the learning environment and is not the preferred approach for a production deployment. SSH agent forwarding or AWS Systems Manager Session Manager can be considered to avoid storing private key material on the Bastion Host.

## Selected Project Evidence

The following screenshots highlight the most important infrastructure and connectivity verification points.

### VPC and Subnet Architecture

![Subnets](screenshots/02-subnets.png)

Shows the public and private subnet layout created inside the custom VPC.

### Route Table Configuration

![Public Route Table](screenshots/04-public-route-table.png)

Shows the routing configuration used by the public network segment.

### Security Group Configuration

![Private Security Group](screenshots/07-private-security-group.png)

Shows the Security Group configuration used to restrict access to the private EC2 instance.

### EC2 Instances

![EC2 Instances](screenshots/08-ec2-instances.png)

Shows the EC2 instances used for the Bastion Host and private workload.

### SSH Access to Private EC2

![SSH Private](screenshots/10-ssh-private-server.png)

Shows successful administrative access to the private EC2 instance through the Bastion Host.

The remaining screenshots are retained in the `screenshots/` directory as additional evidence of the VPC, Internet Gateway, route tables, Security Groups, and SSH connectivity without displaying every AWS console screen in the README.

## Repository Structure

```text
.
├── README.md
├── architecture.png
└── screenshots/
    ├── 01-vpc.png
    ├── 02-subnets.png
    ├── 03-internet-gateway.png
    ├── 04-public-route-table.png
    ├── 05-private-route-table.png
    ├── 06-bastion-security-group.png
    ├── 07-private-security-group.png
    ├── 08-ec2-instances.png
    ├── 09-ssh-bastion.png
    └── 10-ssh-private-server.png
```

## Limitations and Next Steps

This project focuses specifically on VPC network segmentation, routing, Security Groups, and controlled access to a private EC2 instance.

Current limitations and relevant next steps include:

- **Private subnet outbound connectivity:** A NAT Gateway was intentionally not implemented because this was a short-lived learning environment and the additional AWS cost was not necessary for the project's objective. As a result, the private instance does not have general outbound IPv4 internet access. In an environment where private workloads require outbound internet connectivity, a NAT Gateway or another appropriate egress solution could be added.

- **Workload availability:** Although the network contains subnets across two Availability Zones, the project does not deploy redundant application workloads across those zones. Multi-AZ workload deployment and load balancing would be required to demonstrate application-level high availability.

- **Administrative access:** The project uses a Bastion Host to demonstrate controlled SSH access. AWS Systems Manager Session Manager could be evaluated as an alternative approach that avoids exposing SSH and maintaining a dedicated bastion instance.

- **Infrastructure provisioning:** The infrastructure was created manually through the AWS Console to focus on understanding the underlying networking components. A similar infrastructure pattern is automated using Terraform in a separate project, demonstrating the progression from manual infrastructure configuration to Infrastructure as Code.

The project intentionally remains focused on understanding and implementing the underlying AWS VPC networking architecture rather than combining unrelated application deployment or CI/CD components into the same environment.