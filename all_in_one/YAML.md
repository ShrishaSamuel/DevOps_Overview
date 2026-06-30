# YAML — Zero to Hero

> A complete, beginner-to-advanced guide to YAML, the configuration language at the heart of DevOps tooling.

---

## Table of Contents

1. [Introduction & Theory](#1-introduction--theory)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [Hands-on Tasks](#4-hands-on-tasks)
5. [Projects](#5-projects)
6. [Best Practices & Common Pitfalls](#6-best-practices--common-pitfalls)
7. [Interview Questions](#7-interview-questions)
8. [Quizzes](#8-quizzes)
9. [Further Resources](#9-further-resources)

---

## 1. Introduction & Theory

### 1.1 What is YAML?

**YAML** ("**YAML Ain't Markup Language**" — a recursive acronym) is a **human-readable data serialization language**. It's designed to be easy for people to read and write while still being straightforward for machines to parse. YAML is the dominant configuration format across the DevOps ecosystem: Kubernetes manifests, Docker Compose, Ansible playbooks, GitHub Actions/GitLab CI pipelines, Helm charts, and countless application config files all use YAML.

### 1.2 Why YAML dominates DevOps

| Reason | Explanation |
|--------|-------------|
| **Human-readable** | Clean, indentation-based; less noisy than JSON/XML |
| **Expressive** | Represents scalars, lists, and maps naturally |
| **Comments** | Supports `#` comments (JSON does not) |
| **Superset of JSON** | Any valid JSON is valid YAML |
| **Ubiquitous** | The config language of K8s, CI/CD, IaC, and more |
| **Language-agnostic** | Parsers exist for every major language |

### 1.3 YAML vs. JSON vs. TOML

| | YAML | JSON | TOML |
|--|------|------|------|
| Readability | High | Medium | High |
| Comments | Yes (`#`) | No | Yes |
| Structure | Indentation | Braces/brackets | Sections |
| Data types | Rich | Basic | Rich |
| Best for | Config, K8s, CI/CD | APIs, data exchange | App config (Rust/Python) |
| Risk | Whitespace/type gotchas | Verbose, no comments | Less nesting-friendly |

> **Key insight:** YAML is a **superset of JSON** — valid JSON can be pasted into a YAML file and it parses correctly.

### 1.4 How YAML represents data

YAML describes three fundamental structures:

```
   Scalars  → single values (string, number, boolean, null)
   Sequences (lists)  → ordered collections
   Mappings (dicts)   → key/value pairs
```

These nest arbitrarily to form documents:

```yaml
# A mapping containing scalars, a sequence, and a nested mapping
name: my-app
version: 1.2.3
enabled: true
ports:           # a sequence
  - 80
  - 443
metadata:        # a nested mapping
  team: platform
  tier: backend
```

### 1.5 The role of indentation

YAML uses **indentation (spaces, never tabs)** to denote structure and nesting — like Python. Indentation level determines parent/child relationships. This makes YAML clean but also makes it sensitive: a single misplaced space changes meaning or causes a parse error.

```
   key:
     child: value      ← 2-space indent makes 'child' belong to 'key'
   sibling: value      ← back to column 0 = sibling of 'key'
```

### 1.6 Parsing model

```
   YAML text ──► parser ──► native data structures (maps/lists/scalars)
                              │
                  consumed by Kubernetes, Ansible, CI, your app, etc.
```

A YAML parser converts the document into the host language's structures (e.g., Python dict/list, Go map/slice). The consuming tool then acts on that data. Understanding that YAML is *just data* (not logic) clarifies what it can and can't do.

### 1.7 When to use YAML

- Configuration files (apps, services, tools).
- Kubernetes manifests, Helm values, Docker Compose.
- CI/CD pipelines (GitHub Actions, GitLab CI, Azure Pipelines).
- Infrastructure as Code (Ansible, CloudFormation templates).

When **not** ideal: data interchange between systems/APIs (JSON is stricter and faster); highly complex logic (YAML is data, not a programming language); very large datasets (binary/columnar formats).

---

## 2. Installation & Setup

### 2.1 YAML needs no runtime

YAML is a data format, not a program — there's nothing to "install" to write it. You validate and process it with parsers/tools.

### 2.2 Editor setup

Use an editor with YAML support (VS Code + the **YAML extension** by Red Hat), which provides:
- Syntax highlighting and indentation guides.
- Schema validation (e.g., Kubernetes, GitHub Actions schemas).
- Auto-completion and error squiggles.

> Configure your editor to **insert spaces, not tabs**, and show whitespace.

### 2.3 Command-line validation tools

```bash
# yamllint — the standard YAML linter
pip install yamllint
yamllint config.yaml

# yq — jq-like processor for YAML
yq '.metadata.name' deployment.yaml
yq -i '.spec.replicas = 3' deployment.yaml   # in-place edit

# Validate by parsing with Python
python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" config.yaml
```

### 2.4 Quick parse check

```bash
python3 -c "import yaml; print(yaml.safe_load(open('config.yaml')))"
```

**Expected:** Prints the parsed Python data structure (dict/list), confirming valid YAML.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Scalars (single values)

```yaml
string1: hello world          # unquoted string
string2: "double quoted"      # allows escapes \n \t
string3: 'single quoted'      # literal, no escapes
integer: 42
float: 3.14
boolean_true: true
boolean_false: false
null_value: null              # or ~
empty:                        # also null
```

#### 3.1.2 Mappings (key/value)

```yaml
person:
  name: Alice
  age: 30
  email: alice@example.com
```

Keys and values are separated by `: ` (colon + **space**). Nesting is by indentation.

#### 3.1.3 Sequences (lists)

```yaml
# Block style
fruits:
  - apple
  - banana
  - cherry

# Flow style (inline, JSON-like)
fruits: [apple, banana, cherry]
```

Each `- ` (dash + space) is a list item.

#### 3.1.4 Combining maps and lists

```yaml
servers:
  - name: web1
    ip: 10.0.0.1
    roles:
      - frontend
      - cache
  - name: db1
    ip: 10.0.0.2
    roles:
      - database
```

A list of mappings — extremely common (e.g., Kubernetes `containers`, CI `steps`).

#### 3.1.5 Comments

```yaml
# This is a full-line comment
name: my-app        # inline comment after a value
# port: 8080        # commented-out line
```

Comments start with `#` and run to end of line. (JSON has no comments — a major reason YAML is preferred for config.)

#### 3.1.6 Indentation rules

- Use **spaces only — never tabs** (tabs cause errors).
- Be **consistent** (2 spaces is the common convention).
- Indentation level defines structure; siblings share the same indent.

### 3.2 INTERMEDIATE

#### 3.2.1 Multi-line strings

```yaml
# Literal block scalar (|) — preserves newlines
description: |
  Line one
  Line two
  Line three

# Folded block scalar (>) — folds newlines into spaces
summary: >
  This long sentence
  becomes a single
  line when parsed.

# Block chomping indicators
keep: |+      # keep trailing newlines
strip: |-     # strip trailing newline
clip: |       # default: single trailing newline
```

- `|` keeps line breaks (great for scripts, certs, config blobs).
- `>` folds lines into spaces (great for long prose).

#### 3.2.2 Quoting and the Norway problem

```yaml
# These unquoted values become booleans, not strings!
country: NO          # → false (the "Norway problem")
answer: yes          # → true
toggle: on           # → true
version: 1.10        # → 1.1 (float!)
zip: 01234           # may lose the leading zero
phone: "0123456789"  # quote to keep as string
```

> **Quote strings that look like booleans, numbers, dates, or have leading zeros.** YAML's implicit typing causes subtle bugs (`NO`, `yes`, `on`, `off`, `true`, version `1.10`). When in doubt, quote.

#### 3.2.3 Explicit typing with tags

```yaml
string_num: !!str 123        # force string
int_val: !!int "42"          # force int
explicit_null: !!null
```

#### 3.2.4 Multiple documents

```yaml
# Three separate documents in one file, split by ---
apiVersion: v1
kind: Service
---
apiVersion: apps/v1
kind: Deployment
---
apiVersion: v1
kind: ConfigMap
```

`---` separates documents; `...` optionally ends one. Kubernetes uses this to bundle many resources in a single file.

#### 3.2.5 Anchors, aliases, and merge keys (DRY)

```yaml
# Define an anchor (&) and reuse it with an alias (*)
defaults: &defaults
  timeout: 30
  retries: 3
  log_level: info

production:
  <<: *defaults          # merge key: inherit all defaults
  log_level: warn        # override one field

staging:
  <<: *defaults
  timeout: 60
```

- `&name` defines an **anchor**.
- `*name` references it (**alias**).
- `<<:` is the **merge key**, injecting a mapping's keys.

This eliminates duplication — common in CI configs and complex app config.

#### 3.2.6 Flow style and empty structures

```yaml
inline_map: {name: Alice, age: 30}
inline_list: [1, 2, 3]
empty_map: {}
empty_list: []
```

### 3.3 ADVANCED

#### 3.3.1 Complex keys and special cases

```yaml
# Quoted keys with special characters
"key:with:colons": value
"key with spaces": value

# Numeric/boolean-looking keys
"123": numeric-looking key
"true": boolean-looking key
```

#### 3.3.2 Type coercion deep dive

YAML's spec versions differ. **YAML 1.1** (used by many tools, including older PyYAML) treats `yes/no/on/off` as booleans; **YAML 1.2** (JSON-compatible core schema) only treats `true/false`. This inconsistency is why explicit quoting is the safe practice across tools.

```yaml
# Risky values to always quote
octal_like: "0o755"
hex_like: "0xFF"
sexagesimal: "12:30:00"   # old YAML parsed as base-60!
```

#### 3.3.3 Anchors at scale and limitations

- Anchors/aliases work **within a single document** only (not across files).
- Tools like Helm/Kustomize/Ansible add their own templating/overlays because YAML anchors alone don't span files or support logic.
- Beware **"YAML bombs"** (billion laughs) — recursive aliases that explode memory; safe parsers limit this.

#### 3.3.4 Schema validation

Validate YAML against a schema to catch errors early:

```yaml
# JSON Schema (YAML is JSON-compatible) describing allowed structure
type: object
required: [name, replicas]
properties:
  name: { type: string }
  replicas: { type: integer, minimum: 1 }
```

Tools: **yamllint** (style/syntax), **JSON Schema** validators, **kubeconform/kubeval** (Kubernetes), editor schema integration. Schemas turn "valid YAML" into "valid *config*".

#### 3.3.5 Processing YAML programmatically

```python
import yaml

# Always use safe_load (load can execute arbitrary objects!)
with open("config.yaml") as f:
    data = yaml.safe_load(f)          # → dict/list

data["replicas"] = 5
with open("out.yaml", "w") as f:
    yaml.safe_dump(data, f, default_flow_style=False, sort_keys=False)

# Multiple documents
docs = list(yaml.safe_load_all(open("manifests.yaml")))
```

> **Security:** Never use `yaml.load()` on untrusted input — it can instantiate arbitrary Python objects. Always `safe_load`.

#### 3.3.6 yq for pipelines

```bash
yq '.spec.template.spec.containers[0].image' deploy.yaml   # read
yq -i '.spec.replicas = 3' deploy.yaml                     # edit in place
yq '... comments=""' file.yaml                             # strip comments
yq ea '. as $item ireduce ({}; . * $item)' a.yaml b.yaml   # merge
yq -o=json '.' config.yaml                                 # convert to JSON
```

`yq` is to YAML what `jq` is to JSON — essential for scripting config changes in CI.

#### 3.3.7 YAML in Kubernetes and CI/CD (in practice)

```yaml
# A real Kubernetes Deployment — every concept in action
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: "nginx:1.25"      # quoted to be safe
          ports:
            - containerPort: 80
          env:
            - name: LOG_LEVEL
              value: "info"
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
```

#### 3.3.8 Templating layered on YAML

Because YAML is static data, tools add templating/overlays:
- **Helm** — Go templates render YAML from values.
- **Kustomize** — patches/overlays plain YAML (no templating).
- **Ansible/Jinja2** — variables inside YAML playbooks.
- **CI engines** — variable interpolation (`${{ }}`, `$VAR`).

Understanding where YAML ends and templating begins prevents confusion (e.g., indentation bugs when templating multi-line strings).

---

## 4. Hands-on Tasks

### Task 1: Write basic scalars and a mapping

Create `person.yaml` with name, age, email, and a boolean.

**Expected:** Parses into a dict with correct types.

### Task 2: Create a list and a list of maps

Write a `servers` list where each item has `name`, `ip`, and a `roles` list.

**Expected:** Parses into a list of dicts, each with a nested list.

### Task 3: Validate with a parser

```bash
python3 -c "import yaml; print(yaml.safe_load(open('servers.yaml')))"
```

**Expected:** The printed structure matches your file.

### Task 4: Use multi-line strings

Add a `script: |` literal block and a `note: >` folded block.

**Expected:** `|` preserves newlines; `>` folds them into spaces when parsed.

### Task 5: Trigger and fix the Norway problem

Set `country: NO` unquoted, parse it (becomes `false`), then quote it to fix.

**Expected:** Unquoted → boolean `false`; quoted → string `"NO"`.

### Task 6: Multiple documents

Put two documents in one file separated by `---`; load with `safe_load_all`.

**Expected:** Two separate parsed documents returned.

### Task 7: DRY with anchors and merge keys

Define `&defaults` and reuse with `<<: *defaults`, overriding one field.

**Expected:** Both consumers inherit defaults; the override applies.

### Task 8: Lint your YAML

```bash
yamllint config.yaml
```

**Expected:** Reports style/syntax issues (e.g., trailing spaces, indentation).

### Task 9: Edit YAML with yq

```bash
yq -i '.spec.replicas = 5' deployment.yaml
```

**Expected:** The replicas value is updated in place.

### Task 10: Convert YAML to JSON

```bash
yq -o=json '.' config.yaml
```

**Expected:** Equivalent JSON is printed (demonstrating YAML ⊇ JSON).

---

## 5. Projects

### Project 1 (Beginner): Application Config File

**Goal:** Design a clean, well-structured config for an app.

Steps:
1. Model app settings: server (host/port), database, feature flags, logging.
2. Use mappings, lists, and appropriate types; **quote risky values**.
3. Add **comments** documenting each section.
4. Validate with `yamllint` and parse with Python.
5. Provide environment overrides (e.g., `config.dev.yaml`, `config.prod.yaml`).

**Skills:** structure, types, quoting, comments, linting.

### Project 2 (Intermediate): Kubernetes Manifest Set

**Goal:** Author a multi-resource Kubernetes deployment by hand.

Steps:
1. Write Deployment, Service, and ConfigMap in one file using `---`.
2. Use labels/selectors consistently; set resources, env, and probes.
3. **Quote** image tags and numeric-like values to avoid type bugs.
4. Validate with **kubeconform**/`kubectl apply --dry-run=client`.
5. Refactor shared values and verify with `yq` queries.

**Skills:** multi-doc YAML, K8s structure, validation, yq.

### Project 3 (Advanced): DRY CI/CD Pipeline with Anchors and Schema

**Goal:** A maintainable, validated pipeline configuration.

Steps:
1. Build a CI pipeline (e.g., GitLab CI) using **anchors/aliases/merge keys** to DRY repeated job definitions.
2. Add **multiple documents**/included templates where supported.
3. Define a **JSON Schema** for a custom config and validate it in CI.
4. Write a **Python/yq script** to programmatically generate or patch YAML safely (`safe_load`/`safe_dump`).
5. Enforce **yamllint** + schema checks as pipeline gates.
6. Document quoting/typing conventions for the team.

**Skills:** anchors/merge keys, schema validation, programmatic processing, CI integration, conventions.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use spaces, never tabs**; be consistent (2 spaces).
- **Quote ambiguous strings** (booleans-like `NO/yes/on`, numbers, versions, leading zeros, dates).
- **Use comments** to document non-obvious config.
- **Use `safe_load`/`safe_dump`** in code; never `yaml.load` on untrusted input.
- **Validate** with yamllint and schemas (JSON Schema, kubeconform).
- **Use anchors/merge keys** to avoid duplication (where the tool supports them).
- **Keep documents focused**; split large configs logically.
- **Prefer block style** for readability; flow style for short inline values.
- **Be explicit with multi-line scalars** (`|` vs `>` and chomping `+`/`-`).
- **Lint in CI** so bad YAML never merges.
- **Mind tool YAML version** (1.1 vs 1.2) differences in implicit typing.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using tabs for indentation | Parse error | Spaces only |
| Inconsistent indentation | Wrong structure/errors | Consistent 2-space indent |
| Unquoted `NO`/`yes`/`on`/`off` | Becomes boolean (Norway problem) | Quote the string |
| Version like `1.10` unquoted | Parsed as float `1.1` | Quote it |
| Leading-zero values (`0123`) | Lost/octal interpretation | Quote it |
| `yaml.load()` on untrusted data | Code execution | Use `safe_load` |
| Missing space after `:` or `-` | Parse error | `key: value`, `- item` |
| Trailing spaces / mixed line endings | Lint errors, diffs | Lint + editor cleanup |
| Over-nesting | Hard to read | Flatten/split files |
| Misindented multi-line block | Wrong/empty values | Verify `|`/`>` indentation |

---

## 7. Interview Questions

### Beginner

1. **What is YAML?**
   *A human-readable data serialization language used widely for configuration (Kubernetes, CI/CD, Ansible, etc.).*

2. **What three data structures does YAML represent?**
   *Scalars (single values), sequences (lists), and mappings (key/value pairs).*

3. **How does YAML denote structure?**
   *Through indentation using spaces (never tabs); indentation level defines nesting.*

4. **How do you write a comment in YAML?**
   *With `#`, which runs to the end of the line.*

5. **What's the relationship between YAML and JSON?**
   *YAML is a superset of JSON — any valid JSON is also valid YAML.*

### Intermediate

6. **What is the "Norway problem"?**
   *Unquoted values like `NO`, `yes`, `on`, `off` are interpreted as booleans (e.g., `NO` → false), causing bugs; quoting the string fixes it.*

7. **What's the difference between `|` and `>` in multi-line strings?**
   *`|` (literal) preserves newlines; `>` (folded) folds newlines into spaces.*

8. **How do anchors and aliases work?**
   *`&name` defines an anchor on a node; `*name` references (aliases) it; `<<:` merge key injects a mapping's keys — used to avoid duplication within a document.*

9. **How do you put multiple documents in one YAML file?**
   *Separate them with `---` (and optionally end with `...`); parsers load them as a sequence of documents.*

10. **Why should you quote a version like `1.10`?**
    *Unquoted it's parsed as a float and becomes `1.1`, losing the trailing zero; quoting keeps it as the string `"1.10"`.*

### Advanced

11. **Why is `yaml.safe_load` preferred over `yaml.load`?**
    *`yaml.load` can instantiate arbitrary objects from tags, allowing code execution on untrusted input; `safe_load` restricts parsing to basic types.*

12. **How do YAML 1.1 and 1.2 differ, and why does it matter?**
    *1.1 treats `yes/no/on/off` as booleans; 1.2 (JSON-compatible core schema) does not. Tools implement different versions, so explicit quoting ensures consistent behavior.*

13. **What are the limitations of anchors/aliases?**
    *They only work within a single document (not across files) and provide no logic; that's why tools add templating (Helm) or overlays (Kustomize).*

14. **What is a "YAML bomb" / billion laughs attack?**
    *A document with recursively nested aliases that expands exponentially to exhaust memory; safe parsers and limits mitigate it.*

15. **How do you ensure YAML config is not just valid syntax but valid configuration?**
    *Validate against a schema (JSON Schema, kubeconform/kubeval for Kubernetes) in addition to linting, catching structural and value errors early.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** YAML indentation must use:
- A) Tabs  B) Spaces  C) Either  D) Commas

**Q2.** Which character starts a comment?
- A) `//`  B) `#`  C) `--`  D) `;`

