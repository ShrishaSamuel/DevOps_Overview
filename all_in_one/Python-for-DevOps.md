# Python for DevOps — Zero to Hero

> A complete, beginner-to-advanced guide to using Python for automation, tooling, and infrastructure in DevOps.

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

### 1.1 Why Python for DevOps?

**Python** is the de facto language for DevOps automation and tooling. Where Bash excels at gluing CLI commands, Python takes over when you need **real data structures, robust error handling, APIs, and maintainable code**. Many core DevOps tools (Ansible, SaltStack, AWS CLI, parts of OpenStack) are written in Python, and every major cloud and service exposes a Python SDK.

| Reason | Explanation |
|--------|-------------|
| **Readable & maintainable** | Clear syntax; scales far better than large Bash scripts |
| **Rich ecosystem** | SDKs/libraries for every cloud, API, and protocol |
| **Cross-platform** | Runs on Linux, macOS, Windows |
| **Batteries included** | Strong standard library (os, subprocess, json, http, pathlib) |
| **Glue + logic** | Combines system automation with complex logic and data handling |
| **Ubiquitous in tooling** | Ansible, cloud SDKs, CI scripts, monitoring exporters |

### 1.2 Bash vs. Python — when to use which

```
   Task complexity / data structure needs
   low ──────────────────────────────────► high
   │                                          │
   Bash                                     Python
   (glue CLI commands,            (APIs, JSON/YAML, data structures,
    quick file ops, pipelines)     error handling, large automation,
                                    testing, reusable tooling)
```

**Rule of thumb:** if a script needs loops over structured data, talks to APIs, parses JSON/YAML, needs unit tests, or exceeds ~100–200 lines of logic, use Python.

### 1.3 How Python runs

```
   your_script.py
        │ CPython interpreter
        ▼
   Lexer/Parser → bytecode (.pyc) → Python Virtual Machine → OS calls
```

Python is **interpreted** (compiled to bytecode at runtime). You run scripts with `python script.py` or make them executable with a shebang `#!/usr/bin/env python3`.

### 1.4 Core DevOps use cases

- **Automation scripts** — provisioning, deployments, backups, log processing.
- **Cloud automation** — boto3 (AWS), google-cloud (GCP), azure-sdk.
- **API integration** — REST calls to CI/CD, monitoring, ticketing, chat.
- **Config & data** — parse/generate JSON, YAML, TOML, CSV.
- **Infrastructure tooling** — custom CLIs, operators, exporters.
- **CI/CD logic** — build orchestration, test runners, release automation.
- **Monitoring** — Prometheus exporters, health checks, alerting glue.

### 1.5 The Python ecosystem essentials

| Component | Purpose |
|-----------|---------|
| **CPython** | The reference interpreter (`python3`) |
| **pip** | Package installer |
| **venv** | Isolated virtual environments |
| **PyPI** | The Python Package Index (300k+ packages) |
| **Standard library** | Built-in modules (no install) |
| **Type hints** | Optional static typing for safer code |

### 1.6 When NOT to use Python

For trivial one-liners or pure command pipelines, Bash is faster to write. For performance-critical systems code, Go/Rust may be better (and Go produces single static binaries — popular for CLI tools). Python remains the best general-purpose automation language for most DevOps work.

---

## 2. Installation & Setup

### 2.1 Install Python 3

```bash
python3 --version            # check (3.10+ recommended)

# Debian/Ubuntu
sudo apt update && sudo apt install -y python3 python3-pip python3-venv

# macOS
brew install python

# Windows: install from python.org or `winget install Python.Python.3`
```

### 2.2 Virtual environments (always use them)

```bash
python3 -m venv .venv          # create
source .venv/bin/activate      # activate (Linux/macOS)
# .venv\Scripts\activate       # Windows
pip install requests boto3     # installs into the venv only
deactivate                     # leave
```

> **Always** isolate project dependencies in a venv to avoid polluting system Python and dependency conflicts.

### 2.3 Managing dependencies

```bash
pip install requests
pip freeze > requirements.txt     # pin exact versions
pip install -r requirements.txt   # reproduce environment
```

