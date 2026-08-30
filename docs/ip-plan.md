# 📋 IP & VLAN Addressing Plan

> **Document status:** Active — must be updated whenever network addressing changes.
> This document reflects the design; the machine-readable Source of Truth is
> [`group_vars/all.yml`](../group_vars/all.yml) and [`inventory/hosts.yml`](../inventory/hosts.yml).

## 1. Address Space Overview

| Block            | Purpose                          |
|------------------|----------------------------------|
| `10.10.0.0/16`   | Entire internal enterprise space |
| `10.10.x.0/24`   | One /24 per VLAN (x = VLAN ID)   |
| `10.10.254.0/30` | Edge ↔ Core transit link         |

**Convention:** the third octet matches the VLAN ID (VLAN 10 → `10.10.10.0/24`).
This makes subnets self-documenting and simplifies automation templates.

## 2. VLAN Plan

| VLAN ID | Name        | Subnet          | Gateway (SVI on SW-CORE) | Broadcast     | Usable Range               | Est. Hosts |
|---------|-------------|-----------------|--------------------------|---------------|----------------------------|------------|
| 10      | IT          | 10.10.10.0/24   | 10.10.10.1               | 10.10.10.255  | 10.10.10.1 – 10.10.10.254  | ~20        |
| 20      | HR          | 10.10.20.0/24   | 10.10.20.1               | 10.10.20.255  | 10.10.20.1 – 10.10.20.254  | ~15        |
| 30      | SALES       | 10.10.30.0/24   | 10.10.30.1               | 10.10.30.255  | 10.10.30.1 – 10.10.30.254  | ~30        |
| 99      | SERVER-MGMT | 10.10.99.0/24   | 10.10.99.1               | 10.10.99.255  | 10.10.99.1 – 10.10.99.254  | ~10        |

### Reserved / Future VLANs

| VLAN ID | Name      | Subnet          | Status      |
|---------|-----------|-----------------|-------------|
| 40      | MARKETING | 10.10.40.0/24   | 🔒 Reserved |
| 50      | GUEST     | 10.10.50.0/24   | 🔒 Reserved |

## 3. Transit / Point-to-Point Links

| Link            | Subnet           | Side A (IP)           | Side B (IP)            | Type           |
|-----------------|------------------|-----------------------|------------------------|----------------|
| R-EDGE ↔ SW-CORE| 10.10.254.0/30   | r-edge Gi0/1 = .1     | sw-core Gi0/0 = .2     | Routed L3 link |
| R-EDGE ↔ ISP    | DHCP from NAT    | r-edge Gi0/0          | GNS3 NAT Cloud         | WAN uplink     |

## 4. Device Management IPs

All devices are managed in-band via **VLAN 99** (except R-EDGE, reached via the transit link).

| Hostname | Role               | Platform     | Management IP | Reached Via     | SSH |
|----------|--------------------|--------------|---------------|-----------------|-----|
| r-edge   | Edge Router / NAT  | Cisco IOSv   | 10.10.254.1   | Transit link    | ✅  |
| sw-core  | Core L3 Switch     | Cisco IOSvL2 | 10.10.99.1    | SVI VLAN 99     | ✅  |
| sw-acc1  | Access Switch      | Cisco IOSvL2 | 10.10.99.11   | SVI VLAN 99     | ✅  |
| sw-acc2  | Access Switch      | Cisco IOSvL2 | 10.10.99.12   | SVI VLAN 99     | ✅  |
| docker01 | Automation Host    | Ubuntu 22.04 | 10.10.99.10   | Access port     | ✅  |

## 5. Static IP Assignments (VLAN 99 — Server Zone)

| IP           | Assigned To                  | Notes                              |
|--------------|------------------------------|------------------------------------|
| 10.10.99.1   | SW-CORE (SVI / gateway)      | Default gateway for VLAN 99        |
| 10.10.99.2 – 10.10.99.9   | *Reserved*      | Future network devices             |
| 10.10.99.10  | docker01                     | Ansible · Prometheus · Grafana · CI runner |
| 10.10.99.11  | sw-acc1                      | Management SVI                     |
| 10.10.99.12  | sw-acc2                      | Management SVI                     |
| 10.10.99.13 – 10.10.99.49 | *Reserved*      | Future switches / servers          |

> VLAN 99 has **no DHCP pool** — all addresses are statically assigned.

## 6. DHCP Plan (Server: SW-CORE)

