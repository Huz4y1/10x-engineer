---
tags: [azure, storage]
status: not-started
---

# Azure Fundamentals

> **What this is:** how Azure organises things, and how it decides who's allowed to touch what.
> **Why you care:** every "403 Forbidden" and "authorization failed" you'll ever hit in this roadmap comes from this note. Learn it once and those errors become obvious instead of mysterious.

---

## The idea in plain English

Azure is a very large building full of computers you can rent.

To use it, you need three things sorted:

1. **Where does my stuff live?** → subscriptions and resource groups (the filing system)
2. **Who am I?** → Entra ID (the ID badge)
3. **What am I allowed to do?** → RBAC (what doors the badge opens)

Almost every Azure problem is one of these three being wrong.

---

## 1. The filing system

```
Tenant                    ← your whole organisation ("Contoso Ltd")
 └── Subscription         ← the billing boundary. One invoice per subscription.
      └── Resource Group  ← a folder for related things
           └── Resource   ← an actual thing: a storage account, a database, a VM
```

**Tenant** — your organisation's identity space. As an individual learner you have one and can ignore it.

**Subscription** — where the money comes from. Everything inside it appears on one bill. Companies often have several (`prod`, `dev`, `sandbox`) so costs are separated.

**Resource group** — just a folder, but with two real powers:
- **Delete the group, delete everything in it.** This is your best cost-control tool while learning.
- Permissions can be granted at the group level, so they apply to everything inside.

**Rule of thumb:** one resource group per project per environment. `retail-dev-rg`, `retail-prod-rg`. Things with the same lifecycle — created together, deleted together — go in the same group.

### Regions

Every resource lives in a physical **region** (`uksouth`, `westeurope`, `eastus`).

Three things follow from that:

- **Latency.** Your app and your database should be in the same region. Cross-region round trips cost tens of milliseconds each, and they add up fast.
- **Egress cost.** Moving data *out* of a region costs money. Moving it *within* one is free. A pipeline that reads from a lake in `uksouth` and writes to a database in `eastus` is quietly charging you.
- **Data residency.** Some data legally has to stay in a country.

> **Rule of thumb:** pick one region at the start of the capstone and put *everything* in it. Mixing regions on a learning project buys you nothing and costs you both money and speed.

### Naming things

Some Azure names are **globally unique across all of Azure, forever** — storage accounts, SQL servers, container registries. That's because they become part of a public URL (`https://retailstore.blob.core.windows.net`). `retailstore` was taken in 2014.

```bash
# a suffix saves you a lot of "name is already in use"
az storage account create --name retailstore$RANDOM ...
```

Naming convention that scales: `<project>-<env>-<type>`, e.g. `retail-dev-sql`. Storage accounts can't have hyphens or capitals, so: `retaildevstore01`.

---

## 2. Identity — Entra ID

**Microsoft Entra ID** is the new name for Azure Active Directory. You'll see both names everywhere; they're the same thing.

It answers one question: **who are you?**

### The kinds of identity

| Identity type | Who/what it is | Use it for |
|---|---|---|
| **User** | A human. You. | Logging into the portal, `az login` |
| **Group** | A bag of users | Assigning permissions to a team at once |
| **Service principal** | An identity for an *application*, with a secret or certificate | A script, a CI pipeline, an app running outside Azure |
| **Managed identity** | A service principal that Azure creates and manages for you, **with no password you ever see** | An app running *inside* Azure. **Prefer this always.** |

### Why managed identities are the good answer

A service principal has a client secret. That secret is a password. It has to be stored somewhere, rotated before it expires, and it can leak.

A managed identity has none of that. You flip a switch on a resource ("give this Container App an identity"), Azure creates one, and the resource can get tokens automatically. There is no secret to store, leak, or rotate.

```python
# Works identically on your laptop (uses your `az login`)
# and in Azure (uses the managed identity). No code change, no secret.
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

credential = DefaultAzureCredential()
client = BlobServiceClient(
    account_url="https://retailstore.blob.core.windows.net",
    credential=credential,
)
```

