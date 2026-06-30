# HashiCorp Vault — Zero to Hero

> A complete, beginner-to-advanced guide to HashiCorp Vault for secrets management, encryption, and identity-based access.

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

### 1.1 What is Vault?

**HashiCorp Vault** is a tool for **secrets management, data encryption, and identity-based access**. It provides a centralized, secure place to store, access, and tightly control access to tokens, passwords, certificates, API keys, and other secrets. Beyond static storage, Vault can **generate dynamic, short-lived credentials on demand**, encrypt data as a service, and broker identity across systems.

### 1.2 The problem Vault solves

In modern systems, secrets sprawl everywhere — hardcoded in code, committed to Git, pasted in config files, shared in chat, baked into images, scattered across environments. This is dangerous:
- **Secret sprawl** — no central inventory or control.
- **Long-lived static credentials** — if leaked, valid until manually rotated.
- **No audit trail** — who accessed what, when?
- **Hard rotation** — changing a leaked DB password breaks many services.

Vault centralizes secrets behind strong access control, **encrypts everything**, provides a full **audit log**, and replaces static secrets with **dynamic, automatically-expiring** ones.

```
   Before Vault                          With Vault
   ────────────                          ──────────
   secrets in code/Git/env       →       all secrets in Vault (encrypted)
   shared static DB password     →       dynamic per-app DB creds (TTL)
   manual rotation               →       automatic lease/rotation/revocation
   no audit                      →       full audit log of every access
```

### 1.3 Core capabilities

| Capability | Description |
|------------|-------------|
| **Secret storage** | Securely store static key/value secrets (encrypted at rest) |
| **Dynamic secrets** | Generate credentials on demand (DB, cloud, SSH) with TTL |
| **Encryption as a service** | Encrypt/decrypt data without storing it (Transit engine) |
| **Identity & access** | Authenticate via many methods, authorize via policies |
| **Leasing & revocation** | Every secret has a lease; revoke instantly |
| **Audit logging** | Tamper-evident log of all operations |
| **PKI / certificates** | Issue and manage TLS certificates |

### 1.4 Architecture

```
   Clients (apps, humans, CI)
        │ authenticate (auth method) → get a token
        ▼
   +--------------------------------------------------+
   |                   Vault Server                   |
   |                                                  |
   |  Auth Methods   Policies (ACL)   Secrets Engines |
   |  (token, k8s,   (what a token    (kv, db, transit|
   |   approle, ...) can access)       pki, aws, ...) |
   |                                                  |
   |  Barrier (encryption) ── Storage Backend         |
   |                          (Integrated Raft,       |
   |                           Consul, etc.)          |
   +--------------------------------------------------+
        │ all data encrypted by the barrier before storage
        ▼
   Audit Devices (log every request/response)
```

- **Storage backend** holds *encrypted* data (Vault never trusts storage with plaintext). Integrated Storage (Raft) is the standard.
- **Barrier** is the cryptographic boundary; data is encrypted before it leaves Vault.
- **Auth methods** verify identity and issue tokens.
- **Policies** define what a token can do.
- **Secrets engines** store or generate secrets.
- **Audit devices** record everything.

### 1.5 Seal/unseal and the encryption key hierarchy

Vault starts **sealed** — it knows where its encrypted data is but cannot read it.

```
   Unseal keys (Shamir shares)  ──►  reconstruct  ──►  Master/Root key
                                                            │ decrypts
                                                            ▼
                                                      Encryption key
                                                            │ decrypts
                                                            ▼
                                                      Stored data
```

