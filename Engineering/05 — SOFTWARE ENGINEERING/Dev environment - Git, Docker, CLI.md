---
tags: [foundations, git, docker, cli]
status: not-started
---

# Dev Environment: Git, Docker, CLI

> **What this is:** the three tools that surround your code — saving its history, packaging it so it runs anywhere, and creating cloud resources by typing instead of clicking.
> **Why you care:** every later stage assumes these are muscle memory. Nobody teaches them again; they're just expected.

---

## Part 1 — Git

### The idea in plain English

Git is a **save-game system for code**.

You're playing a game. You save before the boss fight. If you die, you reload. If you want to try a completely different strategy, you make a *second* save file and mess about there without touching the first one.

That's Git. The saves are **commits**. The parallel save files are **branches**.

The difference from a game: Git saves *the difference* between states, not a full copy. So you can have 10,000 saves and it costs almost nothing.

### The four places your code lives

This is the bit that confuses everyone. There are four, not two:

```
Working directory  →  Staging area  →  Local repo   →  Remote (GitHub)
(your actual files)   (git add)       (git commit)    (git push)
```

1. **Working directory** — the files you're editing right now.
2. **Staging area** ("the index") — a waiting room. You put changes here to say *"this batch belongs in the next save."* This is why `git add` exists as a separate step: it lets you commit 3 of your 10 changed files.
3. **Local repository** — your machine's full history. `git commit` moves things here. **Still offline.**
4. **Remote** — GitHub. `git push` sends your local history up.

> **The most common beginner mistake:** committing and thinking it's on GitHub. It isn't. `commit` is local. `push` is what leaves your laptop.

### The daily loop

```bash
git status                          # what's changed? RUN THIS CONSTANTLY.
git switch -c feature/add-forecast  # make a new branch and move onto it
# ...edit files...
git add .                           # stage everything changed
git add api/main.py                 # or stage just one file
git commit -m "Add forecast endpoint"
git push -u origin feature/add-forecast   # -u only needed the first time per branch
```

Then open a **pull request** on GitHub: "please merge my branch into main." Someone reviews it, CI runs the tests ([[Testing and CI-CD]]), and it gets merged.

> Write commit messages in the imperative: "Add forecast endpoint", not "Added" or "adding". It reads as *"applying this commit will Add forecast endpoint"*. It's a convention, but it's universal.

### Branching: why not just work on `main`?

`main` should always be working code. If you edit `main` directly and break it, everyone's broken.

A **branch** is a separate line of work. You break things freely, and `main` is untouched until you deliberately merge.

```bash
git switch main            # go back to main
git pull                   # get everyone else's latest work
git switch -c fix/null-customer-ids
```

> `git switch` and `git restore` are the modern replacements for the overloaded `git checkout`. `checkout` did about five unrelated jobs, which is why it was confusing. Use the new ones.

### Merge vs. rebase

Both combine two branches. They differ in what the history *looks like* afterwards.

**Merge** — keeps the true story, including the fact that work happened in parallel. Creates an extra "merge commit".

```
main:    A---B---C-------M
                  \     /
feature:           D---E
```

**Rebase** — rewrites your commits so they *appear* to have been made after the latest `main`. Straight line, tidier history, but it's a fiction.

```bash
git switch feature/add-forecast
git rebase main
```

```
main:    A---B---C
                  \
feature:           D'---E'    (D and E rewritten on top of C)
```

> **The one rule that actually matters:** **never rebase a branch other people have already pulled.** Rebase rewrites commit IDs. If someone else has the old IDs, their history and yours now disagree and it's genuinely horrible to untangle. Rebase your own unpushed local work freely; leave shared branches alone.
>
> Safe default: rebase your feature branch onto main *before* opening a PR, then merge the PR normally.

### Conflicts

A conflict happens when two branches changed **the same lines** of the same file. Git can't guess which one wins, so it asks you.

The file gets markers stuffed into it:

```python
<<<<<<< HEAD
timeout = 30
=======
timeout = 60
>>>>>>> feature/add-forecast
```

- Above `=======` is **what's already there** (the branch you're on).
- Below is **what's coming in**.

You fix it by editing the file so it's correct — **delete all three marker lines**, keep whatever the right answer is (possibly a mix, possibly neither). Then:

```bash
git add the_conflicted_file.py
git rebase --continue     # or `git commit` if you were merging
```

