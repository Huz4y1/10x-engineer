---
tags: [seaweedfs, s3, storage, wsl, guide, data-engineering]
---

# Using SeaweedFS

**The hands-on guide.** What buckets are, running SeaweedFS on WSL, every command you'll actually type, and saving your PySpark DataFrames into it.

Concepts and internals: [[SeaweedFS]] · Spark API: [[PySpark reference]] · Setup: [[Setting up a dev machine]]

---

## What a bucket actually is

**A bucket is a named container for files. That's genuinely it.**

Think of it as a top-level folder that lives on a storage server instead of your laptop:

```
seaweedfs (the storage server)
├── bronze/                 <- a bucket
│   ├── taxi/2026-01-01.parquet
│   └── taxi/2026-01-02.parquet
├── silver/                 <- another bucket
│   └── taxi_clean/
└── gold/                   <- another bucket
    └── daily_revenue/
```

**Why not just use folders on your disk?** Because object storage gives you things a folder can't:

| | A folder on your laptop | A bucket |
|---|---|---|
| Reachable from | That one machine | Anything on the network — Spark, an API, a container |
| Size limit | Your disk | Add more machines |
| Access control | File permissions | Per-bucket keys and policies |
| How you address it | `C:\data\file.parquet` | `s3://bronze/file.parquet` |
| Survives the machine dying | ❌ | ✅ (with replication) |

**The vocabulary, in one table:**

| Term | Means | Example |
|---|---|---|
| **Bucket** | The top-level container | `bronze` |
| **Key** | The full path of an object inside it | `taxi/2026-01-01.parquet` |
| **Object** | The file itself | The actual bytes |
| **Prefix** | The start of a key — acts like a folder | `taxi/` |
| **URI** | How you address it | `s3://bronze/taxi/2026-01-01.parquet` |

> ⚠️ **There are no real folders in object storage.** `taxi/2026-01-01.parquet` is one long key that happens to contain slashes. Tools *display* it as folders, but nothing is nested — which is why you can't have an empty "folder", and why deleting a "folder" means deleting every key with that prefix.

**Why SeaweedFS specifically:** it speaks the **S3 API**, the same protocol as AWS S3, Azure (via a gateway) and Google Cloud Storage. So you write your code once against `s3a://`, run it locally against SeaweedFS for free, and the exact same code runs against real S3 in production. That's the whole point — see [[Cloud comparison dictionary]].

---

## Setting it up on WSL

### Step 1 — Docker

SeaweedFS runs in Docker. You need Docker reachable from inside WSL:

```bash
docker --version           # if this works, you're done
```

If it doesn't: install **Docker Desktop on Windows**, then Settings → Resources → WSL Integration → enable your distro. That's the path of least resistance — Docker Desktop runs the engine on Windows and exposes it inside WSL.

> **Alternative, no Docker Desktop:** install Docker Engine directly inside WSL (`curl -fsSL https://get.docker.com | sh`), then `sudo service docker start` each time you open WSL. Lighter, but you start it manually.

### Step 2 — A folder for the project

```bash
cd ~                                   # your WSL HOME - not /mnt/c
mkdir -p ~/lakehouse && cd ~/lakehouse
```

> ⚠️ **Work in `~/`, never `/mnt/c/`.** Docker volumes and Spark writes on the Windows drive are dramatically slower through the WSL filesystem bridge, and file permissions behave oddly. See [[Setting up a dev machine]].

### Step 3 — Credentials

```bash
cat > s3.json << 'EOF'
{
  "identities": [
    {
      "name": "local",
      "credentials": [
        { "accessKey": "localkey", "secretKey": "localsecret" }
      ],
      "actions": ["Admin", "Read", "Write", "List", "Tagging"]
    }
  ]
}
EOF
```

> ⚠️ **Without this file the S3 endpoint is wide open** — anyone who can reach the port can read and write everything. Fine on a laptop; never expose that port beyond it ([[Security in practice]]).

