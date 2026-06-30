# Ansible — Zero to Hero

> A complete, beginner-to-advanced guide to Ansible for configuration management, automation, and orchestration.

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

### 1.1 What is Ansible?

**Ansible** is an open-source **automation tool** used for **configuration management, application deployment, orchestration, and provisioning**. Created by Michael DeHaan in 2012 and acquired by Red Hat in 2015, Ansible automates IT tasks by describing the desired state of systems in simple, human-readable **YAML** files called **playbooks**. Its tagline is "simple, agentless automation."

### 1.2 What problem does it solve?

Managing dozens, hundreds, or thousands of servers by hand (SSHing in and running commands) is slow, error-prone, and inconsistent. Ansible lets you:
- Configure many machines identically and repeatably.
- Deploy applications consistently across environments.
- Orchestrate multi-tier rollouts (e.g., update web servers, then app servers, then database).
- Enforce a known, documented state ("desired state").

### 1.3 Key characteristics

| Feature | Explanation |
|---------|-------------|
| **Agentless** | No software to install on managed nodes — uses SSH (Linux) or WinRM (Windows) |
| **Declarative + procedural** | Describe desired state; tasks run in defined order |
| **Idempotent** | Running a playbook repeatedly yields the same result; no unintended changes |
| **Push-based** | Control node pushes config to managed nodes (vs. pull like Puppet) |
| **Human-readable** | YAML playbooks are easy to read and write |
| **Extensible** | Thousands of modules + custom modules/plugins |

### 1.4 Agentless architecture

```
   +------------------------+
   |     Control Node       |   (where Ansible is installed)
   |  - Inventory           |
   |  - Playbooks (YAML)    |
   |  - Modules             |
   +-----------+------------+
               │  SSH / WinRM (push)
   ┌───────────┼─────────────────────────┐
   ▼           ▼                          ▼
+--------+  +--------+               +--------+
| node1  |  | node2  |     ...       | nodeN  |   (Managed Nodes)
| (no    |  | (no    |               | (no    |   No Ansible agent required;
| agent) |  | agent) |               | agent) |   just SSH + Python
+--------+  +--------+               +--------+
```

- **Control Node:** The machine where Ansible runs. Only it needs Ansible installed.
- **Managed Nodes:** The servers being configured. They need SSH access and (usually) Python — no Ansible agent.
- **Inventory:** A list of managed nodes, optionally grouped.
- **Modules:** Units of work (e.g., install a package, copy a file) executed on managed nodes.
- **Playbooks:** YAML files defining the automation.

### 1.5 How Ansible runs

1. You run a playbook against an inventory.
2. Ansible connects to each managed node via SSH.
3. It copies the relevant module code to the node, executes it (typically with Python), collects results, and removes the temporary code.
4. Modules are **idempotent** — they check current state and only make changes if needed, reporting `changed` or `ok`.

### 1.6 Ansible vs. other tools

| Tool | Model | Language | Agent | Primary use |
|------|-------|----------|-------|-------------|
| **Ansible** | Push | YAML | Agentless | Config mgmt + orchestration |
| Puppet | Pull | Puppet DSL | Agent | Config management |
| Chef | Pull | Ruby DSL | Agent | Config management |
| Terraform | Push | HCL | Agentless | Infrastructure provisioning |
| SaltStack | Push/Pull | YAML/Python | Agent (or SSH) | Config mgmt + remote exec |

**Terraform vs. Ansible:** Terraform *provisions* infrastructure (creates VMs, networks); Ansible *configures* what's inside (installs software, sets config). They're complementary — often Terraform creates servers and Ansible configures them.

### 1.7 When to use Ansible

- Configuring and maintaining fleets of servers consistently.
- Application deployment and rolling updates.
- Orchestrating multi-step, multi-tier operations.
- One-off ad-hoc administration across many hosts.

