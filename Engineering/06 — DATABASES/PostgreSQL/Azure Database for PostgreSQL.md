---
tags: [postgresql, azure, cloud, databases, guide]
---

# Azure Database for PostgreSQL

**PostgreSQL that Microsoft runs for you.** You get a normal Postgres database — same SQL, same pgAdmin, same Python code — but Azure handles the server, backups, updates and failover.

Hub: [[PostgreSQL]] · The AWS version: [[AWS RDS for PostgreSQL]] · Azure basics: [[Azure fundamentals]] · Azure's other SQL database: [[Azure SQL Database]]

> ✅ **Every `az` command and flag here was checked against Azure CLI 2.90's own help.** Prices change — check the Azure pricing page before creating anything.

---

## What it is, in simple words

Running Postgres yourself means: install it, keep it patched, take backups, restore them when things break, and keep it running at 3 a.m. **A managed database does all of that for you.** You just connect and use it.

| Job | Your own Postgres | Azure Database for PostgreSQL |
|---|---|---|
| Install and patch Postgres | You | Azure |
| Daily backups | You | ✅ Automatic, 7–35 days |
| Restore to 10:42 yesterday | Hard | ✅ A few clicks |
| A second copy in case a data centre fails | You build it | ✅ One setting |
| Pay when you're not using it | Server keeps running | Can **stop** it (it restarts itself after 7 days — see below) |
| SQL, pgAdmin, SQLAlchemy | Same | **Same** |

> **The product to use is "Flexible Server".** Older guides mention "Single Server" — that version has been retired. Every command below is `az postgres flexible-server ...`.

---

## Choosing a size

| Tier | For | Example size |
|---|---|---|
| **Burstable** | Learning, dev, small apps — cheap, can briefly run faster | `Standard_B1ms` (1 vCPU, 2 GB) |
| General Purpose | Real production apps | `Standard_D2s_v3` (2 vCPU, 8 GB) |
| Memory Optimized | Big databases that need lots of RAM | `Standard_E2s_v3` |

> ⚠️ **If you don't say otherwise, `az` creates a General Purpose server with 128 GB of storage** — several times the cost of what a learning project needs. Always pass `--tier Burstable --sku-name Standard_B1ms --storage-size 32` for dev work.

---

## Creating one — Azure CLI

Install the CLI: `winget install Microsoft.AzureCLI` on Windows, or `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash` in WSL. Then:

```bash
az login                                                   # opens a browser to sign in

az group create --name rg-shop-dev --location uksouth      # a "folder" for everything in this project

az postgres flexible-server create \
  --resource-group rg-shop-dev \
  --name shop-pg-dev \
  --location uksouth \
  --tier Burstable \
  --sku-name Standard_B1ms \
  --storage-size 32 \
  --version 17 \
  --admin-user shopadmin \
  --admin-password "$PG_ADMIN_PASSWORD" \
  --public-access "$(curl -s https://api.ipify.org)"
```

| Flag | Means |
|---|---|
| `--name` | The server's name — becomes **`shop-pg-dev.postgres.database.azure.com`**. Lowercase letters, numbers and hyphens, unique across Azure |
| `--tier` / `--sku-name` | How big and how expensive (table above) |
| `--storage-size` | Disk in GB. Minimum 32. **You can grow it later, never shrink it** |
| `--version` | The Postgres major version — use the newest your region offers |
| `--admin-user` / `--admin-password` | The first user. The username can't be changed later |
| `--public-access` | Which IP addresses may connect. Here: just yours (`api.ipify.org` returns your public IP) |

It takes about 5–10 minutes.

> ⚠️ **Don't type the password into the command.** It ends up in your shell history. Put it in an environment variable first (`export PG_ADMIN_PASSWORD='...'` in WSL, `$env:PG_ADMIN_PASSWORD = '...'` in PowerShell), or leave `--admin-password` out and `az` generates one and prints it once.

> ⚠️ **`--public-access 0.0.0.0` does not mean "only me".** It lets **any** service running inside Azure — including other people's — attempt to connect. Give your own IP, a range, or use private networking (below).

### Or in the Azure Portal

**Create a resource** → search **Azure Database for PostgreSQL** → **Flexible server** → **Create**. On **Compute + storage**, choose **Configure server** and pick **Burstable, B1ms, 32 GiB** — the default is much bigger. On **Networking**, choose **Public access** and **Add current client IP address**.

---

## Letting yourself (and your app) connect — the firewall

Nothing can connect until its IP address is allowed:

```bash
az postgres flexible-server firewall-rule create \
  --resource-group rg-shop-dev \
  --server-name shop-pg-dev \
  --name allow-home \
  --start-ip-address 81.2.69.160 \
  --end-ip-address 81.2.69.160
```

> **Note the two names:** `--server-name` is your server; `--name` is just a label for this rule.

> **Your home IP changes.** If pgAdmin suddenly times out one day, your IP moved — add the new one. Connections that just hang (rather than "password failed") are almost always the firewall.

**For an app running in Azure** (App Service, Container Apps): the proper answer is **private networking** — put the database in a virtual network so it has no public address at all, and only things inside that network can reach it. Choose **Private access (VNet integration)** when creating the server. It must be decided at creation time.

---

## Creating a database and connecting

```bash
az postgres flexible-server db create \
  --resource-group rg-shop-dev \
  --server-name shop-pg-dev \
  --name shop

az postgres flexible-server show-connection-string \
  --server-name shop-pg-dev \
  --database-name shop \
  --admin-user shopadmin
```

`show-connection-string` prints ready-made strings for psql, Python, JDBC and more.

