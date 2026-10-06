---
tags: [aws, s3, object-storage, cloud, guide]
---

# AWS S3

**S3 is Amazon's file storage for the cloud.** You put files ("objects") into containers ("buckets"), and any program with permission can read them — from anywhere, at any size. It's where data lakes, backups, model files, images and logs usually live on AWS.

Hub: [[Object storage]] · Same thing on Azure: [[ADLS Gen2]] · Same thing on your laptop: [[SeaweedFS]] · AWS database: [[AWS RDS for PostgreSQL]]

> ✅ **The Python examples were run against `moto`, a local imitation of AWS**, so they're tested without a real account. The CLI commands and flags were checked against the AWS CLI's own help.

---

## The idea, in simple words

S3 isn't a hard drive and it isn't a database. It's closer to **a giant web server for files**: every file has an address, and you put or get whole files by that address.

```
s3://my-shop-data-lake/bronze/orders/2026-03-17.parquet
     └───────┬───────┘ └───────────────┬──────────────┘
          bucket                  key (the file's full name)
```

| Word | Means |
|---|---|
| **Bucket** | A top-level container. **The name is unique across every AWS account in the world** |
| **Object** | A file, plus some information about it (size, type, date) |
| **Key** | The object's full name inside the bucket — `bronze/orders/2026-03-17.parquet` |
| **Prefix** | The start of a key — `bronze/orders/` — which tools *show* as a folder |
| **Region** | Where the bucket physically lives, e.g. `eu-west-2` (London) |

> ⚠️ **There are no real folders.** `bronze/orders/file.parquet` is one long name with slashes in it. That's why "renaming a folder" means copying every object to a new key, and why an empty folder can't exist. Full explanation in [[Using SeaweedFS]] — SeaweedFS copies S3's design exactly.

**What goes where:**

| Store | Put here |
|---|---|
| **S3** | Files: Parquet, CSV, images, PDFs, model weights, backups, logs |
| **A database** ([[AWS RDS for PostgreSQL]]) | Rows your app looks up and updates one at a time |

> **A common, good pattern:** the file lives in S3, the database row stores its key. `users.avatar_key = 'avatars/42.png'`.

---

## The AWS CLI

```bash
aws configure                                   # once: access keys, default region, output format
```

### Buckets

```bash
aws s3 mb s3://my-shop-data-lake --region eu-west-2     # make bucket
aws s3 ls                                               # list your buckets
aws s3 rb s3://my-shop-data-lake                        # remove bucket (must be empty)
aws s3 rb s3://my-shop-data-lake --force                # delete everything in it, then the bucket
```

> **Bucket names:** 3–63 characters, lowercase letters, numbers and hyphens, **unique worldwide**. `data` is taken; `huz-shop-data-lake-2026` probably isn't.

### Files

```bash
aws s3 cp orders.csv s3://my-shop-data-lake/raw/orders.csv              # upload one file
aws s3 cp s3://my-shop-data-lake/raw/orders.csv ./orders.csv            # download one file
aws s3 cp ./exports s3://my-shop-data-lake/raw/ --recursive             # a whole folder
aws s3 sync ./exports s3://my-shop-data-lake/raw/                       # only what changed - best for repeat uploads
aws s3 ls s3://my-shop-data-lake/raw/ --recursive --human-readable --summarize   # what's there, with sizes
aws s3 mv s3://my-shop-data-lake/raw/a.csv s3://my-shop-data-lake/archive/a.csv
aws s3 rm s3://my-shop-data-lake/raw/a.csv                             # delete one object
aws s3 rm s3://my-shop-data-lake/raw/ --recursive                      # delete a whole prefix
```

> ⚠️ **`rm --recursive` has no confirmation and no undo** (unless versioning is on — below). Run the same path with `ls --recursive` first to see exactly what it will delete.

> ⚠️ **`sync --delete` also deletes files in the destination that aren't in the source.** Great for mirroring, dangerous if you point it at the wrong folder.

### Sharing a file for a limited time

```bash
aws s3 presign s3://my-shop-data-lake/reports/march.pdf --expires-in 3600     # a link that works for 1 hour
```

> **A presigned URL lets someone download one object without an AWS account** — and it stops working after the time you set. Use this instead of making anything public.

---

## Python — boto3

```bash
uv add boto3
```

```python
import boto3

s3 = boto3.client("s3", region_name="eu-west-2")         # uses your `aws configure` credentials

s3.create_bucket(
    Bucket="my-shop-data-lake",
    CreateBucketConfiguration={"LocationConstraint": "eu-west-2"},   # needed for every region except us-east-1
)
```