- **Seal:** Vault's data is encrypted and inaccessible (state on startup or when sealed).
- **Unseal:** Provide a quorum of **unseal keys** (Shamir's Secret Sharing splits the master key into shares; e.g., 3 of 5 required) to reconstruct the master key and decrypt the encryption key.
- **Auto-unseal:** In production, use a cloud KMS/HSM to auto-unseal instead of manual key entry.

### 1.6 When to use Vault

- Centralized secrets management across many apps/teams/environments.
- Dynamic, short-lived credentials for databases and cloud providers.
- Encryption-as-a-service so apps never handle encryption keys.
- PKI/certificate automation; SSH credential brokering.
- Compliance needs (audit, least privilege, rotation).

When **not** ideal: a single tiny app (a cloud secrets manager may be simpler); teams unwilling to run/operate a stateful, HA-critical service.

---

## 2. Installation & Setup

### 2.1 Install Vault

```bash
# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/vault

# Linux (HashiCorp apt repo)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vault

vault version
```

### 2.2 Dev server (learning only)

```bash
vault server -dev
```

**Expected:** Vault starts unsealed, in-memory, with a printed **Root Token** and `VAULT_ADDR`. Never use dev mode in production — it's insecure and non-persistent.

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='<root-token-from-output>'
vault status
```

### 2.3 Production init and unseal (concept)

```bash
vault operator init        # outputs 5 unseal keys + initial root token
vault operator unseal      # repeat with 3 different keys to reach the threshold
vault status               # Sealed: false once unsealed
```

> Distribute unseal keys to different trusted people; never keep them all together. Use auto-unseal (KMS) in production.

### 2.4 First secret

```bash
vault secrets enable -path=secret kv-v2
vault kv put secret/myapp/config username="admin" password="s3cr3t"
vault kv get secret/myapp/config
```

**Expected:** The secret is stored and retrievable.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Paths and the API

Everything in Vault is a **path** accessed via a consistent API (CLI, HTTP, or SDK). `vault kv put secret/foo` writes to the `secret/` mount; `sys/` is system config; auth methods mount under `auth/`.

#### 3.1.2 KV secrets engine

```bash
# KV version 2 (versioned secrets)
vault kv put secret/app db_pass="abc123"
vault kv get secret/app
vault kv get -field=db_pass secret/app      # just the value
vault kv put secret/app db_pass="new"        # creates version 2
vault kv get -version=1 secret/app           # read old version
vault kv delete secret/app                   # soft delete
vault kv undelete -versions=2 secret/app
```

KV v2 keeps **version history** and supports soft delete; KV v1 is unversioned.

#### 3.1.3 Tokens

Tokens are the core auth primitive — almost everything in Vault is done with a token.

```bash
vault token create -policy=myapp -ttl=1h
vault token lookup           # info about current token
vault token revoke <token>   # revoke
```

Tokens have a **TTL**, can be **renewable**, and are tied to **policies**.

#### 3.1.4 Policies (authorization)

```hcl
# myapp-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
path "secret/data/shared/*" {
  capabilities = ["read"]
}
```

```bash
vault policy write myapp myapp-policy.hcl
```

Policies are **deny by default** — you grant explicit capabilities (`create`, `read`, `update`, `delete`, `list`, `sudo`, `deny`) on paths.

> For KV v2, data lives under `secret/data/...` in policies (note the `data/` segment).

#### 3.1.5 Auth methods (authentication)

```bash
vault auth enable userpass
vault write auth/userpass/users/alice password="pw" policies="myapp"
vault login -method=userpass username=alice
```

Auth methods verify identity and return a token bound to policies. Examples: `userpass`, `token`, `approle`, `kubernetes`, `ldap`, `github`, cloud IAM (AWS/Azure/GCP), OIDC/JWT.

### 3.2 INTERMEDIATE

#### 3.2.1 Dynamic secrets (databases)

Instead of a shared DB password, Vault generates **unique, temporary credentials per request**:

```bash
vault secrets enable database

vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/app" \
  allowed_roles="readonly" \
  username="vault_admin" password="admin_pw"

vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" max_ttl="24h"

vault read database/creds/readonly    # returns a fresh username/password with TTL
```

When the lease expires (or is revoked), Vault **deletes the DB user** automatically. No long-lived shared credentials.

#### 3.2.2 Leases, renewal, and revocation

```bash
vault lease renew <lease_id>
vault lease revoke <lease_id>
vault lease revoke -prefix database/creds/readonly   # revoke all
```

Every dynamic secret has a **lease** (TTL). Vault tracks them and can **revoke instantly** — critical for incident response (revoke all credentials in seconds).

#### 3.2.3 Transit engine (encryption as a service)

```bash
vault secrets enable transit
vault write -f transit/keys/mykey

# Encrypt (app sends plaintext, gets ciphertext; key never leaves Vault)
vault write transit/encrypt/mykey plaintext=$(echo "secret data" | base64)

# Decrypt
vault write transit/decrypt/mykey ciphertext="vault:v1:..."
```

Apps offload crypto to Vault — they never see or store encryption keys. Supports **key rotation** and **rewrapping** without re-encrypting in the app.

#### 3.2.4 AppRole (machine authentication)

```bash
vault auth enable approle
vault write auth/approle/role/myapp \
  token_policies="myapp" token_ttl=1h token_max_ttl=4h

vault read auth/approle/role/myapp/role-id        # RoleID (like a username)
vault write -f auth/approle/role/myapp/secret-id  # SecretID (like a password)

vault write auth/approle/login role_id=<id> secret_id=<sid>   # returns a token
```

**AppRole** is the standard way for applications/CI to authenticate without humans.

#### 3.2.5 Kubernetes auth

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:443"

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=default \
  policies=myapp ttl=1h
```

Pods authenticate using their **ServiceAccount token** — no secrets to distribute.

#### 3.2.6 Response wrapping

```bash
vault kv get -wrap-ttl=120 secret/app   # returns a single-use wrapping token
vault unwrap <wrapping_token>            # retrieve the actual data once
```

Wrapping lets you pass a secret to a consumer via a **single-use token** — if intercepted/unwrapped by an attacker, the legitimate consumer's unwrap fails (tamper detection).

### 3.3 ADVANCED

#### 3.3.1 Identity: entities and groups

Vault's **Identity** system unifies multiple auth identities (e.g., the same user via LDAP and GitHub) into a single **entity**, and **groups** map to policies — enabling consistent authorization and identity-based access across auth methods. Entities also power **OIDC provider** and identity tokens.

#### 3.3.2 PKI secrets engine

```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=8760h pki
vault write pki/root/generate/internal common_name="example.com" ttl=8760h
vault write pki/roles/web allowed_domains="example.com" allow_subdomains=true max_ttl="72h"
vault write pki/issue/web common_name="app.example.com" ttl="24h"
```

Vault becomes a **certificate authority**, issuing short-lived TLS certs on demand — automating cert rotation and eliminating manual cert management.

#### 3.3.3 High availability and Integrated Storage (Raft)

```hcl
# config.hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "node1"
}
listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = false
}
seal "awskms" {                 # auto-unseal
  kms_key_id = "..."
}
api_addr = "https://node1:8200"
cluster_addr = "https://node1:8201"
```

- **Integrated Storage (Raft)** provides built-in HA and replication (no external Consul needed).
- A cluster has one **active** node and several **standby** nodes (Raft consensus).
- **Auto-unseal** via cloud KMS removes manual unseal toil.

#### 3.3.4 Enterprise features

- **Performance & DR replication** across regions/clusters.
- **Namespaces** for multi-tenancy.
- **HSM integration**, **Sentinel** policy-as-code, control groups.
- **Transform** secrets engine (FPE/tokenization for PCI/PII).

#### 3.3.5 Audit devices

```bash
vault audit enable file file_path=/var/log/vault/audit.log
```

Every request/response is logged (with sensitive values **HMAC'd**, not plaintext). Audit logs are **tamper-evident** and essential for compliance — if all audit devices fail, Vault **stops serving** (fails secure).

#### 3.3.6 Secret rotation and root credential rotation

```bash
vault write -f database/rotate-root/mydb   # Vault rotates the DB admin password
```

Vault can rotate its own privileged credentials so even operators don't know them, and rotate static secrets on a schedule.

#### 3.3.7 Vault Agent and secret injection

- **Vault Agent** runs alongside apps: handles auth (auto-auth), caches tokens, and renders secrets to files via templates.
- **Vault Agent Injector** (Kubernetes) injects a sidecar that fetches secrets into pods automatically via annotations — apps read secrets from a file, unaware of Vault.
- Alternatives: **External Secrets Operator**, **Secrets Store CSI driver**.

#### 3.3.8 Disaster recovery and operations

- Take **snapshots** of Raft storage: `vault operator raft snapshot save backup.snap`.
- Protect and rotate **unseal/recovery keys**; use **rekey** to change the Shamir configuration.
- Monitor seal status, leader, and audit health; alert on seal events.

---

## 4. Hands-on Tasks

### Task 1: Start dev server and store a secret

```bash
vault server -dev    # in one terminal; export VAULT_ADDR/TOKEN in another
vault kv put secret/hello name="world"
vault kv get secret/hello
```

**Expected:** The secret is stored and retrieved.

### Task 2: Read a single field

```bash
vault kv get -field=name secret/hello
```

**Expected:** Prints `world` only.

### Task 3: Version a KV secret

Put a new value and read version 1 back.

**Expected:** KV v2 retains the old version; `-version=1` returns the original.

### Task 4: Write and apply a policy

Create `myapp-policy.hcl`, write it, and create a token with that policy.

**Expected:** The token can read only the permitted paths.

### Task 5: Enable userpass and log in

Enable `userpass`, create a user with the policy, and `vault login`.

**Expected:** Login returns a token bound to the policy.

### Task 6: Generate dynamic database credentials

Configure the database engine and a role; `vault read database/creds/<role>`.

**Expected:** Fresh, unique DB credentials with a TTL are returned.

### Task 7: Revoke a lease

```bash
vault lease revoke -prefix database/creds/readonly
```

**Expected:** The generated DB user is removed immediately.

### Task 8: Use the Transit engine

Enable transit, create a key, encrypt and then decrypt a value.

**Expected:** Ciphertext (`vault:v1:...`) returned; decrypt yields the original base64.

### Task 9: Configure AppRole

Enable approle, create a role, fetch RoleID/SecretID, and log in.

**Expected:** Login with RoleID+SecretID returns a token.

### Task 10: Enable an audit device

```bash
vault audit enable file file_path=/tmp/vault-audit.log
vault kv get secret/hello
tail /tmp/vault-audit.log
```

**Expected:** The access is recorded (sensitive fields HMAC'd).

---

## 5. Projects

### Project 1 (Beginner): Centralize App Secrets

**Goal:** Move an app's secrets out of config files into Vault.

Steps:
1. Run Vault (dev or single-node) and enable KV v2.
2. Store DB/API credentials under `secret/myapp/*`.
3. Write a **policy** granting read to just that path.
4. Authenticate the app via **AppRole**; fetch secrets at startup.
5. Confirm no secrets remain in code/Git.

**Skills:** KV engine, policies, AppRole, secret retrieval.

### Project 2 (Intermediate): Dynamic Database Credentials + Encryption

**Goal:** Eliminate static DB passwords and offload encryption.

Steps:
1. Enable the **database** secrets engine; configure a role with TTLs.
2. Have the app request **dynamic credentials** on startup and renew leases.
3. Use the **Transit** engine to encrypt sensitive fields before storing them.
4. Rotate the database **root** credential via Vault.
5. Enable an **audit device** and review access logs.

**Skills:** dynamic secrets, leases/renewal, Transit, root rotation, auditing.

### Project 3 (Advanced): Production HA Vault on Kubernetes

**Goal:** Operate a secure, highly available Vault serving the cluster.

Steps:
1. Deploy Vault in **HA with Integrated Storage (Raft)** (e.g., Helm chart), 3+ nodes.
2. Configure **auto-unseal** via cloud KMS.
3. Enable **Kubernetes auth**; map ServiceAccounts to policies.
4. Use the **Vault Agent Injector** to inject secrets into pods via annotations.
5. Stand up the **PKI** engine to issue short-lived TLS certs.
6. Configure **audit logging**, **snapshots**, monitoring, and alerting on seal/leader events.
7. Apply least-privilege policies and identity groups.

**Skills:** Raft HA, auto-unseal, Kubernetes auth, Agent Injector, PKI, DR/operations.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Never use dev mode in production** — it's in-memory and unsealed.
- **Use auto-unseal (KMS/HSM)** and protect/distribute unseal & recovery keys.
- **Apply least-privilege policies** (deny by default; grant minimal capabilities).
- **Prefer dynamic, short-lived secrets** over static ones.
- **Use machine auth (AppRole/Kubernetes)** for apps — not root tokens.
- **Never use the root token for day-to-day** — revoke it after setup.
- **Enable audit devices** and ship logs to secure, immutable storage.
- **Set sensible TTLs** and renew/revoke leases; revoke on incidents.
- **Use Transit** so apps never handle encryption keys.
- **Run HA with Raft**, take regular **snapshots**, and test restores.
- **Rotate root/static credentials** regularly (Vault can do it).

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Dev mode in production | Total data loss/exposure | Proper init/unseal, persistent storage |
| Using root token everywhere | Massive blast radius | Scoped tokens + least-privilege policies |
| All unseal keys in one place | Single point of compromise | Distribute shares; use auto-unseal |
| Forgetting `data/` in KV v2 policies | Access denied/confusion | Use `secret/data/...` in policies |
| Long/无 TTLs on dynamic secrets | Stale credentials linger | Set TTLs; renew/revoke |
| No audit device | No forensic trail | Enable audit logging |
| Storing keys in the app (no Transit) | Key leakage risk | Use Transit encryption-as-a-service |
| Single Vault node | Outage = no secrets | HA Raft cluster |
| No snapshots | Unrecoverable disaster | Schedule Raft snapshots, test restore |
| Static long-lived DB password | Hard rotation, leak risk | Dynamic database secrets |

---

## 7. Interview Questions

### Beginner

1. **What is HashiCorp Vault?**
   *A tool for secrets management, encryption as a service, and identity-based access, storing and controlling access to secrets securely.*

2. **What does "sealed" mean?**
   *Vault's data is encrypted and inaccessible; it must be unsealed with a quorum of unseal keys to read its encryption key.*

3. **What is a secrets engine?**
   *A plugin mounted at a path that stores or generates secrets (e.g., KV, database, transit, PKI).*

4. **What is a Vault policy?**
   *An HCL document granting capabilities (read/write/etc.) on paths; Vault is deny-by-default.*

5. **What is a token in Vault?**
   *The core auth credential, tied to policies and a TTL, used to perform operations.*

### Intermediate

6. **What are dynamic secrets and why use them?**
   *Credentials generated on demand with a TTL (e.g., per-app DB users) and auto-revoked on expiry — eliminating shared, long-lived secrets and shrinking the blast radius of a leak.*

7. **What is the Transit engine?**
   *Encryption as a service: Vault encrypts/decrypts data so apps never store or handle encryption keys, and supports key rotation/rewrapping.*

8. **What is AppRole and when is it used?**
   *A machine-oriented auth method using a RoleID and SecretID to obtain a token — used for applications and CI rather than humans.*

9. **How do leases work?**
   *Every dynamic secret/token has a lease (TTL); Vault tracks them and can renew or revoke (instantly, even in bulk) — key for incident response.*

10. **How do Kubernetes pods authenticate to Vault?**
    *Via the Kubernetes auth method using the pod's ServiceAccount token, mapped to a role and policies — no secrets to distribute.*

### Advanced

11. **Explain Shamir's Secret Sharing in Vault's unseal process.**
    *The master key is split into N shares with a threshold K; K of N shares must be provided to reconstruct it and unseal. This prevents any single person from unsealing alone.*

12. **What is auto-unseal and why use it?**
    *Vault delegates decryption of the master key to a cloud KMS/HSM, removing manual unseal toil and the need to handle unseal keys at startup.*

13. **How does Integrated Storage (Raft) provide HA?**
    *Raft consensus replicates encrypted data across nodes with one active and several standby nodes; on failure a standby is promoted — no external storage like Consul required.*

14. **How does response wrapping improve security?**
    *A secret is returned as a single-use wrapping token; the intended consumer unwraps it once. Interception is detectable because a second unwrap fails, revealing tampering.*

15. **How would you respond to a suspected credential leak using Vault?**
    *Revoke the affected leases/tokens immediately (including by prefix), rotate root/static credentials, review audit logs to scope the incident, and rely on short-lived dynamic secrets to limit exposure.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Vault is primarily for:
- A) Container builds  B) Secrets management  C) Load balancing  D) Logging

