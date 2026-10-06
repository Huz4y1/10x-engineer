---
tags: [kubernetes, aks, deployment, containers]
status: not-started
---

# Kubernetes and AKS

> **What this is:** the container orchestrator you'll meet in most large companies, and Azure's managed version of it.
> **Why you care:** you probably **don't need it** for this stack — Container Apps is simpler and does the job. But it's on every job description, and eventually you'll join a team that runs it.

---

## Read this first: do you need Kubernetes?

Almost certainly not, yet.

| You have | Use |
|---|---|
| A few containers, HTTP, scale-to-zero | **Azure Container Apps** ← the capstone |
| One web app, no containers even | Azure App Service |
| A scheduled batch job | Databricks Workflows / Container Apps Jobs |
| Dozens of services, custom networking, multi-team, existing k8s expertise | **Kubernetes / AKS** |

> **Kubernetes is a platform for building platforms.** It's enormously capable and it charges you in complexity: YAML, networking, ingress, RBAC, secrets, storage classes, upgrades, and a team to run it. For a portfolio project or a small team it's a liability, not a credential.
>
> **Azure Container Apps is Kubernetes underneath** — it runs on AKS with KEDA and Dapr — with the complexity hidden. You get autoscaling, scale-to-zero, revisions and traffic splitting without writing a single manifest. Use it, and know what's under it.

Learn Kubernetes so you can work in a team that uses it. Don't adopt it because it's on job adverts.

---

## The idea in plain English

Docker runs a container on **one** machine. Kubernetes runs containers across **many** machines and keeps them running.

You don't tell it "start this container on that server." You tell it **"I want three copies of this running, always"** — and it works out where, restarts them when they die, moves them when a machine fails, and replaces them one at a time when you deploy a new version.

**It's a control loop:** compare desired state to actual state, and act to close the gap. Forever. Everything else in Kubernetes is detail on top of that one idea.

---

## The objects you actually need

There are dozens. You need about six.

```
Deployment  ──manages──▶  ReplicaSet  ──manages──▶  Pods  ──contain──▶  Containers
                                                      ▲
Service  ──routes traffic to──────────────────────────┘
   ▲
Ingress  ──routes external traffic to──┘
```

| Object | What it is | Analogy |
|---|---|---|
| **Pod** | One or more containers that share a network and lifetime | The smallest deployable unit. Usually one container. |
| **Deployment** | "Keep N copies of this pod running, and roll out updates safely" | What you actually write |
| **Service** | A stable internal address + load balancing across pods | Pods die and get new IPs; the Service name doesn't change |
| **Ingress** | HTTP routing from outside the cluster | The front door and URL router |
| **ConfigMap** | Non-secret configuration | Your `.env`, but not secret |
| **Secret** | Secret configuration | ⚠️ base64, **not encrypted** by default |
| **Job / CronJob** | Run to completion, once or on a schedule | Your batch scoring job |
| **HPA** | Horizontal Pod Autoscaler — scale on CPU/memory/custom metrics | Autoscaling |

> **You never create Pods directly.** You create a Deployment, and it creates Pods. A Pod you make by hand isn't replaced when it dies, which defeats the entire point.

---

## A real Deployment

```yaml
# k8s/api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retail-api
spec:
  replicas: 3
  selector:
    matchLabels: {app: retail-api}
  template:
    metadata:
      labels: {app: retail-api}
    spec:
      containers:
        - name: api
          image: myacr.azurecr.io/retail-api:v3      # ← versioned, never :latest
          ports:
            - containerPort: 8000

          # ---- resources: REQUIRED, see the note below ----
          resources:
            requests:                                 # what the scheduler reserves
              memory: "512Mi"
              cpu: "250m"                             # 250m = 0.25 of a core
            limits:                                   # the hard ceiling
              memory: "2Gi"
              cpu: "1000m"

          # ---- health checks ----
          livenessProbe:                              # restart me if this fails
            httpGet: {path: /health, port: 8000}
            initialDelaySeconds: 20
            periodSeconds: 30
          readinessProbe:                             # send me traffic when this passes
            httpGet: {path: /ready, port: 8000}
            initialDelaySeconds: 5
            periodSeconds: 10

          env:
            - name: MODEL_VERSION
              valueFrom:
                configMapKeyRef: {name: retail-config, key: model_version}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: {name: retail-secrets, key: database-url}
```

### The two things that cause most production incidents

**1. Resource requests and limits.**

