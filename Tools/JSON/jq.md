`jq` is sed for JSON. You pipe JSON in, write a filter, get JSON out. It is the fastest way to understand a response you have never seen before.

Install

```bash
sudo apt install jq          # debian / ubuntu
brew install jq              # macos
winget install jqlang.jq     # windows
```

The mental model

A filter takes one input and produces zero, one, or many outputs. `|` pipes between filters, exactly like a shell.

```
  input JSON  ──►  filter  ──►  filter  ──►  output JSON
```

`.` is the identity filter, meaning "the whole input". Everything starts from there.

Pretty printing, the 90% use

```bash
curl -s https://api.example.com/users | jq
```

That alone, with no filter, reformats and colours the response. Often it is all you need.

Selecting fields

```bash
jq '.name'                 # one field
jq '.address.city'         # nested
jq '.tags[0]'              # first array element
jq '.tags[]'               # EVERY element, as separate outputs
jq '.["odd-key"]'          # keys with dashes or spaces need this form
jq '.a?'                   # don't error if .a is missing
```

Note the difference between `.tags` and `.tags[]`. The first gives you one array, the second gives you three separate strings. That matters when you pipe.

Exploring something unfamiliar

These four are what you actually run first on a new API.

```bash
jq 'keys'                  # what fields exist at the top level
jq 'type'                  # object? array? string?
jq 'length'                # how many items
jq '.[0]'                  # just the first item, if it's an array
```

Then to see the whole shape without the noise of values:

```bash
# every path that exists in the document
jq -c 'paths' sample.json | head -40

# just the leaf paths, joined with dots - a map of the structure
jq -r '[paths(scalars) | join(".")] | unique | .[]' sample.json
```

That last one is the single most useful command in this note. On an unfamiliar 3000-line response it prints a clean list of every field path.

Filtering with select

`select` keeps items where a condition is true, drops the rest.

```bash
# users over 30
jq '.[] | select(.age > 30)'

# by string equality
jq '.[] | select(.status == "active")'

# substring match
jq '.[] | select(.name | contains("Huz"))'

# regex
jq '.[] | select(.email | test("@gmail\\.com$"))'

# only items that HAVE a field
jq '.[] | select(has("phone"))'

# combining, and/or/not
jq '.[] | select(.age > 30 and .status == "active")'
```

Wrap in brackets to get an array back instead of a stream:

```bash
jq '[.[] | select(.age > 30)]'
```

map and reshaping

```bash
# pull one field from every item
jq 'map(.name)'
jq '[.[] | .name]'                    # same thing

# build a smaller object from a bigger one
jq '.[] | {name, city: .address.city}'

# {name} is shorthand for {name: .name}
jq 'map({id, name})'

# rename and compute
jq 'map({user: .name, adult: (.age >= 18)})'
```

Object construction is where jq stops being a viewer and starts being a transformer. You can pull an unwieldy API response into exactly the shape your code wants.

Sorting, grouping, counting

```bash
jq 'sort_by(.age)'
jq 'sort_by(-.age)'                   # descending
jq 'group_by(.city)'
jq 'group_by(.city) | map({city: .[0].city, count: length})'
jq 'map(.city) | unique'
jq 'map(.score) | add'                # sum
jq 'map(.score) | add / length'       # mean
jq 'max_by(.score)'
jq '[.[] | select(.status=="error")] | length'
```

Output flags

```bash
jq -r '.name'          # RAW: strings without quotes, for shell use
jq -c '.'              # COMPACT: one line per result, makes JSONL
jq -s '.'              # SLURP: read many inputs into one array
jq -e '.'              # EXIT code reflects the result, for scripts
jq -n '...'            # NULL input, when you're constructing from scratch
```

`-r` is the one you forget and then wonder why your filenames have quotes in them.

```bash
# grab every id into a shell loop
for id in $(jq -r '.[].id' users.json); do
  curl -s "https://api.example.com/users/$id" > "user_$id.json"
done
```

Handling missing data

```bash
jq '.phone // "none"'                 # default if null or false
jq '.a.b.c?'                          # ? suppresses the error
jq 'try .a.b.c catch "failed"'
jq 'map(select(.email != null))'      # drop items missing a field
```

The `//` operator is an alternative, not division. Very common in real filters.

Passing shell values in

Never interpolate shell variables straight into a filter, quoting will bite you.

```bash
# right way
name="Huzayl"
jq --arg n "$name" '.[] | select(.name == $n)' users.json

# numbers need --argjson, since --arg makes a string
jq --argjson min 30 '.[] | select(.age > $min)' users.json
```

CSV and TSV output

```bash
jq -r '.[] | [.id, .name, .age] | @csv' users.json
jq -r '.[] | [.id, .name] | @tsv' users.json

# with a header row
jq -r '["id","name"], (.[] | [.id, .name]) | @csv' users.json
```

Straight into a spreadsheet, no scripting.

JSON Lines

```bash
# each line is its own document, jq handles this by default
cat logs.jsonl | jq 'select(.level == "error")'

# turn an array into JSONL
jq -c '.[]' users.json > users.jsonl

# turn JSONL back into an array
jq -s '.' users.jsonl > users.json
```

Editing values

```bash
jq '.name = "new"'                    # set
jq '.age += 1'                        # update in place
jq '.tags += ["extra"]'               # append to an array
jq 'del(.password)'                   # remove a key
jq 'map(del(.internal_id))'           # remove from every item
jq '.user.city = "London"'            # nested set, creates path if needed
jq 'with_entries(.key |= ascii_downcase)'   # transform every key
```

Useful for stripping secrets out of a sample before you commit it, which comes up in [[Investigating an API]].

Recursive descent

`..` walks the entire tree, at any depth. Good when you know a field exists somewhere but not where.

```bash
# find every "id" anywhere in the document
jq '.. | .id? // empty' sample.json

# find every value that looks like an email
jq -r '.. | strings | select(test("@"))' sample.json
```

Big files

`jq` loads the whole document into memory by default. For anything large, stream it:

```bash
# process one top-level array element at a time
jq -c --stream 'fromstream(1|truncate_stream(inputs))' huge.json

# or just work with JSONL, which never needs this
```

In practice, converting to JSONL once and then filtering line by line is simpler than fighting `--stream`.

Testing a filter without a server

```bash
echo '{"a":{"b":[1,2,3]}}' | jq '.a.b | map(. * 2)'
# [2,4,6]
```

Building filters against a saved sample file rather than a live endpoint is the whole point of saving samples. See [[Investigating an API]].