> Panicking mid-conflict? `git rebase --abort` or `git merge --abort` puts everything back exactly as it was. Nothing is lost. Use it freely — knowing you have an undo button is what makes conflicts non-scary.

### Undoing things — the table you'll come back to

| I want to… | Command | Destroys work? |
|---|---|---|
| Discard changes to one file | `git restore path/to/file.py` | **Yes** — that file's edits are gone |
| Unstage a file (keep the edits) | `git restore --staged file.py` | No |
| Fix the last commit's message | `git commit --amend -m "Better message"` | No (but rewrites history — don't do it after pushing) |
| Undo the last commit, keep the changes as edits | `git reset --soft HEAD~1` | No |
| Undo the last commit and throw the changes away | `git reset --hard HEAD~1` | **Yes** |
| Undo a commit that's already pushed | `git revert <commit-hash>` | No — makes a *new* commit that cancels it out. **This is the safe one for shared branches.** |
| Stash work temporarily to switch branches | `git stash` then later `git stash pop` | No |
| See what you did (even "lost" commits) | `git reflog` | No — this is the emergency undo-the-undo |

> `git reflog` is your safety net. It logs every position `HEAD` has been in for ~90 days, including after a `--hard` reset. If you think you've destroyed work, check reflog *before* despairing — you almost certainly haven't.

### `.gitignore` — the file that saves your job

Never commit: secrets, credentials, virtual environments, data files, model weights, `__pycache__`.

```gitignore
# .gitignore
.env
.venv/
__pycache__/
*.pyc
data/
*.onnx
*.pth
.ipynb_checkpoints/
.DS_Store
```

> **If you ever commit a secret, rotating the secret is the only real fix.** Deleting it in a later commit doesn't help — it's still in the history, forever, and GitHub's history is public if the repo is. Change the password/key at the source.

---

## Part 2 — Python environments

### The problem

Project A needs `pandas 1.5`. Project B needs `pandas 2.2`. If you install packages globally, one of them breaks. Every time.

A **virtual environment** is a private box of packages that belongs to one project. Project A gets its own `pandas`, Project B gets its own. They never see each other.

### Use `uv`

There are five tools that do this (`venv`, `pip`, `virtualenv`, `poetry`, `conda`, `uv`). Pick one and stop thinking about it. **Use `uv`** — it's the current best, it's 10–100× faster than pip, and it replaces all of the above.

```bash
# install uv once (see docs for your OS)
uv init retail-platform          # new project with a pyproject.toml
cd retail-platform
uv add pandas fastapi torch      # installs AND records the dependency
uv run python main.py            # runs inside the env — no "activate" needed
uv sync                          # recreate the exact env from the lockfile
```

The two files that matter, both committed to Git:

- **`pyproject.toml`** — what you *asked for* (`pandas >= 2.0`). Human-written.
- **`uv.lock`** — what you actually *got*, every package pinned to an exact version, including dependencies-of-dependencies. Machine-written.

> **Why the lockfile matters:** it's the difference between "it works on my machine" and "it works on every machine, and on the CI server, and in production, in six months' time." Commit it.

If you're on an older project using pip, the equivalents are `python -m venv .venv`, `.venv\Scripts\activate` (Windows) or `source .venv/bin/activate` (Mac/Linux), and `pip freeze > requirements.txt`.

---

## Part 3 — Docker

### The idea in plain English

A virtual environment solves "which Python packages". Docker solves **everything else**: which Python version, which operating system, which system libraries, which environment variables, which files.

Think of it as **shipping the whole computer, not just the code.**

The classic analogy: a shipping container. It doesn't matter what's inside or whether it's going on a ship, a train, or a lorry — the container is a standard shape, so everything that handles containers can handle yours.

### Image vs. container — get this straight

| | What it is | Analogy |
|---|---|---|
| **Image** | A frozen, read-only blueprint. Built once. | A recipe, or a class in Python |
| **Container** | A running instance of an image. | The actual cake, or an object |

One image → many containers. You **build** an image, you **run** a container.

### A `Dockerfile`, line by line

