# Excel-driven Ansible Configuration

`network-config.xlsx` is the editable source of truth for the network configuration.
Do not manually edit generated files in `inventory/` after adopting this workflow.

## Workbook sheets

| Sheet | Purpose |
|---|---|
| `Devices` | Device name, type, management IP, and whether the device creates SVIs |
| `Settings` | DNS, SNMP, Syslog, domain name, and DHCP server device |
| `VLANs` | VLAN ID, name, subnet, gateway, SVI and DHCP settings |
| `Ports` | Access/trunk interface assignments |
| `Routes` | Static routes |
| `ACLs` | Extended ACL rules and interface assignment |
| `NAT` | NAT ACL networks and outside interface |

## Setup

Install the generator dependencies once:

```bash
python -m pip install -r tools/requirements.txt
```

Place `network-config.xlsx` at the repository root:

```text
netdevops-project/
├── network-config.xlsx
├── tools/
│   ├── generate_inventory.py
│   └── requirements.txt
└── inventory/
```

## Validate before generating

```bash
python tools/generate_inventory.py --check
```

The validator checks, among other things:

- VLAN IDs are unique and within `1-4094`.
- IPv4 addresses and subnets are valid.
- VLAN gateways are usable addresses inside their subnet.
- DHCP exclusion ranges are inside the VLAN subnet.
- Access ports reference an existing VLAN.
- Trunk allowed VLANs reference existing VLANs.
- A device/interface pair is not defined more than once.
- Static routes reference existing devices.
- ACL rows for the same ACL use a consistent interface and direction.

## Generate Ansible YAML

```bash
python tools/generate_inventory.py
```

The command validates the workbook first and then generates:

```text
inventory/
├── hosts.yml
├── group_vars/
│   └── all/
│       └── network.yml
└── host_vars/
    ├── r-edge.yml
    ├── sw-core.yml
    ├── sw-acc1.yml
    └── sw-acc2.yml
```

Generated files contain a warning header. Make configuration changes in Excel and regenerate instead of editing those YAML files directly.

## Example: add VLAN 50 FINANCE

1. In `VLANs`, add an enabled row with VLAN ID `50`, name `FINANCE`, subnet `10.10.50.0/24`, gateway `10.10.50.1`, and the desired DHCP settings.
2. In `Ports`, add VLAN `50` to the relevant trunk `Allowed VLANs` cells.
3. Add or change an access port with `Mode = access` and `Access VLAN = 50` if required.
4. Run `python tools/generate_inventory.py --check`.
5. Run `python tools/generate_inventory.py`.
6. Review the Git diff before deploying.
7. Run the Ansible playbook normally.

```bash
git diff -- inventory/
ansible-playbook playbooks/deploy.yml --check --diff
ansible-playbook playbooks/deploy.yml
```

## Recommended workflow

```text
Excel
  ↓
Validation
  ↓
Generated YAML
  ↓
Git diff / Pull Request
  ↓
Ansible check mode
  ↓
Deploy
```

This keeps Excel convenient for editing while YAML remains reviewable and version-controlled for Ansible and CI/CD.