Modern tooling: **`uv`** (fast installer/resolver), **Poetry**, or **pipenv** with `pyproject.toml` for richer dependency management.

### 2.4 First script

```python
#!/usr/bin/env python3
"""Greet the current user."""
import getpass
from datetime import date

def main() -> None:
    print(f"Hello, {getpass.getuser()}! Today is {date.today()}.")

if __name__ == "__main__":
    main()
```

```bash
chmod +x hello.py && ./hello.py
```

**Expected:** Prints a greeting with username and today's date.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Variables, types, and f-strings

```python
name = "Alice"          # str
count = 5               # int
ratio = 3.14            # float
active = True           # bool
items = ["a", "b"]      # list
config = {"env": "prod"}  # dict

print(f"{name} has {count} items in {config['env']}")   # f-string
```

#### 3.1.2 Data structures

```python
# List (ordered, mutable)
servers = ["web1", "web2"]
servers.append("web3")
for s in servers: print(s)

# Dict (key/value)
ports = {"http": 80, "https": 443}
ports["ssh"] = 22
for k, v in ports.items(): print(k, v)

# Set (unique)
tags = {"prod", "web", "prod"}   # → {'prod', 'web'}

# Tuple (immutable)
coord = (10, 20)
```

#### 3.1.3 Control flow

```python
if count > 10:
    print("many")
elif count == 0:
    print("none")
else:
    print("few")

for i in range(5):
    print(i)

while count > 0:
    count -= 1

# Comprehensions (Pythonic)
squares = [x*x for x in range(5)]
prod_servers = [s for s in servers if "prod" in s]
```

#### 3.1.4 Functions

```python
def deploy(app: str, replicas: int = 1) -> bool:
    """Deploy an app with the given replica count."""
    print(f"Deploying {app} x{replicas}")
    return True

deploy("api", replicas=3)
```

Type hints (`app: str`, `-> bool`) document intent and enable static checkers like **mypy**.

#### 3.1.5 Running shell commands

```python
import subprocess

# Preferred: run, capture output, check errors
result = subprocess.run(
    ["ls", "-l", "/tmp"],
    capture_output=True, text=True, check=True
)
print(result.stdout)

# Check return code
r = subprocess.run(["systemctl", "is-active", "nginx"], capture_output=True, text=True)
if r.returncode == 0:
    print("nginx active")
```

> Use a **list of arguments** (not a string) and avoid `shell=True` with untrusted input to prevent command injection.

#### 3.1.6 Working with files and paths

```python
from pathlib import Path

p = Path("/var/log/app.log")
print(p.exists(), p.name, p.suffix, p.parent)

# Read/write
text = Path("config.txt").read_text()
Path("out.txt").write_text("hello\n")

# Context manager (auto-close)
with open("data.txt") as f:
    for line in f:
        print(line.rstrip())
```

### 3.2 INTERMEDIATE

#### 3.2.1 Working with JSON and YAML

```python
import json

# JSON (standard library)
data = json.loads('{"name": "api", "replicas": 3}')
print(data["replicas"])
text = json.dumps(data, indent=2)

with open("config.json") as f:
    config = json.load(f)

# YAML (pip install pyyaml)
import yaml
with open("config.yaml") as f:
    cfg = yaml.safe_load(f)          # always safe_load (not load)
yaml.safe_dump(cfg, open("out.yaml", "w"))
```

#### 3.2.2 HTTP and APIs with requests

```python
import requests

resp = requests.get(
    "https://api.example.com/status",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10,
)
resp.raise_for_status()              # raise on 4xx/5xx
data = resp.json()

# POST JSON
requests.post("https://api.example.com/deploy",
              json={"app": "api", "version": "1.2.3"},
              timeout=10)
```

#### 3.2.3 Error handling

```python
try:
    resp = requests.get(url, timeout=5)
    resp.raise_for_status()
except requests.Timeout:
    print("request timed out")
except requests.HTTPError as e:
    print(f"HTTP error: {e}")
except requests.RequestException as e:
    print(f"request failed: {e}")
finally:
    print("done")

# Custom exceptions
class DeploymentError(Exception):
    pass
```