`DefaultAzureCredential` tries a list of methods in order — environment variables, managed identity, your Azure CLI login, and a few others — and uses the first that works. That's why the same code works in both places.

> **This is the pattern to internalise.** Whenever you're about to paste a connection string with a key in it, ask: *could this be `DefaultAzureCredential` instead?* Usually yes.

### Creating a service principal (when you genuinely need one)

For GitHub Actions or a script running outside Azure:

```bash
az ad sp create-for-rbac \
  --name retail-ci \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/<sub-id>/resourceGroups/retail-rg
```

That prints a client ID, tenant ID, and a secret — **the only time the secret is ever shown**. Put it straight into GitHub Secrets or Key Vault. Never into a file.

Note the `--scopes`: it's limited to one resource group. Don't scope it to the whole subscription out of laziness.

---

## 3. Permissions — RBAC

**RBAC** = Role-Based Access Control. It answers: **what are you allowed to do?**

Every permission is a sentence with three parts:

> **WHO** gets **WHAT ROLE** on **WHICH SCOPE**

- **Who** — a user, group, service principal, or managed identity
- **What role** — a named bundle of permissions ("Storage Blob Data Reader")
- **Which scope** — subscription, resource group, or a single resource

```bash
az role assignment create \
  --assignee <object-id-or-email> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/retail-rg/providers/Microsoft.Storage/storageAccounts/retailstore01
```

### Permissions flow downhill

A role granted at the **resource group** level applies to every resource in it. Granted at the **subscription** level, it applies to everything. This is why over-granting is easy and dangerous — "just give it Contributor on the subscription" is how a test script ends up able to delete production.

> **Principle of least privilege:** grant the smallest role, at the smallest scope, that makes the thing work. If it breaks, widen deliberately — don't start wide.

### The trap that will get you: control plane vs. data plane

This is the single most confusing thing in Azure, and it costs everyone an afternoon at least once.

There are **two separate permission systems**:

| | Control plane | Data plane |
|---|---|---|
| Governs | The resource *itself* — create, delete, configure, read keys | The *contents* — the actual files and rows |
| Example roles | `Owner`, `Contributor`, `Reader` | `Storage Blob Data Reader`, `Storage Blob Data Contributor` |
| Lets you | Make a storage account, change its settings | Read and write blobs inside it |

**Being `Owner` of a storage account does not let you read the files in it.**

You can create the account, delete the account, change its firewall — and still get `403 AuthorizationPermissionMismatch` when you try to list a blob, because you lack a *Data* role.

> **If you see `AuthorizationPermissionMismatch`, this is almost always why.** Assign yourself `Storage Blob Data Contributor` and try again. Note the word **Data** in the role name — that's the tell.

### Roles you'll actually use

| Role | Gives |
|---|---|
| `Reader` | Look at resources and their config. Not their data. |
| `Contributor` | Create/modify/delete resources. Not their data, and can't grant permissions. |
| `Owner` | Contributor + the ability to grant permissions to others |
| `Storage Blob Data Reader` | Read blobs |
| `Storage Blob Data Contributor` | Read/write/delete blobs |
| `Key Vault Secrets User` | Read secret values |

### Checking what you've got

```bash
az role assignment list --assignee <your-email> --all --output table
```

When something's forbidden, run this first. Usually the answer is right there.

---

## 4. Key Vault — where secrets live

A **Key Vault** is a managed safe for three kinds of thing:

- **Secrets** — connection strings, API keys, passwords
- **Keys** — encryption keys (the key never leaves the vault; you send data *to* it)
- **Certificates** — TLS certs, with auto-renewal

### Why bother, when `.env` works?

- **One place to rotate.** Change the password once in the vault; every app picks it up. No redeploys.
- **Audit log.** You can see exactly who read which secret, and when.
- **No secret in the image, repo, or config.** The app authenticates *as itself* (managed identity) and asks for the value at runtime.

### Using it

```bash
az keyvault create --name retail-kv-01 --resource-group retail-rg --location uksouth

az keyvault secret set \
  --vault-name retail-kv-01 \
  --name sql-connection-string \
  --value "Driver={ODBC Driver 18 for SQL Server};Server=..."

# let yourself read it (data-plane role — see the trap above)
az role assignment create \
  --assignee <your-email> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/retail-rg/providers/Microsoft.KeyVault/vaults/retail-kv-01
```

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