### Step 4 — Start it

```bash
docker run -d --name seaweedfs \
  -p 9333:9333 -p 8080:8080 -p 8888:8888 -p 8333:8333 \
  -v ~/lakehouse/seaweed-data:/data \
  -v ~/lakehouse/s3.json:/etc/seaweedfs/s3.json:ro \
  chrislusf/seaweedfs:latest \
  server -dir=/data -s3 -s3.port=8333 -s3.config=/etc/seaweedfs/s3.json \
         -filer -master.volumeSizeLimitMB=1024
```

**What each part does:**

| Part | Does |
|---|---|
| `-d` | Run in the background (detached) |
| `--name seaweedfs` | Name it, so you can `docker stop seaweedfs` later |
| `-p 8333:8333` | **The S3 API — the port you actually use** |
| `-p 9333:9333` | Master UI, for checking cluster health |
| `-p 8888:8888` | Filer, browse files in a browser |
| `-v ~/lakehouse/seaweed-data:/data` | **Your data lives here, so it survives a container restart** |
| `server ... -s3 -filer` | Run master, volume, filer and S3 gateway in one process |

### Step 5 — Check it's alive

```bash
docker ps                                        # is the container running?
docker logs seaweedfs | tail -20                 # what did it say on startup
curl http://localhost:9333/cluster/status        # master health - should return JSON
curl http://localhost:8333                       # S3 endpoint - empty reply is normal
```

Open **http://localhost:8888** in Windows — WSL forwards the port automatically, and you get a browsable file view.

### Everyday container commands

```bash
docker stop seaweedfs             # stop it (data is kept)
docker start seaweedfs            # start it again
docker restart seaweedfs
docker logs -f seaweedfs          # follow the logs live
docker rm -f seaweedfs            # delete the CONTAINER - data in ~/lakehouse survives
```

> **Your data is in `~/lakehouse/seaweed-data`, not in the container.** Removing the container is safe; deleting that folder is not.

---

## The AWS CLI — your main tool

SeaweedFS speaks S3, so you drive it with the standard AWS CLI.

```bash
sudo apt install awscli            # or: uv tool install awscli
aws --version
```

**Configure it once:**

```bash
aws configure set aws_access_key_id localkey
aws configure set aws_secret_access_key localsecret
aws configure set region us-east-1                # SeaweedFS ignores it, the CLI demands it
```

**Save yourself typing the endpoint every time:**

```bash
echo "alias s3l='aws --endpoint-url http://localhost:8333 s3'" >> ~/.bashrc
source ~/.bashrc

s3l ls                             # now this works
```

> Every command below shows the full form. Substitute `s3l` once you've made the alias.

### Buckets

```bash
aws --endpoint-url http://localhost:8333 s3 mb s3://bronze      # mb = make bucket
aws --endpoint-url http://localhost:8333 s3 mb s3://silver
aws --endpoint-url http://localhost:8333 s3 mb s3://gold

aws --endpoint-url http://localhost:8333 s3 ls                  # list all buckets
aws --endpoint-url http://localhost:8333 s3 rb s3://gold        # rb = remove bucket (must be empty)
aws --endpoint-url http://localhost:8333 s3 rb s3://gold --force   # delete contents too
```

> **Why three buckets — bronze, silver, gold?** It's the **medallion pattern**: raw as it arrived → cleaned and typed → aggregated and ready to use. Separate buckets mean you can always rebuild silver and gold from bronze when you find a bug ([[Databricks and Delta Lake]]).

### Putting data in

```bash
# one file
aws --endpoint-url http://localhost:8333 s3 cp taxi.csv s3://bronze/raw/taxi.csv

# a whole folder
aws --endpoint-url http://localhost:8333 s3 cp ./data s3://bronze/raw/ --recursive

# sync - only uploads what changed. Best for repeat runs.
aws --endpoint-url http://localhost:8333 s3 sync ./data s3://bronze/raw/
```

