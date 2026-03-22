# Multi-Tier Azure Architecture Deployment Script

A bash script that automates the full deployment of a **3-tier network architecture** on Microsoft Azure, including virtual network setup, virtual machines, and Network Security Group (NSG) configurations.

---

## Architecture Overview

```
Internet
   │
   ▼
┌─────────────────┐
│   Web Tier       │  10.0.1.0/24  (WebSubnet)
│   WebVM          │  ← HTTP (80) / HTTPS (443) from Internet
└────────┬────────┘
         │ Port 8080
         ▼
┌─────────────────┐
│   App Tier       │  10.0.2.0/24  (AppSubnet)
│   AppVM          │  ← Port 8080 from Web Tier only
└────────┬────────┘
         │ Port 1433
         ▼
┌─────────────────┐
│  Database Tier   │  10.0.3.0/24  (DataBaseSubnet)
│  DataBaseVM      │  ← SQL (1433) from App Tier only
└─────────────────┘
```

---

## Prerequisites

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- An active Azure subscription
- Sufficient permissions to create resource groups, VNets, VMs, and NSGs
- Ping Tests conducted for traffic control

---

## Resources Deployed

| Resource | Name | Details |
|---|---|---|
| Resource Group | `layeredAppRG` | Region: South Africa North |
| Virtual Network | `Appstack-vnet` | Address space: `10.0.0.0/16` |
| Web Subnet | `WebSubnet` | `10.0.1.0/24` |
| App Subnet | `AppSubnet` | `10.0.2.0/24` |
| DB Subnet | `DataBaseSubnet` | `10.0.3.0/24` |
| Web VM | `WebVM` | Ubuntu 24.04 |
| App VM | `AppVM` | Ubuntu 24.04 |
| Database VM | `DataBaseVM` | Ubuntu 22.04 |
| Web NSG | `nsgweb` | Attached to WebSubnet |
| App NSG | `nsgapp` | Attached to AppSubnet |
| DB NSG | `nsg-db` | Attached to DataBaseSubnet |

---

## NSG Rules Summary

### Web Tier (`nsgweb`)

| Rule Name | Direction | Action | Protocol | Source | Destination | Port |
|---|---|---|---|---|---|---|
| Allow-HTTP-Inbound | Inbound | Allow | TCP | Internet | 10.0.1.0/24 | 80 |
| Allow-HTTPS-Inbound | Inbound | Allow | TCP | Internet | 10.0.1.0/24 | 443 |
| Allow-To-AppTier | Outbound | Allow | TCP | 10.0.1.0/24 | 10.0.2.0/24 | 8080 |
| Deny-To-DBTier | Outbound | Deny | Any | 10.0.1.0/24 | 10.0.3.0/24 | Any |

### App Tier (`nsgapp`)

| Rule Name | Direction | Action | Protocol | Source | Destination | Port |
|---|---|---|---|---|---|---|
| Allow-From-WebTier | Inbound | Allow | TCP | 10.0.1.0/24 | 10.0.2.0/24 | 8080 |
| Deny-All-Other-Inbound | Inbound | Deny | Any | VirtualNetwork | 10.0.2.0/24 | Any |
| Allow-To-DBTier | Outbound | Allow | TCP | 10.0.2.0/24 | 10.0.3.0/24 | 1433 |
| Deny-To-WebTier | Outbound | Deny | TCP | 10.0.2.0/24 | 10.0.1.0/24 | Any |

### Database Tier (`nsg-db`)

| Rule Name | Direction | Action | Protocol | Source | Destination | Port |
|---|---|---|---|---|---|---|
| Allow-From-AppTier | Inbound | Allow | TCP | 10.0.2.0/24 | 10.0.3.0/24 | 1433 |
| Deny-All-Other-Inbound | Inbound | Deny | Any | VirtualNetwork | 10.0.3.0/24 | Any |
| Deny-Internet-Outbound | Outbound | Deny | Any | 10.0.3.0/24 | Internet | Any |

---

## Usage

1. **Clone or download** the script to your local machine.

2. **Make it executable:**
   ```bash
   chmod +x test
   ```

3. **Log in to Azure:**
   ```bash
   az login
   ```

4. **Run the script:**
   ```bash
   ./test
   ```

The script uses `set -e`, so it will exit immediately if any command fails.

---

## Ping Tests

After deployment, Ping tests were conducted to ensure communication to each VMs are controlled. The DB should be isolated from the Webserver.

---

## Security Considerations

- **Passwords** are hardcoded in the script (`kinza@J2123B#2026`). For production deployments, use Azure Key Vault or environment variables instead.
- The Database Tier has no outbound internet access, limiting its exposure.
- The Web Tier is completely blocked from reaching the Database Tier directly, enforcing tier separation.

---

## Cleanup

To delete all deployed resources and avoid ongoing charges:

```bash
az group delete --name layeredAppRG --yes --no-wait
```
