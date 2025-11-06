# 🧩 sbgen — Sing-box Config Generator

`sbgen` is a utility for **generating Sing-box JSON configurations** from templates and YAML profiles.  
It automatically builds the `route.rules` section based on domain pattern lists.

---

## 🚀 Features

- 🧱 Template placeholders: `%%tid:inbound%%` and `%%FINAL%%`.
- 🧾 Pattern sources:
  - Inline lists (`patterns:`),
  - External files (`includes:`),
  - Simple strings (`- "example.com"`).
- 🔄 YAML order preservation.
- 🎯 Multiple `out:` values supported (e.g. `out: [proxy, out-alt]`) — only placeholders present in the template are used.
- 🗂️ Include file search order: `item_dir → base_dir → --base-dir → cwd`.
- 🧩 Supports `base_dir`, `default_direct`, `direct`, `blocked` in YAML profiles.
- 🧼 Cleans up trailing commas and validates JSON.

---

## ⚙️ Template format

A `.tpl` or `.json` file with placeholders:

```json
{
  "route": {
    "rules": [
      { "action": "hijack-dns", "port": [53] },
      %%russia:in-tun%%,
      %%europe:in-tun%%
    ],
    "final": "%%FINAL%%"
  }
}
```

---

## 🧩 YAML profile format

```yaml
russia:
  base_dir: "./lists"
  default_direct: true
  lists:
    - name: world
      out: [out-global, proxy]
      includes: ["world.list"]
      patterns: ["example.org"]
    - name: region
      out: out-alt
      includes: ["region.list"]
  direct:
    - "mysite.net"
    - includes: ["direct.list"]
    - patterns: [".home.arpa", "lan.local"]
  blocked:
    - "*.trackers.com"
    - includes: ["block.list"]
```

`.list` files are plain text files:

```text
# comment
example.com
*.cdnservice.io
```

Empty lines and comments are ignored.

---

## 🧰 Usage

```bash
sbgen <template.tpl> <config.yml> [more.yml...] [options]
```

### Examples

```bash
sbgen singbox.tpl russia.yml > out.json
sbgen singbox.tpl profiles.yml --base-dir /opt/config/lists > out.json
sbgen singbox.tpl profiles.yml --debug
```

---

## 🧩 Command-line options

| Option | Description |
|--------|--------------|
| `--base-dir PATH` | Global base path for `includes:` (if not found in YAML) |
| `--debug` | Enables debug logging to `stderr` |
| `<template>` | JSON template file with placeholders |
| `<config.yml ...>` | One or more YAML profile files |

---

## 🔍 Error handling

- Missing file → warning (skipped).
- Invalid JSON → context printed to stderr.
- Missing `out` → not an error (ignored).
- `--debug` enables detailed tracing of file loads and replacements.

---

## 📂 Example project layout

```
project/
├── templates/
│   └── singbox.tpl
├── profiles/
│   ├── russia.yml
│   └── europe.yml
├── lists/
│   ├── world.list
│   ├── direct.list
│   └── block.list
└── out.json
```

---

© 2025 • `sbgen` — Sing-box route generation utility.