client = SecretClient(
    vault_url="https://retail-kv-01.vault.azure.net",
    credential=DefaultAzureCredential(),
)
conn_str = client.get_secret("sql-connection-string").value
```

> **Fetch secrets once at application startup, not per request.** Every `get_secret` is a network call — doing it inside a FastAPI endpoint adds latency to every single request. Load them in the `lifespan` startup event ([[FastAPI data and deployment]]).

> **Note the name rule:** Key Vault secret names allow letters, digits and hyphens only. No underscores. `sql-connection-string`, not `sql_connection_string`.

---

## 5. Cost control — read this before you build anything

You're using your own money. Three habits, set up today:

**1. Set a budget alert.**
`Cost Management + Billing → Budgets → Add`. Set it to something that would annoy you (£20). You get an email at 80% and 100%. This is the difference between a £30 mistake and a £600 one.

**2. Know what bills by the hour even when idle.**

| Resource | Idle cost |
|---|---|
| Storage account (ADLS) | Pennies. Leave it. |
| Azure SQL (serverless tier) | Auto-pauses. Cheap. |
| Azure SQL (provisioned tier) | **Bills 24/7 regardless of use** |
| Databricks cluster | **Bills per minute while running** |
| Container Apps (scale-to-zero) | Free when nobody's calling it |

**3. Delete the resource group after each session.** Everything in this roadmap is recreatable from a CLI script in a couple of minutes. That's the whole point of doing it in the CLI.

> Set your Databricks clusters to **auto-terminate after 15 minutes** the moment you create them ([[Databricks and Delta Lake]]). A cluster left on over a weekend is the classic expensive learning experience.

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 AuthorizationPermissionMismatch` on a blob | You have a control-plane role but no **Data** role | Assign `Storage Blob Data Contributor` |
| `AuthorizationFailed` creating a resource | No `Contributor` at that scope, or wrong subscription | `az account show`; `az role assignment list --assignee you@x.com --all` |
| Role assigned but still denied | Assignments take a few minutes to propagate | Wait 5 minutes, then `az logout && az login` to refresh the token |
| `DefaultAzureCredential` fails locally | Not logged in | `az login` |
| `DefaultAzureCredential` fails in Azure | Managed identity not enabled, or has no role | Enable identity on the resource, then assign it a data role |
| "Storage account name already taken" | Storage names are globally unique | Add a suffix: `retailstore$RANDOM` |
| Key Vault "secret not found" | Name has an underscore, or wrong vault | Hyphens only; check `az keyvault secret list` |
| Surprise bill | Cluster or provisioned DB left running | `az group delete`; set the budget alert |

---

## Practice checklist

- [ ] Resource groups and regions — how resources are organized and billed
- [ ] Azure Active Directory (Entra ID): users, groups, service principals
- [ ] RBAC: roles, scopes, and assigning access without over-granting it
- [ ] **Control plane vs. data plane** — why `Owner` can't read a blob
- [ ] Managed identities — how services authenticate to each other without stored secrets
- [ ] Azure Key Vault — where secrets, connection strings, and keys actually belong

## Hands-on

- [ ] **Set a budget alert on your subscription. Do this first.**
- [ ] Create a resource group, a service principal, and grant it scoped access to one resource — nothing more
- [ ] Store a connection string in Key Vault and read it from a Python script using `DefaultAzureCredential`
- [ ] Deliberately trigger `AuthorizationPermissionMismatch` by omitting the data role, then fix it — so you recognise the error instantly later

## Resources

- [MS Learn: Azure Fundamentals](https://learn.microsoft.com/training/paths/azure-fundamentals/)
- [MS Learn: Azure Data Fundamentals path](https://learn.microsoft.com/training/paths/azure-data-fundamentals/)
- [Azure built-in roles reference](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
- [DefaultAzureCredential explained](https://learn.microsoft.com/azure/developer/python/sdk/authentication-overview)

## Next

[[ADLS Gen2]]