> ⚠️ **`create_bucket` needs `CreateBucketConfiguration` outside `us-east-1`**, or it fails with *IllegalLocationConstraintException*. The CLI's `aws s3 mb --region` handles this for you; boto3 doesn't.

### Upload and download

```python
import pathlib

pathlib.Path("orders.csv").write_text("order_id,amount\n1,19.99\n2,5.00\n", encoding="utf-8")

s3.upload_file("orders.csv", "my-shop-data-lake", "raw/orders.csv")         # local file -> S3
s3.download_file("my-shop-data-lake", "raw/orders.csv", "orders_copy.csv")  # S3 -> local file

s3.put_object(Bucket="my-shop-data-lake", Key="raw/hello.txt", Body=b"hello")   # bytes straight up
body = s3.get_object(Bucket="my-shop-data-lake", Key="raw/hello.txt")["Body"].read()
print(body)
```

```
b'hello'
```

> **`upload_file` / `download_file` handle big files for you** — they split them into parts and upload in parallel. Use them for files; use `put_object` / `get_object` for small bits of data already in memory.

### Listing — beware the 1,000 limit

```python
for i in range(1500):
    s3.put_object(Bucket="my-shop-data-lake", Key=f"logs/{i:05d}.txt", Body=b"x")

one_call = s3.list_objects_v2(Bucket="my-shop-data-lake", Prefix="logs/")
print("one call returned:", one_call["KeyCount"])            # capped at 1,000

paginator = s3.get_paginator("list_objects_v2")
total = sum(len(page.get("Contents", []))
            for page in paginator.paginate(Bucket="my-shop-data-lake", Prefix="logs/"))
print("with a paginator:", total)
```

```
one call returned: 1000
with a paginator: 1500
```

> ⚠️ **`list_objects_v2` returns at most 1,000 objects per call.** Code that uses one call works in testing and silently misses files in production. **Always use the paginator.**

### Does this object exist?

```python
from botocore.exceptions import ClientError

def exists(bucket: str, key: str) -> bool:
    try:
        s3.head_object(Bucket=bucket, Key=key)          # fetches the details, not the file
        return True
    except ClientError as e:
        if e.response["Error"]["Code"] == "404":
            return False
        raise                                            # anything else (e.g. no permission) is a real error

print(exists("my-shop-data-lake", "raw/orders.csv"), exists("my-shop-data-lake", "raw/nope.csv"))
```

```
True False
```

### A temporary download link

```python
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-shop-data-lake", "Key": "raw/orders.csv"},
    ExpiresIn=3600,                                          # seconds
)
print(url.startswith("https://"))
```

---

## pandas and Parquet straight from S3

```bash
uv add pandas pyarrow s3fs
```

```python
import pandas as pd

df = pd.DataFrame({"order_id": [1, 2, 3], "amount": [19.99, 5.00, 42.50]})
df.to_parquet("s3://my-shop-data-lake/silver/orders.parquet", index=False)    # write

back = pd.read_parquet("s3://my-shop-data-lake/silver/orders.parquet")        # read
print(back)
```

```
   order_id  amount
0         1   19.99
1         2    5.00
2         3   42.50
```

> **`s3fs` is what makes `s3://` paths work in pandas.** It uses the same credentials as the CLI. For [[SeaweedFS]] or MinIO, pass the endpoint: `storage_options={"client_kwargs": {"endpoint_url": "http://localhost:8333"}}` ([[Using SeaweedFS]]).

**PySpark** reads S3 with `s3a://` paths and the `hadoop-aws` connector — the same setup as in [[Using SeaweedFS]], minus the custom endpoint. On AWS EMR and Databricks it's configured for you ([[PySpark reference]]).

---

## Security — the part that goes wrong in the news

### Keep buckets private

New buckets have **Block Public Access** turned on. **Leave it on.** Leaked S3 buckets full of customer data are one of the most common breaches, and nearly always because someone made a bucket public "just for a moment".

```bash
aws s3api get-public-access-block --bucket my-shop-data-lake     # all four settings should be true
```

> **To share a file, use a presigned URL. To serve a website's images, put CloudFront in front.** Neither needs a public bucket.

### Give programs only the access they need

