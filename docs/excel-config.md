# Excel-driven Network Configuration

`network-config.xlsx` คือ **Source of Truth** หลักสำหรับการจัดการ VLAN ของโปรเจกต์นี้

แนวคิดคือผู้ใช้แก้ข้อมูลใน Excel แล้ว GitHub Actions จะตรวจสอบข้อมูล สร้าง Ansible YAML และ deploy ไปยัง Cisco devices โดยอัตโนมัติ

> ไม่ควรแก้ไฟล์ YAML ที่ generator สร้างใน `inventory/` ด้วยมือ เพราะการรัน generator ครั้งถัดไปจะเขียนทับจาก Excel

## VLAN workflow แบบแถวเดียว

ชีตที่ใช้แก้ VLAN หลักคือ `VLANs`

| Column | ตัวอย่าง | หน้าที่ |
|---|---|---|
| `VLAN ID` | `50` | หมายเลข VLAN ระหว่าง 1-4094 |
| `Name` | `FINANCE` | ชื่อ VLAN |
| `Subnet` | `10.10.50.0/24` | Network ของ VLAN |
| `Gateway` | `10.10.50.1` | Gateway / SVI IP |
| `Device` | `sw-acc1` | Access switch ที่ต้องการใช้งาน VLAN |
| `Interface` | `GigabitEthernet0/3` | Access port ที่จะผูกกับ VLAN |
| `Description` | `PC-FINANCE` | คำอธิบาย interface |
| `DHCP` | `TRUE` | เปิด/ปิด DHCP สำหรับ VLAN |

`Device` เลือกได้จาก `sw-acc1` หรือ `sw-acc2` และ `Interface` เลือกจากรายการ interface ที่กำหนดไว้ใน Excel

ถ้าไม่ต้องการผูก VLAN กับ access port สามารถเว้น `Device` และ `Interface` ว่างทั้งคู่ได้

## ตัวอย่าง: เพิ่ม VLAN 50

เพิ่มเพียงหนึ่งแถวในชีต `VLANs`:

```text
VLAN ID     50
Name        FINANCE
Subnet      10.10.50.0/24
Gateway     10.10.50.1
Device      sw-acc1
Interface   GigabitEthernet0/3
Description PC-FINANCE
DHCP        TRUE
```

เมื่อ push ขึ้น GitHub ระบบจะทำให้อัตโนมัติ:

```text
VLAN 50
  ├─ Create VLAN 50
  ├─ Set name FINANCE
  ├─ Ensure VLAN 50 is no shutdown
  ├─ Create SVI Vlan50 = 10.10.50.1/24
  ├─ Create DHCP pool VLAN50
  ├─ Add VLAN 50 to all managed trunks
  └─ sw-acc1 GigabitEthernet0/3 → access VLAN 50
```

## การแก้ VLAN

แก้ค่าที่แถวเดิมใน Excel แล้ว push ขึ้น GitHub เช่นเปลี่ยน:

- Name
- Subnet
- Gateway
- Device
- Interface
- Description
- DHCP

Generator จะสร้าง YAML ใหม่จากข้อมูลล่าสุด และ Ansible จะนำ configuration ใหม่ไป deploy

## การลบ VLAN

Excel เป็น Source of Truth แบบเต็มรูปแบบ

ถ้าลบแถว VLAN ออกจากชีต `VLANs` แล้ว push ขึ้น GitHub ระบบจะตรวจ VLAN ที่มีอยู่จริงบน switch และลบ VLAN ที่ไม่มีอยู่ใน Excel

ตัวอย่าง ถ้า VLAN 50 ถูกลบจาก Excel:

```text
Excel ไม่มี VLAN 50
        ↓
GitHub Actions
        ↓
Ansible gathers current VLANs
        ↓
พบ VLAN 50 บนอุปกรณ์ แต่ไม่มีใน Excel
        ↓
Delete VLAN 50
        ├─ Remove VLAN database entry
        ├─ Remove SVI Vlan50
        └─ Remove DHCP pool VLAN50
```

Cisco default/reserved VLAN ต่อไปนี้ถูกป้องกันและจะไม่ถูกลบโดย workflow นี้:

```text
1
1002
1003
1004
1005
```

