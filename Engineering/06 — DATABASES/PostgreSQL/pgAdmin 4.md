---
tags: [postgresql, pgadmin, tools, databases, guide]
---

# pgAdmin 4

**pgAdmin is a point-and-click app for PostgreSQL.** You can see your databases and tables in a tree, run SQL, look at and edit data, draw diagrams, and back up — without typing `psql` commands.

Hub: [[PostgreSQL]] · The SQL to run in it: [[PostgreSQL SQL]] · Types: [[PostgreSQL data types]]

> ✅ **Written against pgAdmin 4 version 9.17** — the one installed with PostgreSQL 18 on this machine. Menu names and shortcuts come from that version's own manual. Older tutorials online often show different screens.

---

## What pgAdmin is — and isn't

- **pgAdmin is not the database.** It's a window onto one. PostgreSQL runs as a separate program (a *server*) — on your computer, in Docker, or in the cloud. pgAdmin connects to it, the same way a browser connects to a website.
- **One pgAdmin can connect to many servers** — your local Postgres, a test database in Docker, and your Azure or AWS database, all in one tree.
- **Anything you click, pgAdmin turns into SQL.** Most dialogs have an **SQL** tab showing exactly what it will run. That's the best way to learn the SQL behind each action.

---

## Installing it

### Windows

**The easy way:** the PostgreSQL installer from EDB (enterprisedb.com) installs pgAdmin alongside the database — that's how it got onto this machine (`C:\Program Files\PostgreSQL\18\pgAdmin 4`).

**pgAdmin on its own** (for example, if your database is in the cloud):

```powershell
winget install PostgreSQL.pgAdmin
```

Open it from the Start menu: **pgAdmin 4**. The first time, it asks you to set a **master password** — this protects the database passwords pgAdmin saves for you. It isn't your database password.

### WSL, or anywhere with Docker — run pgAdmin as a web page

```bash
docker run -d --name pgadmin -p 5050:80 \
  -e PGADMIN_DEFAULT_EMAIL=you@example.com \
  -e PGADMIN_DEFAULT_PASSWORD=choose-a-password \
  dpage/pgadmin4
```

Open **http://localhost:5050** in your browser and log in with that email and password.

> ⚠️ **Connecting from pgAdmin-in-Docker to Postgres on your computer:** the host is **`host.docker.internal`**, not `localhost`. Inside a container, `localhost` means the container itself. If both run in the same Docker Compose file, use the Postgres **service name** (e.g. `db`) instead ([[Docker deep dive]]).

---

## The screen

```
┌──────────────────────────────────────────────────────────────────┐
│ File  Object  Tools  Help                                          │
├────────────────────┬─────────────────────────────────────────────┤
│ Object Explorer    │  Dashboard | Properties | SQL | Statistics  │
│                    │  | Dependencies | Dependents | Processes    │
│ ▾ Servers          │                                             │
│   ▾ Local PG 18    │   (information about whatever is            │
│     ▾ Databases    │    selected on the left)                    │
│       ▾ shop       │                                             │
│         ▾ Schemas  │   Query Tool tabs open here too             │
│           ▾ public │                                             │
│             ▾ Tables                                             │
│                customers                                         │
│                orders                                            │
└────────────────────┴─────────────────────────────────────────────┘
```

| Part | What it's for |
|---|---|
| **Object Explorer** (left) | The tree: Servers → Databases → Schemas → Tables → Columns. **Right-click anything** for its actions |
| **Dashboard** tab | Live activity: sessions, transactions per second |
| **Properties** tab | Settings of the selected object |
| **SQL** tab | The SQL that would recreate the selected object — great for learning, and for copying a table's definition |
| **Statistics** tab | Size, row counts, how often it's scanned |
| **Dependencies / Dependents** | What this object needs, and what needs it — check before you drop something |

> **Your tables live at `Servers → your server → Databases → your database → Schemas → public → Tables`.** That's the path people get lost in. `public` is the default schema ([[PostgreSQL SQL]]).

---

## 1. Connecting to a server

Right-click **Servers** → **Register** → **Server…**

The dialog has these tabs: **General**, **Connection**, **Parameters**, **SSH Tunnel**, **Advanced**, **Post Connection SQL**, **Tags**. You normally only need the first three.

### General tab