- **Request** = what the scheduler reserves. Too high and pods won't schedule; too low and they get packed onto a node and starve.
- **Limit** = the ceiling. Exceed the memory limit and the pod is **OOMKilled** instantly — no grace, no warning.

> **Omitting them is worse than getting them wrong.** With no request, the scheduler assumes ~nothing and packs your pod onto a full node. With no limit, one leaking pod can take down every other pod on the machine. **Always set both.**
>
> For ML serving: measure actual usage under load, then set request ≈ observed and limit ≈ 2× observed. Remember each replica loads its own copy of the model.

**2. Liveness vs. readiness — they are different, and confusing them is bad.**

| Probe | Question | On failure |
|---|---|---|
| **Liveness** | Is the process wedged? | **Kill and restart the pod** |
| **Readiness** | Can it serve traffic right now? | Remove from the Service; **don't** restart |

> **Never make the liveness probe check your database.** A brief database blip then fails liveness on every pod at once, Kubernetes restarts them all, they come back and fail again — a restart storm that turns a 30-second outage into a total one. **Liveness = "is the process alive" and nothing else.** Dependencies belong in readiness. This is exactly the split from [[FastAPI data and deployment]].

---

## Service and Ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: retail-api
spec:
  selector: {app: retail-api}          # ← matches pods by LABEL, not by name
  ports:
    - port: 80
      targetPort: 8000
```

Other pods now reach it at `http://retail-api` — DNS inside the cluster. That's the same idea as service names in Docker Compose ([[Docker deep dive]]).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: retail.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend: {service: {name: retail-api, port: {number: 80}}}
          - path: /
            pathType: Prefix
            backend: {service: {name: retail-dashboard, port: {number: 80}}}
```

> **Services match pods by label, not name.** A typo in `selector` gives you a Service with no endpoints — it resolves, accepts connections, and times out. `kubectl get endpoints retail-api` showing `<none>` is the tell.

---

## Batch jobs — the ML-relevant one

Your nightly scoring job ([[Deployment patterns]]) is a CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-scoring
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid            # ← don't start if the last run is still going
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2                  # retry twice, then give up
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: scorer
              image: myacr.azurecr.io/retail-scorer:v3
              resources:
                requests: {memory: "2Gi", cpu: "1000m"}
                limits:   {memory: "4Gi", cpu: "2000m"}
```

> **`concurrencyPolicy: Forbid`** is the equivalent of `max_concurrent_runs: 1` in [[Unity Catalog and orchestration]] — it stops a slow run overlapping the next and corrupting your output table.

---

## Secrets — read this before you trust them

```bash
kubectl create secret generic retail-secrets --from-literal=database-url='postgresql://...'
```

> **⚠️ Kubernetes Secrets are base64-encoded, not encrypted.** Anyone with read access to the namespace can decode them in one command. They are *not* a secret manager.
>
> **The right approach on Azure:** use **Workload Identity** + the **Secrets Store CSI driver** to pull from Key Vault at runtime, so no secret is stored in the cluster at all. Same principle as the managed identity pattern in [[Azure fundamentals]] — the goal is always that no secret exists to leak.

---

## GPU workloads

```yaml
spec:
  containers:
    - name: trainer
      image: myacr.azurecr.io/trainer:v1
      resources:
        limits:
          nvidia.com/gpu: 1            # ← whole GPUs only, no fractions
      nodeSelector:
        accelerator: nvidia-tesla-t4
```

Requires the NVIDIA device plugin on the cluster and a GPU node pool. See [[CUDA and GPU programming]].

> **GPUs are allocated whole.** One pod gets the entire GPU — no sharing by default. A GPU node pool that's up and idle is very expensive, so scale it to zero when nothing is training.

---

## AKS specifics

```bash
az aks create --resource-group retail-rg --name retail-aks \
  --node-count 2 --node-vm-size Standard_D2s_v3 \
  --enable-managed-identity --attach-acr myacr \
  --enable-cluster-autoscaler --min-count 1 --max-count 5 \
  --generate-ssh-keys

az aks get-credentials --resource-group retail-rg --name retail-aks
kubectl get nodes
```

- `--attach-acr` lets nodes pull from your registry without image-pull secrets
- `--enable-managed-identity` is the foundation for Key Vault access
- **Node pools:** put GPU nodes in a separate pool with its own autoscaling, so you're not paying for GPUs to run your API

