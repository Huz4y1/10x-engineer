The workflow for an endpoint you have never used, where the docs are wrong, missing, or you just do not trust them.

The rule underneath all of it: **save a real response to a file first, then work against the file.** You get a fixed target, you stop hammering someone's server, and you can iterate on filters offline.

The loop

```
  1. curl it            does it even respond
  2. save a sample      one real response, on disk
  3. map the shape      what fields actually exist
  4. write the filter   against the file, not the network
  5. handle the edges   errors, paging, empty results
  6. model it in code   see the per-language notes
```

Step 1, does it respond

```bash
# -i shows the response headers too
curl -i https://api.example.com/users

# -s silences the progress bar, -S still shows errors
curl -sS https://api.example.com/users | jq

# just the status code, useful when scripting
curl -s -o /dev/null -w "%{http_code}\n" https://api.example.com/users
```

Read the headers before the body. `content-type`, `x-ratelimit-remaining` and any `link` header tell you most of what you need about how the API behaves.

If you get HTML back, you are looking at a login page or an error page. Print the raw body before piping to jq, otherwise the parse error hides the real message.

Step 2, save a sample

```bash
mkdir -p samples
curl -sS https://api.example.com/users > samples/users.json

# pretty-print it as you save, easier to read in an editor
curl -sS https://api.example.com/users | jq > samples/users.json
```

Save the ugly cases too, not just the happy path.

```
  samples/
    users.json              a normal list
    users-empty.json        zero results
    users-one.json          a single item
    error-404.json          the shape of an error
    error-401.json          auth failure looks different, usually
```

That directory is worth committing. Later it becomes your test fixtures, and it documents the API better than the docs do.

Strip secrets before committing:

```bash
jq 'del(.token, .access_token) | .user.email = "redacted@example.com"' \
   samples/me.json > samples/me-safe.json
```

Step 3, map the shape

```bash
# what am I even looking at
jq 'type' samples/users.json
jq 'keys' samples/users.json
jq 'length' samples/users.json

# if it's an array, look at exactly one item
jq '.[0]' samples/users.json

# every field path in the document, deduplicated
jq -r '[paths(scalars) | join(".")] | unique | .[]' samples/users.json
```

That last command is the one to remember. Output looks like:

```
data.0.address.city
data.0.created_at
data.0.id
data.0.name
meta.page
meta.total
```

Now you know the real structure in about two seconds, including the `data` and `meta` wrapper you would otherwise have discovered by crashing.

Check field types and whether anything is inconsistent:

```bash
# what type is each field, across every item
jq -r '.data[] | to_entries[] | "\(.key): \(.value|type)"' samples/users.json \
  | sort -u
```

If a field shows up as both `string` and `null`, that is an optional field and your code needs to handle it. This one command catches a whole class of bug before you write any parsing.

Step 4, auth

Most APIs use a bearer token.

```bash
# keep the token out of your shell history and out of the note
export API_TOKEN="..."

curl -sS -H "Authorization: Bearer $API_TOKEN" \
     https://api.example.com/me | jq
```

Other shapes you will meet:

```bash
# api key in a header
curl -H "X-API-Key: $KEY" ...

# api key in the query string
curl "https://api.example.com/users?api_key=$KEY"

# basic auth
curl -u "user:password" ...

# a cookie session
curl -b "session=abc123" ...
```

Put the token in a `.env` that is gitignored. Never paste a real token into a vault note.

A `.netrc` file works well for machines you use often:

```
machine api.example.com
login your-user
password your-token
```

Then `curl -n https://api.example.com/me` picks it up with no flags.

Step 5, POST and other methods

```bash
# JSON body
curl -sS -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -d '{"name":"Huzayl","age":21}' | jq

# body from a file, cleaner for anything long
curl -sS -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d @body.json | jq

# build the body with jq so quoting can't go wrong
jq -n --arg name "Huzayl" '{name: $name, age: 21}' \
  | curl -sS -X POST https://api.example.com/users \
         -H "Content-Type: application/json" -d @- | jq
```

