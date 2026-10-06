---
tags: [bash, shell, scripting, language]
---

# Bash

The glue. Not a language you write applications in — a language you automate with.

All languages: [[Languages]] · Every command: [[Command reference]] · Setup: [[Setting up a dev machine]]

---

## When to use it, and when to stop

| Use Bash for | Use [[Python]] instead when |
|---|---|
| Running programs in order | You need real data structures |
| Moving and renaming files | You're parsing anything structured |
| CI steps, Docker entrypoints | The script passes ~100 lines |
| One-off pipelines with pipes | You need error handling beyond "exit" |

> **The rule: if you're writing an `if` inside a `for` inside a function, you wanted Python.** Bash is excellent glue and a miserable programming language.

---

## The safety header — put this at the top of every script

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| Flag | Without it |
|---|---|
| `-e` | **The script keeps going after a command fails**, doing the next step on bad data |
| `-u` | A typo'd variable silently becomes empty — `rm -rf "$PREFXI/"` becomes `rm -rf /` |
| `-o pipefail` | `false \| tee log` reports **success**, because only the last command counts |

> ⚠️ **Those three flags prevent the entire category of "my script destroyed something quietly".** There is no reason to omit them.

---

## Variables

```bash
name="huz"                 # NO spaces around = , this is not optional
echo "$name"               # always quote
echo "${name}_suffix"      # braces when the name touches other text

count=$((3 + 4))           # arithmetic
files=$(ls *.csv)          # command substitution
readonly MAX=100
export API_URL="https://…" # visible to child processes
```

> ⚠️ **Quote every variable.** Unquoted `$file` splits on spaces, so `My Report.csv` becomes two arguments. This is the single most common Bash bug.

```bash
${VAR:-default}      # use default if unset or empty
${VAR:?message}      # exit with an error if unset  <- good for required config
${VAR#prefix}        # strip shortest matching prefix
${VAR%.csv}          # strip suffix
${VAR/old/new}       # replace first
${#VAR}              # length
```

---

## Conditionals

```bash
if [[ -f "$file" ]]; then
  echo "exists"
elif [[ -d "$path" ]]; then
  echo "directory"
else
  echo "neither"
fi
```

| Test | True when |
|---|---|
| `-f f` / `-d d` / `-e p` | Regular file / directory / exists |
| `-s f` | File exists **and is not empty** |
| `-z "$s"` / `-n "$s"` | String empty / not empty |
| `"$a" == "$b"` | Strings equal (`=~` for regex) |
| `$a -eq -ne -lt -gt` | Numeric comparison |
| `cmd` | Command exited 0 |

> **Use `[[ ]]`, not `[ ]`.** `[[ ]]` is safer with empty variables, supports `&&`, `||` and `=~`, and doesn't need every variable quoted (though quote them anyway).

---

## Loops

```bash
for f in *.csv; do
  echo "processing $f"
done

for i in {1..10}; do echo "$i"; done

while read -r line; do          # -r means "don't mangle backslashes"
  echo "$line"
done < input.txt

while IFS=, read -r name age; do
  echo "$name is $age"
done < data.csv
```

> ⚠️ **Never `for f in $(ls)`.** It breaks on any filename containing a space. Use a glob: `for f in *.csv`.

> **Always `read -r`.** Without `-r`, backslashes in your data get eaten.

---

## Functions

```bash
process() {
  local input="$1"                       # 'local' or it leaks globally
  local output="${2:-out.txt}"
  [[ -f "$input" ]] || { echo "missing: $input" >&2; return 1; }
  wc -l < "$input" > "$output"
}

process data.csv results.txt || echo "failed"
```

| Variable | Is |
|---|---|
| `$1 $2 …` | Positional arguments |
| `$@` | All arguments, **as separate words** — quote it: `"$@"` |
| `$#` | Number of arguments |
| `$?` | Exit code of the last command |
| `$$` | This script's process ID |

> **`"$@"` not `$@`, and not `$*`.** Only the quoted form preserves arguments containing spaces.

---

## Pipes and redirection

