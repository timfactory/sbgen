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
sbgen <template.tpl> <config.yml> [more.yml...] [options]
```

**Options:**
- `-v, --verbose` — Enable debug logging to stderr
- `-b, --base-dir DIR` — Base directory for resolving relative `includes` paths (used after item/tid base dirs)
- `-a, --append RULE` — Append raw fragment (string or JSON) into `route.rules[]` or `routing.rules[]` before placeholder processing
- `-x, --xray` — Generate Xray/V2Ray style configuration instead of sing-box

**Examples:**
```bash
# Basic usage
./sbgen nekobox-russia.tpl profiles.yml > nekobox-russia.json

# With multiple YAML files
./sbgen standalone-world.tpl profiles.yml local.yml > config.json

# Debug mode
./sbgen nekobox-russia.tpl profiles.yml -v > config.json

# Xray mode
./sbgen template.tpl config.yml -x > xray-config.json

# Append custom rule
./sbgen template.tpl config.yml -a '{"domain":["example.com"],"outbound":"proxy"}' > config.json
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
| `default_direct` | `bool` | If `true`, `%%FINAL%%` becomes `direct`. If `false`, uses first available `out` from `lists`. Default: `true`. |
| `lists` | `list<object>` | A set of routing domain groups with specific outbounds. |
| `direct` | `list<mixed>` | Directly routed domains (`outbound = direct`). |
| `blocked` / `block` | `list<mixed>` | Blocked domains (`outbound = block`). Both keys are supported. |

---

### 🔹 Elements of `lists`

Each `lists` item may contain:

| Field | Type | Description |
|--------|------|-------------|
| `name` | `string` | Optional descriptive name. |
| `out` | `string or list<string>` | One or more outbounds to route traffic to. |
| `patterns` | `list<string>` | Inline domain patterns. |
| `includes` | `list<string>` | Paths to external lists (one domain per line, `#` for comments). |

---

## 🧩 Supported Domain Patterns

| Example | Interpreted As | Generated Rule |
|----------|----------------|----------------|
| `example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `.example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `*.example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `rutracker.*` | `domain_regex` | `"domain_regex": ["^(.+\\.)?rutracker\\.[^.]+$"]` |
| `cdn.*.example.com` | `domain_regex` | `"domain_regex": ["^cdn\\.[^.]+\\.example\\.com$"]` |
| `*cdn.com` | `domain_regex` | `"domain_regex": ["^[^.]*cdn\\.com$"]` |
| `localhost` | `domain_suffix` | `"domain_suffix": ["localhost"]` |

**Note:** In Xray mode (`-x`), these become `domainSuffix` and `domainRegex` instead.

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
- `#` starts a comment (full line or inline with `\#` to escape).  
- Empty lines are ignored.  
- Inline comments are supported: `example.com # comment`  
- Paths are resolved relative to (in priority order):
  1. Directory of the YAML file from which the item was loaded
  2. Profile's `base_dir` (from YAML)
  3. CLI `--base-dir` option
  4. Current working directory

---

## 🧠 sbgen Logic

1. Loads the `.tpl` template and all YAML files.  
2. Sanitizes JSON (removes trailing commas, quotes unquoted placeholders).  
3. Merges all top-level keys (`tid`), e.g., `russia`, `china`.  
4. Collects available outbound tags from the template.  
5. Finds placeholders like `%%russia:in-tun%%` or `"%%russia:in-tun%%"` in the template.  
6. For each placeholder:
   - Loads patterns from `lists`, `direct`, and `blocked` sections
   - Resolves `includes` files (relative to `base_dir`, item base, or CLI `-b`)
   - Classifies patterns as `domain_suffix` or `domain_regex`
   - Generates routing rules for the specified inbound
7. Replaces placeholders with generated rule arrays.  
8. Computes `%%FINAL%%` based on `default_direct` and available outbounds.  
9. Outputs valid JSON to stdout.  

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