Forgetting `Content-Type: application/json` is the single most common reason a POST returns 400 with an unhelpful message.

Step 6, pagination

Three common styles. Check the response `meta` and the `link` header to work out which one you have.

```
  PAGE NUMBER     ?page=2&per_page=100
                  meta gives total pages, easy to loop

  OFFSET          ?offset=100&limit=100
                  same idea, watch for items shifting between calls

  CURSOR          ?after=eyJpZCI6OTF9
                  response gives the next cursor, stop when it's null
                  the only one that's safe on changing data
```

Cursor loop in bash:

```bash
#!/usr/bin/env bash
# 1. start with no cursor
# 2. append each page's items to one file
# 3. stop when the API stops giving a next cursor

cursor=""
: > all.jsonl

while : ; do
  url="https://api.example.com/users?limit=100"
  [ -n "$cursor" ] && url="$url&after=$cursor"

  page=$(curl -sS -H "Authorization: Bearer $API_TOKEN" "$url")

  echo "$page" | jq -c '.data[]' >> all.jsonl

  cursor=$(echo "$page" | jq -r '.meta.next_cursor // empty')
  [ -z "$cursor" ] && break

  sleep 0.3          # be polite, and stay under the rate limit
done

wc -l all.jsonl
```

Writing JSONL as you go means you can start filtering before the crawl finishes, and a crash does not lose everything.

Step 7, errors and rate limits

Look at what a failure actually returns, do not assume.

```bash
curl -sS -i https://api.example.com/users/999999
```

```
  400   your request was malformed, read the body
  401   no or bad credentials
  403   authenticated but not allowed
  404   not found - but some APIs use this for "no permission" too
  422   validation failed, body usually lists which fields
  429   rate limited, RESPECT the Retry-After header
  5xx   their problem, retry with backoff
```

Rate limit headers to watch:

```bash
curl -sS -D - -o /dev/null https://api.example.com/users \
  | grep -i 'ratelimit\|retry-after'
```

```
  x-ratelimit-limit: 5000
  x-ratelimit-remaining: 4998
  x-ratelimit-reset: 1769080080
```

Retry with exponential backoff, not a tight loop. curl can do it for you:

```bash
curl --retry 5 --retry-delay 2 --retry-max-time 60 \
     --retry-all-errors -sS https://api.example.com/users
```

Step 8, turn the sample into code

Once you know the shape, generate the types rather than hand-typing them.

```bash
# TypeScript interfaces from a sample
npx quicktype -s json -o types.ts --lang ts samples/users.json

# Rust structs with serde derives
npx quicktype -s json -o models.rs --lang rust samples/users.json
```

Then check the generated optionality against what you found in step 3. Generators assume a field is required if it was present in the one sample you gave them, which is usually wrong. Anything that came back `null` in any sample should be `Option<T>` or `T | null`.

From here: [[JSON (Rust)]], [[JSON (TypeScript)]], [[JSON (JavaScript)]], [[JSON (C++)]], [[JSON (C)]], [[JSON (SQL)]].

Quick reference

```bash
# explore
curl -sS $URL | jq
curl -sS $URL > samples/thing.json
jq -r '[paths(scalars)|join(".")] | unique | .[]' samples/thing.json
jq -r '.data[] | to_entries[] | "\(.key): \(.value|type)"' samples/thing.json | sort -u

# auth
curl -sS -H "Authorization: Bearer $API_TOKEN" $URL | jq

# filter
jq '[.data[] | select(.active) | {id, name}]' samples/thing.json

# extract for the shell
jq -r '.data[].id' samples/thing.json

# to csv
jq -r '.data[] | [.id,.name] | @csv' samples/thing.json
```

See [[jq]] for the filter language itself, and [[JSON]] for the format's rules and gotchas.
