---
tags: [reference, cheatsheet, cli, bash, git, docker, kubernetes]
---

# Command reference

Every command in this vault, in one place. Section: [[32 — RESOURCES]]

---

## Linux / bash

```bash
# navigate
pwd; ls -lah; cd -                 # cd - goes back to previous dir
tree -L 2                          # directory tree, 2 levels

# find
find . -name "*.py" -mtime -7      # modified in last 7 days
find . -size +100M                 # big files
grep -rn "pattern" --include="*.py" .
rg "pattern" -t py                 # ripgrep - much faster

# inspect files
head -20 f; tail -f log            # -f follows live
less f                             # q to quit, / to search
wc -l f                            # line count
cut -d, -f1,3 f.csv                # columns 1 and 3
sort f | uniq -c | sort -rn        # frequency count - very useful

# processes
ps aux | grep python
top; htop
kill -9 <pid>
lsof -i :8000                      # what's using this port

# disk / memory
df -h                              # disk free
du -sh *                           # size of each item here
free -h                            # memory

# permissions
chmod +x script.sh
chown user:group file

# env
export VAR=value
env | sort
echo $PATH

# archives
tar -czf a.tar.gz dir/             # create
tar -xzf a.tar.gz                  # extract

# network
curl -v https://host/path
curl -X POST -H "Content-Type: application/json" -d '{"a":1}' url
wget url
ssh user@host
scp file user@host:/path
rsync -avz src/ user@host:/dst/
```

> **`sort | uniq -c | sort -rn` is the single most useful pipeline in the list** — it turns any log into a frequency table.

---

## Git

```bash
# daily
git status
git switch -c feature/thing        # create + switch (modern)
git add .                          # or: git add -p  (interactive, by hunk)
git commit -m "Add thing"
git push -u origin feature/thing

# looking
git log --oneline -10
git log --graph --oneline --all
git diff                           # unstaged
git diff --cached                  # staged
git show <hash>
git blame file.py

# undoing
git restore file.py                # discard changes (DESTRUCTIVE)
git restore --staged file.py       # unstage, keep edits
git commit --amend -m "Better"     # fix last message (before push)
git reset --soft HEAD~1            # undo commit, keep changes
git reset --hard HEAD~1            # undo commit, DISCARD changes
git revert <hash>                  # safe undo for PUSHED commits
git stash; git stash pop
git reflog                         # ← THE SAFETY NET. Find "lost" commits.

# branches
git switch main; git pull
git rebase main                    # replay my commits on top of main
git merge feature/thing
git branch -d feature/thing

# remotes
git remote -v
git fetch origin
git push --force-with-lease        # safer than --force
```

> **`git reflog` is the undo-the-undo.** It logs every position HEAD has been in for ~90 days, including after `reset --hard`. Check it before despairing.

---

## Docker

```bash
# build & run
docker build -t name:v1 .
docker build --no-cache -t name:v1 .
docker run -p 8000:8000 --env-file .env name:v1
docker run -d --name api name:v1               # detached
docker run -it --entrypoint bash name:v1       # debug shell
docker run --memory=2g --cpus=1.5 name:v1      # limits
docker run --gpus all name:v1                  # GPU

# inspect
docker ps; docker ps -a
docker logs -f <id>
docker exec -it <id> bash
docker inspect <id>
docker history name:v1                         # which layer is huge
docker stats

# clean
docker system df
docker system prune -a --volumes               # ⚠️ deletes unused images AND volumes

# compose
docker compose up --build
docker compose up -d
docker compose logs -f <service>
docker compose exec <service> bash
docker compose down                            # -v also deletes volumes

# registry
az acr login --name myacr
docker tag name:v1 myacr.azurecr.io/name:v1
docker push myacr.azurecr.io/name:v1
```

---

## Kubernetes

```bash
# look
kubectl get pods -A
kubectl get all -n mynamespace
kubectl get pods -w                            # watch live

# debug — IN THIS ORDER
kubectl describe pod <pod>                     # ← Events at the bottom = why
kubectl logs <pod>
kubectl logs <pod> --previous                  # ← from the CRASHED instance
kubectl exec -it <pod> -- bash
kubectl get events --sort-by=.metadata.creationTimestamp

# apply
kubectl apply -f k8s/
kubectl delete -f k8s/

# rollouts
kubectl rollout status deployment/api
kubectl rollout undo deployment/api            # ← rollback
kubectl rollout history deployment/api

# scale
kubectl scale deployment/api --replicas=5
kubectl autoscale deployment/api --min=2 --max=10 --cpu-percent=70

# access & diagnose
kubectl port-forward svc/api 8000:80
kubectl get endpoints api                      # <none> = selector doesn't match
kubectl top pods

# context
kubectl config get-contexts
kubectl config use-context my-cluster
```

