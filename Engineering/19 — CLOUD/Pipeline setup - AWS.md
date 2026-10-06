---
tags: [cloud, pipeline, setup, aws]
status: not-started
---

# Pipeline setup - AWS

The full pipeline on AWS, end to end, from an empty account.

Overview: [[Pipeline setup - overview]] · Local first: [[Pipeline setup - Local]]

---

## 💸 Before anything else

```
Billing → Budgets → Create budget → Cost budget → $25/month
```

Also enable **Billing alerts** in Billing preferences. Do this before creating a single resource.

> **AWS has more ways to accidentally spend money than the other two clouds** — NAT gateways (~$32/month doing nothing), idle load balancers, unattached EBS volumes, orphaned Elastic IPs. The teardown section at the end matters more here than anywhere.

## What you'll build

```mermaid
flowchart LR
    A["ESP32"] -->|MQTT| B["IoT Core"]
    B --> C["Kinesis<br/>Data Streams"]
    C --> D["EMR<br/>PySpark"]
    D --> E["Delta on S3"]
    E --> F["RDS<br/>PostgreSQL"]
    E --> G["PyTorch +<br/>SageMaker"]
    G --> H["ECR"]
    F --> I["App Runner /<br/>ECS Fargate"]
    H --> I
    I --> J["CloudWatch"]
```

## Variables

```bash
export AWS_REGION=eu-west-2          # London
export SUFFIX=$RANDOM
export BUCKET=pipeline-lake-$SUFFIX  # S3 names are GLOBALLY unique
export ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```

---

## Step 1 — Foundation

```bash
aws configure          # access key, secret, region, output=json
aws sts get-caller-identity
```

### ⚠️ The IAM model is the hard part

This is where AWS differs most from Azure and GCP, and where you'll spend the most time.

| Concept | Meaning |
|---|---|
| **Policy** | A JSON document listing allowed actions on resources |
| **Role** | An identity a *service* assumes — no password |
| **Trust policy** | Says *who is allowed to assume this role* |
| **Attached policy** | Says *what the role can do* |

> **Every AWS role needs both halves.** A trust policy saying "ECS tasks may assume me" **and** a permissions policy saying "and they may read this bucket". Getting one and not the other is the classic AWS failure, and the error message rarely says which is missing.

**Don't use your root account.** Create an IAM user with `AdministratorAccess` for learning, and use that.

---

## Step 2 — Storage

### Object storage — S3

```bash
aws s3 mb s3://$BUCKET --region $AWS_REGION

# block public access (default, but be explicit)
aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# versioning - your undo button
aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# medallion prefixes
for p in bronze silver gold mlflow; do
  aws s3api put-object --bucket $BUCKET --key $p/
done
```

> **S3 has no real folders** — `bronze/` is just a key prefix. Unlike ADLS Gen2 there's no hierarchical-namespace decision to get wrong, but directory renames are also not atomic.

> **Lifecycle rules save real money.** Move bronze to Infrequent Access after 90 days:
> ```bash
> aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
>   --lifecycle-configuration file://lifecycle.json
> ```

### Warehouse — RDS PostgreSQL

```bash
aws rds create-db-instance \
  --db-instance-identifier pipeline-pg \
  --db-instance-class db.t4g.micro \
  --engine postgres --engine-version 16 \
  --master-username pipelineadmin \
  --master-user-password '<a-long-random-password>' \
  --allocated-storage 20 \
  --backup-retention-period 1 \
  --no-multi-az \
  --publicly-accessible          # learning only; use private subnets in production

# wait ~10 minutes
aws rds wait db-instance-available --db-instance-identifier pipeline-pg

aws rds describe-db-instances --db-instance-identifier pipeline-pg \
  --query 'DBInstances[0].Endpoint.Address' --output text
```

Open the security group to your IP:
```bash
MYIP=$(curl -s ifconfig.me)
SG=$(aws rds describe-db-instances --db-instance-identifier pipeline-pg \
     --query 'DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 5432 --cidr $MYIP/32
```

> **Security groups are stateful firewalls attached to resources**, not to the network. If something can't connect in AWS, the security group is the first thing to check — before IAM.