When **not** ideal: pure infrastructure provisioning at large scale (Terraform fits better), or when you need a continuously enforcing agent (Puppet's pull model).

---

## 2. Installation & Setup

### 2.1 Install Ansible (control node)

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# RHEL/Fedora
sudo dnf install -y ansible

# Via pip (any platform)
python3 -m pip install --user ansible

# Verify
ansible --version
ansible-community --version
```

> The control node must be Linux/macOS (not natively Windows; use WSL on Windows). Managed nodes can be Linux (SSH) or Windows (WinRM).

### 2.2 Set up SSH access to managed nodes

```bash
ssh-keygen -t ed25519                       # generate a key (if needed)
ssh-copy-id user@managed-node-ip            # push your public key
ssh user@managed-node-ip                    # confirm passwordless login
```

### 2.3 Create an inventory

```ini
# inventory.ini
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[dbservers]
db1 ansible_host=192.168.1.20

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

### 2.4 Test connectivity

```bash
ansible all -i inventory.ini -m ping
```

**Expected:**

```
web1 | SUCCESS => { "ping": "pong" }
web2 | SUCCESS => { "ping": "pong" }
db1  | SUCCESS => { "ping": "pong" }
```

### 2.5 Configuration file

```ini
# ansible.cfg (in project directory)
[defaults]
inventory = ./inventory.ini
host_key_checking = False
remote_user = ubuntu
retry_files_enabled = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Ad-hoc commands

Quick one-off tasks without writing a playbook:

```bash
ansible all -m ping                              # connectivity
ansible webservers -m command -a "uptime"        # run a command
ansible all -m shell -a "df -h"                  # shell (allows pipes/redirects)
ansible all -m setup                             # gather facts
ansible webservers -m apt -a "name=nginx state=present" --become   # install pkg
ansible all -m copy -a "src=./file dest=/tmp/file" # copy a file
ansible all -m service -a "name=nginx state=started" --become
```

#### 3.1.2 Modules

**Modules** are the building blocks — discrete units of work. Ansible ships thousands. Common ones:

| Module | Purpose |
|--------|---------|
| `ping` | Test connectivity |
| `command` / `shell` | Run commands (shell supports pipes/vars) |
| `apt` / `yum` / `dnf` / `package` | Manage packages |
| `copy` / `template` | Place files (template renders Jinja2) |
| `file` | Manage files/dirs/permissions/links |
| `service` / `systemd` | Manage services |
| `user` / `group` | Manage users/groups |
| `lineinfile` / `blockinfile` | Edit file content |
| `git` | Clone/checkout repos |
| `uri` | HTTP requests |
| `debug` | Print messages/variables |

#### 3.1.3 Your first playbook

A **playbook** is a YAML file describing plays; each play maps a group of hosts to a list of tasks.

```yaml
# webserver.yml
---
- name: Configure web servers
  hosts: webservers
  become: true                       # run with sudo
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled
      service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy a custom index page
      copy:
        content: "<h1>Managed by Ansible</h1>"
        dest: /var/www/html/index.html
```

```bash
ansible-playbook -i inventory.ini webserver.yml
ansible-playbook webserver.yml --check          # dry run (no changes)
ansible-playbook webserver.yml --diff           # show file diffs
ansible-playbook webserver.yml -v               # verbose (-vvv for more)
```

#### 3.1.4 Understanding output

- **ok** — task ran, no change needed (already in desired state).
- **changed** — task made a change.
- **failed** — task errored.
- **skipped** — task condition not met.
- **unreachable** — couldn't connect to the host.

The `PLAY RECAP` at the end summarizes per host.

#### 3.1.5 Idempotency

Running a well-written playbook twice should show `changed` the first time and `ok` (no changes) the second. This is **idempotency** — the cornerstone of safe, repeatable automation. Use proper modules (not raw `shell`) to preserve it.

### 3.2 INTERMEDIATE

#### 3.2.1 Variables

```yaml
- name: Use variables
  hosts: webservers
  vars:
    http_port: 80
    app_name: myapp
  tasks:
    - name: Print variable
      debug:
        msg: "Deploying {{ app_name }} on port {{ http_port }}"
```

Variable sources (precedence low→high, simplified): role defaults → inventory vars → playbook vars → `-e` extra vars (highest).

```bash
ansible-playbook site.yml -e "app_name=prodapp http_port=8080"
```

External variable files:

```yaml
  vars_files:
    - vars/common.yml
    - vars/{{ env }}.yml
```

#### 3.2.2 Facts

**Facts** are system properties gathered automatically (`gather_facts: true` by default).

```yaml
    - name: Show OS family
      debug:
        msg: "OS is {{ ansible_facts['os_family'] }}, {{ ansible_facts['distribution'] }}"

    - name: Show memory
      debug:
        var: ansible_facts['memtotal_mb']
```

```bash
ansible web1 -m setup                          # see all facts
ansible web1 -m setup -a "filter=ansible_distribution*"
```

#### 3.2.3 Conditionals

```yaml
    - name: Install Apache on RedHat
      yum:
        name: httpd
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Install Apache on Debian
      apt:
        name: apache2
        state: present
      when: ansible_facts['os_family'] == "Debian"

    - name: Only when a variable is true
      debug: msg="enabled"
      when: feature_enabled | bool
```

#### 3.2.4 Loops

```yaml
    - name: Install multiple packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - htop

    - name: Create multiple users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.group }}"
      loop:
        - { name: alice, group: devs }
        - { name: bob, group: ops }
