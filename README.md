# Frugal PostgreSQL

A dirt-cheap PostgreSQL deployment on AWS — runs on ECS Fargate Spot with EFS-backed persistent storage and automated native backups to S3.

## Why?

Sometimes you need a PostgreSQL database but don't need (or can't justify) the cost of RDS. This template gives you a fully functional PostgreSQL instance for roughly **~$7/month** — about half the cost of the smallest RDS instance and a fraction of Aurora Serverless.

## What You Get

- PostgreSQL 16 on ECS Fargate Spot (up to 70% cheaper than on-demand)
- Persistent storage on EFS (survives container restarts and spot interruptions)
- Automated per-database native backups (`pg_dump --format=custom`) uploaded to S3 on a configurable interval
- WAL archiving to S3 for point-in-time recovery (PITR)
- Two security groups: one for the database, one to attach to clients
- S3 backup bucket with encryption, versioning, and lifecycle expiry

## Quickstart

### Prerequisites

- An AWS account with a VPC and at least one subnet
- The subnet(s) must have a route to the internet (NAT gateway or public subnet) so Fargate can pull the PostgreSQL container image
- An SSM SecureString parameter containing your PostgreSQL password (min 8 chars)
- AWS CLI configured

### Create the SSM Parameter

```bash
aws ssm put-parameter \
  --name /frugal-postgresql/db-password \
  --type SecureString \
  --value "$(openssl rand -base64 24)"
```

### Deploy

```bash
aws cloudformation deploy \
  --template-file postgresql-template.yaml \
  --stack-name frugal-postgresql \
  --parameter-overrides \
    VpcId=vpc-0123456789abcdef0 \
    SubnetIds=subnet-aaa,subnet-bbb \
    NumberOfSubnets=2 \
    DBPasswordSSMParam=/frugal-postgresql/db-password \
  --capabilities CAPABILITY_IAM
```

### Get Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name frugal-postgresql \
  --query 'Stacks[0].Outputs' \
  --output table
```

Key outputs:
- `DatabaseEndpoint` — DNS name for the database (e.g. `postgresql-dns.frugal-postgresql.local`)
- `ClientSecurityGroupId` — attach this to any resource that needs to connect to the database
- `DatabaseSecurityGroupId` — the SG on the PostgreSQL task
- `BackupBucketName` — where your backups land
- `ClusterName` / `ServiceName` — for ECS management

### Connect a Client

Attach the `ClientSecurityGroupId` to your EC2 instance, Lambda, or other resource. Then connect using the DNS name from the stack outputs:

```bash
psql -h postgresql-dns.frugal-postgresql.local -U postgres
```

The DNS name is stable — it automatically updates when the task restarts with a new IP. No need to look up task IPs.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `VpcId` | *(required)* | VPC to deploy into |
| `SubnetIds` | *(required)* | Comma-separated subnet IDs (min 1, max 3) |
| `NumberOfSubnets` | `1` | Must match the number of subnets provided |
| `DBPasswordSSMParam` | *(required)* | Name or ARN of an SSM SecureString parameter containing the PostgreSQL password |
| `BackupIntervalHours` | `24` | Hours between automated backups |
| `BackupRetentionDays` | `30` | Days to retain backups in S3 before deletion |
| `TaskCpu` | `512` | CPU units for the ECS task (512 = 0.5 vCPU) |
| `TaskMemory` | `1024` | Memory in MiB for the ECS task (1024 = 1 GB) |

## Backups

A sidecar container runs alongside PostgreSQL in the same ECS task. It:

1. Waits for PostgreSQL to become ready (`pg_isready`)
2. Runs `pg_dump --format=custom` for every database (including `postgres`)
3. Uploads each `.dump` file to S3 under `YYYY/MM/DD/HHMMSS/<database>.dump`
4. Deletes the local dump from EFS
5. Sleeps for `BackupIntervalHours`, then repeats

The custom format is PostgreSQL's native backup format — it's compressed and supports selective restore of individual tables, schemas, or data-only/schema-only via `pg_restore`.

### WAL Archiving (Point-in-Time Recovery)

PostgreSQL is configured with `archive_mode=on`. WAL files are written to a shared EFS volume and uploaded to S3 (`s3://<bucket>/wal/`) by the backup sidecar every 60 seconds.

To restore to a specific point in time:

1. Restore the latest `pg_dump` backup from S3
2. Download WAL files from `s3://<bucket>/wal/` that were created after the dump
3. Configure `restore_command` in `postgresql.conf` to read from the downloaded WAL files:
   ```bash
   restore_command = 'cp /path/to/wal/%f %p'
   recovery_target_time = '2026-03-10 08:30:00'
   ```
4. Start PostgreSQL — it will replay WAL files up to the target time

### Restoring a Backup

Download the dump from S3 and restore with `pg_restore`:

```bash
# Full restore to a new database
createdb -h postgresql-dns.frugal-postgresql.local -U postgres mydb_restored
pg_restore -h postgresql-dns.frugal-postgresql.local -U postgres -d mydb_restored mydb.dump

# Restore a single table
pg_restore -h postgresql-dns.frugal-postgresql.local -U postgres -d mydb -t mytable mydb.dump

# List contents of a dump without restoring
pg_restore --list mydb.dump
```