---

## Azure CLI

```bash
az login
az account show
az account set --subscription "Name"

az group create --name rg --location uksouth
az group delete --name rg --yes --no-wait      # deletes EVERYTHING inside

az storage account create --name store$RANDOM --resource-group rg \
  --sku Standard_LRS --enable-hierarchical-namespace true
az storage fs create -n bronze --account-name store --auth-mode login
az storage fs file list --account-name store -f bronze --auth-mode login -o table

az containerapp create --name api --resource-group rg --environment env \
  --image acr.azurecr.io/api:v1 --target-port 8000 --ingress external
az containerapp logs show --name api --resource-group rg --follow
az containerapp ingress traffic set --name api --resource-group rg \
  --revision-weight latest=10

az role assignment list --assignee you@x.com --all -o table
az resource list --resource-group rg -o table
```

> **`--auth-mode login`** on every `az storage` command, or it hunts for account keys and fails confusingly despite correct RBAC.

---

## Terraform

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan        # ← ALWAYS read this
terraform apply tfplan
terraform destroy
terraform state list
terraform show
terraform import azurerm_resource_group.main /subscriptions/.../rg
terraform workspace new prod
```

---

## Python / uv

```bash
uv init myproject
uv add pandas fastapi
uv add --dev pytest ruff mypy
uv sync --frozen
uv run python main.py
uv run pytest -v
uv lock --upgrade

# testing
pytest -v                         # verbose
pytest -x                         # stop at first failure
pytest -k "revenue"               # match name
pytest --lf                       # last failed only
pytest -m "not slow"              # by marker
pytest --cov=api --cov-report=html

# quality
ruff check --fix .
ruff format .
mypy api/

# profiling
python -m cProfile -o out.prof script.py
kernprof -l -v script.py          # line profiler
```

---

## Kafka

```bash
kafka-topics --create --topic t --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092
kafka-topics --list --bootstrap-server localhost:9092
kafka-topics --describe --topic t --bootstrap-server localhost:9092

kafka-console-producer --topic t --bootstrap-server localhost:9092
kafka-console-consumer --topic t --from-beginning --bootstrap-server localhost:9092

kafka-consumer-groups --describe --group g --bootstrap-server localhost:9092   # ← LAG
kafka-consumer-groups --reset-offsets --to-earliest --group g --topic t --execute \
  --bootstrap-server localhost:9092
```

> **`kafka-consumer-groups --describe` is the first thing to run** when a consumer isn't receiving. `LAG` tells you whether the problem is upstream or downstream ([[Kafka]]).

---

## psql

See [[PostgreSQL reference]] for the full list.

```
\dt   \d table   \d+ table   \di   \du   \x   \timing   \q
\copy t FROM 'f.csv' CSV HEADER
```

---

## Spark / Databricks

```python
spark.conf.set("spark.sql.shuffle.partitions", "4")     # 200 is absurd locally
df.explain(mode="formatted")
df.rdd.getNumPartitions()
spark.sql("DESCRIBE HISTORY delta.`path`").show()
spark.sql("OPTIMIZE t ZORDER BY (col)")
spark.sql("RESTORE TABLE t TO VERSION AS OF 3")
```

```bash
databricks bundle validate
databricks bundle deploy --target prod
```

---

## nvidia / GPU

```bash
nvidia-smi                        # what GPU, what's using it
watch -n 0.5 nvidia-smi           # live
nvcc --version                    # CUDA compiler version
```

```python
import torch

torch.cuda.is_available()
torch.cuda.memory_allocated() / 1e9
torch.cuda.empty_cache()
```

---

## The universal first five, when something breaks

```bash
git log --oneline -5              # what changed
docker logs <container>           # what did it say before dying
kubectl describe pod <pod>        # read the Events
env | sort                        # is the config what I think
df -h && free -h                  # out of disk or memory
```

See [[_Troubleshooting template]] for the reasoning method.

## Related

[[32 — RESOURCES]] · [[Dev environment - Git, Docker, CLI]] · [[Docker deep dive]] · [[Kubernetes and AKS]] · [[PostgreSQL reference]] · [[Kafka]] · [[30 — TROUBLESHOOTING]]