#### 3.2.4 Modules, packages, and imports

```python
# mymodule.py
def helper(): ...

# main.py
from mymodule import helper
import os, sys, json
from pathlib import Path

# Project layout
# myproject/
#   __init__.py
#   cli.py
#   utils/
#     __init__.py
#     net.py
```

#### 3.2.5 Command-line interfaces

```python
import argparse

def main() -> None:
    parser = argparse.ArgumentParser(description="Deploy tool")
    parser.add_argument("app", help="application name")
    parser.add_argument("-r", "--replicas", type=int, default=1)
    parser.add_argument("-v", "--verbose", action="store_true")
    args = parser.parse_args()
    print(f"Deploying {args.app} x{args.replicas}")

if __name__ == "__main__":
    main()
```

For richer CLIs: **Click** or **Typer** (type-hint based).

#### 3.2.6 Environment variables and config

```python
import os

env = os.environ.get("APP_ENV", "dev")     # with default
token = os.environ["API_TOKEN"]            # required (KeyError if missing)

# Load .env files (pip install python-dotenv)
from dotenv import load_dotenv
load_dotenv()
```

#### 3.2.7 Logging (not print)

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
log = logging.getLogger(__name__)

log.info("Starting deployment")
log.warning("Retrying...")
log.error("Failed to connect")
```

> Use the `logging` module (levels, formatting, handlers) instead of `print` for real tools.

### 3.3 ADVANCED

#### 3.3.1 Cloud automation (boto3 example)

```python
import boto3

ec2 = boto3.client("ec2", region_name="us-east-1")

# List running instances
resp = ec2.describe_instances(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
)
for r in resp["Reservations"]:
    for inst in r["Instances"]:
        print(inst["InstanceId"], inst["InstanceType"])

# S3 upload
s3 = boto3.client("s3")
s3.upload_file("backup.tar.gz", "my-bucket", "backups/backup.tar.gz")
```

Each cloud has an SDK: **boto3** (AWS), **google-cloud-*** (GCP), **azure-sdk-for-python** (Azure). Prefer instance/workload identities over hardcoded keys.

#### 3.3.2 Concurrency

```python
from concurrent.futures import ThreadPoolExecutor

def check(host: str) -> tuple[str, int]:
    return host, requests.get(f"https://{host}/health", timeout=5).status_code

hosts = ["a.example.com", "b.example.com", "c.example.com"]
with ThreadPoolExecutor(max_workers=10) as pool:
    for host, status in pool.map(check, hosts):
        print(host, status)
```

- **Threads** — good for I/O-bound work (network/API calls); GIL limits CPU parallelism.
- **multiprocessing** — for CPU-bound work.
- **asyncio** — high-concurrency async I/O (`async`/`await`).

#### 3.3.3 Type hints and static analysis

```python
from typing import Optional

def get_config(path: str, default: Optional[dict] = None) -> dict:
    ...
```

Run **mypy** for static type checking, **ruff**/**flake8** for linting, **black** for formatting — catch bugs before runtime.

#### 3.3.4 Testing

```python
# test_deploy.py
import pytest
from deploy import compute_replicas

def test_replicas():
    assert compute_replicas(load=80) == 4

def test_invalid():
    with pytest.raises(ValueError):
        compute_replicas(load=-1)
```

```bash
pytest -v
```

Use **pytest** (fixtures, parametrization, mocking with `unittest.mock`/`monkeypatch`). Tests make automation safe to change.

#### 3.3.5 Dataclasses and structured data

```python
from dataclasses import dataclass

@dataclass
class Server:
    name: str
    cpu: int
    region: str = "us-east-1"

s = Server("web1", cpu=4)
print(s.name, s.cpu, s.region)
```

For validation/parsing config and API payloads, **pydantic** is widely used in DevOps tooling.

#### 3.3.6 Packaging and distribution

```toml
# pyproject.toml
[project]
name = "mytool"
version = "0.1.0"
dependencies = ["requests", "pyyaml"]