## Admin Access with CloudShell VPC Environments

Since the PostgreSQL task runs in a private VPC with no public IP, you can use [CloudShell VPC environments](https://docs.aws.amazon.com/cloudshell/latest/userguide/using-cshell-in-vpc.html) to connect directly from the AWS Console — no bastion host needed.

### Setup

1. Open **CloudShell** in the AWS Console
2. Click the dropdown arrow next to the terminal tab → **Create VPC environment**
3. Enter a name (e.g. `postgresql-admin`)
4. Select the same **VPC** and one of the same **subnets** your stack uses
5. For security groups, select the **ClientSecurityGroupId** from the stack outputs
6. Click **Create**

CloudShell will launch a terminal inside your VPC. From there you can install the PostgreSQL client and connect:

```bash
# Install the client
sudo dnf install -y postgresql16

# Connect using the stable DNS name
psql -h postgresql-dns.frugal-postgresql.local -U postgres
```

### Things to Know About CloudShell VPC Environments

- You get up to **2 VPC environments** per IAM principal
- Storage is **ephemeral** — anything you install is lost when the session ends
- File upload/download via the CloudShell Actions menu is **not available** in VPC mode
- Internet access requires a NAT gateway in the subnet (same as any VPC resource)
- The environment inherits the network config of the VPC/subnet/SGs you assign

### Alternative: ECS Exec

You can also shell directly into the running PostgreSQL container:

```bash
aws ecs execute-command \
  --cluster <ClusterName> \
  --task <TaskArn> \
  --container postgresql \
  --interactive \
  --command "/bin/bash"
```

This drops you into the container itself — useful for running `psql`, checking logs, or inspecting the data directory.

## Cost Comparison (us-east-1, monthly)

All prices are approximate On-Demand rates for the smallest viable configuration running 24/7. Prices will vary by region. Always check the [AWS Pricing Calculator](https://calculator.aws/) for current rates.

| | Frugal PostgreSQL | RDS PostgreSQL | Aurora PostgreSQL Serverless v2 |
|---|---|---|---|
| **Instance** | Fargate Spot 0.5 vCPU / 1 GB | db.t4g.micro (2 vCPU / 1 GB) | 0.5 ACU min (~1 GB) |
| **Compute** | ~$5.40 | $11.68 | $43.80 |
| **Storage** | EFS ~$0.30/GB | gp3 $0.08/GB (20 GB = $1.60) | $0.10/GB |
| **Backups** | S3 ~$0.02/GB | Included (up to DB size) | Included (up to DB size) |
| **I/O** | Included | Included | $0.20/million (Standard) |
| **~Total (1 GB data)** | **~$7** | **~$13** | **~$45** |
| **Managed** | No | Yes | Yes |
| **Auto-failover** | No (Spot re-launch) | Multi-AZ option | Built-in |
| **PITR** | WAL archiving (manual) | Automated | Automated |
| **Scale to zero** | No | Stop instance | Yes (0 ACU) |

### When to Use This

- Dev/test environments
- Side projects and prototypes
- Low-traffic internal tools
- Situations where you need PostgreSQL but can't justify RDS costs
- When you're comfortable managing your own backups and recovery

### When to Use RDS or Aurora Instead

- Production workloads requiring high availability
- When you need automated failover, read replicas, or managed PITR
- Compliance requirements that mandate managed database services
- When the operational overhead of self-managed backups isn't worth the savings

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  VPC                                                │
│  ┌────────────────────────────────────────────────┐ │
│  │ ECS Fargate Spot Task                          │ │
│  │  ┌──────────────┐    ┌──────────────────────┐  │ │
│  │  │ PostgreSQL   │◄───│ Backup Sidecar       │  │ │
│  │  │ :5432        │    │ pg_dump → S3          │  │ │
│  │  └──────┬───────┘    │ WAL upload → S3       │  │ │
│  │         │            └──────────┬───────────┘  │ │
│  └─────────┼───────────────────────┼──────────────┘ │
│  ┌─────────▼───────────────────────▼──────────────┐ │
│  │ EFS: /data (DB files)      /backups (staging)  │ │
│  │                            /backups/wal (WAL)  │ │
│  └────────────────────────────────────────────────┘ │
│  SG:Database ← allows :5432 from → SG:Client       │
└─────────────────────────────────────┬───────────────┘
                                      ▼
                              S3 Backup Bucket
                           (encrypted, versioned)
```

## Caveats

- **Spot interruptions**: Fargate Spot can reclaim your task with 2 minutes notice. EFS ensures data survives, and ECS will re-launch the task automatically. Expect brief outages.
- **No fixed IP**: The task gets a new private IP each time it starts. Use the stable DNS name (`postgresql-dns.<stack-name>.local`) instead — it updates automatically via Cloud Map service discovery.
- **Sidecar installs awscli at startup**: Adds ~30 seconds to initial boot. If you need faster starts, build a custom image with awscli pre-installed.
- **Single instance**: No replication or failover. This is a single-writer setup.

## License

MIT
