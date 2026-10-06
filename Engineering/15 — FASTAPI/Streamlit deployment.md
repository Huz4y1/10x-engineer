---
tags: [streamlit, deployment, docker, azure, aws, gcp]
status: not-started
---

# Streamlit deployment

Getting a Streamlit dashboard off your laptop, on every stack in this vault.

Main note: [[Streamlit]] · Pipelines: [[Pipeline setup - overview]]

---

## The three flags that matter everywhere

```bash
streamlit run app.py \
  --server.port=8501 \
  --server.address=0.0.0.0 \
  --server.headless=true
```

| Flag | Without it |
|---|---|
| `--server.address=0.0.0.0` | Binds to localhost — **unreachable from outside the container** |
| `--server.headless=true` | Tries to open a browser and **prompts for an email**; in a container it just hangs |
| `--server.port` | Defaults to 8501; set it explicitly to match your platform |

> **These three are the entire difference between "works on my machine" and "works deployed".** Every failure below traces back to one of them.

## The Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# dependencies first - layer caching ([[Docker deep dive]])
COPY pyproject.toml uv.lock ./
RUN pip install --no-cache-dir uv && uv sync --frozen --no-dev

COPY dashboard/ ./dashboard/
COPY .streamlit/ ./.streamlit/

RUN useradd --create-home appuser && chown -R appuser /app
USER appuser

ENV PYTHONUNBUFFERED=1
EXPOSE 8501

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s \
  CMD curl -f http://localhost:8501/_stcore/health || exit 1

CMD ["uv", "run", "streamlit", "run", "dashboard/app.py", \
     "--server.port=8501", "--server.address=0.0.0.0", "--server.headless=true"]
```

> **`/_stcore/health` is Streamlit's built-in health endpoint.** Use it for container healthchecks and platform readiness probes — you don't need to write one.

```bash
docker build -t dashboard:v1 .
docker run -p 8501:8501 -e API_URL=http://host.docker.internal:8000 dashboard:v1
```

---

## Local — Docker Compose

Alongside the API, in the same compose file as [[Pipeline setup - Local]]:

```yaml
  dashboard:
    build:
      context: ..
      dockerfile: dashboard/Dockerfile
    ports: ["8501:8501"]
    depends_on: [api]
    environment:
      API_URL: http://api:8000        # <- SERVICE NAME, not localhost
```

> **`http://api:8000`, never `http://localhost:8000`.** Inside a container `localhost` means *that container*. This is the number one Streamlit-in-Docker failure ([[Docker deep dive]]).

---

## Streamlit Community Cloud — free, for portfolios

The fastest way to get a public URL:

1. Push the repo to GitHub
2. [share.streamlit.io](https://share.streamlit.io) -> New app -> pick repo, branch, `dashboard/app.py`
3. Paste `secrets.toml` contents into **Settings -> Secrets**

| Good | Limitations |
|---|---|
| Free, one click, HTTPS, auto-deploys on push | Public by default; sleeps when idle; ~1 GB RAM; no VNet access to a private API |

> **Ideal for a portfolio project.** Not for anything touching private data or a private API — it can only reach the public internet.

---

## Azure — Container Apps

```bash
az acr build --registry $ACR --image dashboard:v1 --file dashboard/Dockerfile .

az containerapp create \
  --name dashboard --resource-group $RG --environment pipeline-env \
  --image $ACR.azurecr.io/dashboard:v1 \
  --registry-server $ACR.azurecr.io \
  --target-port 8501 --ingress external \
  --min-replicas 1 --max-replicas 2 \
  --cpu 1 --memory 2Gi \
  --env-vars "API_URL=https://pipeline-api.internal.${ENV_DOMAIN}"
```

**Two Azure-specific points:**

**1. Set the API's ingress to `internal`, the dashboard's to `external`.** Both live in the same Container Apps environment, so the dashboard reaches the API over the internal network and the API is never exposed publicly.

```bash
az containerapp ingress update --name pipeline-api --resource-group $RG --type internal
```

**2. `--min-replicas 1`, not 0.** Scale-to-zero is right for an API; it's wrong for a dashboard, because a person is waiting through the ~10s cold start and will assume it's broken.

### Free auth in one command

```bash
az containerapp auth microsoft update --name dashboard --resource-group $RG \
  --client-id <app-id> --client-secret <secret> \
  --tenant-id <tenant> --yes

az containerapp auth update --name dashboard --resource-group $RG \
  --unauthenticated-client-action RedirectToLoginPage
```

> **This is the best answer to Streamlit's missing auth.** Entra ID login happens *before* the request reaches your app, so no unauthenticated code runs at all — which a Python password check cannot promise ([[Streamlit]] §14).

---

## AWS — App Runner or ECS

### App Runner — simplest

```bash
aws ecr create-repository --repository-name dashboard
docker build -t $ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/dashboard:v1 -f dashboard/Dockerfile .
docker push $ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com/dashboard:v1

aws apprunner create-service \
  --service-name dashboard \
  --source-configuration '{
     "ImageRepository": {
       "ImageIdentifier": "'$ACCOUNT'.dkr.ecr.'$AWS_REGION'.amazonaws.com/dashboard:v1",
       "ImageRepositoryType": "ECR",
       "ImageConfiguration": {
         "Port": "8501",
         "RuntimeEnvironmentVariables": {"API_URL": "https://<api-url>"}
       }
     },
     "AuthenticationConfiguration": {"AccessRoleArn": "arn:aws:iam::'$ACCOUNT':role/AppRunnerECRAccessRole"}
   }' \
  --health-check-configuration '{"Protocol":"HTTP","Path":"/_stcore/health"}'
```

### ECS Fargate behind an ALB — when you need a VPC

More setup (task definition, service, target group, ALB), but you get VPC placement and **ALB authentication** with Cognito or OIDC.

**Two ALB settings Streamlit needs:**

| Setting | Why |
|---|---|
| **Sticky sessions ON** | Streamlit holds session state **in memory per server**. Without stickiness a user's requests hit different tasks and their state vanishes mid-interaction. |
| Idle timeout raised (~300s) | Streamlit uses a long-lived WebSocket; the default 60s drops it and the page shows "connection error" |

> **Sticky sessions are not optional for multi-replica Streamlit.** This is the most common production Streamlit bug and it looks like random state loss.

---

## GCP — Cloud Run

```bash
gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT/pipeline/dashboard:v1 \
  --file dashboard/Dockerfile .

gcloud run deploy dashboard \
  --image=$REGION-docker.pkg.dev/$PROJECT/pipeline/dashboard:v1 \
  --region=$REGION --platform=managed \
  --port=8501 --cpu=1 --memory=2Gi \
  --min-instances=1 --max-instances=2 \
  --session-affinity \
  --timeout=3600 \
  --set-env-vars="API_URL=https://<api-url>" \
  --allow-unauthenticated
```

**Three Cloud Run flags Streamlit specifically needs:**

| Flag | Why |
|---|---|
| `--session-affinity` | The Cloud Run equivalent of sticky sessions. **Required** for multi-instance Streamlit. |
| `--timeout=3600` | Streamlit's WebSocket is long-lived; the default 300s cuts it |
| `--min-instances=1` | Avoids cold starts for a human user |

### Auth via IAP

```bash
gcloud run deploy dashboard --no-allow-unauthenticated
gcloud run services add-iam-policy-binding dashboard \
  --member="user:someone@example.com" --role="roles/run.invoker" --region=$REGION
```

> Removing `--allow-unauthenticated` means Google handles login before the request reaches Streamlit. Same principle as the Azure approach.

---

## Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: dashboard}
spec:
  replicas: 2
  selector: {matchLabels: {app: dashboard}}
  template:
    metadata: {labels: {app: dashboard}}
    spec:
      containers:
      - name: dashboard
        image: myacr.azurecr.io/dashboard:v1
        ports: [{containerPort: 8501}]
        env:
        - {name: API_URL, value: "http://pipeline-api:8000"}
        resources:
          requests: {memory: "512Mi", cpu: "250m"}
          limits:   {memory: "2Gi",   cpu: "1000m"}
        livenessProbe:
          httpGet: {path: /_stcore/health, port: 8501}
          initialDelaySeconds: 20
        readinessProbe:
          httpGet: {path: /_stcore/health, port: 8501}
---
apiVersion: v1
kind: Service
metadata: {name: dashboard}
spec:
  selector: {app: dashboard}
  sessionAffinity: ClientIP          # <- REQUIRED with replicas > 1
  ports: [{port: 80, targetPort: 8501}]
```

Ingress needs WebSocket support and a raised timeout:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/affinity: "cookie"
```

> `sessionAffinity: ClientIP` plus cookie affinity on the ingress. Same reason as the ALB. See [[Kubernetes and AKS]].

---

## Secrets per platform

| Platform | How |
|---|---|
| Local | `.streamlit/secrets.toml`, gitignored |
| Docker | `-e VAR=value` or `--env-file` |
| Community Cloud | Settings -> Secrets (paste TOML) |
| Azure | `--secrets` + `secretref:`, or managed identity + Key Vault |
| AWS | Secrets Manager, injected via task definition |
| GCP | `--set-secrets=VAR=name:latest` |

```python
import os, streamlit as st

# works locally (secrets.toml) AND deployed (env vars)
API_URL = os.getenv("API_URL") or st.secrets.get("API_URL", "http://localhost:8000")
```

> **That one line makes the same code run everywhere.** Env vars win when present; `secrets.toml` is the local fallback.

---

## The deployment checklist

- [ ] `--server.address=0.0.0.0` and `--server.headless=true`
- [ ] Health probes on `/_stcore/health`
- [ ] **Sticky sessions / session affinity** if replicas > 1
- [ ] **Timeout raised** for the WebSocket (300s+)
- [ ] `min-replicas 1` — no cold start for humans
- [ ] `API_URL` points at the internal API address
- [ ] Secrets injected, never baked into the image
- [ ] Auth in front of the app, not inside it
- [ ] Non-root user in the Dockerfile
- [ ] Versioned image tag, not `:latest`

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Container hangs at startup | Missing `--server.headless=true` | Add it |
| Connection refused from browser | Bound to `127.0.0.1` | `--server.address=0.0.0.0` |
| "Please wait..." forever | WebSocket blocked or timing out | Raise proxy/LB timeout; enable WebSocket support |
| State randomly resets mid-use | **No sticky sessions with >1 replica** | `sessionAffinity` / `--session-affinity` / ALB stickiness |
| Dashboard can't reach the API | Used `localhost` | Use the service/internal name |
| Blank page, no error | App crashed on import | Check container logs |
| Slow first load every time | Scale-to-zero cold start | `--min-instances 1` |
| Works locally, 502 deployed | Health check path wrong | Use `/_stcore/health` |
| Secrets missing when deployed | Only in `secrets.toml`, not injected | Set env vars on the platform |
| Uploads fail over ~200 MB | Default max upload size | `maxUploadSize` in `config.toml` |

More: [[I HAVE A PROBLEM]]

## Related

[[Streamlit]] · [[Streamlit vs Django]] · [[Docker deep dive]] · [[Kubernetes and AKS]] · [[FastAPI data and deployment]] · [[Pipeline setup - Local]] · [[Pipeline setup - Azure]] · [[Pipeline setup - AWS]] · [[Pipeline setup - GCP]]