### Pulling data out

```bash
# one file down
aws --endpoint-url http://localhost:8333 s3 cp s3://bronze/raw/taxi.csv ./taxi.csv

# a whole prefix down
aws --endpoint-url http://localhost:8333 s3 cp s3://silver/taxi_clean/ ./out/ --recursive

# sync down
aws --endpoint-url http://localhost:8333 s3 sync s3://silver/taxi_clean/ ./out/

# read a file WITHOUT saving it - "-" means stdout
aws --endpoint-url http://localhost:8333 s3 cp s3://bronze/raw/taxi.csv - | head -5
```

### Looking around

```bash
aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/              # top level
aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/raw/          # inside a prefix
aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/ --recursive  # everything
aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/ --recursive --human-readable --summarize
#                                                            ^ sizes in MB/GB, plus a total
```

### Moving and deleting

```bash
aws --endpoint-url http://localhost:8333 s3 mv s3://bronze/a.csv s3://bronze/archive/a.csv
aws --endpoint-url http://localhost:8333 s3 rm s3://bronze/raw/taxi.csv         # one object
aws --endpoint-url http://localhost:8333 s3 rm s3://bronze/raw/ --recursive     # a whole prefix
```

> ⚠️ **`rm --recursive` has no confirmation and no undo.** There's no recycle bin. **Always `ls` the exact same path first** to see what you're about to destroy:
> ```bash
> aws --endpoint-url http://localhost:8333 s3 ls s3://bronze/raw/ --recursive   # look
> aws --endpoint-url http://localhost:8333 s3 rm s3://bronze/raw/ --recursive   # then delete
> ```

### The command table

| Command | Does |
|---|---|
| `s3 mb s3://x` | Make bucket |
| `s3 rb s3://x` | Remove bucket (add `--force` to delete contents) |
| `s3 ls` | List buckets |
| `s3 ls s3://x/p/` | List inside a prefix |
| `s3 cp a b` | Copy — either direction, add `--recursive` for folders |
| `s3 sync a b` | Copy only what changed |
| `s3 mv a b` | Move |
| `s3 rm s3://x/k` | Delete an object |
| `s3 presign s3://x/k` | Make a temporary shareable URL |

---

## Saving your PySpark DataFrames into buckets

This is the part you'll use every day.

### The Spark session that talks to SeaweedFS

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = (SparkSession.builder
    .appName("lakehouse")
    .master("local[*]")
    .config("spark.jars.packages",
            "org.apache.hadoop:hadoop-aws:3.3.4,"          # the s3a:// filesystem
            "com.amazonaws:aws-java-sdk-bundle:1.12.262")  # its AWS dependency
    # ---- point Spark's S3 client at SeaweedFS instead of AWS ----
    .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:8333")
    .config("spark.hadoop.fs.s3a.access.key", "localkey")
    .config("spark.hadoop.fs.s3a.secret.key", "localsecret")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")        # REQUIRED - see below
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")  # local, no TLS
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem")
    .config("spark.sql.shuffle.partitions", "4")                    # 200 is absurd on a laptop
    .getOrCreate())
```

> ⚠️ **`path.style.access=true` is mandatory.** Real S3 addresses buckets as `bucket.host/key`; SeaweedFS uses `host/bucket/key`. Without this setting Spark tries to resolve `bronze.localhost` and you get a DNS error that looks nothing like a config problem.

> **The `spark.jars.packages` line downloads jars on first run** — it needs internet once, then they're cached in `~/.ivy2`. If it hangs, that's what it's doing.

### Reading from a bucket

```python
df = spark.read.parquet("s3a://bronze/taxi/")                  # note s3a://, not s3://
df = spark.read.option("header", True).csv("s3a://bronze/raw/taxi.csv")
df = spark.read.format("delta").load("s3a://silver/taxi_clean/")
```

> ⚠️ **Use `s3a://` in Spark, `s3://` in the AWS CLI.** They point at the same place. `s3a` is the Hadoop filesystem connector's scheme; using `s3://` in Spark gives you *No FileSystem for scheme*.

