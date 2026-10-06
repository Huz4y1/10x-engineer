---
tags: [azure, data-lake, storage]
status: not-started
---

# Azure Data Lake Storage Gen2

> **What this is:** a giant, cheap hard drive in the cloud, designed so that Spark can read from it fast.
> **Why you care:** it's where raw data lands before anything touches it. Everything in the pipeline starts here.

---

## The idea in plain English

A **data lake** is a folder on the internet where you dump files.

That's genuinely it. The clever part isn't the storage — it's the discipline about *what goes where*, which is the bronze/silver/gold thing below.

Compare it to a database:

| | Data lake (ADLS) | Database (Azure SQL) |
|---|---|---|
| Holds | Files — CSV, JSON, Parquet, images, anything | Rows in defined tables |
| Schema | Decided when you *read* ("schema on read") | Decided when you *write* ("schema on write") |
| Cost per TB | Very cheap (~£15/TB/month) | Much more expensive |
| Good at | Storing everything, forever, cheaply; big scans | Fast lookups of specific rows |
| Bad at | "Give me customer 17850" — has to scan | Storing 40TB of raw JSON |

You use **both**. Lake for raw and bulk processing, database for serving. That's the whole architecture in [[Data Engineering]].

---

## 1. What makes Gen2 different from plain blob storage

Azure Blob Storage is **flat**. Despite what the portal shows you, there are no real folders — a blob is just named `raw/2011/01/sales.csv`, and the slashes are part of the name, like a very long filename.

That's fine for storing photos. It's terrible for analytics, because:

- Renaming a "folder" means renaming every blob inside it, one at a time. For a job writing 10,000 files, the final rename step could take longer than the actual computation.
- Listing a "folder" means scanning names with a prefix filter.
- You can't set permissions on a directory.

**ADLS Gen2** turns on a setting called the **hierarchical namespace (HNS)**, which gives you real directories:

- Renaming a directory is **one atomic operation**, instantly, regardless of contents.
- Directory listing is a real directory listing.
- You get POSIX-style ACLs per directory and file.

> **The one thing you must not forget:** HNS can only be enabled **when you create the storage account**. You cannot turn it on afterwards. Create a storage account without it and you'll be migrating data to a new account later. This is the most common expensive mistake with ADLS.

```bash
az storage account create \
  --name retailstore$RANDOM \
  --resource-group retail-rg \
  --location uksouth \
  --sku Standard_LRS \
  --enable-hierarchical-namespace true    # ← THIS. Not optional. Not retrofittable.
```

---

## 2. The vocabulary

```
Storage Account          retailstore01          ← the "drive"
 └── Container           raw / bronze / silver  ← the top-level folder
      └── Directory      2011/01/               ← real folders (thanks to HNS)
           └── Blob      sales.csv              ← the actual file
```

A **container** is a top-level folder. Two rules: lowercase letters, digits and hyphens only, and it must be created before you can write into it.

### Paths — and why there are three of them

The same file has different addresses depending on who's asking:

```
https://retailstore01.blob.core.windows.net/raw/2011/sales.csv    ← blob API (REST, SDKs)
https://retailstore01.dfs.core.windows.net/raw/2011/sales.csv     ← datalake API (directory ops)
abfss://raw@retailstore01.dfs.core.windows.net/2011/sales.csv     ← what Spark/Databricks uses
```

Break down the `abfss://` one, because you'll type it constantly:

```
abfss://  raw  @  retailstore01  .dfs.core.windows.net  /2011/sales.csv
  │        │        │                    │                    │
protocol  container  storage account   endpoint            path inside container
```

`abfss` = **A**zure **B**lob **F**ile **S**ystem, **S**ecure (TLS). Always use `abfss`, never `abfs`.

> **The `@` is the bit everyone gets wrong.** The container goes *before* the `@`, not in the path after it. `abfss://retailstore01.dfs.core.windows.net/raw/...` is wrong and gives a confusing error.

---

## 3. Bronze / silver / gold — the actual point of a lake

If you just dump files anywhere, in six months you have a swamp: nobody knows which file is current, which is cleaned, or where a number came from.

The fix is a convention. Three layers, each with one job:

| Layer | Contains | Rules |
|---|---|---|
| **Bronze** (raw) | Exactly what arrived, typed but otherwise untouched | **Append only. Never edit. Never delete.** |
| **Silver** (cleaned) | Deduplicated, nulls handled, types fixed, bad rows removed | One row per real-world event |
| **Gold** (business) | Aggregated tables ready for dashboards and models | Named for what a business person would ask |

```
abfss://bronze@retailstore01.dfs.core.windows.net/online_retail/
abfss://silver@retailstore01.dfs.core.windows.net/sales/
abfss://gold@retailstore01.dfs.core.windows.net/daily_sales/
```