**Q2.** When Vault is "sealed" it:
- A) Is offline  B) Cannot decrypt its data  C) Is deleting data  D) Is read-only

**Q3.** Which splits the master key into shares?
- A) PKI  B) Shamir's Secret Sharing  C) Transit  D) Raft

**Q4.** Which provides encryption as a service?
- A) KV  B) Transit  C) Database  D) PKI

**Q5.** Vault policies are by default:
- A) Allow all  B) Deny all  C) Read only  D) Admin

**Q6.** Which auth method suits applications/CI?
- A) userpass  B) AppRole  C) github  D) okta

**Q7.** Dynamic database credentials are:
- A) Permanent  B) Shared  C) Short-lived per request  D) Stored in Git

**Q8.** Which provides built-in HA storage?
- A) Consul only  B) Integrated Storage (Raft)  C) MySQL  D) S3

**Q9.** Auto-unseal typically uses:
- A) A password  B) Cloud KMS/HSM  C) Git  D) DNS

**Q10.** In KV v2 policies, data paths include:
- A) `secret/`  B) `secret/data/`  C) `kv/`  D) `sys/`

### Short Answer

**S1.** What command initializes a new Vault and outputs unseal keys?

**S2.** What is the term for a dynamic secret's time-to-live tracking object?

**S3.** Which engine turns Vault into a certificate authority?