[project.scripts]
mytool = "mytool.cli:main"
```

```bash
pip install -e .          # editable install
pipx install mytool        # install CLI globally, isolated
```

Package CLIs with `pyproject.toml` + entry points; distribute via PyPI or as a container.

#### 3.3.7 Robust automation patterns

- **Idempotency:** check current state before acting (don't recreate what exists).
- **Retries with backoff:** use `tenacity` for transient failures.
- **Timeouts:** always set timeouts on network calls.
- **Dry-run mode:** preview changes before applying.
- **Structured logging + exit codes** for CI integration.

```python
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(multiplier=1, max=30), stop=stop_after_attempt(5))
def fetch():
    return requests.get(url, timeout=5).json()
```

#### 3.3.8 Building a Prometheus exporter (example)

```python
from prometheus_client import start_http_server, Gauge
import time, random

queue_depth = Gauge("app_queue_depth", "Items in the queue")

if __name__ == "__main__":
    start_http_server(8000)              # exposes /metrics
    while True:
        queue_depth.set(random.randint(0, 100))
        time.sleep(5)
```

Python's `prometheus_client` makes custom metrics/exporters trivial — a common DevOps task.

---

## 4. Hands-on Tasks

### Task 1: Set up a venv and install a package

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install requests && pip freeze > requirements.txt
```

**Expected:** `requests` installs into the venv; `requirements.txt` lists it.

### Task 2: Parse a JSON config

Write a script that loads `config.json` and prints a nested value.

**Expected:** The chosen field is printed.

### Task 3: Run a shell command safely

Use `subprocess.run` with a list and `check=True`; capture and print output.

**Expected:** Command output printed; errors raise an exception.

### Task 4: Iterate files with pathlib

List all `*.log` files in a directory and print their sizes.

**Expected:** One line per log file with its byte size.

### Task 5: Call a REST API

Use `requests` to GET a public API (e.g., GitHub) and print a field from the JSON.

**Expected:** A value from the API response is displayed.

### Task 6: Build a CLI with argparse

Create a tool taking a positional `app` and `--replicas`; print the parsed args.

**Expected:** Arguments parse correctly; `-h` shows help.

### Task 7: Add logging and error handling

Wrap an API call in try/except with timeouts and log at INFO/ERROR levels.

**Expected:** Successes log INFO; failures are caught and logged, not crashing.

### Task 8: Read environment variables

Read `APP_ENV` (with default) and a required `API_TOKEN`; handle the missing case.

**Expected:** Uses default when unset; clear error when required var is missing.

### Task 9: Concurrency for health checks

Use `ThreadPoolExecutor` to check several URLs' health concurrently.

**Expected:** Statuses returned faster than sequential calls.

### Task 10: Write a pytest test

Write a pure function and a `pytest` test (including an error case); run `pytest`.

**Expected:** Tests pass; the error case asserts the raised exception.

---

## 5. Projects

### Project 1 (Beginner): Log Parser / System Report Tool

**Goal:** A maintainable replacement for a fragile Bash script.

Steps:
1. Read a log directory with **pathlib**; count errors/warnings.
2. Summarize top messages using dicts/`collections.Counter`.
3. Build an **argparse** CLI (`--since`, `--level`, `--json`).
4. Add **logging**, error handling, and a `--dry-run`.
5. Output a JSON report; add a **pytest** test.

**Skills:** files, data structures, CLI, logging, testing.

### Project 2 (Intermediate): Cloud Resource Automation

**Goal:** Automate a cloud task via SDK.

Steps:
1. Use **boto3** (or another SDK) to list/tag/clean up resources (e.g., stop untagged instances, delete old snapshots).
2. Authenticate via instance/workload identity (no hardcoded keys).
3. Add **idempotency**, **dry-run**, **retries** (`tenacity`), and timeouts.
4. Parse config from YAML; structured **logging**; meaningful exit codes.
5. Cover logic with **pytest** (mock the SDK).

**Skills:** cloud SDKs, idempotency, retries, config, testing/mocking.