```bash
cmd > file           # stdout to file (overwrite)
cmd >> file          # append
cmd 2> errors.log    # stderr only
cmd &> all.log       # both
cmd 2>&1 | less      # merge stderr into stdout, THEN pipe
cmd < input.txt      # stdin from file
cmd | tee log.txt    # write to file AND pass along
cmd > /dev/null 2>&1 # discard everything
```

> **Order matters:** `cmd > f 2>&1` sends both to the file; `cmd 2>&1 > f` sends stderr to the *terminal*. The redirections apply left to right.

```bash
grep ERROR app.log | awk '{print $4}' | sort | uniq -c | sort -rn | head
```

> **That one line is most of log analysis.** Filter, extract, count, rank. See [[Command reference]].

---

## Error handling

```bash
set -euo pipefail

cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT                # runs on ANY exit, including failure

tmpfile=$(mktemp)

command_that_might_fail || { echo "step 1 failed" >&2; exit 1; }

if ! command -v docker &>/dev/null; then
  echo "docker is not installed" >&2
  exit 1
fi
```

> **`trap cleanup EXIT` is how you guarantee temp files are removed** even when the script dies halfway.

> **Errors go to stderr (`>&2`), and failures exit non-zero.** Otherwise CI thinks your broken script succeeded.

---

## Arrays

```bash
files=(a.csv b.csv "c d.csv")
echo "${files[0]}"
echo "${files[@]}"        # all elements, safely quoted
echo "${#files[@]}"       # count
files+=("e.csv")

for f in "${files[@]}"; do echo "$f"; done
```

> **`"${arr[@]}"` with quotes and `@`.** Any other form breaks on spaces.

---

## A real script

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly DATA_DIR="${DATA_DIR:?DATA_DIR must be set}"
readonly OUT="${1:-./out}"

log() { echo "[$(date -Iseconds)] $*" >&2; }

main() {
  mkdir -p "$OUT"
  local count=0

  for csv in "$DATA_DIR"/*.csv; do
    [[ -e "$csv" ]] || { log "no CSV files found"; exit 1; }   # glob didn't match
    log "processing $(basename "$csv")"
    python clean.py "$csv" > "$OUT/$(basename "$csv" .csv).clean.csv"
    ((count++))
  done

  log "done: $count files"
}

main "$@"
```

> ⚠️ **An unmatched glob expands to the literal pattern**, so `for csv in *.csv` with no CSVs gives you a file called `*.csv`. The `[[ -e "$csv" ]]` guard catches it.

---

## Debugging

```bash
bash -n script.sh        # syntax check, don't run
bash -x script.sh        # print every command as it runs  <- the main tool
set -x ; ... ; set +x    # trace just one section
shellcheck script.sh     # a linter that catches most of this page. USE IT.
```

> **Install `shellcheck` and run it on every script.** It catches unquoted variables, `$(ls)`, missing `-r` and dozens of real bugs. It is the highest-value thing on this page.

---

## Windows and WSL

> ⚠️ **CRLF line endings break Bash scripts** — you get `bad interpreter: /bin/bash^M`. Fix with `git config --global core.autocrlf input`, or `dos2unix script.sh`. See [[Setting up a dev machine]].

> **Run scripts from the Linux filesystem (`~/`), not `/mnt/c/`.** Cross-boundary file access in WSL is dramatically slower.

---

## Common mistakes

| Mistake | Consequence |
|---|---|
| No `set -euo pipefail` | Script continues after failure |
| `name = "x"` with spaces | "command not found" |
| Unquoted `$file` | Breaks on spaces |
| `for f in $(ls)` | Same, worse |
| `read` without `-r` | Backslashes eaten |
| Missing `local` in a function | Variable leaks and collides |
| `$@` unquoted | Arguments with spaces split |
| `[ ]` instead of `[[ ]]` | Fails on empty variables |
| Errors printed to stdout | Pollutes piped output |
| CRLF endings | `bad interpreter` |
| 300-line Bash script | You wanted [[Python]] |

## Related

[[Languages]] · [[Command reference]] · [[Python]] · [[Setting up a dev machine]] · [[Git-GitHub]] · [[Docker deep dive]] · [[CI-CD pipelines]] · [[03 — PROGRAMMING]]
