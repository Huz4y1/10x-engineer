---
tags: [postgresql, aws, rds, cloud, databases, guide]
---

# AWS RDS for PostgreSQL

**PostgreSQL that Amazon runs for you.** RDS = *Relational Database Service*. It's a normal Postgres database — same SQL, same pgAdmin, same Python — and AWS handles the server, backups, patching and failover.

Hub: [[PostgreSQL]] · The Azure version: [[Azure Database for PostgreSQL]] · Storing files on AWS: [[AWS S3]] · Whole AWS pipeline: [[Pipeline setup - AWS]]

> ✅ **Every `aws` command and flag here was checked against the AWS CLI's own help.** Prices and free-tier rules change — check the AWS pricing page before creating anything.

---

## What it is, in simple words

| Job | Your own Postgres | RDS |
|---|---|---|
| Install and patch | You | AWS |
| Backups | You | ✅ Automatic, up to 35 days |
| Restore to 10:42 yesterday | Hard | ✅ Point-in-time restore |
| Standby copy in another data centre | You build it | ✅ `--multi-az` |
| SQL, pgAdmin, SQLAlchemy | Same | **Same** |

**RDS or Aurora?** AWS also sells **Aurora PostgreSQL** — Postgres-compatible, with Amazon's own storage underneath. It's faster to fail over and scales further, but costs more. **Start with RDS**; move to Aurora when you have a reason.

---

## The one concept to understand first — security groups

On AWS, **nothing can reach your database until a security group allows it.** A security group is a firewall attached to the database: a list of *"allow port 5432 from these addresses"* rules.

```
your laptop ──✗── [ security group: allows 5432 from ??? ] ── RDS database
```

Most "I can't connect to RDS" problems are this. Check it first, every time.

---

## Choosing a size

| Class | For |
|---|---|
| `db.t4g.micro` / `db.t3.micro` | Learning and tiny apps — the cheapest |
| `db.t4g.small` / `medium` | Small real apps |
| `db.m7g.large` and up | Production with steady load |

`t` classes are **burstable** — cheap, can briefly run faster, slow down if busy for long. `g` = AWS's own ARM (Graviton) chips, usually cheaper for the same performance.

> ⚠️ **Use a current Postgres version.** Once a major version reaches end of life, AWS moves it to **RDS Extended Support**, which costs extra every hour. Pick the newest version RDS offers.

---

## Creating one — AWS CLI

Install: `winget install Amazon.AWSCLI` on Windows, or AWS's installer in WSL. Then `aws configure` to enter your access keys and default region.

### 1. A security group that lets only you in

```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)

SG_ID=$(aws ec2 create-security-group \
  --group-name shop-pg-sg \
  --description "Postgres from my IP" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp --port 5432 \
  --cidr "$MY_IP/32"                  # /32 = exactly this one address
```

> ⚠️ **Never open 5432 to `0.0.0.0/0`** (the whole internet). Bots scan for open Postgres ports constantly and try common passwords.

### 2. The database

```bash
aws rds create-db-instance \
  --db-instance-identifier shop-pg-dev \
  --engine postgres \
  --engine-version 17 \
  --db-instance-class db.t4g.micro \
  --allocated-storage 20 \
  --storage-type gp3 \
  --master-username shopadmin \
  --manage-master-user-password \
  --db-name shop \
  --vpc-security-group-ids "$SG_ID" \
  --publicly-accessible \
  --backup-retention-period 7 \
  --storage-encrypted \
  --deletion-protection
```

| Flag | Means |
|---|---|
| `--db-instance-identifier` | The database's name in AWS |
| `--engine postgres` / `--engine-version` | Postgres, and which **major** version — RDS picks the current minor version for you. List what's available: `aws rds describe-db-engine-versions --engine postgres --query "DBEngineVersions[].EngineVersion"` |
| `--db-instance-class` | How big (table above) |
| `--allocated-storage 20` / `--storage-type gp3` | 20 GB of the standard SSD type |
| `--master-username` | The first user |
| **`--manage-master-user-password`** | ✅ **AWS creates the password and keeps it in Secrets Manager** — you never type or store it |
| `--db-name shop` | Create a database called `shop` inside it |
| `--vpc-security-group-ids` | Attach the firewall from step 1 |
| `--publicly-accessible` | Give it an address reachable from the internet (still limited by the security group). Leave this out for private-only |
| `--backup-retention-period 7` | Keep automatic backups for 7 days |
| `--storage-encrypted` | Encrypt the disk |
| `--deletion-protection` | Refuse to delete until you turn this off — stops accidents |

It takes about 5–15 minutes. Check progress:

```bash
aws rds describe-db-instances \
  --db-instance-identifier shop-pg-dev \
  --query "DBInstances[0].[DBInstanceStatus, Endpoint.Address]" --output text
```

When the status is `available`, the second value is your **host name** — something like `shop-pg-dev.abc123xyz.eu-west-2.rds.amazonaws.com`.

### Getting the password AWS created

```bash
SECRET_ARN=$(aws rds describe-db-instances \
  --db-instance-identifier shop-pg-dev \
  --query "DBInstances[0].MasterUserSecret.SecretArn" --output text)

aws secretsmanager get-secret-value \
  --secret-id "$SECRET_ARN" \
  --query SecretString --output text          # prints {"username":"shopadmin","password":"..."}
```

> **Your app should read the secret the same way at startup** (with `boto3`), instead of having the password in its config. Secrets Manager can also rotate it automatically.

### Or in the AWS Console

**RDS → Create database → Standard create → PostgreSQL**. Under **Templates** choose **Free tier** or **Dev/Test** (Production turns on expensive options). Under **Credentials**, choose **Managed in AWS Secrets Manager**. Under **Connectivity**, set **Public access: Yes** only for learning, and choose your security group.

