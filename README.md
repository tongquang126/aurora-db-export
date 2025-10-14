# 🧩 Aurora DB Export Automation (via Ansible)

This project automates the **export of a specific Aurora MySQL database** from an **AWS Backup snapshot** to **Amazon S3**, using **Ansible**.

It is designed for CloudOps/SRE workflows where you need to:
- Restore an Aurora cluster from a snapshot (e.g. AWS Backup-managed snapshot),
- Create a temporary DB instance,
- Run `mysqldump` for one database,
- Upload the dump to S3,
- And automatically delete the temporary resources afterward.

---

## 🚀 Features

- **Minimal inputs required**: only `aws_profile`, `aws_region`, and `snapshot_identifier`
- **Automatic metadata discovery** (engine, subnet group, SGs, encryption, etc.)
- **End-to-end automation**: restore → dump → upload → cleanup
- **Safe and idempotent** — re-runs won’t duplicate resources
- **Works with AWS Backup snapshots** (e.g. `awsbackup:job-xxxxx`)


---

## ⚙️ Prerequisites

### 1️⃣ Install required tools
```bash
brew install ansible awscli mysql
pip3 install boto3 botocore

### 2️⃣ Install Ansible AWS collections
ansible-galaxy collection install -r collections/requirements.yml