**Why bronze must be immutable:** it's your audit trail and your undo button. When someone asks "why does the January revenue number look wrong?", you re-run the pipeline from bronze and find out. If you cleaned the data in place, the evidence is gone and you can never answer the question.

> If you learn one thing from this note: **never overwrite bronze.** Storage is cheap. Being unable to reproduce a number is expensive.

### Partitioning — how to lay out directories

Don't put 500,000 files in one directory. Split by something you'll filter on — usually date:

```
bronze/online_retail/year=2011/month=01/part-0001.parquet
bronze/online_retail/year=2011/month=02/part-0001.parquet
```

That `key=value` naming is **Hive-style partitioning**, and Spark understands it natively. When you write `WHERE year = 2011 AND month = 1`, Spark skips every other directory without opening a single file. That's called **partition pruning**, and it's the difference between a 4-second query and a 4-minute one.

> **Getting the size right:** aim for files of roughly **128MB–1GB**. Thousands of tiny files ("the small file problem") is the classic data lake performance killer — every file has fixed overhead to open, so 100,000 × 1MB files are dramatically slower than 800 × 128MB files, despite being the same data. [[Databricks and Delta Lake]] covers `OPTIMIZE`, which fixes this after the fact.

### File formats: use Parquet, not CSV

| | CSV | Parquet |
|---|---|---|
| Layout | Row by row, text | **Column by column**, binary |
| Types | Everything's a string | Real types, preserved |
| Size | Baseline | ~5–10× smaller (compressed) |
| Reading 2 of 50 columns | Reads all 50 | **Reads only those 2** |
| Human-readable | Yes | No |

That "reads only those 2" is why Parquet exists. Because it's stored column by column, a query touching 2 columns physically reads 4% of the bytes. On a 200GB table that's the whole ballgame.

> **Rule of thumb:** CSV only at the very edge, where an external system hands you one. The instant it's in bronze, it's Parquet (or Delta, which is Parquet plus a transaction log).

---

## 4. Access tiers — the cost dial

| Tier | Storage cost | Read cost | Use for |
|---|---|---|---|
| **Hot** | Highest | Lowest | Data you touch regularly |
| **Cool** | ~50% less | Higher | Accessed less than monthly. **30-day minimum.** |
| **Cold** | Lower still | Higher still | Rarely accessed. 90-day minimum. |
| **Archive** | ~90% less | **Hours to retrieve** | Compliance. 180-day minimum. |

The trap: those minimums are real. Delete a blob from Cool after 3 days and you're charged for 30. Moving data around to save pennies can cost more than leaving it.

> **For this roadmap: leave everything Hot.** Your dataset is under a gigabyte. Tiering matters at terabyte scale; below that it's noise. Set a **lifecycle management policy** if you want to see how it works — "move bronze to Cool after 90 days" — but don't expect savings on a learning project.

---

## 5. Access control — four ways in, ranked

| Method | How it works | Verdict |
|---|---|---|
| **Managed identity + RBAC** | The app has an Azure identity with a data role | ✅ **Use this.** No secret exists. |
| **Service principal + RBAC** | App ID + secret in Key Vault | ✅ Fine when the app runs outside Azure |
| **SAS token** | A time-limited URL granting specific access | ⚠️ OK for handing a file to an outsider |
| **Account key** | One master password for the whole account | ❌ Avoid. Full control, can't be scoped, rarely rotated. |

The account key is what every tutorial uses, because it's one line. It's also the thing that ends up committed to GitHub and gives a stranger delete access to everything. Learn the pattern once and never use it:

```python
from azure.identity import DefaultAzureCredential
from azure.storage.filedatalake import DataLakeServiceClient

service = DataLakeServiceClient(
    account_url="https://retailstore01.dfs.core.windows.net",
    credential=DefaultAzureCredential(),   # no secret anywhere
)

fs = service.get_file_system_client("bronze")
with open("online_retail_II.csv", "rb") as f:
    fs.get_file_client("online_retail/raw.csv").upload_data(f, overwrite=True)
```

Remember the **data-plane role** requirement from [[Azure fundamentals]]: you need `Storage Blob Data Contributor`. Being `Owner` is not enough, and the error message won't tell you that.

### ACLs — the second permission layer

With HNS on, you *also* get POSIX ACLs per directory and file (read/write/execute for owner/group/other), on top of RBAC. Effective access is RBAC **plus** ACLs.

For this roadmap, RBAC at the container level is enough. Just know ACLs exist, because when a permission behaves oddly in a corporate environment, this is usually why.

---

## 6. Connecting Databricks to ADLS

Three approaches, in order of preference:

