# JFrog Interview Preparation — Hands-On Tasks

> A curated set of real-world JFrog challenges organized by difficulty.  
> Covers Artifactory, Xray, JFrog CLI, and CI/CD integration scenarios.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Architecture & Design Tasks](#architecture--design-tasks)

---

## Beginner Tasks

---

### Task B1 — Set Up a Local Repository and Push Artifacts

**Problem Statement**  
Your team needs a central place to store internally built Docker images and Python packages. Create a local Docker repository and a local PyPI repository in Artifactory. Push a Docker image and a Python `.whl` package to each. Verify both artifacts are browsable in the Artifactory UI and downloadable via CLI.

**Expected Outcome**
- A local Docker repository named `docker-local` exists.
- A local PyPI repository named `pypi-local` exists.
- `docker push myartifactory.example.com/docker-local/myapp:v1.0` succeeds.
- `pip install --index-url https://myartifactory.example.com/artifactory/api/pypi/pypi-local/simple/ mypackage` succeeds.
- Both artifacts appear in the Artifact Repository Browser with size, SHA256, and upload timestamp.

**Hints**
- Create repositories via: `Admin > Repositories > Add Repositories > Local Repository`.
- For Docker: configure the Artifactory server as a Docker registry — set the subdomain or port routing.
- `docker login myartifactory.example.com` — use your Artifactory credentials or an API key.
- For PyPI upload use `twine`: `twine upload --repository-url https://.../artifactory/api/pypi/pypi-local/ dist/*.whl`.
- Use JFrog CLI: `jf rt u "dist/*.whl" pypi-local/` after configuring `jf config add`.

**Skills Tested**
- Repository types (local, remote, virtual)
- Docker registry configuration in Artifactory
- PyPI repository setup
- JFrog CLI basic upload/download
- Artifact browsing and metadata inspection

---

### Task B2 — Configure a Remote Repository as a Proxy Cache

**Problem Statement**  
Developers are downloading packages directly from `https://registry.npmjs.org` and `https://repo1.maven.org`. This causes slow builds, blocked downloads when the internet is unavailable, and no control over what external packages enter the organization. Configure Artifactory remote repositories to proxy and cache both registries.

**Expected Outcome**
- A remote npm repository named `npm-remote` proxying `https://registry.npmjs.org`.
- A remote Maven repository named `maven-remote` proxying `https://repo1.maven.org/maven2`.
- First `npm install express` pulls from npmjs.org and caches in Artifactory.
- Second `npm install express` (with internet blocked) still succeeds from the Artifactory cache.
- Cached artifacts are visible in the Repository Browser under `npm-remote-cache`.

**Hints**
- Create remote repo: `Admin > Repositories > Remote > Add Remote > npm`.
- Set `URL` to the upstream registry URL.
- Configure `.npmrc`: `registry=https://myartifactory.example.com/artifactory/api/npm/npm-remote/`.
- For Maven: update `settings.xml` mirror to point to the Artifactory remote repo URL.
- Remote repos cache downloaded artifacts — check `npm-remote-cache` in the UI after first download.

**Skills Tested**
- Remote repository as proxy/cache concept
- npm and Maven client configuration to use Artifactory
- Cache-miss vs cache-hit flow understanding
- Package manager `.npmrc`/`settings.xml` configuration
- Internet dependency reduction strategy

---

### Task B3 — Create a Virtual Repository for Unified Access

**Problem Statement**  
Your developers need to pull packages from both a local repository (internal packages) and a remote repository (external packages) using a single URL endpoint — without knowing which repository the package comes from. Create virtual repositories for npm and Maven that aggregate their respective local and remote repositories.

**Expected Outcome**
- A virtual npm repository `npm-virtual` includes both `npm-local` and `npm-remote`.
- A virtual Maven repository `maven-virtual` includes `maven-local` and `maven-remote`.
- `npm install` from `npm-virtual` resolves internal packages from `npm-local` first, then falls back to `npm-remote`.
- The `Default Deployment Repository` for `npm-virtual` is set to `npm-local` — publishes go to local, not remote.
- Developers use only ONE URL — they don't need to know about the underlying repositories.

**Hints**
- Create virtual: `Admin > Repositories > Virtual > Add Repository`.
- Add both the local and remote repos as `Included Repositories`.
- Repository resolution order matters — move local repos to the top of the list.
- `Default Deployment Repository` determines where `jf rt u` or `npm publish` lands when using the virtual URL.
- Test with `jf rt s "npm-virtual/*express*"` to search for a package across the virtual.

**Skills Tested**
- Virtual repository aggregation concept
- Resolution order and priority
- Default deployment repository setting
- Single-URL developer experience design
- `jf rt search` command

---

### Task B4 — Manage Users, Groups, and Permissions

**Problem Statement**  
Your organization has three teams: `dev-team` (read-only on all repos), `release-team` (read + deploy on release repos), and `admin-team` (full control). Create users, groups, and permission targets in Artifactory to enforce this access model. Verify that a `dev-team` member cannot upload artifacts.

**Expected Outcome**
- Three groups created: `developers`, `releasers`, `admins`.
- Permission target `dev-permissions` grants `Read` and `Download` on all repos to `developers`.
- Permission target `release-permissions` grants `Read`, `Download`, `Deploy`, and `Delete` on release repos to `releasers`.
- A test user in `developers` group gets `403 Forbidden` when attempting `jf rt u file.jar libs-release-local/`.
- The same user successfully downloads with `jf rt dl libs-release-local/myapp-1.0.jar`.

**Hints**
- Create groups: `Admin > Identity and Access > Groups`.
- Permission targets: `Admin > Identity and Access > Permissions`.
- In permission targets, select repositories and assign groups with specific actions (Read, Write, Deploy, Delete, Manage).
- Test permissions: create a user, assign to `developers` group, generate an API key, test with `jf rt u/dl`.
- Use `Anonymous` access setting carefully — disable it for security if all users must authenticate.

**Skills Tested**
- Artifactory RBAC: users, groups, permission targets
- Principle of least privilege in Artifactory
- API key generation and CLI authentication
- Permission verification testing
- Anonymous access controls

---

## Intermediate Tasks

---

### Task I1 — Integrate Artifactory into a CI/CD Pipeline

**Problem Statement**  
A GitLab CI pipeline currently downloads dependencies from the public internet on every build and has no artifact storage for its outputs. Integrate Artifactory so that: all dependency downloads go through Artifactory virtual repos, built artifacts are uploaded with build info, and the pipeline can be re-run reproducibly using the same artifact versions.

**Expected Outcome**
- Pipeline uses Artifactory virtual repos for all downloads (npm, Maven, Docker).
- `jf rt bp` (build publish) uploads a build record to Artifactory with modules, dependencies, and environment info.
- `jf rt bs` (build scan) runs Xray on the build.
- Build info is visible in Artifactory UI under `Builds`.
- Re-running the pipeline with the same build number fails unless `--build-name` is unique or `--overwrite` is set.

**Hints**
- Configure JFrog CLI in CI: `jf config add --url $ARTIFACTORY_URL --user $ART_USER --password $ART_PASSWORD --interactive=false`.
- Use `jf mvn`, `jf npm`, `jf docker` wrappers instead of native tools — they auto-capture build info.
- Build name and number: `--build-name=$CI_PROJECT_NAME --build-number=$CI_PIPELINE_ID`.
- Publish build info: `jf rt bp $BUILD_NAME $BUILD_NUMBER`.
- Collect environment variables (non-secret): `jf rt bce $BUILD_NAME $BUILD_NUMBER`.

**Skills Tested**
- JFrog CLI integration in CI/CD
- Build info collection and publishing
- `jf mvn`/`jf npm`/`jf docker` build wrappers
- Build reproducibility via Artifactory
- CI/CD environment variable management

---

### Task I2 — Configure Xray Policies and Watches for Security Scanning

**Problem Statement**  
Your company has no visibility into CVEs in artifacts stored in Artifactory. Set up JFrog Xray with: a security policy that blocks artifacts with `Critical` CVEs, a license policy that blocks `GPL-3.0` licenses, a watch on all production repositories, and an alert that notifies a Slack channel when a policy is violated. Test by uploading a known-vulnerable artifact.

**Expected Outcome**
- Xray security policy `block-critical-cves` fails builds/downloads with `Critical` severity CVEs.
- Xray license policy `block-gpl3` blocks artifacts with `GPL-3.0` license.
- Watch `prod-watch` covers all repos with `-prod-` in the name.
- Uploading `log4j-2.14.1.jar` (Log4Shell-vulnerable) triggers a policy violation visible in Xray Violations.
- Slack notification arrives within 5 minutes of the violation.
- `jf rt dl log4j-2.14.1.jar` returns `403` due to the active block rule.

**Hints**
- Create policies: `Xray > Policies > New Policy > Security Policy`.
- Set `Minimum Severity: Critical` and `Block Download: true`.
- Create watch: `Xray > Watches > New Watch` — add repositories and attach policies.
- Webhook for Slack: `Admin > Xray > Webhooks` — use a Slack incoming webhook URL.
- Index existing repos: `Xray > Indexed Resources > Add` — Xray only scans indexed repositories.
- Force re-scan: `jf xr scan --watches prod-watch` or trigger from the Xray UI.

**Skills Tested**
- Xray policy types (security vs license)
- Watch configuration and scope
- Policy enforcement (block download, fail build)
- Webhook/notification setup
- CVE vulnerability database and package indexing

---

### Task I3 — Implement Artifact Promotion Between Environments

**Problem Statement**  
Your release pipeline produces artifacts in `libs-snapshot-local`. After passing all tests, the artifact should be promoted (not re-built) to `libs-release-local` with a status change from `snapshot` to `release`. The promotion must update the artifact's properties, copy (not move) the artifact, and the build info must reflect the promotion.

**Expected Outcome**
- Artifact exists in `libs-snapshot-local` with property `status=snapshot`.
- After promotion, artifact exists in `libs-release-local` with property `status=release`.
- Original artifact still exists in `libs-snapshot-local` (copy, not move).
- Build info in Artifactory shows the promotion event with timestamp and promoted-by user.
- `jf rt s "libs-release-local/*.jar" --props "status=release"` finds the artifact.

**Hints**
- Promote via CLI: `jf rt bpr $BUILD_NAME $BUILD_NUMBER libs-release-local --status release --copy=true`.
- Set properties on artifacts: `jf rt sp "libs-release-local/myapp-1.0.jar" "status=release;env=production"`.
- Search by property: `jf rt s --props "status=release" "libs-release-local/"`.
- Promotion via REST API: `POST /artifactory/api/build/promote/{buildName}/{buildNumber}`.
- Use `--fail-fast=false` to promote even if some artifacts fail — then investigate individually.

**Skills Tested**
- Artifact promotion workflow (snapshot → release)
- `jf rt bpr` (build promote) command
- Artifact properties as metadata
- Copy vs move during promotion
- Build info promotion tracking

---

### Task I4 — Set Up Replication Between Two Artifactory Instances

**Problem Statement**  
Your company has a primary Artifactory in `us-east-1` and needs a read-only mirror in `eu-west-1` for European developers (to reduce latency). Configure push replication from the primary to the secondary for the `libs-release-local` and `docker-prod-local` repositories. Verify that artifacts pushed to the primary appear in the secondary within 5 minutes.

**Expected Outcome**
- Replication configured from primary `libs-release-local` → secondary `libs-release-local`.
- Replication configured from primary `docker-prod-local` → secondary `docker-prod-local`.
- Pushing an artifact to the primary triggers replication automatically (event-based) or on schedule.
- Secondary shows the artifact within 5 minutes.
- Replication status is visible in `Admin > Repositories > <repo> > Replication`.
- The secondary repository is set to `Read Only` — no direct uploads allowed.

**Hints**
- Push replication: configured on the source (primary) repo under `Edit Repository > Replication`.
- Set `Target URL` to the secondary Artifactory URL + repository key.
- Replication modes: `Event-driven` (immediate on push) vs `Cron-based` (scheduled).
- The replication user on the secondary must have `Deploy` permission on the target repo.
- Use `jf rt rplc` command for replication configuration via CLI.
- Test: `jf rt u testfile.jar libs-release-local/` on primary, then check secondary with `jf rt s "libs-release-local/testfile.jar"`.

**Skills Tested**
- Artifactory replication (push vs pull)
- Multi-site HA/DR architecture
- Event-based vs scheduled replication
- Read-only repository configuration
- Replication monitoring and troubleshooting

---

### Task I5 — Manage Artifact Lifecycle with Retention Policies

**Problem Statement**  
Your Artifactory storage has grown from 500 GB to 4 TB in 6 months. Old snapshot builds accumulate indefinitely. Implement: an Artifactory retention policy that deletes snapshot artifacts older than 30 days, a cleanup AQL query that identifies artifacts not downloaded in the last 90 days, and a script that dry-runs the deletion and requires confirmation before proceeding.

**Expected Outcome**
- Retention policy deletes `libs-snapshot-local` artifacts older than 30 days automatically.
- An AQL query returns all artifacts in `libs-release-local` not downloaded in 90+ days.
- Dry-run script outputs a list of artifacts to be deleted with their sizes.
- Interactive confirmation: `Delete 47 artifacts (12.3 GB)? [y/N]:` before actual deletion.
- After cleanup, `df` on the Artifactory storage shows measurable reduction.

**Hints**
- AQL (Artifactory Query Language) for stale artifacts:
  ```
  items.find({
    "repo": "libs-release-local",
    "stat.downloaded": {"$before": "90d"}
  }).include("name", "repo", "path", "size", "stat.downloaded")
  ```
- Run AQL: `jf rt cl -m POST "/api/search/aql" -d @query.aql`.
- Delete by AQL result: parse JSON output and pipe to `jf rt del`.
- Artifactory retention policies: `Admin > Repositories > <repo> > Xray` (for Xray-integrated cleanup) or use the Cleanup Policies UI (JFrog Platform 7.x+).
- Always test with `--dry-run` first: `jf rt del "libs-snapshot-local/*.jar" --dry-run`.

**Skills Tested**
- AQL (Artifactory Query Language)
- Artifact lifecycle and retention policies
- `jf rt del` with filters
- Storage capacity management
- Safe dry-run before destructive operations

---

## Advanced Tasks

---

### Task A1 — Implement a Secure Supply Chain with Artifact Signing

**Problem Statement**  
Your security team mandates that all release artifacts must be cryptographically signed and the signature verified before deployment. Implement: GPG signing of Maven JARs before upload, signature verification as a mandatory step in the deployment pipeline, Artifactory properties storing the signing key fingerprint, and Xray policy that blocks unsigned artifacts.

**Expected Outcome**
- Release JAR is signed: `myapp-1.0.jar.asc` uploaded alongside `myapp-1.0.jar`.
- CI pipeline verifies the signature before deploying: `gpg --verify myapp-1.0.jar.asc myapp-1.0.jar`.
- Property `gpg.fingerprint=ABCD1234...` set on the artifact in Artifactory.
- Deployment script fails if the signature file is missing or verification fails.
- Artifact provenance is traceable: who signed it, when, and with which key.

**Hints**
- Sign: `gpg --armor --detach-sign --default-key $KEY_ID myapp-1.0.jar`.
- Upload both files: `jf rt u "myapp-1.0.jar" libs-release-local/` and `jf rt u "myapp-1.0.jar.asc" libs-release-local/`.
- Set property: `jf rt sp "libs-release-local/myapp-1.0.jar" "gpg.fingerprint=$(gpg --fingerprint $KEY_ID | grep fingerprint | tr -d ' =' | cut -c12-)"`.
- Verify in deploy script: `gpg --verify "$artifact.asc" "$artifact" || { echo "Signature invalid!"; exit 1; }`.
- Store the public GPG key in Artifactory as a generic artifact for key distribution.

**Skills Tested**
- GPG signing and verification workflow
- Artifact signing in CI/CD pipelines
- Custom properties for artifact metadata
- Supply chain security concepts
- Software artifact provenance

---

### Task A2 — Build a Multi-Type Repository Strategy for a Large Organization

**Problem Statement**  
Design and implement a complete repository naming convention and structure for a 500-developer organization with 10 teams, 3 environments (dev, staging, prod), 4 package types (Docker, npm, Maven, Python), and policies that prevent dev artifacts from reaching prod without going through the promotion pipeline.

**Expected Outcome**
- Repository naming convention documented and implemented: `{team}-{type}-{env}-{local|remote|virtual}`.
- Virtual repositories per environment aggregate the correct local + remote repos.
- Permission targets enforce that only the `release-pipeline` service account can promote to `*-prod-*` repos.
- AQL query confirms no artifact with property `env=dev` exists in any prod repository.
- A repository template (via REST API) allows creating a new team's repos in under 5 minutes.

**Hints**
- Example naming: `teamalpha-docker-dev-local`, `teamalpha-docker-dev-virtual`, `teamalpha-docker-prod-local`.
- Use Artifactory's repository template API: `POST /artifactory/api/repositories/{repoKey}` with JSON body.
- Script repo creation: `for team in alpha beta gamma; do for type in docker npm maven pypi; do create_repo $team $type; done; done`.
- Enforce promotion gates via permission targets: `*-prod-*` repos have `Deploy` only for `ci-service-account` group.
- Property gate: set `env=dev` on build artifacts at creation; promotion script changes it to `env=prod`.

**Skills Tested**
- Enterprise repository strategy and naming conventions
- Multi-team, multi-environment Artifactory governance
- Repository templating via REST API
- Promotion gate enforcement via permissions
- AQL for compliance verification

---

### Task A3 — Configure Artifactory as a Helm Chart Repository

**Problem Statement**  
Your Kubernetes team manages 15 Helm charts and currently stores them as tarballs in a shared S3 bucket with no versioning UI, no search, and no security scanning. Migrate to Artifactory: set up a local Helm repository, push all charts, configure Xray scanning for Helm chart dependencies, and update all CI/CD pipelines to pull charts from Artifactory.

**Expected Outcome**
- Local Helm repository `helm-local` created and configured.
- All 15 charts uploaded: `jf rt u "charts/*.tgz" helm-local/`.
- `helm repo add myrepo https://myartifactory.example.com/artifactory/helm-local` works.
- `helm search repo myrepo/` lists all 15 charts with versions.
- `helm install myapp myrepo/myapp --version 1.2.3` installs from Artifactory.
- Xray scans the chart's container image dependencies and flags vulnerabilities.

**Hints**
- Artifactory Helm repo type: when creating, select `Helm` as the package type.
- Charts must be properly packaged tarballs: `helm package ./mychart/`.
- Helm repo index is auto-generated by Artifactory — no manual `helm repo index` needed.
- For Xray to scan Helm charts, enable Helm indexing: `Xray > Indexed Resources`.
- Update CI: replace `helm repo add stable https://charts.helm.sh/stable` with the Artifactory URL.
- Use `helm push` plugin or `jf rt u` for uploading chart tarballs.

**Skills Tested**
- Helm repository management in Artifactory
- Helm chart packaging and publishing
- Xray integration for Helm dependency scanning
- CI/CD pipeline migration to Artifactory-hosted Helm
- Helm client configuration (`helm repo add`)

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Debug a 403 Forbidden When Pushing Artifacts

**Problem Statement**  
A CI pipeline suddenly starts failing with `HTTP/1.1 403 Forbidden` when pushing artifacts to `libs-snapshot-local`. It worked yesterday. No code changes were made. Systematically diagnose all possible causes and fix the root issue.

**Expected Outcome**
- Root cause identified from a checklist of possible causes.
- Push succeeds after the fix.
- All findings documented so the team can prevent recurrence.

**Hints**
- Checklist to investigate (work through all of these):
  1. **API key/password expired** — check token expiry in `Admin > Identity and Access > Access Tokens`.
  2. **User's group membership changed** — check if the user was removed from the deploy group.
  3. **Permission target modified** — check if `Deploy` permission was removed from the repo.
  4. **Repository changed to Read Only** — check `Edit Repository > Basic Settings > Read Only`.
  5. **License expired** — Artifactory disables writes if the license is expired or invalid.
  6. **Quota exceeded** — `Admin > Storage` shows if the storage quota has been hit.
  7. **IP allowlist** — check if a network access policy was added.
- Test the token directly: `curl -u user:token https://art.example.com/artifactory/api/system/ping`.
- Check Artifactory access logs: `$ARTIFACTORY_HOME/var/log/artifactory-access.log`.

**Skills Tested**
- Systematic 403 diagnosis methodology
- Artifactory access log analysis
- Token/API key lifecycle management
- Permission target and repository settings audit
- License and quota monitoring

---

### Task T2 — Fix Replication Lag and Failures

**Problem Statement**  
The EU Artifactory mirror hasn't received new artifacts for 8 hours. European developers are getting stale packages. The replication shows status `Error` in the admin panel. Diagnose and fix the replication failure without data loss.

**Expected Outcome**
- Root cause of replication failure identified.
- Replication manually triggered and backlog caught up.
- Monitoring added so replication failures are alerted within 15 minutes in the future.
- Documentation updated with the runbook for this failure mode.

**Hints**
- Check replication status: `GET /artifactory/api/replication/{repoKey}` — look at `lastCompleted` and `status`.
- Common causes:
  - Network connectivity between primary and secondary (firewall rule change).
  - Credentials for the replication user expired or changed.
  - Target repository deleted or set to read-only incorrectly.
  - Artifactory version mismatch causing API incompatibility.
- Replication logs: `$ARTIFACTORY_HOME/var/log/artifactory.log` — filter for `replication`.
- Manual trigger: `POST /artifactory/api/replication/execute/{repoKey}`.
- Verify the replication user's credentials on the secondary still have `Deploy` on the target repo.

**Skills Tested**
- Replication failure diagnosis
- Artifactory REST API for replication status
- Network and credential troubleshooting
- Log analysis for replication errors
- Monitoring and alerting for replication health

---

### Task T3 — Resolve an Xray Scan That Never Completes

**Problem Statement**  
A build uploaded 6 hours ago still shows `Scanning` status in Xray and the pipeline is blocked waiting for the scan result. Other artifacts scan fine. Diagnose why this specific artifact is stuck and resolve it without re-uploading.

**Expected Outcome**
- Root cause identified (e.g., Xray indexer queue backed up, artifact type not indexed, Xray service degraded, corrupted artifact metadata).
- Scan completed or manually triggered.
- Pipeline unblocked.
- Xray indexer queue health added to monitoring.

**Hints**
- Check Xray system health: `GET /xray/api/v1/system/ping` and `GET /xray/api/v1/system/metrics`.
- View indexer queue depth: Xray admin metrics show pending indexing jobs.
- Force re-index: `POST /xray/api/v1/index/forceReindex` with the artifact details.
- Verify the artifact's package type is in Xray's indexed resources list — if not, add it and wait.
- Xray logs: `$JFROG_HOME/xray/var/log/xray-server.log` — look for indexing errors for that artifact SHA.
- Workaround for blocked pipelines: set `FAIL_BUILD_ON_SCAN_FAILURE=false` temporarily while investigating.

**Skills Tested**
- Xray service health monitoring
- Indexer queue management
- Force re-index via REST API
- Xray log analysis
- Pipeline unblocking strategies during scanner issues

---

### Task T4 — Diagnose and Clean Up a Full Artifactory Disk

**Problem Statement**  
Artifactory's storage is at 99% capacity. New uploads are failing with `No space left`. You cannot immediately add more storage. Safely free up space without deleting any artifact that was downloaded in the last 30 days or has the property `keep=true`.

**Expected Outcome**
- AQL query identifies candidate artifacts for deletion (not downloaded in 30 days, no `keep=true` property).
- Dry-run shows total recoverable space and artifact count.
- After deletion, storage is below 70% and uploads succeed.
- Garbage collection run to reclaim space from deleted artifacts (deleted artifacts still consume binaries until GC runs).
- A retention policy is created to prevent this situation recurring.

**Hints**
- Find candidates with AQL:
  ```
  items.find({
    "$and": [
      {"stat.downloaded": {"$before": "30d"}},
      {"@keep": {"$nmatch": "true"}}
    ]
  }).include("repo", "path", "name", "size")
  ```
- `jf rt del` with `--dry-run` first.
- After deletion, run garbage collection: `POST /artifactory/api/system/storage/gc` — this reclaims binary storage.
- GC can be slow — check `Admin > Storage > Garbage Collection` for progress.
- Check for orphaned binaries: `POST /artifactory/api/system/storage/optimize`.

**Skills Tested**
- AQL with compound conditions and property filters
- `jf rt del` safe deletion workflow
- Artifactory garbage collection mechanics
- Storage optimization and orphaned binary cleanup
- Proactive retention policy design

---

## Architecture & Design Tasks

---

### Task D1 — Design a Secure Software Supply Chain with JFrog

**Problem Statement**  
Design an end-to-end secure software supply chain using JFrog platform for a company shipping a containerized SaaS product. The design must cover: artifact ingestion from public registries (with Xray scanning), internal build and storage, signed release promotion, and secure distribution to customers. Document the flow, repository structure, and security gates at each stage.

**Expected Outcome**
- Architecture diagram (or written description) covering: Remote repos (proxying public registries) → Local build repos → Xray scan gate → Promotion to release repos → Customer distribution repos.
- Security gates defined: Xray blocks `Critical` CVEs and banned licenses, GPG signing required for promotion, only the release pipeline can write to release repos.
- Repository naming convention covers all artifact types and environments.
- Customer distribution uses a `Distribution` repository (JFrog Distribution / Release Bundles) with access tokens scoped per customer.
- Audit trail: every artifact traceable from source commit → build → scan result → promotion → distribution.

**Hints**
- Use JFrog Release Bundles for customer distribution — they're immutable, signed, and trackable.
- `jf ds rbc` (distribution release bundle create) builds a signed bundle from promoted artifacts.
- Customer edge nodes (JFrog Edge) can receive release bundles without exposing the full Artifactory.
- Document each stage with: what enters, what gate applies, what exits, who has access.
- Consider SBOMs: `jf rt build-add-dependencies` and Xray's SBOM export per build.

**Skills Tested**
- End-to-end supply chain architecture
- JFrog Release Bundles and Distribution
- Multi-stage security gate design
- Customer-facing artifact distribution patterns
- SBOM and artifact traceability concepts

---

### Task D2 — Migrate from Nexus Repository to JFrog Artifactory

**Problem Statement**  
Your company uses Sonatype Nexus 3 and has decided to migrate to JFrog Artifactory. Plan and execute a migration for: 500 GB of Maven artifacts, 200 GB of npm packages, and all existing user/group/permission configurations. The migration must have zero downtime for developers and a rollback plan if the migration fails.

**Expected Outcome**
- All artifacts migrated from Nexus to Artifactory with checksums verified.
- All users and groups recreated in Artifactory with equivalent permissions.
- CI/CD pipelines switched from Nexus URLs to Artifactory URLs with a 5-minute config change.
- A 2-week parallel-run period where both systems are active (Nexus read-only, Artifactory primary).
- Rollback plan: switch `settings.xml` and `.npmrc` back to Nexus URLs if issues arise.

**Hints**
- Use JFrog's `jf-nexus-migrator` tool or the JFrog Import tool (Artifactory Pro feature).
- Manual migration: `jf rt u --flat=false "**/*.jar"` after downloading from Nexus API.
- Verify checksums: `jf rt s --props ""` and compare SHA256 from Nexus.
- Nexus API to list artifacts: `GET /service/rest/v1/assets?repository={name}`.
- URL switch strategy: update only CI/CD config files (`settings.xml`, `.npmrc`) — no code changes needed.
- Use Artifactory's `Smart Remote Repository` to proxy Nexus during parallel-run (Artifactory fetches from Nexus if not locally available yet).

**Skills Tested**
- Artifact repository migration planning
- Checksum verification for data integrity
- Zero-downtime migration strategy
- Parallel-run / cut-over planning
- Rollback strategy design

---

## Quick Reference — JFrog CLI Commands

| Command | Purpose |
|---|---|
| `jf config add` | Configure Artifactory server connection |
| `jf rt ping` | Test connectivity to Artifactory |
| `jf rt u <file> <repo/path>` | Upload artifact |
| `jf rt dl <repo/path>` | Download artifact |
| `jf rt del <repo/path>` | Delete artifact (use `--dry-run` first) |
| `jf rt s <repo/pattern>` | Search artifacts by path |
| `jf rt sp <path> "key=val"` | Set properties on artifact |
| `jf rt cl -m GET /api/system/ping` | Raw REST API call |
| `jf rt bce $BUILD $NUM` | Collect environment variables into build info |
| `jf rt bp $BUILD $NUM` | Publish build info to Artifactory |
| `jf rt bpr $BUILD $NUM <target-repo>` | Promote build to another repository |
| `jf rt bs $BUILD $NUM` | Scan build with Xray |
| `jf xr scan --watches <watch>` | Trigger Xray watch scan |
| `jf ds rbc <bundle> <ver> --spec=spec.json` | Create release bundle |
| `jf ds rbd <bundle> <ver>` | Distribute release bundle |
| `jf rt build-clean $BUILD $NUM` | Clear local build info cache |

---

## Key AQL Query Patterns

```bash
# Find artifacts not downloaded in 90 days
items.find({
  "repo": "libs-release-local",
  "stat.downloaded": {"$before": "90d"}
}).include("name", "repo", "path", "size", "stat.downloaded")

# Find artifacts by property
items.find({
  "@status": {"$match": "release"},
  "repo": {"$match": "*-release-*"}
}).include("name", "repo", "path", "created")

# Find Docker images by tag
items.find({
  "repo": "docker-local",
  "name": {"$match": "manifest.json"},
  "path": {"$match": "myapp/*"}
}).include("path", "created", "size")

# Find artifacts created in the last 24 hours
items.find({
  "repo": "libs-snapshot-local",
  "created": {"$last": "24h"}
}).include("name", "path", "size", "created")

# Find artifacts with no downloads ever
items.find({
  "repo": "libs-release-local",
  "stat.downloaded": {"$eq": null}
}).include("name", "path", "size")
```

---

## Interview Cheat Sheet

- **"What is the difference between local, remote, and virtual repositories?"**  
  Local: stores artifacts internally — your published artifacts live here. Remote: a proxy/cache for an external registry (npmjs, Maven Central) — artifacts are downloaded on demand and cached. Virtual: an aggregation layer that combines local + remote repos behind a single URL. Developers should always use virtual repos.

- **"What is build info and why does it matter?"**  
  Build info is a JSON metadata record that links a CI build to every artifact it produced and consumed — including dependencies, modules, environment variables, and VCS info. It enables full traceability: given a running artifact, you can trace back to the exact commit and build that produced it.

- **"How does Xray differ from a traditional vulnerability scanner?"**  
  Xray is deeply integrated with Artifactory — it knows the exact dependency graph of every artifact (recursive deep inspection), not just surface-level CVEs. It can block downloads and fail builds at the repository level. Traditional scanners analyze files at a point in time; Xray continuously monitors as new CVEs are disclosed.

- **"Explain artifact promotion."**  
  Promotion moves (or copies) an artifact from one repository to another as it progresses through the pipeline — e.g., snapshot → staging → release. The artifact binary is NOT rebuilt — the same binary is promoted, ensuring what you tested is what you ship. Promotion updates properties and build info status.

- **"What is an AQL and when do you use it?"**  
  AQL (Artifactory Query Language) is a domain-specific query language for searching artifacts with complex criteria: by property, download stats, creation date, path, size, and Xray scan results. Used for cleanup scripts, compliance checks, and audit queries. More powerful than the simple search API.

- **"What is a JFrog Release Bundle?"**  
  A Release Bundle is an immutable, signed collection of artifacts used for secure distribution. It is cryptographically signed by the source Artifactory, distributed to edge nodes or external parties, and the signature can be verified at the destination. Used for customer-facing distribution where integrity guarantees are required.

- **"How does Artifactory handle high availability?"**  
  Active/Active clustering: multiple Artifactory nodes share a common database (PostgreSQL) and a shared NFS/S3 binary store. A load balancer distributes traffic. All nodes are equally capable of read and write. The database holds metadata; the binary store holds artifact content. JFrog's cloud (JFrog Platform) handles this automatically.

---

## Study Path

```
Week 1  → B1, B2, B3, B4         (Repository types, virtual repos, permissions)
Week 2  → I1, I2, I3             (CI/CD integration, Xray, promotion)
Week 3  → I4, I5, T1, T2         (Replication, retention, 403 debug, replication lag)
Week 4  → T3, T4, A1             (Xray stuck scan, disk full, artifact signing)
Week 5  → A2, A3                 (Repository strategy, Helm charts)
Week 6  → D1, D2 + AQL practice  (Supply chain design, Nexus migration, AQL queries)
```

---

*Set up a free JFrog Cloud instance (14-day trial or free tier at jfrog.com) and practice every CLI command hands-on. Being able to write AQL queries from memory and explain the local/remote/virtual model confidently are the two skills that distinguish strong candidates in JFrog interviews.*