```dockerfile
# 1. Start from someone else's image, so you don't build Linux + Python yourself
FROM python:3.12-slim

# 2. Where inside the container we'll work
WORKDIR /app

# 3. Copy ONLY the dependency files first (see the caching note below)
COPY pyproject.toml uv.lock ./

# 4. Install dependencies
RUN pip install uv && uv sync --frozen --no-dev

# 5. NOW copy the rest of the code
COPY . .

# 6. Document which port the app uses
EXPOSE 8000

# 7. What to run when the container starts
CMD ["uv", "run", "uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Why steps 3–5 are split — the single most useful Docker trick.** Each line is a cached **layer**. Docker reuses cached layers until it hits one whose inputs changed, then rebuilds everything below.

Your code changes fifty times a day. Your dependencies change once a month. Copying dependencies first means editing a Python file only invalidates layers 5–7 — the slow `pip install` stays cached. Rebuild goes from 3 minutes to 3 seconds.

Copy everything in one `COPY . .` before installing, and every one-character code change reinstalls PyTorch. People genuinely lose hours a week to this.

> **`--host 0.0.0.0` is not optional.** Inside a container, `127.0.0.1` means "only reachable from inside this container" — you'd get connection refused from your own browser. `0.0.0.0` means "accept connections from outside". This catches everyone once.

### `.dockerignore`

Same idea as `.gitignore`, for build context. Without it, Docker copies your `.venv/`, `.git/`, and 4GB of data into the build.

```
.venv/
.git/
data/
__pycache__/
*.pth
```

### The commands

```bash
docker build -t retail-api .            # build image named "retail-api" from Dockerfile here
docker run -p 8000:8000 retail-api      # run it, map my port 8000 → its port 8000
docker run -p 8000:8000 --env-file .env retail-api    # pass in secrets
docker ps                               # what's running
docker logs <container_id>              # why did it die
docker exec -it <container_id> bash     # get a shell INSIDE the running container
docker system prune -a                  # reclaim disk (images pile up fast)
```

> `-p 8000:8000` is `HOST:CONTAINER`. The left number is what *you* type in your browser. They don't have to match — `-p 3000:8000` works fine.

### `docker-compose` — several containers at once

Your capstone ends up with a FastAPI container *and* a Streamlit container, and locally maybe a database too. Starting three by hand each time is tedious.

```yaml
# docker-compose.yml
services:
  api:
    build: ./api
    ports: ["8000:8000"]
    env_file: .env

  dashboard:
    build: ./dashboard
    ports: ["8501:8501"]
    environment:
      API_URL: http://api:8000      # ← reach the other service by its SERVICE NAME
    depends_on: [api]
```

```bash
docker compose up --build      # start everything
docker compose down            # stop everything
```

> **The networking bit that's genuinely magic:** inside a compose network, services find each other by **service name**. The dashboard talks to `http://api:8000`, not `localhost:8000` — `localhost` inside a container means *that container*, not your machine. This exact mistake accounts for a large share of "my dashboard can't reach my API" problems.

---

## Part 4 — Azure CLI

### Why not just click in the portal?

Clicking works once. But:

- You can't remember in three months what you clicked.
- You can't code-review a click.
- You can't recreate the environment for a colleague.
- You can't put a click in a script.

Typing it means it's repeatable, reviewable, and version-controllable. This is the mindset that eventually becomes Terraform ([[Adjacent tools you will meet]]).

### Getting started

```bash
az login                                   # opens a browser to sign in
az account show                            # which subscription am I in?
az account set --subscription "My Sub"     # switch
```

### The hierarchy

```
Subscription  (the billing account)
  └── Resource Group  (a labelled folder for related things)
        ├── Storage Account
        ├── SQL Server → Database
        └── Databricks Workspace
```

A **resource group** is just a folder. Its superpower: `az group delete` removes *everything* inside it. When you're learning on your own card, putting a whole experiment in one resource group and deleting it afterwards is how you avoid a surprise bill.

### The capstone provisioning, for real

```bash
RG=retail-rg
LOC=uksouth

az group create --name $RG --location $LOC

# storage account with hierarchical namespace = ADLS Gen2 (see [[ADLS Gen2]])
az storage account create \
  --name retailstore$RANDOM \
  --resource-group $RG \
  --location $LOC \
  --sku Standard_LRS \
  --enable-hierarchical-namespace true

# see what you've made
az resource list --resource-group $RG --output table

# tear the whole lot down when you're done
az group delete --name $RG --yes --no-wait
```

> `--output table` on any `az` command turns a wall of JSON into something readable. `--output tsv --query "..."` extracts a single value for use in a script.

