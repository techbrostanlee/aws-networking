# aws-networking

A hands-on AWS networking project demonstrating the setup and configuration of a Virtual Private Cloud (VPC) with multi-AZ subnets, a NAT Gateway, Security Groups, and Network Access Control Lists (NACLs).

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [VPC and Subnet Configuration](#vpc-and-subnet-configuration)
3. [Internet Gateway](#internet-gateway)
4. [Route Tables](#route-tables)
5. [NAT Gateway](#nat-gateway)
6. [Security Group](#security-group)
7. [Network ACL (NACL)](#network-acl-nacl)
8. [Testing the NAT Gateway](#testing-the-nat-gateway)
9. [Key Concepts](#key-concepts)
10. [Repository Structure](#repository-structure)
11. [Screenshots](#screenshots)

---

## Architecture Overview

This project builds a production-style AWS network with:

- A **VPC** spanning two Availability Zones (AZs)
- **Public subnets** (internet-accessible via an Internet Gateway)
- **Private subnets** (outbound-only internet access via a NAT Gateway)
- **Security Groups** for instance-level traffic control
- **NACLs** for subnet-level traffic control

```
Internet
    │
    ▼
Internet Gateway (my-igw)
    │
    ▼
┌─────────────────────────────────────────────┐
│                 VPC (10.0.0.0/16)           │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ public-sub-1 │    │ public-sub-2 │       │
│  │ 10.0.1.0/24  │    │ 10.0.2.0/24  │       │
│  │  us-east-1a  │    │  us-east-1b  │       │
│  │  [NAT GW]    │    │              │       │
│  └──────┬───────┘    └──────────────┘       │
│         │ (outbound)                         │
│  ┌──────▼───────┐    ┌──────────────┐       │
│  │ private-sub-1│    │ private-sub-2│       │
│  │ 10.0.3.0/24  │    │ 10.0.4.0/24  │       │
│  │  us-east-1a  │    │  us-east-1b  │       │
│  └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────┘
```

---

## VPC and Subnet Configuration

### VPC

| Setting     | Value           |
|-------------|-----------------|
| Name        | `my-vpc`        |
| IPv4 CIDR   | `10.0.0.0/16`   |
| Tenancy     | Default         |

The `/16` CIDR gives us 65,536 IP addresses, split across subnets.

### Subnets

| Name              | CIDR Block     | Availability Zone | Type    |
|-------------------|----------------|-------------------|---------|
| `public-subnet-1` | `10.0.1.0/24`  | us-east-1a        | Public  |
| `public-subnet-2` | `10.0.2.0/24`  | us-east-1b        | Public  |
| `private-subnet-1`| `10.0.3.0/24`  | us-east-1a        | Private |
| `private-subnet-2`| `10.0.4.0/24`  | us-east-1b        | Private |

Subnets are spread across two AZs to support high availability. Each subnet provides 256 IP addresses (254 usable — AWS reserves 5 per subnet).

Both public subnets have **Auto-assign Public IPv4** enabled so instances launched in them receive a public IP automatically.

Configuration reference: [`vpc/vpc-config.json`](vpc/vpc-config.json)

---

## Internet Gateway

An **Internet Gateway (IGW)** is attached to `my-vpc` to allow communication between the public subnets and the internet.

| Setting | Value   |
|---------|---------|
| Name    | `my-igw` |
| VPC     | `my-vpc` |

The IGW performs NAT for instances in public subnets that have public IP addresses.

---

## Route Tables

Two route tables control how traffic flows within the VPC.

### Public Route Table (`public-rt`)

| Destination   | Target   | Purpose                        |
|---------------|----------|--------------------------------|
| `10.0.0.0/16` | local    | Internal VPC traffic           |
| `0.0.0.0/0`   | `my-igw` | All other traffic → internet   |

Associated with: `public-subnet-1`, `public-subnet-2`

### Private Route Table (`private-rt`)

| Destination   | Target      | Purpose                              |
|---------------|-------------|--------------------------------------|
| `10.0.0.0/16` | local       | Internal VPC traffic                 |
| `0.0.0.0/0`   | `my-nat-gw` | Outbound internet via NAT Gateway    |

Associated with: `private-subnet-1`, `private-subnet-2`

Private subnets have no route to the IGW directly — instances in them cannot be reached from the internet, but can initiate outbound connections through the NAT Gateway.

---

## NAT Gateway

A **NAT Gateway** allows instances in private subnets to reach the internet (for software updates, API calls, etc.) without exposing them to inbound connections.

| Setting      | Value              |
|--------------|--------------------|
| Name         | `my-nat-gw`        |
| Subnet       | `public-subnet-1`  |
| Elastic IP   | Allocated EIP      |
| Type         | Public             |

**Important:** The NAT Gateway must be placed in a **public subnet** and is assigned an **Elastic IP (EIP)** — a static public IP address. Private subnet instances appear to the internet as if their traffic originates from this EIP.

Setup notes: [`nat/nat-setup.md`](nat/nat-setup.md)

---

## Security Group

Security Groups act as a virtual firewall at the **instance level**. They are **stateful** — return traffic is automatically allowed, so you only need to define inbound rules.

| Setting | Value               |
|---------|---------------------|
| Name    | `my-security-group` |
| VPC     | `my-vpc`            |

### Inbound Rules

| Type   | Protocol | Port | Source        | Reason                        |
|--------|----------|------|---------------|-------------------------------|
| SSH    | TCP      | 22   | `YOUR_IP/32`  | Admin access (restricted)     |
| HTTP   | TCP      | 80   | `0.0.0.0/0`   | Web traffic                   |
| HTTPS  | TCP      | 443  | `0.0.0.0/0`   | Secure web traffic            |

### Outbound Rules

| Type       | Protocol | Port | Destination | Reason               |
|------------|----------|------|-------------|----------------------|
| All traffic| All      | All  | `0.0.0.0/0` | Allow all outbound   |

SSH access is restricted to a specific IP address (`/32`) to prevent unauthorised access.

Configuration reference: [`security/security-group.json`](security/security-group.json)

---

## Network ACL (NACL)

NACLs operate at the **subnet level** and are **stateless** — both inbound and outbound rules must be explicitly defined, including return (ephemeral) traffic on ports 1024–65535.

| Setting | Value      |
|---------|------------|
| Name    | `my-nacl`  |
| VPC     | `my-vpc`   |

Associated with all four subnets.

### Inbound Rules

| Rule # | Type       | Protocol | Port Range  | Source        | Action |
|--------|------------|----------|-------------|---------------|--------|
| 100    | SSH        | TCP      | 22          | `YOUR_IP/32`  | ALLOW  |
| 110    | HTTP       | TCP      | 80          | `0.0.0.0/0`   | ALLOW  |
| 120    | HTTPS      | TCP      | 443         | `0.0.0.0/0`   | ALLOW  |
| 130    | Custom TCP | TCP      | 1024–65535  | `0.0.0.0/0`   | ALLOW  |
| *      | All        | All      | All         | `0.0.0.0/0`   | DENY   |

### Outbound Rules

| Rule # | Type       | Protocol | Port Range  | Destination   | Action |
|--------|------------|----------|-------------|---------------|--------|
| 100    | HTTP       | TCP      | 80          | `0.0.0.0/0`   | ALLOW  |
| 110    | HTTPS      | TCP      | 443         | `0.0.0.0/0`   | ALLOW  |
| 120    | Custom TCP | TCP      | 1024–65535  | `0.0.0.0/0`   | ALLOW  |
| *      | All        | All      | All         | `0.0.0.0/0`   | DENY   |

**Rule #130 / #120 (ephemeral ports)** — TCP connections use short-lived ports (1024–65535) for return traffic. Because NACLs are stateless, these must be explicitly allowed or responses will be silently dropped.

Rules are evaluated in **ascending order by rule number**. The first matching rule wins; the default `*` DENY catches everything else.

Configuration reference: [`security/nacl.json`](security/nacl.json)

---

## Testing the NAT Gateway

To confirm that private subnet instances can reach the internet through the NAT Gateway:

**1. Launch a Bastion Host** in `public-subnet-1` with a public IP.

**2. Launch a Private Instance** in `private-subnet-1` with no public IP.

**3. SSH via the Bastion:**

```bash
# From your local machine
ssh -A ec2-user@<BASTION_PUBLIC_IP>

# From the bastion, SSH into the private instance
ssh ec2-user@<PRIVATE_INSTANCE_PRIVATE_IP>
```

**4. Verify outbound internet access from the private instance:**

```bash
curl https://checkip.amazonaws.com
```

**Expected output:** The Elastic IP address assigned to the NAT Gateway — confirming that outbound traffic is routed through it.

```
3.92.XXX.XXX   ← This should match your NAT Gateway's Elastic IP
```

---

## Key Concepts

| Concept | Description |
|---|---|
| **VPC** | A logically isolated virtual network within AWS. All resources are launched inside it. |
| **Public Subnet** | A subnet with a route to an Internet Gateway. Instances can have public IPs. |
| **Private Subnet** | A subnet with no direct route to the internet. Instances are only reachable from within the VPC. |
| **Internet Gateway** | Enables communication between the VPC and the internet for public subnets. |
| **NAT Gateway** | Allows private subnet instances to initiate outbound internet connections without being reachable inbound. |
| **Security Group** | Stateful, instance-level firewall. Return traffic is automatically allowed. |
| **NACL** | Stateless, subnet-level firewall. Both directions must be explicitly defined, including ephemeral ports. |
| **Elastic IP** | A static public IPv4 address assigned to the NAT Gateway. |
| **Availability Zone** | Physically separate data centres within a region. Spreading resources across AZs improves resilience. |

---

## Repository Structure

```
aws-networking/
├── README.md                  # This file
├── vpc/
│   └── vpc-config.json        # VPC, subnet, IGW, and NAT Gateway configuration
├── security/
│   ├── security-group.json    # Security Group inbound/outbound rules
│   └── nacl.json              # NACL inbound/outbound rules
├── nat/
│   └── nat-setup.md           # NAT Gateway setup steps and notes
└── screenshots/
    ├── vpc-dashboard.png
    ├── subnets.png
    ├── route-tables.png
    ├── nat-gateway.png
    ├── security-group.png
    ├── nacl-inbound.png
    ├── nacl-outbound.png
    └── nat-test-curl.png
```

---

## Screenshots

All screenshots are located in the [`screenshots/`](screenshots/) folder and include:

- **VPC Dashboard** — `my-vpc` with CIDR `10.0.0.0/16`
- **Subnets** — All four subnets with their CIDRs and AZs
- **Route Tables** — Public (IGW route) and private (NAT Gateway route)
- **NAT Gateway** — Status: Available, with Elastic IP
- **Security Group** — Inbound and outbound rules
- **NACL** — Inbound and outbound rules including ephemeral port range
- **NAT Test** — Terminal output of `curl https://checkip.amazonaws.com` returning the NAT Gateway EIP

---

## Author

> Replace with your name, student ID, and course details.