### Writing to a bucket

```python
# the basic write
df.write.mode("overwrite").parquet("s3a://silver/taxi_clean/")

# partitioned - writes into date=2026-01-01/ folders so readers can skip files
(df.write
   .mode("overwrite")
   .partitionBy("pickup_date")
   .parquet("s3a://silver/taxi_clean/"))

# CSV, if a human needs to open it
df.write.mode("overwrite").option("header", True).csv("s3a://gold/report/")
```

| Mode | Does |
|---|---|
| `"overwrite"` | **Deletes what's there**, then writes |
| `"append"` | Adds to it |
| `"error"` *(default)* | Fails if the path exists |
| `"ignore"` | Does nothing if the path exists |

> ⚠️ **`overwrite` deletes the whole target path first.** Re-running a job that writes to `s3a://silver/taxi_clean/` destroys every partition, not just today's. See the idempotent pattern below.

### The full worked pipeline

Raw CSV in bronze → cleaned Parquet in silver → aggregated in gold:

```python
from pyspark.sql import functions as F

# ---- 1. READ the raw data out of bronze ----
raw = (spark.read
    .option("header", True)
    .option("inferSchema", True)          # fine for exploring; declare a schema in production
    .csv("s3a://bronze/raw/taxi.csv"))

print(f"read {raw.count():,} rows")
raw.printSchema()

# ---- 2. PREPROCESS ----
clean = (raw
    .drop(                                             # remove columns you don't need
        "RatecodeID",
        "store_and_fwd_flag",
        "extra",
        "mta_tax",
        "improvement_surcharge",
    )
    .filter(F.col("trip_distance") > 0)                # drop impossible trips
    .filter(F.col("total_amount") > 0)                 # drop refunds and errors
    .filter(F.col("passenger_count").isNotNull())      # we can't use rows without it
    .withColumn("pickup_date", F.to_date("tpep_pickup_datetime"))       # for partitioning
    .withColumn("trip_minutes",                                          # derive duration
        (F.unix_timestamp("tpep_dropoff_datetime")
       - F.unix_timestamp("tpep_pickup_datetime")) / 60)
    .withColumn("price_per_mile",
        F.round(F.col("total_amount") / F.col("trip_distance"), 2))
    .filter(F.col("trip_minutes").between(1, 240))     # 1 min to 4 hrs is plausible
    .dropDuplicates())

print(f"kept {clean.count():,} rows after cleaning")

# ---- 3. SAVE the cleaned data to silver ----
(clean.write
   .mode("overwrite")
   .partitionBy("pickup_date")
   .parquet("s3a://silver/taxi_clean/"))

# ---- 4. AGGREGATE and save to gold ----
daily = (clean
    .groupBy("pickup_date")
    .agg(
        F.count("*").alias("trips"),
        F.round(F.avg("trip_distance"), 2).alias("avg_miles"),
        F.round(F.sum("total_amount"), 2).alias("revenue"),
    )
    .orderBy("pickup_date"))

daily.write.mode("overwrite").parquet("s3a://gold/daily_summary/")
daily.show()
```

### Making it safe to re-run

```python
from pyspark.sql import functions as F

# overwrite ONLY one day's partition, leave the rest alone
(clean.filter(F.col("pickup_date") == run_date)
      .write.mode("overwrite")
      .option("partitionOverwriteMode", "dynamic")     # Parquet
      .partitionBy("pickup_date")
      .parquet("s3a://silver/taxi_clean/"))
```

With Delta it's cleaner:

```python
(clean.write.format("delta").mode("overwrite")
   .option("replaceWhere", f"pickup_date = '{run_date}'")   # replace just this day
   .save("s3a://silver/taxi_clean/"))
```

> **A job you can safely re-run is a job you can recover from.** Without this, a retry after a half-finished run either duplicates data or wipes history ([[PySpark reference]]).