```

#### 3.2.5 Handlers and notifications

**Handlers** run only when notified by a changed task — ideal for restarting services after config changes.

```yaml
  tasks:
    - name: Deploy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx          # only fires if the file changed

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

Handlers run **once**, at the end of the play, even if notified multiple times.

#### 3.2.6 Templates (Jinja2)

`template` renders **Jinja2** templates with variables/facts.

```jinja
{# templates/nginx.conf.j2 #}
server {
    listen {{ http_port }};
    server_name {{ ansible_facts['hostname'] }};
    root /var/www/{{ app_name }};

    {% for loc in locations %}
    location {{ loc.path }} {
        proxy_pass {{ loc.backend }};
    }
    {% endfor %}
}
```

```yaml
    - name: Render nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
      notify: Restart nginx
```

#### 3.2.7 Tags

Run or skip subsets of tasks.

```yaml
    - name: Install packages
      apt: { name: nginx, state: present }
      tags: [install]

    - name: Configure
      template: { src: app.j2, dest: /etc/app.conf }
      tags: [config]
```

```bash
ansible-playbook site.yml --tags config
ansible-playbook site.yml --skip-tags install
```

#### 3.2.8 Error handling

```yaml
    - name: Try something risky
      command: /opt/maybe-fails.sh
      register: result
      ignore_errors: true

    - name: React to failure
      debug: msg="It failed but we continue"
      when: result.rc != 0

    - name: Block with rescue
      block:
        - command: /opt/step1.sh
        - command: /opt/step2.sh
      rescue:
        - debug: msg="Recovering from failure"
      always:
        - debug: msg="This always runs"
```

### 3.3 ADVANCED

#### 3.3.1 Roles

**Roles** are the standard way to organize and reuse Ansible content. A role has a fixed directory structure:

```
roles/
└── webserver/
    ├── tasks/main.yml          # tasks
    ├── handlers/main.yml       # handlers
    ├── templates/              # Jinja2 templates
    ├── files/                  # static files
    ├── vars/main.yml           # role variables (high precedence)
    ├── defaults/main.yml       # default variables (low precedence)
    ├── meta/main.yml           # role metadata/dependencies
    └── README.md
```

```bash
ansible-galaxy role init roles/webserver        # scaffold a role
```

Use roles in a playbook:

```yaml
- name: Configure web tier
  hosts: webservers
  become: true
  roles:
    - common
    - webserver
    - { role: monitoring, when: enable_monitoring }
```

#### 3.3.2 Ansible Galaxy

**Galaxy** is the public hub for sharing roles and collections.

```bash
ansible-galaxy role install geerlingguy.nginx
ansible-galaxy collection install community.general
ansible-galaxy role list
```

Use a `requirements.yml`:

```yaml
roles:
  - name: geerlingguy.docker
collections:
  - name: community.general
  - name: amazon.aws
```

```bash
ansible-galaxy install -r requirements.yml
```

#### 3.3.3 Collections

**Collections** bundle modules, roles, and plugins (the modern distribution format). Reference modules by FQCN (Fully Qualified Collection Name):

```yaml
    - name: Create an S3 bucket
      amazon.aws.s3_bucket:
        name: my-bucket
        state: present
```

#### 3.3.4 Ansible Vault (secrets)

Encrypt sensitive data (passwords, keys) at rest.

```bash
ansible-vault create secrets.yml          # create encrypted file
ansible-vault edit secrets.yml            # edit
ansible-vault encrypt vars/prod.yml       # encrypt existing file
ansible-vault decrypt secrets.yml
ansible-vault encrypt_string 'sekret' --name 'db_password'   # inline

ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

```yaml
# Use the encrypted variable normally
    - name: Configure DB
      template:
        src: db.conf.j2
        dest: /etc/db.conf
      vars:
        password: "{{ db_password }}"
```

#### 3.3.5 Orchestration: serial, delegation, run_once

```yaml
- name: Rolling update web tier
  hosts: webservers
  serial: 2                          # update 2 hosts at a time
  max_fail_percentage: 25
  tasks:
    - name: Remove from load balancer
      delegate_to: loadbalancer
      command: /opt/lb-drain.sh {{ inventory_hostname }}

    - name: Deploy new version
      copy: { src: app.jar, dest: /opt/app.jar }
      notify: Restart app

    - name: Run a one-time DB migration
      command: /opt/migrate.sh
      run_once: true                 # only on the first host
      delegate_to: "{{ groups['dbservers'][0] }}"
