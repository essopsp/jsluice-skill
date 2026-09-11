---
name: jsluice
description: "Use when extracting URLs, paths, endpoints, secrets, API keys, or interesting data from JavaScript files with JSLuice (Bishop Fox) during reconnaissance, bug bounty, or web app assessments. Triggers: JS analysis, javascript recon, extract endpoints from js, find secrets in javascript, JSLuice, jsluice CLI."
---

# JSLuice — Extract Secrets & URLs from JavaScript Files

JSLuice is Bishop Fox's Go package and CLI tool for statically extracting URLs, paths, secrets, and other interesting data from JavaScript source code. It parses JavaScript with `go-tree-sitter` so values are found based on **how they are used**, not just how they look — e.g. assignments to `document.location`, calls to `window.open()`, `fetch()`, `XMLHttpRequest`, and jQuery's `$.get`/`$.post`/`$.ajax`.

Source: https://github.com/BishopFox/jsluice

## Installation

Requires Go. After installing, verify with `jsluice --help`.

```
go install github.com/BishopFox/jsluice/cmd/jsluice@latest
```

## Command shape

```
jsluice <mode> [options] [file...]
```

Files can be local paths, any `http(s)://` URL (fetched automatically), WARC files (with `-w`), or read one-per-line from stdin:

```
find . -name '*.js' | jsluice <mode> [options]
```

Output is JSONL; pipe to `jq` for readability/filtering.

## Modes

- `urls` — extract URLs and paths
- `secrets` — find secrets and interesting data
- `tree` — print syntax trees (helpful when writing queries)
- `query` — run tree-sitter queries against files
- `format` — format (beautify) JavaScript source

## Mode: urls

Extracts from:
- assignments to `document.location`, `val.href`, `val.src`, etc.
- calls to `location.replace`, `window.open`, `fetch`
- `XMLHttpRequest` usage
- jQuery `$.get`, `$.post`, `$.ajax`
- any string literal that looks like a URL (disable with `-I`/`--ignore-strings`)

Where possible it also extracts HTTP method, headers, query params, and body params.

```
jsluice urls demo.js | jq
{
  "url": "/api/users?id=EXPR&format=json",
  "queryParams": ["id", "format"],
  "method": "GET",
  "headers": { "X-Env": "stage" },
  "type": "fetch",
  "filename": "demo.js"
}
```

Unknown expressions in string concatenation are replaced with `EXPR` (change with `-P`/`--placeholder`).

Useful flags:
- `-R <url>` / `--resolve-paths <url>` — resolve relative paths against a base URL
- `-S` / `--include-source` — add a `source` field with the originating code
- `-I` / `--ignore-strings` — skip the string-literal URL matcher
- `-H` / `--header` — headers for HTTP-fetched inputs (repeatable)
- `-C` / `--cookie` — cookies for HTTP-fetched inputs
- `-c N` / `--concurrency N` — concurrent file processing (default 1)

## Mode: secrets

Built-in extractors:
- AWS keys (pairs the access key with its secret in the `data` field)
- GCP keys
- GitHub keys
- Firebase configurations

```
jsluice secrets awskey.js | jq
{
  "kind": "AWSAccessKey",
  "data": {
    "key": "AKIAIOSFODNN7EXAMPLE",
    "secret": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  },
  "filename": "awskey.js",
  "severity": "high",
  "context": { ... }
}
```

The enclosing object is included as `context`.

### Custom secret patterns (`-p` / `--patterns`)

Pattern file is a JSON array. Each pattern may have:
- `name` — used in the output
- `severity` — `info`, `low`, `medium`, or `high`
- `value` — regex matched against string values
- `key` — regex matched against key names
- `object` — array of `{key, value?}` patterns matched against all keys/values of a whole object

All regexes use Go regex syntax. Anchor them (`^...$`) when matching whole values.

```json
[
  { "name": "genericSecret", "key": "(secret|private|key)", "value": "[%a-zA-Z0-9+/]+" },
  {
    "name": "firebaseConfig",
    "severity": "high",
    "object": [
      {"key": "apiKey", "value": "^AIza.+"},
      {"key": "authDomain"},
      {"key": "projectId"},
      {"key": "storageBucket"}
    ]
  }
]
```

Run:

```
jsluice secrets -p patterns.json target.js | jq
```

## Mode: tree

Prints a textual syntax tree, invaluable for designing tree-sitter queries:

```
jsluice tree hello.js
program
  expression_statement
    call_expression
      function: member_expression
        object: identifier (console)
        property: property_identifier (log)
        arguments: arguments
          string ("Hello, world!")
```

## Mode: query

Run tree-sitter queries; `@name` capture(s) determine what is output. Query syntax docs: https://tree-sitter.github.io/tree-sitter/using-parsers#query-syntax

```
jsluice query -q '(string) @str' config.js
jsluice query -q '(object) @match' config.js | jq
```

JSON-encodes matched objects/arrays/strings for valid JSONL; use `-r`/`--raw-output` for raw text.

## Mode: format

Beautify minified JavaScript with jsbeautifier-go:

```
jsluice format minified.js
```

## Workflow for recon / bug bounty

1. Discover: crawl the target and collect `.js` file URLs (e.g. from HTML, `*.js` fingerprints, JS maps).
2. Extract endpoints and secrets:
   ```
   cat js_urls.txt | jsluice urls -c 10 -R https://target.com   | tee urls.jsonl
   cat js_urls.txt | jsluice secrets -c 10 -p patterns.json     | tee secrets.jsonl
   ```
3. Process with jq: dedupe URLs, isolate query/body params for XXE/IDOR/parameter fuzzing lists, group secrets by severity.
4. For finer control, dump a syntax tree with `jsluice tree` and target specific node types via `jsluice query`.

## Notes

- Static analysis cannot know runtime values: variables in concatenation become `EXPR` (or your `-P` placeholder). A URL like `/api/users?id=EXPR&format=json` still reveals both query params.
- `jsluice` does NOT match `mailto:` or other scheme URLs by default — expect false-negative types such as these.
- Remote inputs fetch over HTTP; repeat `-H` to add multiple headers.
- `--warc` treats inputs as WARC archives, with the requested URL as `filename`.