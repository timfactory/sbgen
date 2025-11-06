# 🧩 sbgen — sing-box configuration generator from YAML templates

**Author:** Ivan Tarasov  
**Year:** 2025  
**License:** MIT  

---

## 📘 Overview

`sbgen` is a Python utility for generating [sing-box](https://sing-box.sagernet.org/) JSON configurations from template (`.tpl`) files and YAML profiles.

It allows you to:
- define proxy routes, domain lists, and rules in YAML;
- include external domain lists (`includes`);
- combine multiple profiles (e.g., `russia.yml`, `china.yml`);
- manage priorities and fallback logic inside templates;
- ensure valid, clean JSON output with no syntax issues.

---

## ⚙️ Usage

```bash
sbgen <template.tpl> <config.yml> [more.yml...]
```

**Examples:**
```bash
sbgen nekobox.tpl russia.yml > sing-box.json
sbgen server.tpl global.yml local.yml > config.json
```

If any of the specified files do not exist, the tool exits with an error.

---

## 🧱 YAML Structure

Each YAML file defines one or more **profiles** (e.g., `russia`, `china`, `office`).

Example:

```yaml
russia:
  base_dir: ./lists
  default_direct: true
  lists:
    - name: world
      out: [ proxy, proxy2 ]
      fallback: false
      patterns:
        - "google.com"
        - "youtube.com"
        - "linkedin.com"
      includes:
        - "world.list"
  direct:
    - "myserver.local"
    - includes: ["direct.list"]
  blocked:
    - includes: ["block.list"]
```

---

## 🔄 Profile Parameters

| Parameter | Type | Description |
|------------|------|-------------|
| `base_dir` | `string` | Optional. Base directory for relative `includes` paths. |
| `default_direct` | `bool` | Defines the final outbound behavior (`direct` or first list `out`). |
| `lists` | `list<object>` | A set of routing domain groups. |
| `direct` | `list<mixed>` | Directly routed domains. |
| `blocked` | `list<mixed>` | Blocked domains (`outbound = block`). |

---

### 🔹 Elements of `lists`

Each `lists` item may contain:

| Field | Type | Description |
|--------|------|-------------|
| `name` | `string` | Optional descriptive name. |
| `out` | `string or list<string>` | One or more outbounds to route traffic to. |
| `patterns` | `list<string>` | Inline domain patterns. |
| `includes` | `list<string>` | Paths to external lists (one domain per line, `#` for comments). |
| `fallback` | `bool` | Optional flag (not used by sbgen, but preserved). |

---

## 🧩 Supported Domain Patterns

| Example | Interpreted As |
|----------|----------------|
| `example.com` | `domain_suffix` |
| `.example.com` | `domain_suffix` |
| `*.example.com` | wildcard suffix |
| `rutracker.*` | regex (any TLD) |
| `cdn.*.example.com` | regex |
| `*cdn.com` | regex |
| `localhost` | single label |

---

## 📂 `includes` File Format

Example `world.list`:

```text
# streaming
youtube.com
netflix.com
*.hulu.com

# social
twitter.com
facebook.com
```

- Each line defines a domain or pattern.  
- `#` starts a comment.  
- Empty lines are ignored.

---

## 🧠 sbgen Logic

1. Loads the `.tpl` template and all YAML files.  
2. Merges all top-level keys (`tid`), e.g., `russia`, `china`.  
3. Finds placeholders like `%%russia:in-tun%%` in the template.  
4. Generates domain rules for each matching section.  
5. Replaces placeholders with formatted JSON.  
6. Computes `%%FINAL%%` if present.  
7. Validates the resulting JSON for syntax correctness.  

---

## 🧩 Example 1 — Standalone Server

**Template (`server.tpl`):**
```json
{
  "route": {
    "rules": [
      %%russia:in-tun%%
    ],
    "final": "%%FINAL%%"
  }
}
```

**YAML (`russia.yml`):**
```yaml
russia:
  lists:
    - out: proxy
      patterns: [ "youtube.com", "twitter.com" ]
  direct: [ "intranet.local" ]
```

**Result:**
```json
{
  "route": {
    "rules": [
      { "inbound": "in-tun", "outbound": "proxy", "domain_suffix": ["twitter.com", "youtube.com"] },
      { "inbound": "in-tun", "outbound": "direct", "domain_suffix": ["intranet.local"] }
    ],
    "final": "direct"
  }
}
```

---

## 🌐 Example 2 — NekoBox (TUN Mode)

**Template (`nekobox.tpl`):**
```json
{
  "inbounds": [
    {
      "tag": "in-tun",
      "type": "tun",
      "endpoint_independent_nat": true,
      "inet4_address": ["172.19.0.1/28"],
      "mtu": 9000,
      "sniff": true,
      "stack": "mixed"
    }
  ],
  "route": {
    "rules": [ %%russia:in-tun%% ],
    "final": "%%FINAL%%"
  }
}
```

---

## 🌍 Example 3 — Multiple YAML Profiles

```bash
sbgen global.tpl russia.yml israel.yml > sing-box.json
```

- Placeholders `%%russia:in-tun%%` and `%%israel:in-tun%%` will both be resolved.  
- YAML files are merged by their top-level identifiers (`tid`).

---

## 🧩 Debugging

To enable debug mode:
```bash
DEBUG=1 sbgen template.tpl config.yml
```

This prints detailed file loading, merging, placeholder detection, and generated rule data.

---

## ⚠️ Common Issues

| Error | Cause |
|--------|--------|
| `Usage: sbgen <template> <yaml>` | Missing required arguments |
| `File not found:` | One of the input files does not exist |
| `JSON error at line ...` | Template contains invalid JSON (trailing commas or missing replacements) |

---

## 📜 License

MIT © Ivan Tarasov, 2025  
Free to use, modify, and distribute with author attribution.