## 🌐 Example 3 — Standalone V2Ray/Xray

**Template (`v2ray.tpl`):**
```json
{
  "log": { "level": "warn" },
  "inbounds": [
    {
      "tag": "in-socks",
      "port": 1080,
      "protocol": "socks",
      "settings": {
        "auth": "noauth"
      }
    }
  ],
  "outbounds": [
    { "tag": "proxy", "protocol": "vmess", ... },
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ],
  "routing": {
    "rules": [
      %%myprofile:in-socks%%
    ],
    "domainStrategy": "IPIfNonMatch"
  }
}
```

**Generate Xray/V2Ray configuration:**
```bash
./sbgen v2ray.tpl config.yml -x > v2ray-config.json
```

**Note:** In Xray mode (`-x`):
- Use `routing` instead of `route`
- Use `routing.rules[]` instead of `route.rules[]`
- Field names use camelCase: `outboundTag`, `domainSuffix`, `domainRegex`, `inboundTag`
- `%%FINAL%%` is replaced with the selected outbound tag (e.g., `"direct"` or `"proxy"`) and can be used in `routing.final` or anywhere in the template

---

## 🌍 Example 4 — Multiple YAML Profiles

```bash
./sbgen standalone-world.tpl profiles.yml > config.json
```

- Placeholders `%%russia:in-tun%%` and `%%world:in-tun%%` will both be resolved if present in template.  
- YAML files are merged by their top-level identifiers (`tid`).  
- If the same `tid` appears in multiple files, their sections are merged (lists, direct, blocked are combined).

---

## 🔧 Example 5 — Using `-a` (--append) Option

The `-a` option allows you to inject custom rules into `route.rules[]` (or `routing.rules[]` in Xray mode) **before** placeholder processing. This is useful for adding priority rules or rules that don't fit the YAML structure.

### Example 5.1 — Append a Single Rule (String Placeholder)

Append a placeholder that will be processed later:

```bash
./sbgen nekobox-russia.tpl profiles.yml -a '%%russia:in-extra%%' > config.json
```

This adds `%%russia:in-extra%%` to the rules array, which will be replaced during placeholder processing if it exists in the template context.

### Example 5.2 — Append a Single Rule (JSON Object)

Add a custom rule directly:

```bash
./sbgen standalone-russia.tpl profiles.yml \
  -a '{"domain":["example.com","test.com"],"outbound":"direct"}' \
  > config.json
```

**Result:** The rule is appended to the end of `route.rules[]`. If placeholders are processed, they are replaced in place, so the order depends on where placeholders are located in the template.

---

## 📁 Project Structure Example

```
.
├── sbgen                    # Python script
├── profiles.yml            # YAML configuration with multiple profiles
├── nekobox-russia.tpl      # Template for NekoBox (mobile)
├── standalone-russia.tpl   # Template for standalone (desktop)
├── lists/                  # Domain lists directory
│   ├── world.list          # International services
│   ├── russia.list         # Russian services
│   ├── israel.list         # Israeli services
│   ├── direct.list         # Direct connection domains
│   └── block.list          # Blocked domains
└── README.md
```

**Typical workflow:**
```bash
# Generate NekoBox config
./sbgen nekobox-template.tpl config.yml > nekobox-config.json

# Generate standalone config
./sbgen standalone-template.tpl config.yml > standalone-config.json
```

---

## 📱 NekoBox Workflow Example

This section shows how to modify an existing NekoBox configuration using `sbgen`.

**Note:** By default, NekoBox uses:
- Inbound tag: `tun-in`
- Outbound tag: `proxy`

When creating templates or using placeholders, use these default names (e.g., `%%myprofile:tun-in%%`).

**Example YAML configuration (`config.yml`):**

```yaml
myprofile:
  base_dir: ./lists
  default_direct: true
  lists:
    - name: world
      out: [ proxy ]
      includes: [ world.list ]
      patterns:
        - "google.com"
        - "youtube.com"
  direct:
    - includes: [ direct.list ]
  block:
    - includes: [ block.list ]
```