### 💸 The cost habit

You are spending your own money. Build these two reflexes now:

- **Set a budget alert** the day you create the subscription: `Cost Management → Budgets`. Set it low (£20). It emails you before, not after.
- **Delete the resource group when you finish a session.** Databricks clusters and SQL databases bill by the hour whether you're using them or not. Everything in this roadmap can be recreated from a script in two minutes.

---

## Part 5 — Secrets

**Never put a password, key, or connection string in your code.** Not even temporarily. Not even "I'll take it out before I commit."

### Locally: a `.env` file

```bash
# .env — LISTED IN .gitignore
AZURE_SQL_CONNECTION_STRING=Driver={ODBC Driver 18 for SQL Server};Server=...
STORAGE_ACCOUNT_KEY=abc123...
```

```python
import os
from dotenv import load_dotenv

load_dotenv()
conn_str = os.environ["AZURE_SQL_CONNECTION_STRING"]
```

> Use `os.environ["KEY"]`, not `os.environ.get("KEY")`. The first crashes immediately with a clear error if the variable is missing. The second returns `None` and you get a baffling failure five functions later.

Commit a `.env.example` with the *names* and no values, so the next person knows what to fill in.

### In the cloud: Key Vault

`.env` files don't belong on servers. Azure Key Vault stores secrets centrally, and your app authenticates to it with a **managed identity** — an identity Azure hands the app automatically, with no password to store anywhere. Covered properly in [[Azure fundamentals]].

---

## When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| Pushed but nothing on GitHub | You committed but didn't push, or you're on a different branch | `git status`, then `git push` |
| "Updates were rejected" on push | Someone else pushed first | `git pull --rebase`, resolve, push again |
| Deleted work by accident | — | `git reflog`, find the hash, `git reset --hard <hash>` |
| Rebase went badly | — | `git rebase --abort` — you're back where you started |
| Docker: connection refused from browser | App bound to `127.0.0.1` inside the container | Add `--host 0.0.0.0` |
| Docker: rebuild takes minutes every time | `COPY . .` before installing deps | Copy dependency files first, install, then copy code |
| Docker: "no such file or directory" | Path is relative to `WORKDIR`, or `.dockerignore` excluded it | `docker exec -it <id> ls -la` and look |
| Container exits instantly | The `CMD` process ended or crashed | `docker logs <id>` — the error is there |
| Compose: service can't reach another | Used `localhost` instead of the service name | `http://api:8000`, not `http://localhost:8000` |
| `az` command "not found" after install | Not on PATH | Restart the terminal |
| Azure "authorization failed" | Wrong subscription, or your account lacks the role | `az account show`; see RBAC in [[Azure fundamentals]] |
| Unexpected Azure bill | A cluster or database left running | `az group delete`; set a budget alert |

---

## Practice checklist

- [ ] Git branching workflow: feature branches, rebasing vs. merging, resolving conflicts
- [ ] The four states of a file: working dir → staged → committed → pushed
- [ ] `git reflog` — knowing your safety net exists
- [ ] A clean per-project Python environment (uv or poetry — pick one and use it everywhere)
- [ ] Docker: images vs. containers, writing a `Dockerfile`, layer caching, `docker-compose` for local multi-service setups
- [ ] Azure CLI: `az login`, `az group create`, `az storage account create` — provisioning from the terminal, not just the portal
- [ ] Environment variables and secrets: never commit a connection string

## Hands-on

- [ ] Containerize a trivial Python script — write the `Dockerfile`, build it, run it, confirm it works with no local Python installed
- [ ] Deliberately create a merge conflict between two branches and resolve it
- [ ] Break the layer cache on purpose, then fix the `Dockerfile` and watch the rebuild time drop
- [ ] Provision an Azure resource group and storage account entirely from the CLI — then delete the group
- [ ] Set a budget alert on your Azure subscription **before** you go any further

## Resources

- [Pro Git (free book)](https://git-scm.com/book/en/v2) — chapters 2 and 3 are the ones that matter
- [Oh Sh*t, Git!?!](https://ohshitgit.com/) — plain-English fixes for "I've broken it"
- [Docker: Get Started](https://docs.docker.com/get-started/)
- [uv documentation](https://docs.astral.sh/uv/)
- [Azure CLI docs](https://learn.microsoft.com/cli/azure/)

## Next

[[Azure fundamentals]]