```

- `serial` — control batch size for rolling updates.
- `delegate_to` — run a task on a different host.
- `run_once` — run a task only once across the batch.

#### 3.3.6 Dynamic inventory

Instead of a static file, generate inventory from a cloud provider.

```bash
# aws_ec2.yml (inventory plugin)
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
keyed_groups:
  - key: tags.Role
    prefix: role
```

```bash
ansible-inventory -i aws_ec2.yml --graph
ansible-playbook -i aws_ec2.yml site.yml
```

#### 3.3.7 Custom modules and plugins

For logic not covered by existing modules, write a custom module (typically Python) that reads JSON args and returns JSON with a `changed` flag. Plugins (filter, lookup, callback) extend Ansible's behavior.

#### 3.3.8 Performance tuning

```ini
# ansible.cfg
[defaults]
forks = 50                      # parallelism (default 5)
gathering = smart               # cache facts
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts

[ssh_connection]
pipelining = True               # fewer SSH operations per task
 control_path = /tmp/ansible-%%h-%%p-%%r
```

Other techniques: limit fact gathering (`gather_facts: false` when not needed), use `async`/`poll` for long tasks, and `--limit` to target subsets.

#### 3.3.9 Testing and quality

```bash
ansible-playbook site.yml --syntax-check
ansible-lint site.yml                       # best-practice linter
ansible-playbook site.yml --check --diff    # dry run with diffs
# Molecule — test roles in containers/VMs
molecule init role myrole
molecule test
```

---

## 4. Hands-on Tasks

### Task 1: Ping all hosts

```bash
ansible all -i inventory.ini -m ping
```

**Expected:** Each host returns `SUCCESS => { "ping": "pong" }`.

### Task 2: Run ad-hoc commands

```bash
ansible webservers -m command -a "uptime"
ansible all -m shell -a "free -h | head -2"
```

**Expected:** Uptime and memory output per host.

### Task 3: Install a package via ad-hoc

```bash
ansible webservers -m apt -a "name=htop state=present" --become
```

**Expected:** `changed` on first run, `ok` on second (idempotent).

### Task 4: Write and run a playbook

Use `webserver.yml` from section 3.1.3:

```bash
ansible-playbook -i inventory.ini webserver.yml
ansible-playbook -i inventory.ini webserver.yml      # run again
```

**Expected:** First run shows `changed`; second shows `ok` (no changes).

### Task 5: Use variables and loops

```yaml
---
- hosts: all
  become: true
  vars:
    packages: [git, curl, vim]
  tasks:
    - name: Install tools
      package:
        name: "{{ item }}"
        state: present
      loop: "{{ packages }}"
```

**Expected:** All three packages installed across hosts.

### Task 6: Templates and handlers

Create `templates/motd.j2`:

```jinja
Welcome to {{ ansible_facts['hostname'] }} ({{ ansible_facts['os_family'] }})
Managed by Ansible.
```

```yaml
    - name: Deploy MOTD
      template: { src: motd.j2, dest: /etc/motd }
      notify: Log change
  handlers:
    - name: Log change
      debug: msg="MOTD updated"
```

**Expected:** Rendered MOTD per host; handler fires only on change.

### Task 7: Conditionals across OS families

Use the conditional Apache example from section 3.2.3.

**Expected:** Correct package installed depending on each host's OS family.

### Task 8: Create and use a role

```bash
ansible-galaxy role init roles/nginx
# add tasks/handlers/templates, then:
```

```yaml
- hosts: webservers
  become: true
  roles:
    - nginx