> ระวัง: VLAN ที่สร้างด้วยมือบน switch แต่ไม่ได้อยู่ใน Excel จะถูกมองว่าไม่ต้องการและอาจถูกลบโดย deployment

## เพิ่ม VLAN กลับหลังจากเคยลบ

เมื่อเพิ่ม VLAN กลับเข้า Excel ระบบจะสร้าง VLAN ใหม่และสั่ง `no shutdown` ใน VLAN context ให้อัตโนมัติ เช่น:

```text
vlan 50
 no shutdown
```

จึงไม่ต้องเข้า Cisco CLI เพื่อสั่ง `no shutdown` ด้วยมือ

SVI จะถูกสร้างพร้อม `no shutdown` ตาม configuration ของ routing role เช่นกัน

## Trunk interfaces

ผู้ใช้ไม่จำเป็นต้องแก้รายการ Allowed VLAN ในชีต `Ports` ตอนเพิ่ม VLAN

Trunk ที่กำหนดไว้ในระบบจะนำ VLAN ที่อยู่ใน Excel ไปสร้างรายการ allowed VLAN อัตโนมัติ

ชีต `Ports` จึงใช้สำหรับเก็บโครงสร้าง trunk ของ lab เป็นหลัก

## DHCP defaults

ถ้า `DHCP` เปิดใช้งาน แต่ไม่ได้กำหนด DHCP exclusion range แบบละเอียด ระบบจะใช้ค่าพื้นฐานจาก subnet โดยอัตโนมัติ เช่นสำหรับ `10.10.50.0/24` จะ reserve ช่วงต้นของ subnet สำหรับ gateway และ infrastructure

DHCP server ของ lab กำหนดด้วย `dhcp_server_device` ในชีต `Settings`

## Validation

ตรวจ Excel โดยไม่เขียน YAML:

```bash
python tools/generate_inventory.py --check
```

Validator ตรวจอย่างน้อย:

- VLAN ID ต้องเป็น integer และอยู่ระหว่าง 1-4094
- VLAN ID ต้องไม่ซ้ำ
- Subnet ต้องเป็น IPv4 network ที่ถูกต้อง
- Gateway ต้องอยู่ใน subnet และเป็น usable host
- Device ต้องมีอยู่ใน inventory
- Device และ Interface ต้องกรอกคู่กัน
- Device สำหรับ access port ต้องไม่ใช่ router
- Device/interface pair ต้องไม่ซ้ำ
- Static route, ACL และ NAT references ต้องถูกต้อง

Generator จะอ่านเฉพาะตารางหลักของแต่ละชีต ดังนั้นกล่องคำแนะนำด้านขวาใน Excel จะไม่ถูกนำไป validate เป็น VLAN row

## Generate YAML

รัน:

```bash
python tools/generate_inventory.py
```

ไฟล์หลักที่สร้างคือ:

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

## CI/CD workflow

### Push เข้า `main`

```text
network-config.xlsx
        ↓
GitHub Actions
        ↓
Create Python virtual environment
        ↓
Install openpyxl + PyYAML
        ↓
Validate Excel
        ↓
Generate inventory YAML
        ↓
Ansible syntax check
        ↓
Deploy
        ↓
Verify
        ↓
Backup running-config
```

Self-hosted runner ใช้ Python virtual environment (`.venv`) เพื่อหลีกเลี่ยงปัญหา PEP 668 / `externally-managed-environment`

### Pull Request

PR workflow จะทำ validation และ dry-run ก่อน deploy จริง:

```text
Excel
  ↓
Validate
  ↓
Generate YAML
  ↓
yamllint / ansible-lint
  ↓
ansible-playbook --check --diff
```

## Workflow ที่แนะนำ

```text
แก้ Excel
   ↓
Save
   ↓
git add network-config.xlsx
   ↓
git commit
   ↓
git push
   ↓
CI/CD validate และ deploy
```

ตัวอย่าง:

```bash
git add network-config.xlsx
git commit -m "feat: add vlan 50 finance"
git push origin main
```

## Secrets

Password หรือข้อมูลสำคัญไม่ควรเก็บใน Excel

Ansible Vault และ GitHub Secret เช่น `VAULT_PASSWORD` ยังคงใช้แยกจาก `network-config.xlsx`
