# Oracle Data Guard Out-of-Place Patching using Gold Image

Ansible automation for **zero-downtime out-of-place patching** of an Oracle
19c Data Guard environment using a pre-built Gold Image (a fully patched
Oracle Home staged ahead of time, so patching a node becomes: stop services
→ point config at the new home → restart services → datapatch → validate,
rather than patching binaries in place on a live home).

---

## Features

- Out-of-place patching using a Gold Image (new Oracle Home)
- Standby-first patching approach
- Automatic copy of `dbs/*` and `network/admin/*` from old home to new home
- Updates `/etc/oratab`, `/home/oracle/.bash_profile`, and `listener.ora`
- Listener management (`LISTENER` + `LISTENER2`)
- Data Guard Broker aware (`dgmgrl`)
- Uses `sqlplus` primarily for DB-level operations

---

## Directory Structure

```
oracle-ansible-dg-outofplace/
├── ansible.cfg
├── inventory/
│   └── dataguard-prim-stby/
│       └── hosts
├── playbooks/
│   └── patching/
│       ├── 01_prepare_new_home.yml       # Stage new Oracle Home on both servers
│       ├── 02_standby_outofplace.yml     # Patch standby
│       ├── 03_switchover_to_standby.yml  # Switchover: standby becomes primary
│       ├── 04_primary_outofplace.yml     # Patch (former) primary
│       ├── 05_switchback_to_primary.yml  # Switch roles back, if required
│       ├── 06_datapatch_primary.yml      # Run datapatch on new primary
│       ├── 07_validate.yml               # Post-patch validation
│       └── rollback.yml                  # Revert to pre-patch Oracle Home
├── files/
│   └── 19c/
│       └── gold_images/
│           └── RHEL8_DB1930_GoldImage/
└── README.md
```

---

## Quick Start

### 1. Prerequisites

- Ansible installed on the automation/bastion host
- SSH private key access to both primary and standby servers as `cloud-user`
- Gold Image `.tar.gz` / `.zip` built and ready to copy in

### 2. Configuration

**Edit `ansible.cfg`:**

```ini
private_key_file = /home/youruser/.ssh/id_rsa   # change this
```

**Edit `inventory/dataguard-prim-stby/hosts`:**

```ini
[primary]
NARESH-POC-PRIM-DATABASE ansible_host=10.1.1.1

[standby]
NARESH-POC-STBY-DATABASE ansible_host=10.1.1.2

[all:vars]
oracle_home=/mnt/app/oracle/product/19.3.0/db_1          # Existing Oracle Home
new_oracle_home=/mnt/app/oracle/product/19.3.0/db_1930    # New Oracle Home
gold_image_zip=db-home-2026-03-27-03-06-11AM.zip           # Exact name of Gold Image zip file
patch_stage=/mnt/softwareRepo                              # Patch staging location on primary/standby
listener_name=LISTENER
listener2_name=LISTENER2
```

### 3. Place Gold Image

Copy your gold image archive to:

```
files/19c/gold_images/
```

### 4. Run Patching (in order)

```bash
cd ~/oracle-ansible-dg-outofplace

# Prepare new home on both servers
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/01_prepare_new_home.yml

# Standby patching
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/02_standby_outofplace.yml

# Switchover
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/03_switchover_to_standby.yml

# Primary patching
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/04_primary_outofplace.yml

# Switchback (if roles should return to original)
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/05_switchback_to_primary.yml

# Datapatch on new primary
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/06_datapatch_primary.yml

# Validation
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/07_validate.yml
```

Always run `--syntax-check` before real execution on any playbook you've
edited:

```bash
ansible-playbook --syntax-check -i inventory/dataguard-prim-stby/hosts playbooks/patching/02_standby_outofplace.yml
```

### 5. Rollback (if needed)

```bash
ansible-playbook -i inventory/dataguard-prim-stby/hosts playbooks/patching/rollback.yml
```

*(Document exactly what this reverts — Oracle Home symlink/profile only, or
also DB state — so it's unambiguous during an incident.)*

---

## Required Variables

| Variable                  | Description                                              |
|----------------------------|-----------------------------------------------------------|
| `oracle_env`               | Path to Oracle profile to source (e.g. `~/.bash_profile`)  |
| `oracle_home`               | Current (pre-patch) `ORACLE_HOME`                          |
| `new_oracle_home`           | Path to the new/patched `ORACLE_HOME`                      |
| `gold_image_zip`            | Exact filename of the gold image archive                   |
| `patch_stage`               | Staging directory for patch files on target hosts           |
| `listener_name`             | Primary listener name                                       |
| `listener2_name`            | Secondary listener name                                     |
| `standby_db_unique_name`    | Set automatically via `set_fact` in `02_standby_outofplace.yml` — no manual definition needed |

---

## Author

Naresh Sajwani — Senior Oracle DBA, Data Guard / GoldenGate / RAC / RMAN,
building toward Cloud DevOps DBA tooling.