| Pool   | VLAN | Network         | Default Router | DNS     | Excluded (static/reserved)  |
|--------|------|-----------------|----------------|---------|-----------------------------|
| VLAN10 | 10   | 10.10.10.0/24   | 10.10.10.1     | 8.8.8.8 | 10.10.10.1 – 10.10.10.10    |
| VLAN20 | 20   | 10.10.20.0/24   | 10.10.20.1     | 8.8.8.8 | 10.10.20.1 – 10.10.20.10    |
| VLAN30 | 30   | 10.10.30.0/24   | 10.10.30.1     | 8.8.8.8 | 10.10.30.1 – 10.10.30.10    |

**Convention:** `.1` = gateway, `.2 – .10` = reserved for static devices (printers,
APs), `.11 – .254` = DHCP dynamic range.

## 7. Physical Interface Map

### r-edge (Cisco IOSv)

| Interface | Connected To  | Mode / IP                  | Description  |
|-----------|---------------|----------------------------|--------------|
| Gi0/0     | GNS3 NAT Cloud| DHCP, `ip nat outside`     | TO-INTERNET  |
| Gi0/1     | sw-core Gi0/0 | 10.10.254.1/30, `nat inside` | TO-CORE    |

### sw-core (Cisco IOSvL2)

| Interface | Connected To   | Mode                          | VLANs        |
|-----------|----------------|-------------------------------|--------------|
| Gi0/0     | r-edge Gi0/1   | Routed port — 10.10.254.2/30  | —            |
| Gi0/1     | sw-acc1 Gi0/0  | Trunk (802.1Q)                | 10, 20, 99   |
| Gi0/2     | sw-acc2 Gi0/0  | Trunk (802.1Q)                | 30, 99       |
| Gi0/3     | docker01       | Access                        | 99           |

### sw-acc1 (Cisco IOSvL2)

| Interface | Connected To  | Mode          | VLAN(s)     |
|-----------|---------------|---------------|-------------|
| Gi0/0     | sw-core Gi0/1 | Trunk (802.1Q)| 10, 20, 99  |
| Gi0/1     | PC-IT (VPCS)  | Access        | 10          |
| Gi0/2     | PC-HR (VPCS)  | Access        | 20          |

### sw-acc2 (Cisco IOSvL2)

| Interface | Connected To    | Mode          | VLAN(s)  |
|-----------|-----------------|---------------|----------|
| Gi0/0     | sw-core Gi0/2   | Trunk (802.1Q)| 30, 99   |
| Gi0/1     | PC-SALES (VPCS) | Access        | 30       |

## 8. Routing Plan

| Device  | Route                            | Next Hop      | Purpose                    |
|---------|----------------------------------|---------------|----------------------------|
| r-edge  | `10.10.0.0/16`                   | 10.10.254.2   | Return traffic to LAN      |
| r-edge  | `0.0.0.0/0`                      | DHCP (Gi0/0)  | Internet via NAT cloud     |
| sw-core | `0.0.0.0/0`                      | 10.10.254.1   | Default route to edge      |
| sw-core | Connected SVIs (VLAN 10/20/30/99)| —             | Inter-VLAN routing         |

**NAT:** PAT (overload) on r-edge Gi0/0 for source `10.10.0.0/16`.

## 9. Security / ACL Policy

| # | Policy                          | Implementation                                  | Applied On          |
|---|---------------------------------|--------------------------------------------------|--------------------|
| 1 | HR must NOT reach Server Zone   | ACL `BLOCK-HR-TO-SERVER`: deny 10.10.20.0/24 → 10.10.99.0/24 | sw-core, `Vlan20 in` |
| 2 | Management access via SSH only  | `transport input ssh` on all VTY lines           | All devices        |
| 3 | Credentials never in plaintext  | Ansible Vault (AES256)                           | Git repository     |

## 10. Credentials & Access

| Item             | Value / Location                                  |
|------------------|---------------------------------------------------|
| SSH user         | `admin` (local auth, privilege 15)                |
| SSH password     | 🔒 `group_vars/vault.yml` (Ansible Vault)         |
| Domain name      | `lab.local`                                       |
| SSH key exchange | RSA 2048                                          |

## 📝 Change Log

| Date       | Change                              | By     |
|------------|-------------------------------------|--------|
| 2026-08-30 | Initial IP plan (Phase 1 baseline)  | *kritsakorn* |