| Field | Put |
|---|---|
| **Name** | Anything — it's just the label in the tree, e.g. `Local PG 18` or `Azure prod` |
| Server group | Leave as `Servers` |

### Connection tab

| Field | Local Postgres | Postgres in Docker | Azure | AWS RDS |
|---|---|---|---|---|
| **Host name/address** | `localhost` | `localhost` (if you mapped `-p 5432:5432`) | `yourserver.postgres.database.azure.com` | `yourdb.xxxx.eu-west-2.rds.amazonaws.com` |
| **Port** | `5432` | `5432` | `5432` | `5432` |
| **Maintenance database** | `postgres` | `postgres` | `postgres` | `postgres` |
| **Username** | `postgres` | whatever `POSTGRES_USER` was | your admin user | your master username |
| **Password** | the one you set when installing | `POSTGRES_PASSWORD` | your admin password | from Secrets Manager |
| **Save password?** | ✅ on your own machine | ✅ | Your choice | Your choice |

> **"Maintenance database"** is just the database pgAdmin connects to first. `postgres` always exists. Your own databases still show up in the tree.

### Parameters tab — encryption

For **cloud databases, set SSL mode to `require`** here. Azure and AWS refuse or warn about unencrypted connections, and without it your password crosses the internet readable. For your own computer, the default is fine.

| SSL mode | Means |
|---|---|
| `prefer` *(default)* | Encrypt if the server supports it |
| **`require`** | Always encrypt — **use for cloud** |
| `verify-full` | Encrypt **and** check the server's certificate — the most secure; needs the provider's CA certificate |

Click **Save**. The server appears in the tree; click the ▸ to connect.

### SSH Tunnel tab

For a database that's only reachable through a jump server (a "bastion"): turn on **Use SSH tunneling**, and give the jump server's host, username and key. pgAdmin connects through it.

---

## 2. Creating a database and a table

**A database:** right-click **Databases** → **Create** → **Database…** → type a name (e.g. `shop`) → **Save**.

**A table** — the clicking way: open the database → **Schemas** → **public** → right-click **Tables** → **Create** → **Table…**

| Tab | Do this |
|---|---|
| **General** | Name: `customers` |
| **Columns** | Click **+** for each column: name, data type, *Not NULL?*, *Primary key?* |
| **Constraints** | Unique, check and foreign-key constraints |
| **SQL** | ✅ **Read this before saving** — it's the `CREATE TABLE` pgAdmin will run |

> **Faster: just write the SQL in the Query Tool.** Clicking through columns is fine for learning, but a `CREATE TABLE` statement is quicker, can be saved in a file, and can go into git. The **SQL** tab exists to teach you that SQL.

**Identity (auto-number) columns:** in the Columns tab, edit the column → its **Constraints** tab → set the type to **IDENTITY** (the choices are NONE / IDENTITY / GENERATED). See [[PostgreSQL data types]] for why that beats `serial`.

---

## 3. The Query Tool — where you'll spend most of your time

Open it: select a database in the tree, then **Tools → Query Tool** (or **Alt+Shift+Q**).

```
┌──────────────────────────────────────────────────────┐
│ ▶ Execute   ⚡ Explain   💾   ⟲ Commit  ⟲ Rollback   │  ← toolbar
├──────────────────────────────────────────────────────┤
│ SELECT name, country                                  │
│ FROM customers                                        │  ← the SQL editor
│ WHERE country = 'GB';                                 │
├──────────────────────────────────────────────────────┤
│ Data Output | Messages | Notifications | Explain     │  ← results
│  name │ country                                       │
│  Ada  │ GB                                            │
└──────────────────────────────────────────────────────┘
```

### Running SQL — the three ways

| Key | Runs | Use when |
|---|---|---|
| **F5** | **Everything** in the editor — or only your **highlighted** text | Running a whole script, or a highlighted part of it |
| **Alt+F5** | Only **the statement your cursor is in** | You've got many queries in one tab and want to run just one |
| **F7** | `EXPLAIN` — shows the plan as a diagram, doesn't run the query | "Why is this slow?" |
| **Shift+F7** | `EXPLAIN ANALYZE` — runs it and shows real timings | Same, with real numbers |

> **Highlight, then F5, is the habit that saves you.** Keep a scratchpad of queries in one tab and run them one at a time, instead of running the whole file by accident.