**S4.** Name the two credentials used in AppRole login.

**S5.** What happens to serving if all audit devices fail?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-B, Q6-B, Q7-C, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `vault operator init`.
- **S2.** A **lease**.
- **S3.** The **PKI** secrets engine.
- **S4.** RoleID and SecretID.
- **S5.** Vault **stops serving requests** (fails secure).

---

## 9. Further Resources

### Official Documentation
- Vault docs — https://developer.hashicorp.com/vault/docs
- Tutorials — https://developer.hashicorp.com/vault/tutorials
- API docs — https://developer.hashicorp.com/vault/api-docs
- Vault on Kubernetes — https://developer.hashicorp.com/vault/docs/platform/k8s

### Learning
- HashiCorp Learn (Vault) — https://developer.hashicorp.com/vault/tutorials
- Vault Associate certification guide
- Vault Agent & Injector tutorials

### Tools & Ecosystem
- Vault Helm chart, Vault Agent Injector
- External Secrets Operator, Secrets Store CSI driver
- consul-template, vault-cli

### Books & Guides
- *Running HashiCorp Vault in Production* (HashiCorp guides)
- *HashiCorp Vault* learning paths

### Certifications
- HashiCorp Certified: Vault Associate
- HashiCorp Certified: Vault Operations Professional

---

*End of HashiCorp Vault — Zero to Hero.*