This configuration will generate rules for the `%%myprofile:tun-in%%` placeholder, routing domains from `world.list` through the `proxy` outbound.

### Step 1: Export Configuration from NekoBox

1. Open NekoBox app
2. Select your server configuration
3. Tap the **Share** icon (📤)
4. Choose **Configuration** → **Export as file**
5. Save the exported JSON file (e.g., `exported-config.json`)

### Step 2: Process the Configuration

You have two options to add routing rules:

#### Option A: Create a Template

1. **Extract the base configuration:**
   - Copy the exported JSON
   - Remove or replace the `route.rules[]` section with placeholders
   - Save as a template (e.g., `custom-template.tpl`)

   **Example template:**
   ```json
   {
     "log": { "level": "warn" },
     "dns": { ... },
     "inbounds": [ ... ],
     "outbounds": [ ... ],
     "route": {
       "rules": [
         %%myprofile:tun-in%%,
         { "inbound": ["tun-in"], "outbound": "proxy" }
       ],
       "final": "%%FINAL%%"
     }
   }
   ```

2. **Generate new configuration:**
   ```bash
   ./sbgen custom-template.tpl config.yml > updated-config.json
   ```

#### Option B: Use `-a` to Append Rules

Add rules directly to the exported configuration. You can pass placeholders or JSON rules via `-a`:

```bash
# Add a placeholder that will be processed
./sbgen exported-config.json config.yml \
  -a '%%myprofile:tun-in%%' \
  > updated-config.json

# Or add a JSON rule directly
./sbgen exported-config.json config.yml \
  -a '{"domain_suffix":["example.com"],"outbound":"direct"}' \
  > updated-config.json
```

**Note:** When using `-a` with an exported config, the script will:
- Process any placeholders in the file (if present)
- Process placeholders passed via `-a` (if any)
- Add your custom rule to `route.rules[]`
- Preserve all other settings (DNS, inbounds, outbounds, etc.)

### Step 3: Import Back to NekoBox

1. Open NekoBox app
2. Tap the **Add** icon (+)
3. Select **Manual Settings** → **Custom Config**
4. Choose **Import from file** or paste the JSON content
5. Select the generated file (e.g., `updated-config.json`)
6. Save and activate the configuration

### Complete Example

```bash
# 1. Export from NekoBox → exported-config.json

# 2. Process with sbgen (using -a option with placeholder)
./sbgen exported-config.json config.yml \
  -a '%%myprofile:tun-in%%' \
  > updated-config.json

# 3. Import updated-config.json back to NekoBox
```

**Tip:** Use `-v` flag to see what rules are being added:
```bash
./sbgen exported-config.json config.yml -a '...' -v > updated-config.json
```

---

## 🧩 Debugging

To enable debug mode:
```bash
./sbgen template.tpl config.yml -v
```

This prints to stderr:
- Template and YAML file paths
- Base directories for each profile
- Available outbound tags
- Found placeholders
- Pattern loading from includes
- Generated rules count
- FINAL outbound selection

---

## ⚠️ Common Issues

| Error | Cause | Solution |
|--------|--------|----------|
| `Template not found` | Template file doesn't exist | Check file path |
| `JSON error: ...` | Invalid JSON syntax | Check for unclosed brackets, quotes, or invalid placeholders |
| `Include not found: ...` | `includes` file path cannot be resolved | Check `base_dir` or use absolute paths |
| `Skip list out='...' (not present in template outbounds)` | Outbound tag in YAML doesn't exist in template | Verify outbound tags match between template and YAML |

**Note:** The script automatically handles trailing commas in JSON templates, but other JSON syntax errors must be fixed manually.

---

## 📜 License

MIT © Ivan Tarasov, 2025  
Free to use, modify, and distribute with author attribution.