### Checking what actually landed

```bash
aws --endpoint-url http://localhost:8333 s3 ls s3://silver/taxi_clean/ --recursive --human-readable --summarize
```

```python
back = spark.read.parquet("s3a://silver/taxi_clean/")
print(back.count())
back.show(2, vertical=True)
```

> **Always read it back once.** A write that "succeeded" but produced zero rows, or landed in the wrong prefix, is a genuinely common and quiet failure.

### Controlling the number of files

```python
df.coalesce(1).write.parquet("s3a://gold/report/")     # ONE file - small results only
df.repartition(4).write.parquet("s3a://silver/x/")     # 4 files, evenly sized
```

> ⚠️ **Spark writes a *folder* of `part-0000...` files, not a single file.** That's normal and correct. Use `coalesce(1)` only when the result is genuinely small and a human needs one file.

### Reading it with pandas instead

For small results you don't need Spark at all:

```python
import pandas as pd

storage = {"key": "localkey", "secret": "localsecret",
           "client_kwargs": {"endpoint_url": "http://localhost:8333"}}

df = pd.read_parquet("s3://gold/daily_summary/", storage_options=storage)   # needs s3fs
```

```bash
uv add s3fs pyarrow
```

> Note pandas uses `s3://` here, while Spark uses `s3a://`. Different libraries, different scheme names, same storage ([[pandas]]).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No FileSystem for scheme "s3"` | Used `s3://` in Spark | Use `s3a://` |
| DNS error mentioning `bronze.localhost` | Virtual-host addressing | `path.style.access=true` |
| `Connection refused` | Container not running | `docker ps`, then `docker start seaweedfs` |
| `403 Forbidden` | Wrong keys, or `s3.json` not mounted | Check the keys match `s3.json` |
| `ClassNotFoundException: S3AFileSystem` | Missing jars | Add `hadoop-aws` + `aws-java-sdk-bundle` |
| Jar versions clash | `hadoop-aws` must match Spark's Hadoop | `spark.sparkContext._jvm.org.apache.hadoop.util.VersionInfo.getVersion()` |
| `NoSuchBucket` | Bucket not created | `s3 mb s3://bronze` |
| Write succeeded, no data | Wrong prefix, or the DataFrame was empty | `df.count()` before writing |
| Everything is very slow | Project on `/mnt/c/` | Move it to `~/` |
| Data gone after restart | No `-v` volume mount | Mount `~/lakehouse/seaweed-data:/data` |
| First run hangs on startup | Downloading jars | Wait; they cache in `~/.ivy2` |

```bash
docker logs seaweedfs | tail -50            # the first place to look
curl http://localhost:9333/cluster/status   # is the master healthy?
```

---

## The daily cheat sheet

```bash
docker start seaweedfs                                         # start of day

s3l ls                                                         # what buckets exist
s3l ls s3://bronze/ --recursive --human-readable --summarize   # what's in one
s3l cp local.csv s3://bronze/raw/                              # push data up
s3l cp s3://gold/report/ ./out/ --recursive                    # pull results down
s3l rm s3://silver/taxi_clean/ --recursive                     # clear before a rebuild

docker stop seaweedfs                                          # end of day
```

```python
from pyspark.sql import functions as F

# read -> preprocess -> write, the shape of every job
df = spark.read.parquet("s3a://bronze/taxi/")
clean = df.drop("junk_col").filter(F.col("amount") > 0)
clean.write.mode("overwrite").partitionBy("date").parquet("s3a://silver/taxi_clean/")
```

---

## Related

[[SeaweedFS]] · [[PySpark reference]] · [[Databricks and Delta Lake]] · [[Running the whole stack locally]] · [[Cloud comparison dictionary]] · [[Setting up a dev machine]] · [[Docker deep dive]] · [[Data modeling]] · [[Security in practice]] · [[Pipeline setup - Local]] · [[07 — DATA ENGINEERING]]
