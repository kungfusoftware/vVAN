# Hub Networking — Azure Verified Modules (AVM)

Production-ready hub networking built exclusively on **[Azure Verified Modules (AVM)](https://azure.github.io/Azure-Verified-Modules/)** using the Virtual WAN (vWAN) pattern. This module deploys a Secured vHub per region with Azure Firewall Premium, VPN/ExpressRoute Gateways, DNS Private Resolver, and ~40 Private DNS zones — forming the centralized network control plane for all spoke workloads.

> **Module:** `Azure/avm-ptn-alz-connectivity-virtual-wan/azurerm` `~> 0.14`  
> **File:** `connectivity.tf`  
> **Subscription:** Connectivity (`corp-connectivity`)

---

## Architecture Overview

```
Azure Virtual WAN (single instance, shared across all hubs)
├── vHub — eastus2                    (10.0.0.0/23)
│   ├── Azure Firewall Premium        (Secured vHub, IDPS, TLS inspection)
│   ├── VPN Gateway                  (VpnGw2AZ — IPsec failover)
│   ├── ExpressRoute Gateway         (ErGw1AZ — primary on-prem path)
│   └── DNS Sidecar VNet             (10.0.2.0/25 — auto-peered to vHub)
│       ├── dns-resolver-inbound     (10.0.2.0/28)
│       └── dns-resolver-outbound    (10.0.2.16/28)
│
└── vHub — centralus                  (10.1.0.0/23)
    ├── Azure Firewall Premium
    ├── VPN Gateway
    ├── ExpressRoute Gateway
    └── DNS Sidecar VNet             (10.1.2.0/25)
        ├── dns-resolver-inbound     (10.1.2.0/28)
        └── dns-resolver-outbound    (10.1.2.16/28)
```

Cross-region routing between hubs, and to on-premises via ER/VPN, is handled automatically by the vWAN platform — no UDRs required on spoke subnets for cross-hub traffic.

---

## Resources Deployed

All resources below are provisioned by `module.connectivity` (`avm-ptn-alz-connectivity-virtual-wan`) via a single `virtual_hubs` map in `connectivity.tf`.

| Resource | SKU / Notes | Controlled by |
|---|---|---|
| `azurerm_virtual_wan` | Standard; single instance shared across all hubs | `virtual_wan_settings` |
| `azurerm_virtual_hub` | One per region; Microsoft-managed routing; no subnets inside | `virtual_hubs.<key>.location` |
| `azurerm_firewall_policy` | Premium SKU; TLS inspection, IDPS, URL filtering | `enabled_resources.firewall = true` |
| `azurerm_firewall` | Premium SKU (`AZFW_Hub`); zone-redundant; 4 public IPs | `firewall.sku_tier = "Premium"` |
| `azurerm_vpn_gateway` | VpnGw2AZ; IPsec failover for ER | `enabled_resources.virtual_network_gateway_vpn = true` |
| `azurerm_express_route_gateway` | ErGw1AZ; primary on-prem path | `enabled_resources.virtual_network_gateway_express_route = true` |
| `azurerm_virtual_network` | DNS sidecar VNet; auto-peered to vHub via `azurerm_virtual_hub_connection` | `sidecar_virtual_network` block |
| `azurerm_private_dns_resolver` | Inbound + outbound endpoints in sidecar VNet | `enabled_resources.private_dns_resolver = true` |
| `azurerm_private_dns_zone` × ~40 | All Azure Private Link zones; linked to sidecar VNet | `enabled_resources.private_dns_zones = true` |
| `azurerm_network_ddos_protection_plan` | Optional; shared across hubs | `virtual_wan_settings.enabled_resources.ddos_protection_plan` |

---

## Address Space

| Component | eastus2 CIDR | centralus CIDR | Notes |
|---|---|---|---|
| vHub routing prefix | `10.0.0.0/23` | `10.1.0.0/23` | Minimum /23; advertised to spokes and on-prem via BGP |
| DNS sidecar VNet | `10.0.2.0/25` | `10.1.2.0/25` | Auto-peered to vHub |
| `dns-resolver-inbound` subnet | `10.0.2.0/28` | `10.1.2.0/28` | Minimum /28 required by Azure |
| `dns-resolver-outbound` subnet | `10.0.2.16/28` | `10.1.2.16/28` | Minimum /28; used for on-prem DNS forwarding |

> **Note:** The vHub is a Microsoft-managed routing construct — you do not define subnets inside it. `GatewaySubnet` and `AzureFirewallSubnet` are managed internally by the platform.

---

## Azure Firewall Premium

The hub deploys **Azure Firewall Premium** (`AZFW_Hub` SKU) in every vHub. Premium enables TLS inspection, Intrusion Detection and Prevention System (IDPS), and URL/FQDN filtering.

### Traffic Flow

All spoke subnets (except `snet-pep-*`) have a UDR with a single default route pointing to the Firewall private IP as next hop. Every packet leaving a spoke hits the Firewall first.

| Traffic Type | Path | Rule Type |
|---|---|---|
| Internet egress (HTTPS) | Spoke → Firewall → Application rules → Internet | `firewall_application_rule_collections` |
| Internet egress (non-HTTP) | Spoke → Firewall → Network rules → Internet | `firewall_network_rule_collections` |
| Inbound from internet | Internet → Firewall public IP → DNAT → Spoke private IP | `firewall_nat_rule_collections` |
| East-west (spoke-to-spoke) | Spoke A → Firewall → Firewall rules → Spoke B | `private_traffic_inspection_enabled` |
| On-premises traffic | Spoke → BGP-propagated routes → vHub ER/VPN (bypasses Firewall) | vHub routing table |
| Private endpoints | `snet-pep-*` has no UDR — routes directly via vHub | Subnet excluded by design |

### Firewall Configuration

```hcl
firewall = {
  sku_tier                         = "Premium"
  threat_intel_mode                = "Alert"   # Change to "Deny" after validating allowlist
  idps_mode                        = "Alert"   # Change to "Deny" for active blocking
  dns_proxy_enabled                = true      # Required for FQDN rules and Private DNS
  zones                            = ["1", "2", "3"]
}
```

### Why `snet-pep-*` Is Excluded from the UDR

Azure does not support UDRs on subnets containing private endpoints — the platform enforces direct routing to private endpoint NICs. DNS resolution via the Private Resolver returns the private IP, so traffic reaches private endpoints without traversing the Firewall. NSG rules on `snet-pep-*` provide the security boundary.

---

## TLS Inspection

TLS inspection is **conditionally enabled** via `firewall_tls_inspection_enabled` in `terraform.tfvars`. It defaults to `false` until prerequisites are provisioned.

### Enable TLS Inspection

Set one flag — everything else is automated by `tls_inspection.tf`:

```hcl
firewall_tls_inspection_enabled = true
```

This automatically provisions:

| Resource | Name | Purpose |
|---|---|---|
| `azurerm_resource_group` | `rg-tls-inspection-<location>-001` | Container for TLS resources |
| `azurerm_user_assigned_identity` | `id-firewall-tls-<location>-001` | Identity the Firewall uses to read the cert from Key Vault |
| `azurerm_key_vault` | `kv-fw-tls-<location>-001` | Premium SKU; purge protection enabled; stores intermediate CA cert |
| `azurerm_key_vault_certificate` | `firewall-intermediate-ca` | Self-signed intermediate CA (RSA 4096, 24 months, `CA:true`) |
| `azurerm_role_assignment` × 3 | — | Terraform: Cert + Secrets Officer; Firewall identity: Secrets User |

### One Manual Step — Distribute the Root CA

Because this uses a self-signed certificate, workload VMs will reject HTTPS connections re-signed by the Firewall. After `terraform apply`, distribute the certificate to all workload VMs as a trusted root CA:

```bash
# Download the certificate
az keyvault certificate download \
  --vault-name kv-fw-tls-eastus2-001 \
  --name firewall-intermediate-ca \
  --file firewall-intermediate-ca.cer

# Linux VMs — deploy via VM extension, Ansible, or cloud-init
# Windows VMs — deploy via GPO: Computer Configuration → Trusted Root CAs
```

### What Is and Is Not Inspected

| Traffic | TLS Inspected? | Notes |
|---|---|---|
| Spoke → internet (HTTPS) | Yes | Decrypted, IDPS-scanned, re-encrypted |
| Spoke → spoke (east-west HTTPS) | Yes | When `private_traffic_inspection_enabled = true` |
| On-premises traffic | No | BGP-propagated routes bypass the Firewall |
| Private endpoints (`snet-pep-*`) | No | Subnet excluded from UDR by design |

---

## DNS Resolution

### Resolution Chain

All spoke VNets are configured with the DNS Private Resolver inbound endpoint IP as their custom DNS server. The resolution chain is:

```
Workload VM
  → DNS query to Resolver inbound IP (e.g. 10.0.2.4)
    → Azure DNS (168.63.129.16)
      → Private DNS zone (e.g. privatelink.blob.core.windows.net)
        → Private endpoint IP returned
```

### On-Premises DNS Forwarding

When `dns_forwarding_rules` is non-empty, the resolver deploys an outbound endpoint with a forwarding ruleset. Queries for on-premises domains are forwarded via the outbound endpoint:

```
Workload VM
  → DNS query for corp.local to Resolver inbound IP
    → Forwarding ruleset matches corp.local.
      → Resolver outbound endpoint
        → On-premises DNS server (e.g. 192.168.1.10:53)
          → On-premises hostname resolved
```

Configure forwarding rules in `terraform.tfvars`:

```hcl
dns_forwarding_rules = {
  corp_local = {
    domain_name = "corp.local."          # trailing dot required
    dns_servers = ["192.168.1.10", "192.168.1.11"]
  }
}
```

Set `dns_forwarding_rules = {}` to skip the outbound endpoint entirely — only Private Link DNS resolution is active.

### DNS Resolver Inbound IP (Output)

```hcl
module.connectivity.dns_server_ip_address["primary"]
# Exposed as output: dns_resolver_inbound_ip
```

Set this as the DNS server on **all spoke VNets** so Private DNS zones resolve correctly.

### Why a Sidecar VNet Is Required

The vHub is Microsoft-managed — custom subnets cannot be placed inside it. The DNS Private Resolver requires dedicated `/28` subnets. A sidecar VNet is provisioned alongside each hub and auto-connected via `azurerm_virtual_hub_connection`, making the resolver IP reachable from all spokes through the vHub routing fabric.

| Subnet | Purpose | Min Size |
|---|---|---|
| `dns-resolver-inbound` | Receives queries from spoke VNets | /28 |
| `dns-resolver-outbound` | Forwards queries to on-premises DNS | /28 |

---

## Routing Intent — Disabled (Deliberate)

Routing Intent is **intentionally disabled** in `connectivity.tf`.

When Routing Intent with `private_traffic_routing_policy` is enabled, vWAN advertises RFC 1918 summary prefixes across hubs instead of specific on-premises routes learned from ER/VPN. This prevents the centralus hub's on-premises routes from being visible in the eastus2 hub routing table, **breaking cross-region ER failover**.

Instead, firewall enforcement is achieved via explicit UDRs on every spoke subnet:

```
0.0.0.0/0 → Firewall private IP (VirtualAppliance)
```

With BGP route propagation **enabled** (`disable_bgp_route_propagation = false`) on spoke route tables, the vHub injects specific on-premises prefixes into spoke subnets:

- Default internet traffic → Firewall (enforced by the UDR)
- On-premises specific prefixes → BGP-propagated routes from vHub (enabling cross-region failover)

To re-enable Routing Intent (if cross-region failover is not a requirement), uncomment the `routing_intent` block in `connectivity.tf` and set `disable_bgp_route_propagation = true` in the landing zone vending module.

---

## On-Premises Connectivity & Failover

### Steady State — All Paths Up

```
On-Premises
  ← ER circuit (primary)    → eastus2 vHub (ER Gateway → Firewall → Spokes)
  ← IPsec VPN (failover)   → eastus2 vHub (VPN Gateway → Firewall → Spokes)
  ← ER circuit (primary)    → centralus vHub (ER Gateway → Firewall → Spokes)
  ← IPsec VPN (failover)   → centralus vHub (VPN Gateway → Firewall → Spokes)

vWAN Backbone connects both hubs automatically — cross-region routing is platform-managed.
```

### Failover Matrix

| Failure Scenario | Automatic Failover | Path |
|---|---|---|
| eastus2 ER circuit down | Yes — within region | eastus2 spoke → Firewall → VPN Gateway → on-prem |
| eastus2 Azure ER infrastructure outage | Yes — within region | eastus2 spoke → Firewall → VPN Gateway → on-prem |
| eastus2 ER + VPN both down | Yes — cross-region | eastus2 spoke → vWAN backbone → centralus Firewall → ER/VPN → on-prem |
| Entire eastus2 vHub down | Yes — cross-region | eastus2 spokes → vWAN backbone → centralus vHub → ER/VPN → on-prem |
| Both regions' ER + VPN down simultaneously | No | All spokes lose on-prem connectivity |

> **Note:** When the eastus2 vHub is fully down, eastus2 spokes lose local internet egress. On-premises connectivity is preserved via centralus gateways.

---

## Inputs

| Variable | Default | Description |
|---|---|---|
| `hub_networks` | see tfvars | Map of vWAN hub configs keyed by name — one entry per region |
| `primary_hub_key` | `primary` | Key from `hub_networks` used for single-value outputs and spoke vending |
| `hub_firewall_enabled` | `true` | Deploy Azure Firewall Premium in all hubs |
| `vpn_gateway_enabled` | `true` | Deploy VPN Gateway (VpnGw2AZ) in all hubs |
| `er_gateway_enabled` | `false` | Deploy ExpressRoute Gateway (ErGw1AZ) in all hubs |
| `ddos_protection_plan_enabled` | `false` | Deploy DDoS Protection Standard (shared across hubs) |
| `firewall_tls_inspection_enabled` | `false` | Enable TLS inspection — requires Key Vault and cert (auto-provisioned by `tls_inspection.tf`) |
| `dns_forwarding_rules` | `{}` | Map of on-premises DNS forwarding rules — leave empty to skip outbound endpoint |
| `private_traffic_inspection_enabled` | `true` | Route all RFC 1918 traffic through the Firewall for east-west inspection |
| `private_bypass_address_prefixes` | `[]` | RFC 1918 prefixes to bypass the Firewall when `private_traffic_inspection_enabled = false` |

### `hub_networks` Map Structure

Each key represents one hub (e.g. `"primary"`, `"secondary"`). Each entry contains:

```hcl
hub_networks = {
  primary = {
    location                       = "eastus2"
    vwan_hub_address_prefix        = "10.0.0.0/23"
    dns_sidecar_address_space      = ["10.0.2.0/25"]
    dns_resolver_inbound_subnet    = "10.0.2.0/28"
    dns_resolver_outbound_subnet   = "10.0.2.16/28"
  }
  secondary = {
    location                       = "centralus"
    vwan_hub_address_prefix        = "10.1.0.0/23"
    dns_sidecar_address_space      = ["10.1.2.0/25"]
    dns_resolver_inbound_subnet    = "10.1.2.0/28"
    dns_resolver_outbound_subnet   = "10.1.2.16/28"
  }
}
```

---

## Outputs

| Output | Description |
|---|---|
| `hub_virtual_network_ids` | Map of hub name → VNet resource ID (all regions) |
| `firewall_private_ips` | Map of hub name → Firewall private IP (all regions) |
| `dns_resolver_inbound_ips` | Map of hub name → DNS Resolver inbound IP (all regions) |
| `hub_virtual_network_id` | Primary hub VNet resource ID (convenience — uses `primary_hub_key`) |
| `firewall_private_ip` | Primary hub Firewall private IP — UDR next-hop for spokes |
| `dns_resolver_inbound_ip` | Primary hub DNS Resolver inbound IP — set as DNS server on spokes |

---

## Firewall Rules

Firewall rules are configured in `terraform.tfvars` and applied by `firewall_rules.tf`.

### Application Rule Collections

| Collection | Priority | Rules |
|---|---|---|
| `arc-windows-updates` | 100 | Windows Update FQDNs, Microsoft CRL |
| `arc-linux-updates` | 110 | Ubuntu/Debian repos, RHEL/CentOS CDN |
| `arc-container-registries` | 120 | MCR, Docker Hub, GHCR, Quay.io, ACR |
| `arc-azure-monitoring` | 130 | Azure Monitor Agent, Defender for Endpoint, Azure Policy |
| `arc-azure-platform` | 140 | management.azure.com, login.microsoftonline.com, Key Vault, Storage |
| `arc-aks` | 150 | AKS control plane, MCR, Azure AD (for AKS clusters in spokes) |

### Network Rule Collections

| Collection | Priority | Rules |
|---|---|---|
| `nrc-dns` | 200 | UDP/TCP 53 → Azure resolver (168.63.129.16) + public DNS (8.8.8.8, 1.1.1.1) |
| `nrc-ntp` | 210 | UDP 123 → Google and NIST NTP servers |
| `nrc-adds-replication` | 220 | Kerberos (88), LDAP/LDAPS (389/636), GC (3268/3269), SMB/RPC (445/135), dynamic RPC (49152–65535), W32TM (123) — Identity VNet → on-premises DCs |
| `nrc-azure-services` | 230 | TCP 443 → `AzureMonitor` + `AzureActiveDirectory` service tags |

### DNAT Rule Collections

| Collection | Priority | Rules |
|---|---|---|
| `narc-inbound-custom-app` | 300 | TCP 8443 inbound → Application Gateway private IP on 443 |
| `narc-inbound-ssh-jumpbox` | 310 | TCP 22 from restricted source → jump box in Management subnet (example — use Bastion instead) |

> After first apply, replace `FIREWALL_PUBLIC_IP` placeholder in DNAT rules with the actual Firewall public IP:
> ```bash
> az network public-ip show \
>   --resource-group rg-connectivity-eastus2-001 \
>   --name pip-afw-hub-eastus2-001 \
>   --query ipAddress -o tsv
> ```

---

## NAT Gateway — Denied by Policy

NAT Gateway is **explicitly prohibited** on all workload landing zones and sandboxes via `Deny-NatGateway-Deployment` in `custom_policies.tf`.

**Why:** NAT Gateway provides outbound internet egress independently of the hub Firewall. Without the policy, a workload team could deploy a NAT Gateway on their spoke subnet and bypass Firewall inspection entirely — no FQDN filtering, no threat intelligence, no IDPS, no egress logs.

```
Without policy (dangerous):
  Spoke VM → NAT Gateway → internet   ← no inspection, no logging

With policy enforced (correct):
  Spoke VM → UDR → Firewall → internet   ← inspected, logged, FQDN-filtered
```

| Assignment | Scope | Covers |
|---|---|---|
| `Deny-NatGW-LZ` | `corp-landingzones` MG | All corp and online workload subscriptions |
| `Deny-NatGW-SBX` | `corp-sandboxes` MG | All sandbox subscriptions |

`corp-platform` is intentionally excluded — the Connectivity subscription owns the hub Firewall public IPs and may legitimately need NAT Gateway in edge cases.

---

## Connecting a New Spoke

Every new spoke VNet must configure three things to integrate with the hub:

**1. DNS server** — set to the DNS Private Resolver inbound IP:
```hcl
dns_servers = [module.root.dns_resolver_inbound_ip]
```

**2. UDR on all subnets** (except `snet-pep-*`):
```hcl
routes = [{
  name                   = "default-to-firewall"
  address_prefix         = "0.0.0.0/0"
  next_hop_type          = "VirtualAppliance"
  next_hop_in_ip_address = module.root.firewall_private_ip
}]
```

**3. VNet peering** — use `avm-res-network-virtualnetwork` with `create_reverse_peering = true` to create both sides of the peering in one module call.

Refer to `modules/landing-zone-vending/main.tf` for the complete spoke onboarding pattern.

---

## Provider Requirements

| Provider | Version | Purpose |
|---|---|---|
| `hashicorp/azurerm` | `~> 4.0` | Azure resource management — AVM modules require v4.x |
| `Azure/azapi` | `~> 2.4` | Direct Azure REST API access for operations not yet in azurerm |

The Connectivity provider alias is used throughout:

```hcl
provider "azurerm" {
  alias           = "connectivity"
  subscription_id = var.subscription_id_connectivity
}
```

---

## Deployment Notes

- **Azure Firewall Premium** takes approximately **10 minutes** to provision.
- **VPN Gateway (VpnGw2AZ)** takes approximately **25 minutes** to provision.
- **ExpressRoute Gateway** takes approximately **20–30 minutes** to provision.
- Total connectivity module apply time: **25–45 minutes** depending on which gateways are enabled.
- The `azurerm_resource_group.connectivity` resource must be pre-created (one per region) before the connectivity module is called.

---

## Related Modules

| Module | File | Purpose |
|---|---|---|
| `avm-ptn-alz` | `main.tf` | Management group hierarchy and ALZ policy library |
| `avm-ptn-alz-management` | `management.tf` | Log Analytics, Sentinel, Defender for Cloud |
| `avm-res-network-bastionhost` | `identity.tf` | Azure Bastion in Identity VNet |
| `lz-vending` | `subscriptions.tf` | Subscription creation and MG association |
| `landing-zone-vending` (local) | `modules/landing-zone-vending/` | Spoke VNet, NSGs, UDRs, peering, budget per workload |