### Project 3 (Advanced): Custom DevOps CLI / Operator Tool

**Goal:** A packaged, tested, production-grade tool.

Steps:
1. Build a multi-command CLI with **Typer/Click** (deploy, status, rollback).
2. Integrate multiple APIs (CI/CD, cloud, chat notifications).
3. Use **dataclasses/pydantic** for config and validation.
4. Add **concurrency** (threads/async) for parallel operations.
5. Full quality gate: **type hints + mypy**, **ruff**, **black**, **pytest** with coverage.
6. **Package** with `pyproject.toml` + entry points; distribute via PyPI/pipx or a container; wire into CI.

**Skills:** advanced CLIs, API integration, validation, concurrency, packaging, full tooling/quality.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Always use virtual environments**; pin dependencies (`requirements.txt`/`pyproject.toml`).
- **Use `subprocess.run` with arg lists**, `check=True`, timeouts; avoid `shell=True` with untrusted input.
- **Use `pathlib`** for paths and **context managers** (`with`) for files/resources.
- **Use `logging`, not `print`**, in real tools.
- **Add type hints** and run **mypy/ruff/black**.
- **Handle errors explicitly**; set timeouts and retries on network calls.
- **Use `json`/`yaml.safe_load`** for structured data (never `yaml.load`/`eval`).
- **Write tests with pytest**; mock external systems.
- **Make automation idempotent** with dry-run modes.
- **Never hardcode secrets**; use env vars / secret managers / cloud identities.
- **Follow PEP 8**; keep functions small and `if __name__ == "__main__":` guarded.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Installing into system Python | Dependency conflicts | Use `venv` |
| `shell=True` with user input | Command injection | Arg lists, validate input |
| `yaml.load` / `eval` on input | Code execution | `yaml.safe_load`, `json` |
| No timeouts on requests | Hanging scripts | Set `timeout=` |
| Bare `except:` | Hides bugs, swallows Ctrl-C | Catch specific exceptions |
| `print` instead of logging | No levels/control | Use `logging` |
| Hardcoded secrets | Credential leak | Env vars / secret managers |
| Mutable default args (`def f(x=[])`) | Shared state bugs | Use `None` + create inside |
| Ignoring exit codes in CI | Failures pass silently | `sys.exit(code)` on errors |
| No tests for automation | Risky changes | pytest + mocks |

---

## 7. Interview Questions

### Beginner

1. **Why use Python over Bash for DevOps?**
   *Python offers real data structures, robust error handling, API/SDK access, testing, and maintainability — better for anything beyond simple command gluing.*

2. **What is a virtual environment and why use it?**
   *An isolated Python environment (`venv`) that keeps project dependencies separate from system Python, preventing conflicts.*

3. **What is `pip`?**
   *Python's package installer that fetches libraries from PyPI; use `requirements.txt` to pin and reproduce them.*

4. **What does `if __name__ == "__main__":` do?**
   *Runs the block only when the file is executed directly, not when imported as a module.*

5. **How do you read a file safely?**
   *Use a context manager: `with open(path) as f:` so the file is automatically closed.*

### Intermediate

6. **How do you run shell commands safely in Python?**
   *Use `subprocess.run` with an argument list, `check=True`, and a timeout; avoid `shell=True` with untrusted input to prevent injection.*

7. **How do you parse and emit JSON/YAML?**
   *Use the `json` module (`loads`/`dumps`) and PyYAML's `safe_load`/`safe_dump` — never `yaml.load` on untrusted data.*

8. **How should errors be handled around API calls?**
   *Wrap calls in try/except catching specific exceptions (Timeout, HTTPError), set timeouts, and use `raise_for_status()`; add retries for transient failures.*

9. **What is the difference between a list, tuple, set, and dict?**
   *List: ordered/mutable; tuple: ordered/immutable; set: unordered unique elements; dict: key/value mapping.*

10. **Why use `logging` instead of `print`?**
    *`logging` provides levels, formatting, handlers, and configurable output suitable for production tooling and CI.*

### Advanced

