# 🧩 Aurora DB Export – Automated MySQL Database Export from AWS Backup Snapshot to S3

This project automates the process of exporting a ***specific MySQL database*** from an ***AWS Backup snapshot*** to ***Amazon S3***, using **Ansible**.
It’s designed for **CloudOps / SRE** workflows where you need to quickly restore and extract database data without manual steps.

# Workflow Overview
The Ansible playbook orchestrates the following steps:
- Discover snapshot metadata (via AWS APIs).
- Retrieve database credentials from AWS Secrets Manager.
- Restore an Aurora RDS cluster or instance from a given AWS Backup snapshot.
- Create a temporary writer DB instance.
- Dump a specific MySQL database via mysqldump.
- Upload the dump file to Amazon S3.
- Teardown temporary resources automatically after completion

## Architecture 
```pgsql
[ Ansible Control Node (EC2 w/ IAM Role) ]
                │
                │ AWS API calls (boto3 via amazon.aws collection)
                ▼
+------------------------------------------------------------+
|                  AWS RDS / Aurora Snapshot                 |
+------------------------------------------------------------+
                │
                ▼
+------------------------------------------------------------+
| Restored Aurora Cluster  ──► Temporary Writer Instance     |
|  │                                                           
|  ├─ Retrieve credentials from AWS Secrets Manager           
|  ├─ mysqldump target DB → local /tmp/testdb_backup.sql      
|  └─ Upload to S3 bucket: s3://bucket4databackup/            |
+------------------------------------------------------------+
                │
                ▼
+------------------------------------------------------------+
|             Cleanup (delete temp instance & cluster)        |
+------------------------------------------------------------+
```

## ⚙️ Prerequisites
### Ansible Control Node (EC2)
You need an **EC2 instance** (Amazon Linux 2023 recommended) with Ansible installed and attached IAM Role.

### IAM Role Policy
Attach a role to the EC2 instance with the following minimal permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSBackupAndRestore",
      "Effect": "Allow",
      "Action": [
        "rds:Describe*",
        "rds:ListTagsForResource",
        "rds:AddTagsToResource",
        "rds:CreateDBInstance",
        "rds:DeleteDBInstance",
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:RestoreDBClusterFromSnapshot",
        "rds:CreateDBSnapshot",
        "rds:DeleteDBSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3UploadDownloadBackup",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:GetObject",
        "s3:GetObjectTagging",
        "s3:PutObjectTagging",
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::bucket4databackup",
        "arn:aws:s3:::bucket4databackup/*"
      ]
    },
    {
      "Sid": "SecretsManagerReadAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecrets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLoggingOptional",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```
### Installation Steps (on EC2)
```bash
# 1. Update system and install dependencies
sudo dnf update -y
sudo dnf install -y git python3-pip mariadb105-client

# 2. Clone the repository
git clone https://github.com/tongquang126/aurora-db-export.git
cd aurora-db-export

# 3. Install Python dependencies
pip3 install --user -r requirements.txt

# 4. Install Ansible collections
ansible-galaxy collection install -r collections/requirements.yml

```
## ▶️ Running the Playbook
#### Modify the global variables to meet your requirement (group_vars/all.yml)
Example configuration 
```yaml
region: us-east-1

# RDS / snapshot info
source_db_identifier: webdatabase
snapshot_identifier: webdatabase-snapshot-2025-10-17
restored_db_identifier: webdatabase-backup-temp
database_name: testdb

# Secret & S3
secret_name: prod/webapp/rds
s3_bucket: bucket4databackup
backup_file_path: /tmp/testdb_backup.sql
```

#### Run the full workflow
```bash
ansible-playbook playbooks/main.yml -vvv
```
#### Or run specific steps via tags 
```bash
ansible-playbook playbooks/main.yml  --tags restore -vvv
```

## Notes & Best Practices
- Run this playbook on EC2 with proper IAM role — no local AWS credentials required.
- The project uses amazon.aws collection to interact with RDS, S3, and Secrets Manager.
- Ensure mysqldump is installed (mariadb105-client package).
- The backup file is first saved to /tmp/ then uploaded to your S3 bucket.
- Temporary RDS resources are cleaned up automatically in the teardown step.
- Use tags to rerun specific stages (useful for debugging).
## License
MIT License © 2025 Tony Tong

## Credits
Developed by **Tony Tong**
Designed for **Cloud Operations / Site Reliability Engineering** teams automating AWS RDS snapshot exports with Ansible.