```

**Expected:** Role-organized deployment runs successfully.

### Task 9: Encrypt secrets with Vault

```bash
ansible-vault create secrets.yml          # add: db_password: s3cret
ansible-playbook site.yml --ask-vault-pass
```

**Expected:** Secret stays encrypted on disk; decrypted only at runtime.

### Task 10: Rolling update with serial

```yaml
- hosts: webservers
  serial: 1
  become: true
  tasks:
    - name: Upgrade nginx
      apt: { name: nginx, state: latest }
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      service: { name: nginx, state: restarted }
```

**Expected:** Hosts updated one at a time, preserving availability.

---

## 5. Projects

### Project 1 (Beginner): Automated LAMP/LEMP Server Setup

**Goal:** A playbook that configures a complete web server from scratch.

Steps:
1. Define an inventory of one or more target VMs.
2. Write a playbook to install nginx (or Apache), PHP, and a database.
3. Use the `template` module for the web server config.
4. Use a handler to restart the service on config change.
5. Deploy a sample page and verify with `curl`.
6. Confirm idempotency by running twice.

**Skills:** playbooks, modules, templates, handlers, idempotency.

### Project 2 (Intermediate): Role-Based Multi-Tier Deployment

**Goal:** Configure a web tier, app tier, and database tier using reusable roles.

Steps:
1. Create roles: `common` (users, packages, hardening), `webserver`, `appserver`, `database`.
2. Use group_vars/host_vars to parameterize per environment.
3. Store DB credentials in **Ansible Vault**.
4. Use conditionals/facts to support multiple OS families.
5. A single `site.yml` orchestrates all tiers.
6. Add tags so you can run individual tiers.

**Skills:** roles, group_vars, Vault, conditionals, orchestration, tags.

### Project 3 (Advanced): Zero-Downtime Rolling Deployment with Dynamic Inventory

**Goal:** Deploy an application update across a cloud fleet with no downtime.

Steps:
1. Use a **dynamic inventory** (e.g., `amazon.aws.aws_ec2`) to discover hosts by tag.
2. Implement a rolling update with `serial` and `max_fail_percentage`.
3. For each batch: drain the load balancer (`delegate_to`), deploy, health-check, re-add.
4. Run DB migrations exactly once (`run_once` + `delegate_to`).
5. Use `block/rescue/always` for safe error handling and rollback.
6. Encrypt all secrets with Vault; lint with `ansible-lint`; test roles with **Molecule**.
7. Integrate the playbook into CI/CD (triggered on release).

**Skills:** dynamic inventory, rolling updates, delegation, error handling, testing, CI/CD.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Prefer modules over `command`/`shell`** to preserve idempotency.
- **Make tasks idempotent**; verify by running playbooks twice (expect `ok`).
- **Organize with roles** and a clear directory layout; keep tasks small and named.
- **Use `group_vars`/`host_vars`** instead of hard-coding values.
- **Encrypt secrets with Ansible Vault**; never commit plaintext credentials.
- **Use handlers** for service restarts triggered by changes.
- **Name every task** clearly for readable output.
- **Test with `--check --diff`**, `--syntax-check`, `ansible-lint`, and Molecule.
- **Pin role/collection versions** in `requirements.yml`.
- **Use tags** for selective runs and `--limit` to scope hosts.
- **Use `serial`** for safe rolling updates.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Overusing `shell`/`command` | Non-idempotent, fragile | Use proper modules; add `creates`/`changed_when` |
| Hard-coding values in tasks | Not reusable | Use variables, group_vars/host_vars |
| Committing secrets in plaintext | Credential leak | Ansible Vault / external secret store |
| Ignoring idempotency | Unintended repeated changes | Use idempotent modules; test twice |
| One giant playbook | Hard to maintain | Refactor into roles |
| Forgetting `become` | Permission denied | Add privilege escalation |
| No `--check` before prod | Surprise changes | Dry-run with `--check --diff` |
| Gathering facts when unnecessary | Slow runs | `gather_facts: false` |
| Not pinning Galaxy versions | Breaking changes | Pin in `requirements.yml` |
| Misusing handlers (expecting immediate run) | Confusing timing | Remember handlers run at play end |

---

## 7. Interview Questions

### Beginner

1. **What is Ansible and what is it used for?**
   *An agentless automation tool for configuration management, application deployment, and orchestration, using YAML playbooks.*

2. **What does "agentless" mean?**
   *Managed nodes need no special Ansible software — Ansible connects over SSH (or WinRM) and uses Python on the node.*

3. **What is a playbook?**
   *A YAML file defining one or more plays that map hosts to ordered tasks.*

4. **What is an inventory?**
   *A list of managed hosts, optionally grouped, that Ansible targets.*

5. **What is a module?**
   *A unit of work executed on managed nodes (e.g., install a package, copy a file).*

### Intermediate

6. **What is idempotency and why does it matter?**
   *Running a playbook multiple times produces the same result without unintended changes. It makes automation safe and repeatable.*

7. **What are handlers?**
   *Tasks that run only when notified by a changed task, typically used to restart services after config changes; they run once at the end of the play.*

8. **What are facts?**
   *System properties (OS, IP, memory, etc.) gathered automatically from managed nodes, usable as variables.*

9. **How do you manage secrets in Ansible?**
   *Use Ansible Vault to encrypt sensitive data at rest, supplying the vault password at runtime.*

10. **What is the difference between `command` and `shell` modules?**
    *`command` runs a command without a shell (no pipes/redirection/env expansion); `shell` runs through a shell, enabling those features but with more risk.*

### Advanced

11. **What are roles and why use them?**
    *Roles are a standardized directory structure that packages tasks, handlers, templates, files, and variables for reuse and maintainability.*

12. **How do you perform a zero-downtime rolling deployment?**
    *Use `serial` to update in batches, drain/re-add via `delegate_to` the load balancer, run migrations with `run_once`, health-check between batches, and handle failures with `block/rescue` and `max_fail_percentage`.*

13. **What is dynamic inventory?**
    *Inventory generated at runtime from an external source (e.g., a cloud provider), so hosts are discovered automatically rather than listed statically.*

14. **Explain variable precedence in Ansible.**
    *From lowest to highest (simplified): role defaults < inventory vars < group_vars < host_vars < play vars < task vars < extra vars (`-e`). Extra vars always win.*

15. **How do you test and validate Ansible content?**
    *`--syntax-check`, `--check --diff` dry runs, `ansible-lint`, and Molecule for role testing in containers/VMs, ideally gated in CI.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Ansible is best described as:
- A) Agent-based  B) Agentless  C) Compiled  D) Kernel module

**Q2.** What language are playbooks written in?
- A) Ruby  B) HCL  C) YAML  D) JSON

**Q3.** Which property means re-running yields the same result?
- A) Atomicity  B) Idempotency  C) Concurrency  D) Immutability

**Q4.** What connects the control node to Linux managed nodes?
- A) HTTP  B) WinRM  C) SSH  D) gRPC

**Q5.** Which runs only when notified by a changed task?
- A) Task  B) Handler  C) Role  D) Fact

**Q6.** What gathers system information from nodes?
- A) Vault  B) Facts  C) Tags  D) Inventory

**Q7.** Which tool encrypts secrets in Ansible?
- A) Ansible Vault  B) Galaxy  C) Molecule  D) Tower

**Q8.** Which keyword limits how many hosts update at once?
- A) `forks`  B) `serial`  C) `loop`  D) `delegate_to`

**Q9.** A standardized, reusable directory of automation content is a:
- A) Module  B) Play  C) Role  D) Handler

**Q10.** Which renders Jinja2 templates onto managed nodes?
- A) `copy`  B) `template`  C) `file`  D) `lineinfile`

### Short Answer

**S1.** Write an ad-hoc command to ping all hosts.

**S2.** What flag performs a dry run of a playbook?

**S3.** How do you run a playbook prompting for the Vault password?

**S4.** Which directive runs a task on a different host than the current one?

**S5.** Name the typical task statuses shown in Ansible output.

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-C, Q5-B, Q6-B, Q7-A, Q8-B, Q9-C, Q10-B

**Short Answer:**
- **S1.** `ansible all -m ping`
- **S2.** `--check` (often with `--diff`).
- **S3.** `ansible-playbook site.yml --ask-vault-pass`
- **S4.** `delegate_to`
- **S5.** ok, changed, failed, skipped, unreachable.

---

## 9. Further Resources

### Official Documentation
- Ansible Docs — https://docs.ansible.com
- Ansible Getting Started — https://docs.ansible.com/ansible/latest/getting_started/
- Module index — https://docs.ansible.com/ansible/latest/collections/
- Best practices — https://docs.ansible.com/ansible/latest/tips_tricks/

### Learning
- Ansible for DevOps — Jeff Geerling (book + free chapters)
- Red Hat Ansible learning paths
- Jeff Geerling's example roles (geerlingguy.*) on Galaxy

### Tools
- Ansible Galaxy — https://galaxy.ansible.com
- ansible-lint — https://ansible.readthedocs.io/projects/lint/
- Molecule (role testing) — https://ansible.readthedocs.io/projects/molecule/
- AWX / Ansible Automation Platform (web UI, RBAC, scheduling)

### Books
- *Ansible for DevOps* — Jeff Geerling
- *Ansible: Up & Running* — Lorin Hochstein & René Moser
- *Mastering Ansible* — James Freeman

### Certifications
- Red Hat Certified Specialist in Ansible Automation (EX374 / EX294)

---

*End of Ansible — Zero to Hero.*