> **`db.t4g.micro` is free-tier eligible for 12 months.** After that it bills continuously. Stop it between sessions:
> `aws rds stop-db-instance --db-instance-identifier pipeline-pg`
> (RDS auto-restarts after 7 days — you can't stop it indefinitely.)

---

## Step 3 — Ingest and streaming

```bash
# Kinesis (= Kafka)
aws kinesis create-stream --stream-name telemetry --shard-count 1

# IoT Core - no cluster to create, it's fully managed
aws iot create-thing --thing-name sensor-01
aws iot create-keys-and-certificate --set-as-active \
  --certificate-pem-outfile cert.pem \
  --public-key-outfile public.key --private-key-outfile private.key
```

> **Kinesis shards ≠ Kafka partitions.** A shard is a throughput unit (1 MB/s in, 2 MB/s out). You pay **per shard-hour** whether you use it or not — one shard is ~$11/month. Delete the stream when you're done.

Route IoT messages into Kinesis with an IoT **rule** (SQL over the MQTT topic):
```bash
aws iot create-topic-rule --rule-name to_kinesis --topic-rule-payload '{
  "sql": "SELECT * FROM \"sensors/+/telemetry\"",
  "actions": [{"kinesis": {"roleArn": "arn:aws:iam::'$ACCOUNT':role/iot-kinesis-role",
                            "streamName": "telemetry", "partitionKey": "${device_id}"}}]
}'
```

> That `roleArn` must exist first, with a trust policy allowing `iot.amazonaws.com` to assume it. This is the two-halves IAM pattern in practice.

---

## Step 4 — Process: EMR

```bash
aws emr create-cluster \
  --name pipeline-spark \
  --release-label emr-7.0.0 \
  --applications Name=Spark \
  --instance-type m5.xlarge --instance-count 2 \
  --use-default-roles \
  --auto-termination-policy IdleTimeout=1800 \
  --log-uri s3://$BUCKET/emr-logs/
```

> ⚠️ **`--auto-termination-policy IdleTimeout=1800` is not optional.** EMR bills per instance-hour. A two-node cluster left overnight is real money. 1800 seconds = 30 minutes idle.

> **Cheaper alternative for learning: skip EMR entirely.** Run Spark locally against S3 — the S3A connector works identically. You only need EMR when data genuinely exceeds one machine ([[When to leave Python]]).

### Spark + Delta on S3

```python
from pyspark.sql import SparkSession

spark = (SparkSession.builder.appName("pipeline")
    .config("spark.jars.packages",
            "io.delta:delta-spark_2.12:3.2.0,org.apache.hadoop:hadoop-aws:3.3.4")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog",
            "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    # on EMR, credentials come from the instance role - no keys needed
    .config("spark.hadoop.fs.s3a.aws.credentials.provider",
            "com.amazonaws.auth.DefaultAWSCredentialsProviderChain")
    .getOrCreate())

LAKE = f"s3a://{BUCKET}"      # vs abfss:// on Azure, gs:// on GCP
```

> **On EMR you don't set access keys.** The EC2 instance profile provides credentials automatically — the AWS equivalent of managed identity. If you find yourself pasting keys into Spark config, something is wrong.

The bronze/silver/gold code is **identical to [[Pipeline setup - Local]]**.

---

## Step 5 — Train + registry

**SageMaker** is the managed option, but it's the heaviest of the three clouds' ML platforms.

> **Honest recommendation for learning: run MLflow yourself** on a small EC2 instance or locally, with artifacts in S3. Everything in [[MLflow experiment tracking]] then works unchanged, and you avoid SageMaker's considerable surface area.

```python
import mlflow

mlflow.set_tracking_uri("http://<your-mlflow-host>:5000")
# artifacts go to s3://$BUCKET/mlflow
```

If you do use SageMaker, its Experiments and Model Registry are the equivalents — see [[Azure ML and the MLOps stack]] for the concepts, which transfer.

---

## Step 6 — Serve

### Container registry — ECR

```bash
aws ecr create-repository --repository-name pipeline-api

aws ecr get-login-password --region $AWS_REGION \
  | docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com

docker build -t $ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/pipeline-api:v1 .
docker push $ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/pipeline-api:v1
```

### App Runner — the simplest option

```bash
aws apprunner create-service \
  --service-name pipeline-api \
  --source-configuration '{
     "ImageRepository": {
       "ImageIdentifier": "'$ACCOUNT'.dkr.ecr.'$AWS_REGION'.amazonaws.com/pipeline-api:v1",
       "ImageRepositoryType": "ECR",
       "ImageConfiguration": {"Port": "8000"}
     },
     "AuthenticationConfiguration": {"AccessRoleArn": "arn:aws:iam::'$ACCOUNT':role/AppRunnerECRAccessRole"}
   }' \
  --instance-configuration '{"Cpu":"1024","Memory":"2048"}'
```

> **App Runner is AWS's closest thing to Container Apps / Cloud Run** — give it an image, get an HTTPS URL. **ECS Fargate** gives more control (VPC placement, sidecars, load balancer) at the cost of much more configuration: task definition, service, target group, ALB.
>
> **Use App Runner for learning.** Move to ECS when you need VPC networking.

### Secrets

```bash
aws secretsmanager create-secret --name pipeline/db-url \
  --secret-string "postgresql://pipelineadmin:...@pipeline-pg...:5432/pipeline"
```

Grant the task role permission to read it, then use `boto3` — or better, give the role direct RDS IAM auth so no password exists at all.

> [[FastAPI data and deployment]]

---

## Step 7 — Observe

CloudWatch collects container logs automatically.

```bash
aws logs tail /aws/apprunner/pipeline-api --follow
```

Metrics and traces:
```python
# X-Ray for traces, or point OpenTelemetry at the ADOT collector
```

> Your OTel instrumentation is vendor-neutral — the collector routes to CloudWatch/X-Ray instead of Grafana. Same application code. [[Observability for data and ML pipelines]]

---

## The five verification checks

```bash
# 1 - data arrives
aws s3 ls s3://$BUCKET/bronze/ --recursive | head

# 2 - pipeline ran (check the Delta table)

# 3 - database serves
PGHOST=$(aws rds describe-db-instances --db-instance-identifier pipeline-pg \
         --query 'DBInstances[0].Endpoint.Address' --output text)
psql "host=$PGHOST user=pipelineadmin dbname=pipeline sslmode=require" \
  -c "SELECT count(*) FROM daily_readings;"

# 4 - API predicts
URL=$(aws apprunner list-services --query "ServiceSummaryList[?ServiceName=='pipeline-api'].ServiceUrl" --output text)
curl -s https://$URL/health

# 5 - monitoring
aws logs tail /aws/apprunner/pipeline-api --since 10m
```

---

## 🔻 Teardown — read this carefully

**AWS has no "delete the resource group" button.** You must delete each resource. This is the single biggest cost risk of the three clouds.

```bash
# compute
aws apprunner delete-service --service-arn <arn>
aws emr terminate-clusters --cluster-ids <id>

# streaming - bills per shard-hour
aws kinesis delete-stream --stream-name telemetry

# database
aws rds delete-db-instance --db-instance-identifier pipeline-pg \
  --skip-final-snapshot --delete-automated-backups

# registry + storage
aws ecr delete-repository --repository-name pipeline-api --force
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET

# IoT
aws iot delete-thing --thing-name sensor-01
```

**Then check for the expensive leftovers:**

```bash
aws ec2 describe-nat-gateways --filter Name=state,Values=available   # ~$32/mo each
aws ec2 describe-addresses                                           # unattached Elastic IPs
aws ec2 describe-volumes --filters Name=status,Values=available      # orphaned EBS
aws elbv2 describe-load-balancers                                    # idle ALBs
```

> **Use Terraform for anything beyond a one-off experiment** ([[Terraform]]). `terraform destroy` does what `az group delete` does — and on AWS that's worth considerably more.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `AccessDenied` on S3 | Role has no policy, or trust policy missing | Check **both** halves of the role |
| Service can't assume a role | Trust policy doesn't name the service principal | Add `ecs-tasks.amazonaws.com` etc. |
| Can't connect to RDS | Security group, not IAM | `authorize-security-group-ingress` on port 5432 |
| Bucket name rejected | S3 names are globally unique | Add `$RANDOM` |
| ECR push denied | Not logged in; token expires in 12h | Re-run `get-login-password` |
| App Runner can't pull image | Missing `AppRunnerECRAccessRole` | Create it with the ECR trust policy |
| Kinesis throttling | One shard = 1 MB/s | Add shards (each costs) |
| EMR bill larger than expected | No idle timeout | `--auto-termination-policy` |
| $32 charge doing nothing | **NAT gateway** | Delete it; use public subnets for learning |
| RDS restarted itself | Stopped instances auto-start after 7 days | Delete rather than stop, long-term |

More: [[I HAVE A PROBLEM]]

## Next

[[Pipeline setup - GCP]]

## Related

[[Pipeline setup - overview]] · [[Cloud comparison dictionary]] · [[Terraform]] · [[Docker deep dive]] · [[PySpark core]]