> ⚠️ **Shift+F7 really runs the query.** On an `UPDATE` or `DELETE` it changes data. Wrap it in `BEGIN; ... ROLLBACK;` ([[PostgreSQL SQL]]).

### Results tabs

| Tab | Shows |
|---|---|
| **Data Output** | The rows your query returned |
| **Messages** | Errors, and "UPDATE 3" / how long it took |
| **Explain** | The plan diagram after F7 / Shift+F7 — hover each box for details |
| **Notifications** | Messages from `LISTEN` / `NOTIFY` |

### Auto-commit — is my change saved?

Click the small arrow next to **Execute** to see **Auto commit?** and **Auto rollback on error?**.

- **Auto commit on** *(the default)*: every statement is saved the moment it runs. There's no undo.
- **Auto commit off**: your changes are only saved when you click **Commit** (**Shift+Ctrl+M**); **Rollback** (**Shift+Ctrl+R**) throws them away.

> ⚠️ **With auto-commit on, `DELETE FROM customers;` without a `WHERE` is permanent the moment you press F5.** When you're working on real data, either turn auto-commit off, or type `BEGIN;` first and check the row count in **Messages** before running `COMMIT;`.

> ⚠️ **With auto-commit off, remember to commit.** Changes you haven't committed are invisible to everyone else, and they're lost when you close the tab. An uncommitted transaction can also hold locks that block other people.

### Downloading results

**F8** saves the results to a CSV file (the format is set in **File → Preferences → Query Tool → CSV/TXT Output**).

### Handy editor keys