| Connecting from | Settings |
|---|---|
| **psql** | `psql "host=shop-pg-dev.postgres.database.azure.com port=5432 dbname=shop user=shopadmin sslmode=require"` |
| **[[pgAdmin 4]]** | Host `shop-pg-dev.postgres.database.azure.com`, port 5432, user `shopadmin`, **Parameters → SSL mode: require** |
| **Python** | `postgresql+psycopg://shopadmin:PASSWORD@shop-pg-dev.postgres.database.azure.com:5432/shop?sslmode=require` — see [[PostgreSQL with SQLAlchemy]] |

> ⚠️ **Always use `sslmode=require`.** Azure encrypts connections by default and may refuse unencrypted ones.

---

## Don't let your app use the admin account

The admin user can do anything. Create a separate login for the app, with only the rights it needs — connect as `shopadmin` once and run:

```sql
CREATE ROLE shop_app LOGIN PASSWORD 'a-long-random-password';
GRANT CONNECT ON DATABASE shop TO shop_app;
GRANT USAGE ON SCHEMA public TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO shop_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_app;
```

Why each line matters: [[PostgreSQL SQL]] (users and permissions) and [[Security in practice]].

### Better: no password at all — Microsoft Entra ID

Instead of a password stored in your app's settings, an app running in Azure can log in with its **managed identity** — Azure hands it a short-lived token automatically. Nothing to leak, nothing to rotate.

1. Create the server with `--microsoft-entra-auth Enabled` (or turn it on in **Settings → Authentication**)
2. Add yourself as the Entra admin: `az postgres flexible-server microsoft-entra-admin create --server-name ... --display-name ... --object-id ...`
3. Give the app's managed identity a Postgres role, and connect with a token as the password

> **This is the recommended setup for production on Azure** — see [[Azure fundamentals]] for managed identities. For learning, a password is fine.

---

## Extensions — you have to allow them first

On Azure you can't just `CREATE EXTENSION`. The extension must first be on the server's **allow-list**:

```bash
az postgres flexible-server parameter set \
  --resource-group rg-shop-dev \
  --server-name shop-pg-dev \
  --name azure.extensions \
  --value "vector,pg_trgm,citext"
```

Then, connected to your database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;     -- pgvector: AI embeddings
CREATE EXTENSION IF NOT EXISTS pg_trgm;    -- fast ILIKE '%...%'
```

> ⚠️ **"extension is not allow-listed" means you skipped the first step.** The value **replaces** the list — include every extension you want, not just the new one.

Some extensions (like `pg_stat_statements`, for finding slow queries) also need to be in `shared_preload_libraries`, which requires a server restart. In the Portal: **Settings → Server parameters**.

---

## Connection pooling — PgBouncer is built in

Small servers allow a limited number of connections, and apps with several workers use them up fast ([[PostgreSQL with SQLAlchemy]]). Flexible Server includes **PgBouncer**, which lets many app connections share a few real ones:

1. Set the server parameter **`pgbouncer.enabled`** to `true` (Portal: **Server parameters**)
2. Connect your **app** to port **6432** instead of 5432

> Keep pgAdmin and migrations on port **5432** — some admin operations don't work through PgBouncer.

---

## Backups and restoring

- Backups are **automatic**, kept **7 days** by default (up to 35: `--backup-retention 35`)
- You can restore to **any point in time** within that window — a new server is created from it:

```bash
az postgres flexible-server restore \
  --resource-group rg-shop-dev \
  --name shop-pg-restored \
  --source-server shop-pg-dev \
  --restore-time "2026-03-17T10:42:00Z"
```

> **Restore makes a new server; it doesn't overwrite the old one.** Check the restored data, then point your app at it. This is what saves you after an accidental `DELETE` without a `WHERE`.

---

## Saving money

```bash
az postgres flexible-server stop  --resource-group rg-shop-dev --name shop-pg-dev   # stop paying for compute
az postgres flexible-server start --resource-group rg-shop-dev --name shop-pg-dev
```

> ⚠️ **A stopped server starts itself again after 7 days** — and you're billed again from then. For a server you've really finished with, delete it.

> ⚠️ **Storage is billed even while stopped.** Only deleting stops all charges.

**When you're done with the whole project, delete the resource group** — it deletes the server and everything else inside it:

```bash
az group delete --name rg-shop-dev --yes
```

> ⚠️ **Deleting the server deletes its automatic backups too.** Take a `pg_dump` first if you might want the data ([[pgAdmin 4]]).

---

## Common problems

| You see | Cause | Fix |
|---|---|---|
| Connection **times out** | Your IP isn't in the firewall | Add a firewall rule |
| *no pg_hba.conf entry … no encryption* | Connecting without SSL | `sslmode=require` |
| *password authentication failed* | Wrong user or password | Username is exactly as created — e.g. `shopadmin` |
| *extension "vector" is not allow-listed* | Skipped `azure.extensions` | `parameter set --name azure.extensions` |
| *remaining connection slots are reserved* | Too many connections | Smaller pools; PgBouncer on 6432 |
| Unexpectedly high bill | Default General Purpose / 128 GB, or a forgotten server | Burstable + 32 GB; stop or delete when idle |
| App can't reach the server | Server has private access, app isn't in the VNet | VNet integration for the app |

## Related

[[PostgreSQL]] · [[AWS RDS for PostgreSQL]] · [[PostgreSQL with SQLAlchemy]] · [[pgAdmin 4]] · [[PostgreSQL SQL]] · [[Azure fundamentals]] · [[Azure SQL Database]] · [[Pipeline setup - Azure]] · [[Cloud comparison dictionary]] · [[Security in practice]]
