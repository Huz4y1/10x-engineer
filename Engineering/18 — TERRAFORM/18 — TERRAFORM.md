---
tags: [moc, terraform]
---

# 18 — TERRAFORM

> Infrastructure defined in files instead of clicked in a portal.

**Why it matters:** Clicking works once. You can't code-review a click, can't repeat it, and can't remember it in six months.

Home: [[ULTIMATE ENGINEER]] · Roadmap: [[THE ULTIMATE ENGINEER ROADMAP]]

---

## Topics

Infrastructure as Code · providers · resources · variables · outputs · modules · **state** · plan · apply · destroy · drift · remote state · dependencies

## Declarative, not imperative

You describe the desired end state; Terraform works out the steps. Run it twice and the second run does nothing — it's already correct. That's **idempotency**, and it's why it beats a shell script of `az` commands.

```hcl
resource "azurerm_storage_account" "lake" {
  name                     = "retailstore01"
  resource_group_name      = azurerm_resource_group.main.name
  account_tier             = "Standard"
  account_replication_type = "LRS"
  is_hns_enabled           = true
}
```

```bash
terraform init
terraform plan     # <- ALWAYS read this
terraform apply
terraform destroy
```

> **`plan` before every `apply`.** It shows exactly what will be created, changed and **destroyed**. A `-/+` on a database means data loss.

## The same infrastructure, four clouds

Change the provider and resource types; the structure stays. See [[Cloud comparison dictionary]].

## State

Terraform tracks what it made in a **state file**. Local by default — which breaks with two people. Real setups store it remotely with locking. **Never commit it** — it can contain secrets.

## Related
[[19 — CLOUD]] · [[Adjacent tools you will meet]] · [[CI-CD pipelines]] · [[26 — SECURITY]]