| Key | Does |
|---|---|
| **Ctrl+Space** | Auto-complete table and column names |
| **Ctrl+/** | Comment / uncomment the selected lines |
| **Ctrl+F** / **Ctrl+Shift+F** | Find / replace |
| **Ctrl+L** | Go to line |
| **Ctrl+Alt+L** | Clear the editor |
| **Shift+Ctrl+U** | Change the selected text's case |
| **Ctrl+S** / **Ctrl+O** | Save / open a `.sql` file |
| **Alt+Shift+Q** | Cancel a query that's taking too long |

---

## 4. Looking at and editing data

Right-click a table and pick **View First 100 Rows**, **View Last 100 Rows**, **View All Rows** or **View Filtered Rows…** (older versions put these under a **View/Edit Data** submenu).

You get a spreadsheet-like grid:

- **Change a value:** double-click the cell, type, press Enter
- **Add a row:** the add-row button (**Alt+Ctrl+A**)
- **Delete rows:** select them, then the delete button (**Alt+Shift+D**)
- **Save:** **F6**, or the **Save Data** button — **nothing is written until you save**

> ⚠️ **Editing needs a primary key.** Without one pgAdmin can't tell which row you changed, so the grid is read-only. That's one of many reasons every table should have a primary key.

> **Use "View First 100 Rows" on big tables.** "All Rows" on a table with 50 million rows tries to load all of them.

---

## 5. Importing and exporting CSV files

Right-click a table → **Import/Export Data…**

| Setting | For a normal CSV |
|---|---|
| **Import / Export** switch | Import = file → table. Export = table → file |
| **Filename** | Pick the file |
| **Format** | `csv` (also `text` and `binary`) |
| **Options → Header** | ✅ On if the first row is column names |
| **Options → Delimiter** | `,` — or `;`, tab or `|` if your file uses those |
| **Columns** | Untick columns the file doesn't have (e.g. an auto-numbered `id`) |

This runs Postgres's `\copy` behind the scenes — the fast way to load data ([[PostgreSQL SQL]]).

> ⚠️ **"invalid input syntax" on import** means a value in the file doesn't match the column's type — a date in the wrong format, text in a number column. The **Messages** show the line number. Fix the file, or load into an all-`text` staging table first and convert with SQL.

---

## 6. Backing up and restoring

### Backup

Right-click a database → **Backup…**

| Setting | Choose |
|---|---|
| **Filename** | Where to save it |
| **Format** | **Custom** — see the table below |
| **Data Options** tab | *Only data*, *only schema*, or both (default) |

| Format | Is | Restore with |
|---|---|---|
| **Custom** | Compressed archive | pgAdmin **Restore**, or `pg_restore` — can restore just one table ✅ |
| Plain | A readable `.sql` text file | Open it in the Query Tool, or `psql` |
| Tar | A `.tar` archive | `pg_restore` |
| Directory | A folder, one file per table | `pg_restore` — can back up in parallel for big databases |

pgAdmin runs the standard `pg_dump` tool for you. The command-line version:

```bash
pg_dump -h localhost -U postgres -Fc -f shop.dump shop         # -Fc = custom format
pg_restore -h localhost -U postgres -d shop_copy shop.dump      # into an existing empty database
```

### Restore

Right-click a database (usually a new, empty one) → **Restore…** → pick the file → **Restore**.

> ⚠️ **A backup you've never restored is only a hope.** Restore it into a new database once, and check your tables are there ([[Security in practice]]).

> **Cloud databases back themselves up automatically** — Azure and AWS keep daily snapshots and let you restore to any minute in the last week or more. A `pg_dump` is still useful for copying data to your laptop ([[Azure Database for PostgreSQL]], [[AWS RDS for PostgreSQL]]).

---

## 7. The ERD tool — diagrams of your tables

Right-click a database → **ERD For Database**.

You get a diagram of every table, its columns, and lines for the foreign keys between them. You can also **design** tables here: add tables and relationships on the diagram, then use the **Generate SQL** button to get the `CREATE TABLE` script.

> **Great for understanding a database someone else built** — the foreign-key lines show how everything connects in seconds ([[Data modeling]]).

---

## 8. Seeing what the server is doing

Select a server → **Dashboard** tab: live charts of sessions and transactions.

**A query stuck for minutes?** Find it with SQL in the Query Tool — this works in every version of pgAdmin and in `psql`:

```sql
SELECT pid, usename, state, now() - query_start AS running_for, left(query, 60) AS query
FROM pg_stat_activity
WHERE state <> 'idle' AND pid <> pg_backend_pid()      -- skip idle sessions and this one
ORDER BY running_for DESC NULLS LAST;
```

Then, using the `pid` from that list:

```sql
SELECT pg_cancel_backend(12345);      -- stop that one query (the session stays connected) - try this first
SELECT pg_terminate_backend(12345);   -- disconnect the whole session - if cancelling didn't work
```

Both return `true` if they found the process. More admin queries: [[PostgreSQL reference]].

> ⚠️ **"idle in transaction" sessions are a warning sign.** Someone started a transaction and never committed — often a Query Tool tab with auto-commit off. It can hold locks and block other queries.

---

## Main-window shortcuts

| Key | Does |
|---|---|
| **Alt+Shift+Q** | Open the Query Tool |
| **Alt+Shift+V** | View data of the selected table |
| **Alt+Shift+N** | Create an object |
| **Alt+Shift+E** | Edit the selected object's properties |
| **Alt+Shift+D** | Delete the selected object |
| **Alt+Shift+S** | Search objects — find a table by name |
| **Ctrl+Shift+F** | Quick search |
| **F5** *(in the tree)* | Refresh — **use this when a new table doesn't show up** |

> **New table not in the tree?** The tree doesn't refresh itself after you run `CREATE TABLE` in the Query Tool. Right-click **Tables → Refresh** (or F5 with the tree selected).

> Every shortcut can be changed in **File → Preferences → Keyboard shortcuts**.

---

## Common problems

| You see | Cause | Fix |
|---|---|---|
| *could not connect to server: Connection refused* | Postgres isn't running, or wrong port | Start the service (Windows: *Services → postgresql-x64-18*); check port 5432 |
| *password authentication failed* | Wrong password or username | Re-type it; right-click the server → **Clear Saved Password** |
| *no pg_hba.conf entry* | The server doesn't allow your IP or requires SSL | Cloud: add your IP to the firewall **and** set SSL mode `require` |
| Timeout connecting to Azure/AWS | Your IP isn't allowed | Add it to the Azure firewall / AWS security group |
| New table not visible | Tree not refreshed | Right-click → **Refresh** |
| Can't edit data in the grid | Table has no primary key | Add one |
| From Docker pgAdmin, `localhost` fails | `localhost` = the container | Use `host.docker.internal` |
| Forgot the master password | It protects saved passwords | Click **Reset Master Password** on the password prompt — this also clears the saved database passwords |

## Related

[[PostgreSQL]] · [[PostgreSQL SQL]] · [[PostgreSQL data types]] · [[PostgreSQL reference]] · [[Azure Database for PostgreSQL]] · [[AWS RDS for PostgreSQL]] · [[Data modeling]] · [[Setting up a dev machine]]