1. **Unity Catalog external location** (best — set up once, everyone uses plain paths). Covered in [[Unity Catalog and orchestration]].
2. **Service principal + secret scope** — works everywhere, some config.
3. **Account key in a notebook** — never do this.

Option 2 looks like:

```python
spark.conf.set("fs.azure.account.auth.type.retailstore01.dfs.core.windows.net", "OAuth")
spark.conf.set("fs.azure.account.oauth.provider.type.retailstore01.dfs.core.windows.net",
               "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider")
spark.conf.set("fs.azure.account.oauth2.client.id.retailstore01.dfs.core.windows.net",
               dbutils.secrets.get("retail-scope", "sp-client-id"))
spark.conf.set("fs.azure.account.oauth2.client.secret.retailstore01.dfs.core.windows.net",
               dbutils.secrets.get("retail-scope", "sp-secret"))   # ← from a secret scope, never inline
spark.conf.set("fs.azure.account.oauth2.client.endpoint.retailstore01.dfs.core.windows.net",
               "https://login.microsoftonline.com/<tenant-id>/oauth2/token")

df = spark.read.parquet("abfss://bronze@retailstore01.dfs.core.windows.net/online_retail/")
```

Verbose, but it's copy-paste once per workspace. `dbutils.secrets.get` reads from a Databricks secret scope, which can be backed by Key Vault — so the secret is never in the notebook.

---

## Useful commands

```bash
# create containers
az storage fs create -n bronze --account-name retailstore01 --auth-mode login
az storage fs create -n silver --account-name retailstore01 --auth-mode login
az storage fs create -n gold   --account-name retailstore01 --auth-mode login

# upload
az storage fs file upload \
  --account-name retailstore01 --file-system bronze \
  --source ./online_retail_II.csv --path online_retail/raw.csv --auth-mode login

# list
az storage fs file list --account-name retailstore01 -f bronze --auth-mode login -o table

# how big is it
az storage fs file list --account-name retailstore01 -f bronze --auth-mode login \
  --query "sum(@[].contentLength)"
```

> `--auth-mode login` tells the CLI to use your `az login` identity instead of hunting for an account key. Without it you'll get key-related errors even when your RBAC is perfect. Add it to every `az storage` command by reflex.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 AuthorizationPermissionMismatch` | Missing **data-plane** role | Assign `Storage Blob Data Contributor` |
| `az storage` asks for a key despite being logged in | Missing `--auth-mode login` | Add it |
| Spark: "Configuration property … not found" | The `fs.azure.*` config doesn't match the account name exactly | The account name appears in every property key — check for typos |
| `abfss` path "container not found" | Container put after the `@` instead of before | `abfss://container@account.dfs.core.windows.net/path` |
| Directory rename is slow | HNS isn't enabled — it's a flat blob account | Create a new account with HNS; you can't retrofit it |
| Spark job crawls on lots of small files | Small file problem | Repartition before writing; `OPTIMIZE` on Delta tables |
| Query reads everything despite a date filter | Not partitioned, or filtering on a non-partition column | Partition by `year=`/`month=`; filter on those columns |
| Unexpected storage bill | Old versions/snapshots piling up, or archive early-deletion fees | Check lifecycle policy; look at soft-delete retention |
| Can't delete a container | Soft delete is on | Wait out the retention period, or disable soft delete |

---

## Practice checklist

- [ ] Hierarchical namespace — what makes Gen2 different from plain blob storage, and why it can't be enabled later
- [ ] Containers vs. blobs vs. directories
- [ ] `abfss://` path anatomy — container before the `@`
- [ ] Bronze / silver / gold, and why bronze is immutable
- [ ] Parquet vs. CSV, and why columnar storage wins
- [ ] Partitioning with `key=value` directories, and the small file problem
- [ ] Access tiers (hot/cool/archive) and when each is worth it
- [ ] Access control: account key vs. SAS vs. service principal vs. managed identity — know which you're using and why

## Hands-on

- [ ] Create a storage account with hierarchical namespace enabled
- [ ] Set up bronze/silver/gold containers for the capstone
- [ ] Ingest the Online Retail II dataset via the Azure CLI or Python SDK using `DefaultAzureCredential` (not the portal upload button, not an account key)
- [ ] Convert that CSV to Parquet and compare the file sizes — see the compression for yourself
- [ ] Write a partitioned dataset (`year=`/`month=`) and confirm a filtered read only touches the right directories

## Resources

- [ADLS Gen2 introduction](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
- [Azure Storage Python SDK](https://learn.microsoft.com/azure/storage/blobs/storage-quickstart-blobs-python)
- [Apache Parquet: file format explained](https://parquet.apache.org/docs/file-format/)

## Next

[[Azure SQL Database]]
