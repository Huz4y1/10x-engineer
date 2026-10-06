  

**There are 3 "zones" your code sits in:**

**1. Working Directory** — where you're just editing files normally. Git can see changes but isn't tracking them yet.

**2. Staging Area** — a "waiting room" where you put changes you want to include in your next commit. This is what `git add` does — it moves changes from zone 1 to zone 2.

**3. Committed** — changes are saved into git's history permanently.

Edit files → git add → git commit  
(zone 1) (zone 2) (zone 3)

  

  

initialise a git

```bash
git init  
```

Connecting the git to github

```bash
git remote add origin https://github.com/yourname/project.git
```

Check status of the repo

```bash
git status
```

Add all folders and sub folders that you changes into the repo

```bash
git add . 
```

Can add individual files too

```bash
git add main.cpp
```

Create a snapshot

```bash
git commit -m "Initial project"
```

View history

```bash
git log --oneline
```

Create a new branch and switch to it

```bash
git checkout -b feature 
```

Check which branch you are in

```bash
git branch
```

Switching back to main

```bash
git checkout main
```

upload the snapshot to gitHub

```bash
git push
```

pushing to a branch on gitHub

```bash
git push -u origin feature
```

pulling the latest version of the branch you’re on

```bash
git pull
```

  

Goes back to the last commit but **keeps all your changes staged** (ready to commit again)

```bash
git reset --soft HEAD~1
```

Goes back to that commit and **unstages your changes** but keeps the actual file edits  
So when `git reset --mixed` **unstages** your changes, it's just pushing them back from zone 2 to zone 1 — your code edits are still there, they just need `git add`ing again.

```bash
git reset --mixed HEAD~1
```

Goes back to that commit and **wipes everything** — staged and unstaged changes are gone

```bash
git reset --hard HEAD~1
```

  

Checking exact changes beyond git status

```bash
git diff
```

Show exactly what someone has committed

```bash
git show <commit hash>
```

  

  

**What is a PR (Pull Request) and what is a merge?**

A pull request is sending work upward, you are asking if your work on your branch can be pulled into another branch like from your branch to main/dev. Happens on GitHub.

A merge is pulling work downwards, you are bringing others work from a branch into your branch like from main/dev to your branch. Happens locally in git.

  

How to perform a Pull Request:

First save your changes locally to your git, adding files and commiting, then push your changes to your branch on the actual GitHub platform

```bash
git add .
```

```bash
git commit -m "message about what you have done"
```

```bash
git push origin your--branch
```

now your changes exist both locally in git and on GitHub

Then create the pull request on the GitHub platform

your—branch —> develop

This means I want to pull my work into the develop branch and thats all the steps

  

How to perform a merge:

First go onto the branch you want things to merge into

```bash
git switch name_of_your_branch
```

then do this command

```bash
git fetch origin
git merge origin/name_of_branch_you_want_the_code_from
```
---

[[Dev environment - Git, Docker, CLI]] — branching, rebase vs merge, conflicts, undo, and `.gitignore` for data projects

[[CI-CD pipelines]] — GitHub Actions and Azure DevOps
