# 🌐 Network as Code — Enterprise Network Automation Lab

Automated enterprise network management using **Infrastructure as Code (IaC)** principles.
A simulated company network built in **GNS3**, fully managed through **Git**, **Ansible**,
and **CI/CD pipelines** — no manual CLI configuration required.

![Topology](docs/topology.png)

## 📖 Overview

Traditionally, network engineers configure routers and switches manually via SSH,
one device at a time. This project applies **DevOps practices to network operations
(NetDevOps)**:

- 📦 All network configuration lives in this Git repository (**Single Source of Truth**)
- 🤖 **Ansible** renders and pushes configurations to all devices automatically
- 🔀 Changes go through **branch → Pull Request → review → CI/CD → deploy**
- 📊 **Prometheus + Grafana** monitor the network in real time
- ⏪ Every change is versioned — full history and easy rollback

> **Example:** Adding a new department (VLAN) = editing ~5 lines of YAML
> and opening a Pull Request. The pipeline handles the rest.

## 🗺️ Network Topology

A small enterprise LAN with 3 departments, simulated in GNS3:

```
                        Internet (NAT Cloud)
                              │
                        [ R-EDGE ]  Cisco IOSv — NAT, ACL, SSH
                              │  10.10.254.0/30 (routed link)
                        [ SW-CORE ] Cisco IOSvL2 — Inter-VLAN Routing, DHCP
                        ┌─────┼──────────┐
                  [ SW-ACC1 ] [ SW-ACC2 ] [ Docker Host ]
                   VLAN 10,20   VLAN 30     VLAN 99
                   IT / HR      Sales       Ansible · Monitoring · CI Runner
```

### IP / VLAN Plan

| VLAN | Name        | Subnet          | Gateway (SVI) | Purpose                     |
|------|-------------|-----------------|---------------|-----------------------------|
| 10   | IT          | 10.10.10.0/24   | 10.10.10.1    | IT department workstations  |
| 20   | HR          | 10.10.20.0/24   | 10.10.20.1    | HR department workstations  |
| 30   | SALES       | 10.10.30.0/24   | 10.10.30.1    | Sales department            |
| 99   | SERVER-MGMT | 10.10.99.0/24   | 10.10.99.1    | Servers & device management |
| —    | Transit     | 10.10.254.0/30  | —             | Edge ↔ Core routed link     |

### Management IPs

| Device   | Role              | Platform     | Management IP |
|----------|-------------------|--------------|---------------|
| r-edge   | Edge Router (NAT) | Cisco IOSv   | 10.10.254.1   |
| sw-core  | Core L3 Switch    | Cisco IOSvL2 | 10.10.99.1    |
| sw-acc1  | Access Switch     | Cisco IOSvL2 | 10.10.99.11   |
| sw-acc2  | Access Switch     | Cisco IOSvL2 | 10.10.99.12   |
| docker01 | Automation Host   | Ubuntu 22.04 | 10.10.99.10   |

### Security Policy

- HR (VLAN 20) is **denied** access to the Server Zone (VLAN 99) via ACL
- All devices accessible via SSH only (no telnet), local auth
- Secrets encrypted with **Ansible Vault** — no plaintext credentials in this repo

## 🛠️ Tech Stack

| Tool                     | Role                                              |
|--------------------------|---------------------------------------------------|
| **GNS3**                 | Network simulation (Cisco IOSv / IOSvL2)          |
| **Git / GitHub**         | Version control, Source of Truth, code review     |
| **Ansible**              | Configuration management & deployment             |
| **GitHub Actions**       | CI/CD — lint, dry-run, deploy, post-deploy tests  |
| **Docker**               | Runs monitoring stack & CI runner                 |
| **Prometheus + Grafana** | Network monitoring & dashboards                   |

## 📁 Repository Structure

```
├── inventory/          # Device inventory (hosts, management IPs)
├── group_vars/         # Shared variables — VLANs, credentials (vaulted)
├── host_vars/          # Per-device variables (SVIs, DHCP pools)
├── roles/              # Ansible roles: vlan, routing, dhcp, acl
├── playbooks/          # deploy.yml, backup.yml
├── backups/            # Auto-backed-up device configurations
├── docs/               # Topology diagram, IP plan, design notes
└── tests/              # Post-deployment validation
```

## 🚀 Usage

### Prerequisites

- GNS3 lab is running (see [docs/ip-plan.md](docs/ip-plan.md))
- Ansible ≥ 2.15 on the automation host
- Vault password file (not in repo — ask the maintainer)

### Common Operations

```bash
# Verify connectivity to all devices
ansible all -m ping

# Preview changes without applying (dry-run)
ansible-playbook playbooks/deploy.yml --check --diff

# Deploy configuration to all devices
ansible-playbook playbooks/deploy.yml

# Backup running-configs from all devices
ansible-playbook playbooks/backup.yml
```

### ✏️ How to Make a Change (e.g., add a new VLAN)

1. Create a branch: `git checkout -b feature/add-vlan-40`
2. Edit `group_vars/all.yml` — add the new VLAN entry:
   ```yaml
   - id: 40
     name: MARKETING
     subnet: 10.10.40.0/24
     gateway: 10.10.40.1
   ```
3. Commit and push, then open a **Pull Request**
4. CI automatically runs **lint** and **dry-run** checks
5. After review and merge, the pipeline **deploys to all devices** and runs
   post-deployment tests

> ⚠️ Direct pushes to `main` are blocked. All changes must go through a PR.

## 🔀 CI/CD Pipeline

```
 Push / PR ──► Lint ──► Dry-run ──► [Manual Approve] ──► Deploy ──► Verify
              yamllint   ansible      on merge to        ansible     ping /
              ansible-   --check      main               playbook    OSPF /
              lint                                                   SSH tests
```

## 🗓️ Project Roadmap

- [x] **Phase 1** — Build GNS3 topology, manual baseline configuration
- [x] **Phase 2** — Git repository structure, Source of Truth, Vault
- [ ] **Phase 3** — Ansible roles & playbooks (VLAN, routing, DHCP, ACL, backup)
- [ ] **Phase 4** — CI/CD pipeline with GitHub Actions + self-hosted runner
- [ ] **Phase 5** — Monitoring stack (Prometheus, Grafana, SNMP exporter, syslog)
- [ ] **Phase 6** *(optional)* — Terraform GNS3 provider, NetBox integration

## 📚 Documentation

- [IP & VLAN Plan](docs/ip-plan.md)
- [Topology Diagram](docs/topology.png)