Don't give your app your own admin keys. Give its role a policy listing exactly what it may do:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-shop-data-lake/silver/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-shop-data-lake",
      "Condition": {"StringLike": {"s3:prefix": ["silver/*"]}}
    }
  ]
}
```

> ⚠️ **Note the two different `Resource` forms.** Reading and writing objects uses `bucket/*` (the objects). **Listing** uses the bucket itself — `bucket` without `/*`. Getting this wrong is the classic *Access Denied* when listing.

> **On AWS compute (EC2, ECS, Lambda), use an IAM role — never access keys in code.** boto3 picks the role's credentials up automatically ([[Security in practice]]).

### Encryption

Everything new is **encrypted at rest automatically** (SSE-S3). For stricter control you can use your own KMS key per bucket.

---

## Versioning — an undo button

```bash
aws s3api put-bucket-versioning --bucket my-shop-data-lake --versioning-configuration Status=Enabled
```

With versioning on, **overwriting or deleting an object keeps the old version**. A delete just adds a "delete marker" — remove the marker and the file is back.

> ⚠️ **Old versions are stored — and billed — forever** unless a lifecycle rule removes them. Always pair versioning with a rule like "delete non-current versions after 30 days".

---

## Storage classes and lifecycle rules — the cost dial

| Class | Cheaper storage, but… | Use for |
|---|---|---|
| **S3 Standard** | — | Data you read often |
| **S3 Intelligent-Tiering** | Small monitoring fee | ✅ **When you're not sure** — AWS moves objects between tiers for you |
| S3 Standard-IA | Fee each time you read | Read about once a month |
| S3 Glacier Instant Retrieval | Higher read fee | Rarely read, but needed instantly |
| S3 Glacier Flexible / Deep Archive | Retrieval takes minutes to hours | Backups kept for compliance |

A **lifecycle rule** moves or deletes objects automatically as they age:

```json
{
  "Rules": [
    {
      "ID": "tidy-raw-data",
      "Status": "Enabled",
      "Filter": {"Prefix": "raw/"},
      "Transitions": [{"Days": 30, "StorageClass": "GLACIER_IR"}],
      "Expiration": {"Days": 365},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
    }
  ]
}
```

```bash
aws s3api put-bucket-lifecycle-configuration --bucket my-shop-data-lake --lifecycle-configuration file://lifecycle.json
```

> **This one rule** — archive after 30 days, delete after a year, clean old versions after 30 days — prevents most "why is our S3 bill growing every month?" surprises.

---

## What costs money

| You pay for | Notes |
|---|---|
| **Storage** | Per GB per month, by storage class |
| **Requests** | Per thousand `PUT` / `GET` / `LIST` — millions of tiny files add up |
| **Data out to the internet** | Per GB downloaded — often the surprise item |
| Data in | Free |
| Data to other AWS services in the same region | Usually free |

> ⚠️ **Millions of tiny files are expensive and slow.** Every one costs a request, and Spark struggles to list them. Combine small files into bigger Parquet files (roughly 100 MB–1 GB each) — see [[Databricks and Delta Lake]].

---

## S3 on other clouds and locally

S3's API became the standard, so the same code works almost everywhere:

| Where | Service | Speaks S3? |
|---|---|---|
| AWS | **S3** | ✅ It *is* S3 |
| Your laptop / own servers | **[[SeaweedFS]]**, MinIO | ✅ Change the endpoint only |
| Azure | **[[ADLS Gen2]]** / Blob Storage | ❌ Its own API (`abfss://`) |
| Google Cloud | Cloud Storage | Mostly — via its interoperability mode |

Full comparison: [[Object storage]] and [[Cloud comparison dictionary]].

---

## Common problems

| You see | Cause | Fix |
|---|---|---|
| *BucketAlreadyExists* | Someone, anywhere, has that name | Pick a more unique name |
| *IllegalLocationConstraintException* | boto3 `create_bucket` outside us-east-1 | Add `CreateBucketConfiguration` |
| *AccessDenied* on list, but get works | Policy has `bucket/*` but not `bucket` | Add `s3:ListBucket` on the bucket ARN |
| *NoSuchKey* | Wrong key — often a missing or extra `/` | `aws s3 ls` the prefix |
| Only 1,000 files found | One `list_objects_v2` call | Use the paginator |
| *No FileSystem for scheme "s3"* in Spark | Spark wants `s3a://` | `s3a://` + `hadoop-aws` |
| Bill keeps growing | Old versions, no lifecycle rule, lots of downloads | Lifecycle rules; check the Cost Explorer |
| Bucket can't be deleted | Not empty — including old versions | `rb --force`; for versioned buckets delete all versions first |

## Related

[[Object storage]] · [[ADLS Gen2]] · [[SeaweedFS]] · [[Using SeaweedFS]] · [[AWS RDS for PostgreSQL]] · [[Pipeline setup - AWS]] · [[Cloud comparison dictionary]] · [[Databricks and Delta Lake]] · [[pandas]] · [[PySpark reference]] · [[Security in practice]]
