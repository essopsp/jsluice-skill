# JSLuice Skill for opencode

An [opencode](https://opencode.ai) skill for extracting URLs, paths, endpoints, secrets, and API keys from JavaScript files using **[JSLuice](https://github.com/BishopFox/jsluice)** by Bishop Fox.

Designed for security researchers doing **reconnaissance**, **bug bounty**, and **web application assessments**. The skill goes beyond regex brute-force: it drills into how URLs are *used* (`fetch`, `document.location`, `XMLHttpRequest`, jQuery `$.ajax`, …) using tree-sitter parsing, and understands string concatenation so it can still extract query params from dynamic URLs.

## What it covers

- Full `jsluice` CLI: all modes — `urls`, `secrets`, `tree`, `query`, `format`
- Key flags: `-R` path resolution, `-S` include source, `-I` ignore strings, `-H`/`-C` for remote fetches, `-c` concurrency, `-w` WARC input
- Custom secret pattern files (`-p`): AWS, GCP, GitHub, Firebase built-ins plus user-defined `value`/`key`/`object` patterns
- Tree-sitter queries for custom extraction of nearly anything in a JS file
- End-to-end recon workflow: discover JS files → extract endpoints & secrets → dedupe/process with `jq`

## Requirements

- [Go](https://go.dev/dl/) (to install the CLI)
- The `jsluice` CLI:

  ```
  go install github.com/BishopFox/jsluice/cmd/jsluice@latest
  ```

- An opencode-compatible agent that supports skills (this is a pure-prompt skill; no code to run).

## Installation

Copy the skill folder into your opencode skills directory:

```
mkdir -p ~/.config/opencode/skills
cp -r jsluice ~/.config/opencode/skills/
```

Or clone this repo somewhere and register that path in your `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/path/to/clone/of/jsluice-skill"]
  }
}
```

Restart opencode, then ask for things like:

- "Extract all endpoints from these JavaScript files"
- "Find secrets in this app's JS bundles"
- "Run JSLuice on https://target.com/app.js"

## Quick start

```
# Extract endpoints from a JS file (JSONL output → jq)
jsluice urls demo.js | jq

# Secrets with custom patterns from a crawled list of JS URLs
cat js_urls.txt | jsluice secrets -c 10 -p patterns.json | jq

# Resolve relative paths against the target origin
jsluice urls app.js -I -R https://target.com/ | jq

# Pull every string literal with a tree-sitter query
jsluice query -q '(string) @str' config.js
```

Example output from `jsluice urls`:

```json
{
  "url": "/api/users?id=EXPR&format=json",
  "queryParams": ["id", "format"],
  "method": "GET",
  "headers": { "X-Env": "stage" },
  "type": "fetch",
  "filename": "demo.js"
}
```

## License

MIT — see [LICENSE](LICENSE).