---

## Connecting

| From | Settings |
|---|---|
| **psql** | `psql "host=shop-pg-dev.abc123xyz.eu-west-2.rds.amazonaws.com port=5432 dbname=shop user=shopadmin sslmode=require"` |
| **[[pgAdmin 4]]** | Host = the endpoint, port 5432, user `shopadmin`, **Parameters → SSL mode: require** |
| **Python** | `postgresql+psycopg://shopadmin:PASSWORD@<endpoint>:5432/shop?sslmode=require` — [[PostgreSQL with SQLAlchemy]] |

> **Recent Postgres versions on RDS require encrypted connections by default** (the `rds.force_ssl` setting). Always add `sslmode=require`; for full certificate checking use `sslmode=verify-full` with AWS's CA bundle.

> **Connection just hangs?** It's the security group (your IP changed) or the database isn't publicly accessible. *"Password authentication failed"* means you did reach it.

---

## Users — don't let the app use the master account

The master user has the `rds_superuser` role — powerful, and note that on RDS **nobody gets true superuser**. Create a login for the app with only what it needs:

```sql
CREATE ROLE shop_app LOGIN PASSWORD 'a-long-random-password';
GRANT CONNECT ON DATABASE shop TO shop_app;
GRANT USAGE ON SCHEMA public TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO shop_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_app;
```

### Better: IAM authentication — no stored password

Turn on `--enable-iam-database-authentication`, then an app with the right IAM role gets a **15-minute token** to use as its password:

```bash
aws rds generate-db-auth-token \
  --hostname shop-pg-dev.abc123xyz.eu-west-2.rds.amazonaws.com \
  --port 5432 --username shop_app
```

```sql
GRANT rds_iam TO shop_app;                 -- this role now logs in with IAM tokens
```

> **Recommended for apps running on AWS** (ECS, Lambda, EC2) — nothing to leak or rotate ([[Security in practice]]).

---

## Extensions

Most common extensions just work — no allow-list step like Azure:

```sql
SHOW rds.extensions;                        -- everything this RDS version supports
CREATE EXTENSION IF NOT EXISTS vector;      -- pgvector
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

Settings that normally live in `postgresql.conf` (like `shared_preload_libraries` for `pg_stat_statements`) are changed through a **parameter group** — a named set of settings attached to the database. Create one, change it, attach it with `modify-db-instance`, and reboot if the setting needs it.

---

## Connection pooling — RDS Proxy

Small instances allow few connections, and apps with many workers — or AWS Lambda, which opens a connection per call — run out fast. **RDS Proxy** sits in front of the database and shares a small pool of real connections among many clients. Point your app at the proxy's endpoint instead of the database's.

---

## Backups and restoring

- Automatic daily backups plus continuous logs, kept for `--backup-retention-period` days (up to 35)
- **Point-in-time restore** creates a **new** database as it was at any second in that window:

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier shop-pg-dev \
  --target-db-instance-identifier shop-pg-restored \
  --restore-time 2026-03-17T10:42:00Z
```

- **Manual snapshots** last until you delete them — take one before anything risky:

```bash
aws rds create-db-snapshot \
  --db-instance-identifier shop-pg-dev \
  --db-snapshot-identifier shop-before-migration
```

---

## Saving money — and not getting surprise bills

```bash
aws rds stop-db-instance  --db-instance-identifier shop-pg-dev     # stop paying for compute
aws rds start-db-instance --db-instance-identifier shop-pg-dev
```

> ⚠️ **A stopped RDS instance starts itself again after 7 days.** Storage and snapshots are billed even while stopped.

**Deleting it when you're finished:**

```bash
aws rds modify-db-instance --db-instance-identifier shop-pg-dev \
  --no-deletion-protection --apply-immediately

aws rds delete-db-instance --db-instance-identifier shop-pg-dev \
  --final-db-snapshot-identifier shop-final        # keep one last snapshot
  # or --skip-final-snapshot to keep nothing
```

> ⚠️ **Manual snapshots are kept — and billed — after the database is deleted.** List them with `aws rds describe-db-snapshots` and delete the ones you don't need.

> ⚠️ **Deleting the database doesn't delete the security group or secret** — the secret costs a little each month. AWS has no "delete everything for this project" button like Azure resource groups, so tag resources and check the **Billing** page after cleaning up ([[Pipeline setup - AWS]]).

---

## Common problems

| You see | Cause | Fix |
|---|---|---|
| Connection **hangs / times out** | Security group doesn't allow your IP | `authorize-security-group-ingress` with your current IP |
| Times out even with the right SG | Not publicly accessible, or in a private subnet | Make it public (dev), or connect from inside the VPC |
| *no pg_hba.conf entry … no encryption* | SSL required | `sslmode=require` |
| *password authentication failed* | Wrong password | Read it from Secrets Manager |
| *permission denied to create extension* | Needs `rds_superuser` | Run it as the master user |
| *remaining connection slots are reserved* | Too many connections | Smaller pools, RDS Proxy |
| Bill still going after "deleting" | Snapshots, secrets, Extended Support | Delete snapshots and secrets; use a current version |
| `delete-db-instance` refuses | Deletion protection is on | `modify-db-instance --no-deletion-protection` |

## Related

[[PostgreSQL]] · [[Azure Database for PostgreSQL]] · [[AWS S3]] · [[PostgreSQL with SQLAlchemy]] · [[pgAdmin 4]] · [[PostgreSQL SQL]] · [[Pipeline setup - AWS]] · [[Cloud comparison dictionary]] · [[Security in practice]]