> **💸 An AKS cluster bills continuously**, unlike Container Apps' scale-to-zero. Even idle, the nodes cost money. If you spin one up to learn, `az aks delete` when you're finished.

---

## kubectl cheat sheet

```bash
kubectl apply -f k8s/                  # apply everything in a folder
kubectl get pods                       # what's running
kubectl get pods -w                    # watch live
kubectl get all                        # everything in the namespace

# debugging — in this order
kubectl describe pod <pod>             # events at the bottom = WHY it won't start
kubectl logs <pod>                     # app output
kubectl logs <pod> --previous          # logs from the crashed instance ← crucial
kubectl exec -it <pod> -- bash         # shell inside
kubectl get events --sort-by=.metadata.creationTimestamp

# rollouts
kubectl rollout status deployment/retail-api
kubectl rollout undo deployment/retail-api          # ← rollback
kubectl rollout history deployment/retail-api

# scaling
kubectl scale deployment/retail-api --replicas=5
kubectl autoscale deployment/retail-api --min=2 --max=10 --cpu-percent=70

# local access without an ingress
kubectl port-forward svc/retail-api 8000:80

kubectl top pods                       # actual CPU/memory usage
```

> **`kubectl describe pod` first, always.** The Events section at the bottom tells you exactly why a pod won't start — image pull failure, insufficient memory, failing probe, missing secret. It answers the question about 80% of the time.

---

## When something goes wrong

| Pod status / symptom | Meaning | Fix |
|---|---|---|
| `ImagePullBackOff` | Can't fetch the image | Wrong tag, or no registry access — `--attach-acr` |
| `CrashLoopBackOff` | Starts, crashes, repeats | `kubectl logs <pod> --previous` |
| `Pending` forever | Nothing can schedule it | `describe` — usually insufficient CPU/memory, or no GPU node |
| `OOMKilled` | Exceeded the memory limit | Raise the limit, or use less memory |
| `CreateContainerConfigError` | Missing ConfigMap or Secret key | Check the referenced names exist |
| Service resolves but times out | Selector doesn't match pod labels | `kubectl get endpoints` — `<none>` confirms it |
| All pods restart at once | Liveness probe checks a dependency | Liveness = process only |
| Deploy takes down the service | No readiness probe, so traffic hit unready pods | Add readiness |
| Restart storm during a DB blip | Same liveness mistake | Same fix |
| Rollout stuck | New pods never become ready | `describe` the new pod; check the readiness probe |
| Secrets readable by anyone | They're base64, not encrypted | Key Vault + CSI driver |
| Cluster bill is large | Nodes bill continuously | Autoscale to zero; delete when learning |
| Works in Compose, fails in k8s | `localhost` between services | Use the Service name |

---

## Practice checklist

- [ ] **Whether you need Kubernetes at all** — and that Container Apps is AKS with the complexity hidden
- [ ] The control-loop idea: desired state vs. actual state
- [ ] Pod / Deployment / Service / Ingress / ConfigMap / Secret / Job / HPA
- [ ] Why you never create Pods directly
- [ ] **Resource requests and limits** — and why omitting them is worse than guessing
- [ ] **Liveness vs. readiness**, and never checking dependencies in liveness
- [ ] Services select by label; `get endpoints` to diagnose
- [ ] CronJobs and `concurrencyPolicy: Forbid`
- [ ] **Kubernetes Secrets are not encrypted** — use Key Vault + CSI on Azure
- [ ] GPU scheduling and whole-GPU allocation
- [ ] AKS creation flags, node pools, and continuous billing
- [ ] `kubectl describe pod` as the first debugging move

## Hands-on

- [ ] Install `k3d` or enable Kubernetes in Docker Desktop — a local cluster, free
- [ ] Deploy the capstone API as a Deployment + Service; reach it with `port-forward`
- [ ] Set a 128Mi memory limit and watch it get OOMKilled
- [ ] Break the Service selector and confirm `get endpoints` shows `<none>`
- [ ] Deploy a bad image, watch `CrashLoopBackOff`, and diagnose with `logs --previous`
- [ ] Do a rolling update, then `kubectl rollout undo`
- [ ] Convert your nightly scoring job into a CronJob

## Resources

- [Kubernetes: concepts](https://kubernetes.io/docs/concepts/)
- [AKS documentation](https://learn.microsoft.com/azure/aks/)
- [k3d](https://k3d.io/) — a real cluster in Docker, ideal for learning

## Next

[[MLOps and CI-CD]]