11. **When would you use threads vs. multiprocessing vs. asyncio?**
    *Threads for I/O-bound concurrency (limited by the GIL for CPU), multiprocessing for CPU-bound parallelism, and asyncio for very high-concurrency async I/O.*

12. **What is the GIL and why does it matter?**
    *The Global Interpreter Lock allows only one thread to execute Python bytecode at a time, so threads don't speed up CPU-bound work — use multiprocessing for that.*

13. **How do you make automation idempotent and resilient?**
    *Check current state before acting, provide dry-run modes, set timeouts, and add retries with exponential backoff (e.g., tenacity) for transient errors.*

14. **What is the mutable default argument pitfall?**
    *Default values like `def f(x=[])` are created once and shared across calls, causing state leakage; use `x=None` and create the object inside the function.*

15. **How do you ensure code quality in a Python DevOps tool?**
    *Type hints checked by mypy, linting (ruff/flake8), formatting (black), tests with pytest and mocks, pinned dependencies, and packaging with `pyproject.toml`.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Which isolates project dependencies?
- A) pip  B) venv  C) PyPI  D) GIL

**Q2.** Safe way to run a command:
- A) `os.system(userinput)`  B) `subprocess.run([...], check=True)`  C) `eval(cmd)`  D) `shell=True` always

**Q3.** Which loads YAML safely?
- A) `yaml.load`  B) `yaml.safe_load`  C) `eval`  D) `json.loads`

**Q4.** `if __name__ == "__main__":` ensures code runs when:
- A) Imported  B) Executed directly  C) Always  D) Never

**Q5.** Which library makes HTTP requests?
- A) pathlib  B) requests  C) argparse  D) logging

**Q6.** For I/O-bound concurrency you'd use:
- A) multiprocessing  B) threads/asyncio  C) GIL  D) venv

**Q7.** Which is a mutable, ordered collection?
- A) tuple  B) set  C) list  D) frozenset

**Q8.** Which builds a CLI in the standard library?
- A) Click  B) Typer  C) argparse  D) pydantic

**Q9.** You should log with:
- A) print  B) logging  C) sys.stdout.write  D) echo

**Q10.** The GIL primarily limits:
- A) I/O concurrency  B) CPU-bound thread parallelism  C) Memory  D) Disk

### Short Answer

**S1.** Command to create a virtual environment.

**S2.** Which method raises an exception on a 4xx/5xx HTTP response in requests?

**S3.** What's the safe way to read a required environment variable?

**S4.** Name the standard testing framework commonly used.

**S5.** What file pins exact dependency versions?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-B, Q6-B, Q7-C, Q8-C, Q9-B, Q10-B

**Short Answer:**
- **S1.** `python3 -m venv .venv`.
- **S2.** `response.raise_for_status()`.
- **S3.** `os.environ["VAR"]` (raises KeyError if missing) — or validate explicitly.
- **S4.** pytest.
- **S5.** `requirements.txt` (or a locked `pyproject.toml`).

---

## 9. Further Resources

### Official Documentation
- Python docs — https://docs.python.org/3/
- Standard library — https://docs.python.org/3/library/
- subprocess — https://docs.python.org/3/library/subprocess.html
- PEP 8 (style) — https://peps.python.org/pep-0008/

### Learning
- Real Python — https://realpython.com/
- Automate the Boring Stuff — https://automatetheboringstuff.com/
- Python for DevOps (O'Reilly resources)

### Tools & Libraries
- requests, boto3, PyYAML, pydantic, tenacity
- Click / Typer (CLIs), prometheus_client
- pytest, mypy, ruff, black, uv, pipx, Poetry

### Books
- *Python for DevOps* — Gift, Behrman, Deza, Forbes (O'Reilly)
- *Automate the Boring Stuff with Python* — Al Sweigart
- *Fluent Python* — Luciano Ramalho
- *Effective Python* — Brett Slatkin

### Certifications / Paths
- PCEP/PCAP (Python Institute)
- Cloud provider Python SDK learning paths

---

*End of Python for DevOps — Zero to Hero.*
