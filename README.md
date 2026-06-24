# Azure Virtual Network – Three-Tier Network Configuration

Configuration of a three-tier Azure Virtual Network (VNet) with logically isolated subnets, Network Security Groups (NSGs), and traffic control rules for Web, Application, and Database tiers.

## Overview

This project implements a software-defined three-tier network architecture in Microsoft Azure, demonstrating IP addressing with CIDR notation, subnet segmentation, and security boundary enforcement using NSG rules. The design follows the principle of least privilege — each tier only accepts traffic from the tier directly above it, and the Database tier explicitly blocks direct access from the Web tier even though they share the same VNet.

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────┐
│  snet-web  (10.0.1.0/24)    │  ← Public-facing, SSH + HTTP allowed
│  VM: vm-web                 │
└────────────┬────────────────┘
             │ (allowed)
             ▼
┌─────────────────────────────┐
│  snet-app  (10.0.2.0/24)    │  ← Internal only, accepts from Web tier
│  VM: vm-app                 │
└────────────┬────────────────┘
             │ (allowed)
             ▼
┌─────────────────────────────┐
│  snet-db   (10.0.3.0/24)    │  ← Most restricted, accepts from App only
│  VM: vm-db                  │  ← Web tier explicitly denied
└─────────────────────────────┘
```

## Virtual Network

| Property | Value |
|---|---|
| Name | vnet-three-tier |
| Address Space | 10.0.0.0/16 |
| Region | West Europe |
| Resource Group | network-assignment-group |

The /16 address space provides 65,536 IP addresses, allowing significant room for future subnet expansion beyond the initial three tiers.

## Subnets

| Subnet | CIDR | Available IPs | Purpose |
|---|---|---|---|
| snet-web | 10.0.1.0/24 | 251 | Public-facing web tier |
| snet-app | 10.0.2.0/24 | 251 | Internal application tier |
| snet-db | 10.0.3.0/24 | 251 | Database tier (most restricted) |

No overlapping address spaces exist between subnets. Each /24 provides 256 addresses (251 usable after Azure reserves 5).

See: `screenshots/vnet-overview.png`, `screenshots/vnet-subnets.png`

## Network Security Groups

### nsg-web (associated with snet-web)

| Priority | Name | Port | Protocol | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-SSH-Inbound | 22 | TCP | Any | Allow |
| 110 | Allow-HTTP-Inbound | 80 | TCP | Any | Allow |
| 65500 | DenyAllInBound | Any | Any | Any | Deny |

The Web tier accepts SSH (for administration) and HTTP (for web traffic) from any source. All other inbound traffic is denied by the default rule.

See: `screenshots/nsg-web-rules.png`, `screenshots/nsg-web-subnet.png`

### nsg-app (associated with snet-app)

| Priority | Name | Port | Protocol | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-SSH-From-Web | 22 | TCP | 10.0.1.0/24 | Allow |
| 110 | Allow-Web-To-App | Any | Any | 10.0.1.0/24 | Allow |
| 200 | Deny-All-Inbound | Any | Any | Any | Deny |

The App tier only accepts traffic originating from the Web subnet (10.0.1.0/24). All other sources including the internet are denied.

See: `screenshots/nsg-app-rules.png`

### nsg-db (associated with snet-db)

| Priority | Name | Port | Protocol | Source | Action |
|---|---|---|---|---|---|
| 100 | Allow-SSH-From-App | 22 | TCP | 10.0.2.0/24 | Allow |
| 110 | Allow-App-To-DB | Any | Any | 10.0.2.0/24 | Allow |
| 120 | Deny-Web-To-DB | Any | Any | 10.0.1.0/24 | Deny |
| 200 | Deny-All-Inbound | Any | Any | Any | Deny |

The Database tier is the most restricted. It explicitly denies the Web subnet (priority 120) even though both are within the same VNet — this overrides Azure's default "AllowVnetInBound" behavior and enforces a hard security boundary between the Web and Database tiers.

See: `screenshots/nsg-db-rules.png`, `screenshots/nsg-db-subnet.png`

## IP Address Strategy

| Resource | IP Type | Value |
|---|---|---|
| vm-web | Public IP (pip-web) | Assigned at deployment |
| vm-web | Private IP | 10.0.1.x (DHCP from snet-web) |
| vm-app | Private IP only | 10.0.2.x (DHCP from snet-app) |
| vm-db | Private IP only | 10.0.3.x (DHCP from snet-db) |

Only the Web tier VM is assigned a public IP — App and DB tiers are intentionally private, reachable only from within the VNet through their respective tier.

## Task 5 & 6: VM Deployment and Connectivity

VM deployment was attempted but blocked by vCPU quota limits on the Azure free trial subscription across multiple regions (West Europe, East US, North Europe). The subscription's Standard Bsv2 Family vCPU quota was exhausted (4/4 used in West Europe) and unavailable for increase in North Europe due to high regional demand.

The NSG rules are correctly configured to enforce the following connectivity matrix when VMs are deployed:

| Source | Destination | Expected Result |
|---|---|---|
| Internet | vm-web (port 22/80) | ✅ Allowed |
| vm-web | vm-app | ✅ Allowed |
| vm-app | vm-db | ✅ Allowed |
| vm-web | vm-db | ❌ Denied (explicit Deny-Web-To-DB rule) |
| Internet | vm-app | ❌ Denied |
| Internet | vm-db | ❌ Denied |

## Troubleshooting Notes

- **West Europe vCPU quota exhausted**: The subscription had 4/4 regional vCPUs used in West Europe from previous assignment VMs (Linux-VM-Assessment using Standard_B2s_v2). All VMs were deallocated to free quota, but the Standard Bsv2 family-level limit of 4 vCPUs remained a hard cap — insufficient for 3 new VMs (6 vCPUs needed).
- **East US and North Europe also restricted**: Quota increase requests for North Europe were rejected due to high regional demand for the Bsv2 family. East US showed all B-series sizes as unavailable.
- **VNet region mismatch**: Initial attempt deployed the VNet in West Europe while trying to deploy VMs in East US — VMs can only join VNets in the same region, requiring VNet recreation in the target region.

## Summary

This project successfully demonstrates the core networking concepts required for a three-tier cloud architecture: CIDR-based IP addressing, subnet segmentation with no overlapping ranges, and NSG-enforced security boundaries between tiers. The explicit `Deny-Web-To-DB` rule at priority 120 in nsg-db is a particularly important security detail — it prevents lateral movement from a compromised web server directly to the database tier, which would otherwise be permitted by Azure's default VNet-internal allow rule.
