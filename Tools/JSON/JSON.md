JSON is the format almost every API speaks. It is a text format, so everything you get back is a string until you parse it into real values.

[[jq]] — the command line tool for exploring and filtering it

[[Investigating an API]] — the workflow for a new endpoint you know nothing about

Per language:

[[JSON (JavaScript)]] · [[JSON (TypeScript)]] · [[JSON (Rust)]] · [[JSON (C++)]] · [[JSON (C)]] · [[JSON (SQL)]]

The whole type system

Six types, that is all there is.

```
  object    { "key": value }        keys MUST be double-quoted strings
  array     [ value, value ]        ordered, can mix types
  string    "text"                  double quotes only, never single
  number    42   3.14   -1e5        no distinction between int and float
  boolean   true  false             lowercase only
  null      null                    lowercase only
```

No dates, no comments, no functions, no undefined. A date is just a string that both sides agreed to interpret as a date, normally ISO 8601: `"2026-07-22T11:08:00Z"`.

A realistic response

```json
{
  "id": 91,
  "name": "Huzayl",
  "active": true,
  "score": 4.5,
  "tags": ["rust", "hardware"],
  "address": {
    "city": "London",
    "postcode": null
  },
  "created_at": "2026-07-22T11:08:00Z"
}
```

Nesting is the thing to get comfortable with. `address` is an object inside an object, `tags` is an array of strings. Every access is just walking down that tree.

```
  data.name              -> "Huzayl"
  data.address.city      -> "London"
  data.tags[0]           -> "rust"
  data.address.postcode  -> null      <- exists, but empty
  data.phone             -> missing   <- different from null
```

That last distinction causes real bugs. `null` means the key is there with no value, missing means the key was never sent. Some languages collapse both to the same thing, some do not.

Rules that trip people up

```
  NOT VALID JSON                        WHY
  ──────────────                        ───
  { name: "Bob" }                       keys must be quoted
  { 'name': 'Bob' }                     single quotes are never allowed
  { "a": 1, }                           no trailing commas
  { "a": 1 } // comment                 no comments, at all
  { "a": NaN }                          NaN and Infinity do not exist
  { "a": 01 }                           no leading zeros
  { "a": .5 }                           needs a leading digit: 0.5
```

If you want comments in a config file, you want JSON5, YAML or TOML instead. JSON deliberately has none.

The big number problem

JSON numbers have no declared precision. Most parsers turn them into a 64-bit float, which can hold integers exactly only up to 2^53.

```
  9007199254740993        the real id the server sent
  9007199254740992        what JavaScript gives you back

  off by one, silently, no error
```

This bites when APIs use large integer ids (Twitter, Discord, database bigints). Well-designed APIs send those as strings for exactly this reason:

```json
{ "id": "9007199254740993" }
```

If you control the API, do that. If you do not, parse with something that supports big integers.

Duplicate keys

```json
{ "a": 1, "a": 2 }
```

Technically allowed by the spec, and every parser handles it differently. Most take the last one. Never rely on it.

JSON Lines

For logs and streaming, one JSON object per line, no wrapping array.

```
{"level":"info","msg":"started"}
{"level":"error","msg":"payment declined","user_id":91}
{"level":"info","msg":"done"}
```

Called JSONL or NDJSON. The point is you can process it a line at a time without loading the whole file, and appending is just adding a line. This is what most log pipelines emit, including the ones in [[Loki]].

`jq` handles it natively, and `jq -c` produces it.

Content types

```
  application/json          the normal one
  application/x-ndjson      JSON Lines
  application/problem+json  RFC 7807 error responses
```

If a server returns HTML when you expected JSON, your parser will fail with something confusing like "unexpected token <". That is almost always a login page or an error page, so print the raw body before you parse.

Where to go next

Start with [[Investigating an API]] for the practical workflow, and [[jq]] for the tool that makes it fast.
