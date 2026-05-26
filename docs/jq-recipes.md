# jq recipes

A practical reference for the JSON-querying patterns you reach for most often:
picking fields, filtering arrays, mapping/transforming, grouping, sorting, string
operations, recursive descent, raw output, passing args, and combining files. All
examples use `jq` 1.7.x — most also work on 1.6.

## 1. Pretty-print and identity

The simplest jq filter is `.` — it reads JSON, validates it, and pretty-prints
it back out. Useful as a sanity check that a file is valid JSON.

```bash
jq . data.json
```

Compact output (single line, no spaces) — handy when piping into other tools or
storing JSON one-per-line:

```bash
jq -c . data.json
```

Sort object keys alphabetically:

```bash
jq -S . data.json
```

## 2. Pick a field

Drill into an object with `.field`. Chain with dots for nested access:

```bash
jq '.name' data.json
jq '.user.email' data.json
```

Use `?` when a field may be missing or the value may be of a wrong type — jq
returns `null` instead of erroring:

```bash
jq '.user.email?' data.json
```

The alternative operator `//` provides a fallback when a value is `null` or
`false`:

```bash
jq '.user.email // "unknown"' data.json
```

## 3. Array indexing and slicing

Index into an array with `[N]`. Negative indices count from the end:

```bash
jq '.[0]' data.json
jq '.[-1]' data.json
```

Iterate every element of an array with `[]` — emits each element as a separate
output:

```bash
jq '.items[]' data.json
```

Slice an array with `[start:end]` (end exclusive, like Python):

```bash
jq '.items[0:3]' data.json
```

## 4. Filter an array with select

`select(condition)` keeps inputs where the condition is truthy and drops the rest.
Combine with `[]` to filter array elements:

```bash
jq '.items[] | select(.status == "active")' data.json
```

Wrap the result back into an array with `[...]`:

```bash
jq '[.items[] | select(.status == "active")]' data.json
```

Numeric / boolean / multi-condition examples:

```bash
jq '.items[] | select(.score > 0.8 and .active == true)' data.json
```

## 5. Map and transform

`map(f)` is sugar for `[.[] | f]` — apply a filter to every element of an array
and collect the results.

Pluck a single field from each element:

```bash
jq '.items | map(.name)' data.json
```

Project to a smaller object (object shorthand `{name, status}` is the same as
`{name: .name, status: .status}`):

```bash
jq '.items | map({name, status})' data.json
```

Rename / compute fields:

```bash
jq '.items | map({id, label: (.name | ascii_upcase), ok: (.status == "active")})' data.json
```

## 6. group_by and count

`group_by(f)` sorts the input array by `f` and partitions it into runs that
share the same key. The classic histogram pattern:

```bash
jq '.events | group_by(.type) | map({type: .[0].type, count: length})' data.json
```

Sum a numeric field per group:

```bash
jq '.events | group_by(.user_id) | map({user_id: .[0].user_id, total: map(.amount) | add})' data.json
```

## 7. Sort and unique

Sort by a field (ascending). `sort_by(f)` is stable.

```bash
jq '.items | sort_by(.created_at)' data.json
```

Reverse for descending:

```bash
jq '.items | sort_by(.created_at) | reverse' data.json
```

Dedupe by a key — `unique_by` sorts then collapses duplicates:

```bash
jq '.items | unique_by(.id)' data.json
```

## 8. String operations

Case conversion:

```bash
jq '.name | ascii_upcase' data.json
jq '.name | ascii_downcase' data.json
```

Split and join:

```bash
jq '.email | split("@")' data.json
jq '.tags | join(", ")' data.json
```

Regex with `test` (boolean) and `match` (full match info):

```bash
jq '.items[] | select(.email | test("@example\\.com$"))' data.json
```

`startswith` / `endswith` / `contains` for simple substring checks:

```bash
jq '.items[] | select(.name | startswith("test-"))' data.json
```

## 9. Recursive descent

`..` walks every value in the tree. Combine with type filters and `select` to
find values anywhere in a deeply nested structure:

```bash
jq '.. | objects | select(has("email")) | .email' data.json
```

Find every string under a given key, regardless of depth:

```bash
jq '.. | .id? // empty' data.json
```

`paths(f)` returns the path to every value matching `f` — useful when you need
to know *where* something lives, not just its value:

```bash
jq '[paths(type == "string")]' data.json
```

## 10. Raw output and string interpolation

`-r` strips the surrounding quotes from string outputs. Combined with string
interpolation `"\(.field)"`, this is how you produce shell-friendly text:

```bash
jq -r '.items[] | "\(.id): \(.name)"' data.json
```

Generate a CSV row with `@csv` (input must be an array per row):

```bash
jq -r '.items[] | [.id, .name, .status] | @csv' data.json
```

TSV is the same idea:

```bash
jq -r '.items[] | [.id, .name, .status] | @tsv' data.json
```

## 11. Read from environment and arguments

Pass a string in with `--arg name value`. It becomes `$name` inside the filter:

```bash
jq --arg key "email" '.[$key]' data.json
```

Pass a JSON value (number, bool, object, array) with `--argjson`:

```bash
jq --argjson n 3 '.items[$n]' data.json
```

Read environment variables via the built-in `$ENV` (whole map) or `env.NAME`:

```bash
jq -n 'env.HOME'
```

## 12. Multiple inputs and slurp

`-s` (slurp) reads *all* inputs into a single array. Useful for combining
multiple JSON documents:

```bash
jq -s '.' a.json b.json c.json
```

Merge an array of objects into one with `add`:

```bash
jq -s 'add' a.json b.json
```

Read newline-delimited JSON (one object per line) and aggregate:

```bash
jq -s 'map(.amount) | add' events.ndjson
```

`-n` ("null input") runs the filter without consuming stdin — use it when
generating JSON from scratch or working purely from `--arg` / `--argjson`:

```bash
jq -n --arg name "alice" --argjson age 30 '{name: $name, age: $age}'
```
