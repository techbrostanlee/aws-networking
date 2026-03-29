# aws-networking

A hands-on AWS networking project demonstrating the setup and configuration of a Virtual Private Cloud (VPC) with multi-AZ subnets, a NAT Gateway, Security Groups, and Network Access Control Lists (NACLs).



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