**Q3.** YAML is a superset of:
- A) XML  B) JSON  C) TOML  D) INI

**Q4.** A list item begins with:
- A) `*`  B) `- `  C) `:`  D) `>`

**Q5.** `description: |` will:
- A) Fold newlines  B) Preserve newlines  C) Strip text  D) Error

**Q6.** Unquoted `NO` parses as:
- A) "NO"  B) false  C) null  D) 0

**Q7.** Documents in one file are separated by:
- A) `***`  B) `---`  C) `//`  D) `==`

**Q8.** `&name` defines an:
- A) Alias  B) Anchor  C) Tag  D) Comment

**Q9.** Which is safe for parsing untrusted YAML in Python?
- A) `yaml.load`  B) `yaml.safe_load`  C) `eval`  D) `exec`

**Q10.** Version `1.10` unquoted becomes:
- A) "1.10"  B) 1.1  C) 110  D) error

### Short Answer

**S1.** Which whitespace character must never be used for indentation?

**S2.** What is the merge key syntax for inheriting a mapping?

**S3.** Name the folded block scalar indicator.

**S4.** Which CLI tool is "jq for YAML"?

**S5.** Name the standard YAML linter.

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-B, Q6-B, Q7-B, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** The tab character.
- **S2.** `<<: *anchor`.
- **S3.** `>` (greater-than sign).
- **S4.** `yq`.
- **S5.** `yamllint`.

---

## 9. Further Resources

### Official Documentation
- YAML spec — https://yaml.org/spec/
- YAML.org — https://yaml.org/
- PyYAML docs — https://pyyaml.org/wiki/PyYAMLDocumentation

### Learning
- Learn YAML in Y Minutes — https://learnxinyminutes.com/docs/yaml/
- YAML tutorial (Kubernetes docs) — https://kubernetes.io/docs/concepts/
- "Norway problem" explainer — https://hitchdev.com/strictyaml/why/implicit-typing-removed/

### Tools
- yamllint — https://github.com/adrienverge/yamllint
- yq — https://github.com/mikefarah/yq
- kubeconform / kubeval (K8s schema validation)
- Red Hat YAML extension for VS Code
- StrictYAML (typed, safer subset)

### Books & Guides
- *YAML for DevOps* style guides
- Kubernetes / Ansible / GitHub Actions docs (practical YAML)

---

*End of YAML — Zero to Hero.*
