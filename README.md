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
2️⃣ Install Ansible AWS collections
bash
Copy code
ansible-galaxy collection install -r collections/requirements.yml
3️⃣ IAM permissions
Your AWS role (e.g. CloudOps) must include:

Service	Required Permissions
RDS	Describe*, RestoreDBClusterFromSnapshot, CreateDBInstance, DeleteDBCluster, DeleteDBInstance
Secrets Manager	GetSecretValue
S3	PutObject, ListBucket
Backup	ListRecoveryPointsByResource (optional for lookup)

⚙️ Configuration
Edit the file group_vars/all.yml:

yaml
Copy code
aws_profile: "jpro-stage.CloudOps"
aws_region: "us-east-2"
snapshot_identifier: "awsbackup:job-012ac1a6-cecd-8a7a-d41b-ff552382858e"
📝 Only these three variables are required.
All other settings (engine, subnet group, security groups, etc.) are automatically discovered from the snapshot metadata.

▶️ How to Run
Run the playbook locally:

bash
Copy code
cd aurora-db-export
AWS_PROFILE=jpro-stage.CloudOps ansible-playbook playbooks/main.yml
This will:

Discover snapshot metadata automatically

Restore a temporary Aurora cluster

Create a temporary DB instance (writer)

Run mysqldump on the target database

Upload the dump to S3

Delete all temporary resources

🧰 Example Output
markdown
Copy code
TASK [discover_snapshot : Generate temporary resource names] ********************
ok: [localhost] => {
    "restored_cluster_id": "tmp-awsbackup-job-012ac1a6-cecd-8a7a-d41b-ff552382858e",
    "restored_instance_id": "tmp-awsbackup-job-012ac1a6-cecd-8a7a-d41b-ff552382858e-01"
}

TASK [rds_restore_cluster : Restore cluster from snapshot] **********************
changed: [localhost]

TASK [rds_create_instance : Wait until instance becomes available] **************
ok: [localhost]

TASK [dump_and_upload : Upload dump file to S3] *********************************
ok: [localhost] => (s3://my-company-db-exports/aurora_exports/jamfsoftware_20251014.sql)

TASK [teardown : Delete cluster] ************************************************
ok: [localhost]
🔐 Secrets and Credentials
Database credentials are securely retrieved from AWS Secrets Manager.

Default lookup (example in dump_and_upload role):

yaml
Copy code
lookup('amazon.aws.aws_secret', 'cloudops/aurora/admin',
       region=aws_region, profile=aws_profile)
Your secret should be in JSON format:

json
Copy code
{
  "username": "admin",
  "password": "supersecretpassword"
}
🧠 Design Highlights
Dynamic Discovery: Snapshot metadata is queried via rds_cluster_snapshot_info.

Safe Cleanup: The teardown role deletes both cluster and instance automatically.

Resilient: If a task fails mid-way, re-running the playbook resumes cleanup.

Idempotent: Existing resources are reused or safely revalidated.

Naming Convention:
Temporary resource names are derived from the snapshot:

php-template
Copy code
tmp-<snapshot_identifier>
tmp-<snapshot_identifier>-01
🧪 Testing
Before production use, test with a small Aurora snapshot and ensure:

Your local host (or bastion) can reach the cluster’s port 3306

S3 bucket permissions are correct

Secrets Manager entry exists and is accessible

