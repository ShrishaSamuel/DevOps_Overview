# Kubernetes Interview Preparation — Hands-On Tasks

> A curated set of real-world Kubernetes challenges organized by difficulty.  
> Each task mirrors scenarios you will encounter in DevOps interviews and on the job.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Cluster Design & Architecture Tasks](#cluster-design--architecture-tasks)

---

## Beginner Tasks

---

### Task B1 — Deploy and Expose a Stateless Application

**Problem Statement**  
A development team has a containerized nginx application (`nginx:1.25`) that needs to run with 3 replicas and be accessible from inside the cluster. Create a `Deployment` and a `ClusterIP` `Service` in a dedicated namespace called `web`.

**Expected Outcome**
- Namespace `web` exists.
- A `Deployment` named `nginx-app` runs 3 replicas with CPU request of `100m` and memory request of `128Mi`.
- A `ClusterIP` `Service` named `nginx-svc` forwards port `80` to the pods.
- `kubectl rollout status deployment/nginx-app -n web` reports success.

**Hints**
- Use `kubectl create namespace web` or a namespace manifest.
- Set resource requests under `spec.containers[].resources.requests`.
- The `Service` selector must match the `Deployment` pod labels exactly.
- Use `kubectl run` with `--dry-run=client -o yaml` to scaffold the manifest quickly.

**Skills Tested**
- Namespace management
- Deployment creation and rollout
- Service types and selector matching
- Resource requests and limits

---

### Task B2 — Manage Application Configuration with ConfigMaps and Secrets

**Problem Statement**  
An application reads its database host from the environment variable `DB_HOST` and its password from `DB_PASSWORD`. Store the host in a `ConfigMap` and the password in a `Secret`. Inject both into a pod without hardcoding any values in the pod spec.

**Expected Outcome**
- A `ConfigMap` named `app-config` with key `DB_HOST=postgres.internal`.
- A `Secret` named `app-secret` with key `DB_PASSWORD` (base64-encoded value).
- A `Deployment` whose containers consume both via `envFrom` or `valueFrom`.
- `kubectl exec` into a running pod and `echo $DB_HOST` prints `postgres.internal`.

**Hints**
- Use `kubectl create configmap` and `kubectl create secret generic` with `--from-literal`.
- You can inject a single key with `valueFrom.configMapKeyRef` or all keys with `envFrom.configMapRef`.
- Secrets are base64-encoded, not encrypted by default — understand the security implication.
- Verify with `kubectl exec <pod> -- env | grep DB`.

**Skills Tested**
- ConfigMap and Secret creation and injection
- Environment variable patterns in pod specs
- Understanding Secret encoding vs. encryption

---

### Task B3 — Perform a Rolling Update and Rollback

**Problem Statement**  
You have a running `Deployment` with image `myapp:v1`. Update it to `myapp:v2`, verify the rollout, then simulate a bad deployment by updating to a non-existent image `myapp:v99` and roll back to `v1`.

**Expected Outcome**
- `myapp:v2` rolls out with zero downtime (observe `kubectl get pods -w`).
- `myapp:v99` causes pods to enter `ImagePullBackOff`.
- `kubectl rollout undo` restores `myapp:v1` and pods become `Running` again.
- Rollout history shows at least 3 revisions.

**Hints**
- Use `kubectl set image deployment/<name> <container>=<image>` to trigger updates.
- Set `spec.strategy.rollingUpdate.maxUnavailable: 0` and `maxSurge: 1` for zero-downtime.
- Use `kubectl rollout history deployment/<name>` to list revisions.
- Use `kubectl rollout undo deployment/<name> --to-revision=<n>` to target a specific revision.
- Annotate deployments with `--record` (deprecated) or use `kubernetes.io/change-cause` annotation instead.

**Skills Tested**
- Rolling update strategy
- Rollback mechanics
- Deployment history and revision tracking

---

### Task B4 — Expose an Application Externally with NodePort and Ingress

**Problem Statement**  
Expose the `nginx-app` deployment from Task B1 externally. First use a `NodePort` service to verify basic external access, then replace it with an `Ingress` resource using an nginx ingress controller so the app is reachable at `http://myapp.local/`.

**Expected Outcome**
- `NodePort` service is accessible via `<node-ip>:<nodeport>`.
- After switching to `Ingress`, `curl -H "Host: myapp.local" http://<ingress-ip>/` returns nginx's default page.
- The `NodePort` service is cleaned up after the `Ingress` is working.

**Hints**
- Install the nginx ingress controller via its official manifest or Helm chart.
- The `Ingress` resource needs `spec.ingressClassName: nginx`.
- The backend service referenced by `Ingress` must be of type `ClusterIP`.
- Use `/etc/hosts` or `--resolve` with curl for local hostname testing.

**Skills Tested**
- Service types (ClusterIP, NodePort, LoadBalancer)
- Ingress resource and ingress controller concepts
- Layer 7 routing

---

## Intermediate Tasks

---

### Task I1 — Implement RBAC for a Developer Team

**Problem Statement**  
A team of developers should be able to `get`, `list`, `watch`, `create`, and `update` `Deployments`, `Pods`, and `Services` in the `development` namespace, but must have no access to other namespaces or cluster-wide resources like `Nodes` or `PersistentVolumes`.

**Expected Outcome**
- A `Role` named `dev-role` in namespace `development` with the above permissions.
- A `RoleBinding` that binds `dev-role` to a `ServiceAccount` named `dev-user`.
- `kubectl auth can-i list pods --namespace=development --as=system:serviceaccount:development:dev-user` returns `yes`.
- `kubectl auth can-i list nodes --as=system:serviceaccount:development:dev-user` returns `no`.

**Hints**
- Use `Role` (namespaced) not `ClusterRole` to limit blast radius.
- `RoleBinding` ties a `Role` to a subject; the subject here is a `ServiceAccount`.
- Use `kubectl auth can-i` extensively to verify each permission.
- Avoid wildcards (`*`) in `verbs` or `resources` — enumerate explicitly.

**Skills Tested**
- RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding
- Principle of least privilege
- ServiceAccount management
- `kubectl auth can-i` for verification

---

### Task I2 — Configure Horizontal Pod Autoscaler (HPA)

**Problem Statement**  
A backend API deployment runs 2 replicas and frequently gets CPU spikes during peak hours. Configure an HPA that scales the deployment between 2 and 10 replicas, targeting 50% CPU utilization. Simulate load and observe the autoscaler in action.

**Expected Outcome**
- `HorizontalPodAutoscaler` is created and shows `TARGETS` populating within a minute.
- Running a load generator (`kubectl run -it load --image=busybox`) causes replicas to scale up.
- Removing the load generator causes replicas to scale back down to 2 after the cooldown.
- `kubectl describe hpa` shows scaling events in the `Events` section.

**Hints**
- Metrics Server must be installed for HPA to work (`kubectl top pods` should work first).
- The `Deployment` must have CPU `requests` set — HPA calculates utilization as `usage/request`.
- Use `kubectl run load-gen --image=busybox --restart=Never -it -- /bin/sh -c "while true; do wget -q -O- http://<svc>; done"` for load.
- HPA scale-down is intentionally slow (default 5 min stabilization window) — check `--horizontal-pod-autoscaler-downscale-stabilization`.

**Skills Tested**
- HPA configuration and behavior
- Metrics Server dependency
- Resource requests as the basis for autoscaling
- Load testing with kubectl

---

### Task I3 — Run a Stateful Application with PersistentVolumes

**Problem Statement**  
Deploy a single-instance PostgreSQL database using a `StatefulSet`. The data must survive pod restarts. Use a `PersistentVolumeClaim` backed by a `StorageClass` to provision storage dynamically. Verify persistence by writing data, deleting the pod, and reading the data after the pod restarts.

**Expected Outcome**
- A `StatefulSet` named `postgres` with 1 replica in namespace `data`.
- A `PersistentVolumeClaim` of 5Gi mounted at `/var/lib/postgresql/data`.
- After `kubectl delete pod postgres-0`, the new pod reconnects to the same PVC.
- Data written before the delete is still present after restart.

**Hints**
- Use `volumeClaimTemplates` inside the `StatefulSet` spec — not a standalone PVC.
- Pass the PostgreSQL password via a `Secret` as an environment variable.
- Use `kubectl exec postgres-0 -- psql -U postgres -c "CREATE TABLE test (id int);"` to write data.
- Check `kubectl get pvc -n data` to confirm the PVC is `Bound`.

**Skills Tested**
- StatefulSet vs Deployment differences
- PersistentVolume, PVC, and StorageClass
- Dynamic provisioning
- Data persistence verification

---

### Task I4 — Package and Deploy an Application with Helm

**Problem Statement**  
Your team wants to deploy the same application across `dev`, `staging`, and `prod` environments with different replica counts, image tags, and resource limits. Package the application as a Helm chart and create a `values-dev.yaml`, `values-staging.yaml`, and `values-prod.yaml`.

**Expected Outcome**
- A valid Helm chart in `./charts/myapp/` with `Chart.yaml`, `values.yaml`, and templates.
- `helm install myapp-dev ./charts/myapp -f values-dev.yaml -n dev` deploys 1 replica with `myapp:dev`.
- `helm install myapp-prod ./charts/myapp -f values-prod.yaml -n prod` deploys 5 replicas with `myapp:v2.1.0` and higher resource limits.
- `helm list --all-namespaces` shows all three releases.
- Upgrading the chart version updates all three environments independently.

**Hints**
- Start with `helm create myapp` to scaffold the chart structure.
- Use `{{ .Values.replicaCount }}` and `{{ .Values.image.tag }}` in your templates.
- Use `helm template` to render manifests locally before installing.
- Use `helm lint` to validate the chart before deployment.
- Environment-specific `values-*.yaml` files only override what differs from `values.yaml`.

**Skills Tested**
- Helm chart structure and templating
- Values override pattern for multi-environment deploys
- Chart linting and dry-run
- Helm release lifecycle management

---

### Task I5 — Implement Network Policies to Isolate Namespaces

**Problem Statement**  
By default, pods in Kubernetes can communicate freely across namespaces. Lock down the `production` namespace so that only pods in the `production` namespace can talk to each other, and only pods with the label `role: frontend` can receive traffic from outside the namespace. All egress is allowed.

**Expected Outcome**
- A `NetworkPolicy` that denies all ingress to `production` pods by default.
- A second `NetworkPolicy` that allows ingress only from within `production` and from pods labeled `role: frontend` in any namespace.
- `kubectl exec` from a pod in a different namespace fails to reach `production` pods.
- A `frontend`-labeled pod in another namespace can still reach `production`.

**Hints**
- Require a CNI plugin that enforces `NetworkPolicy` (e.g., Calico, Cilium, Weave Net) — standard `kubenet` does not enforce them.
- Start with a default-deny-all ingress policy (`spec.ingress: []`).
- Use `podSelector` and `namespaceSelector` together with `matchLabels` in the ingress rules.
- Test with `kubectl exec <pod> -- curl http://<target-svc>.<namespace>.svc.cluster.local` and expect a timeout on blocked paths.

**Skills Tested**
- NetworkPolicy ingress/egress rules
- Label-based pod selection
- Namespace-level network isolation
- CNI plugin awareness

---

## Advanced Tasks

---

### Task A1 — Set Up Multi-Tenancy with Namespace Quotas and LimitRanges

**Problem Statement**  
Your cluster hosts three teams: `team-alpha`, `team-beta`, and `team-gamma`. Each team must be isolated to its own namespace and capped at: 4 CPUs, 8Gi memory, and no more than 20 pods cluster-wide per team. Individual containers must not exceed 1 CPU and 2Gi memory. Enforce these constraints using Kubernetes-native resources.

**Expected Outcome**
- A `ResourceQuota` per namespace capping CPU, memory, and pod count.
- A `LimitRange` per namespace setting default requests/limits and maximums per container.
- Attempting to create a pod that exceeds the `LimitRange` max is rejected with a clear error.
- Attempting to create pods beyond the quota causes the new pod to be pending.

**Hints**
- `ResourceQuota` enforces namespace-level totals; `LimitRange` enforces per-object limits and injects defaults.
- Without `LimitRange` defaults, pods with no resource spec don't count against CPU/memory quotas correctly.
- Use `kubectl describe resourcequota -n team-alpha` to monitor usage.
- Test quota exhaustion by deploying a deployment with more replicas than the quota allows and observe events.

**Skills Tested**
- ResourceQuota and LimitRange
- Multi-tenancy patterns
- Admission control understanding
- Resource governance

---

### Task A2 — Implement Pod Disruption Budgets for Zero-Downtime Maintenance

**Problem Statement**  
You have a critical payment-processing deployment running 5 replicas. During node maintenance (`kubectl drain`), you must ensure at least 4 replicas are always available. Implement a `PodDisruptionBudget` and verify it blocks the drain if the constraint cannot be satisfied.

**Expected Outcome**
- A `PodDisruptionBudget` named `payment-pdb` with `minAvailable: 4`.
- `kubectl drain <node> --ignore-daemonsets` is blocked when draining would violate the PDB.
- The drain succeeds gracefully once the PDB constraint can be satisfied (e.g., scale up to 6 replicas first).

**Hints**
- PDB `selector` must match the deployment's pod labels.
- You can use either `minAvailable` (absolute or percentage) or `maxUnavailable`.
- Force-draining with `--disable-eviction` bypasses PDB — understand why this is dangerous.
- Check PDB status with `kubectl get pdb` — `ALLOWED DISRUPTIONS` shows the current safe disruption count.

**Skills Tested**
- PodDisruptionBudget configuration
- Voluntary vs involuntary disruptions
- Node drain and cordon workflow
- High-availability production mindset

---

### Task A3 — Deploy a Service Mesh with Istio Traffic Management

**Problem Statement**  
You have two versions of a microservice (`v1` and `v2`) deployed simultaneously. Use Istio to implement a canary release that routes 90% of traffic to `v1` and 10% to `v2`. After confirming `v2` is stable (no errors in the `v2` pods), shift 100% of traffic to `v2` with a single manifest change.

**Expected Outcome**
- Istio installed and both deployments have the sidecar injected (2 containers per pod).
- `VirtualService` and `DestinationRule` configured for the 90/10 split.
- Repeated `curl` against the service URL shows approximately 10% responses from `v2` (`x-version: v2` header).
- Updating the `VirtualService` to `100% v2` redirects all traffic with no restart needed.

**Hints**
- Label pods with `version: v1` and `version: v2` and use these in `DestinationRule` subsets.
- The `VirtualService` `weight` fields across all routes must sum to 100.
- Use `istioctl analyze` to validate your Istio config.
- Check `istioctl proxy-status` to confirm sidecars are synced.
- Use `kubectl logs <pod> -c istio-proxy` to debug routing issues.

**Skills Tested**
- Service mesh concepts
- Canary deployments via traffic splitting
- Istio VirtualService, DestinationRule, and Gateway
- Progressive delivery patterns

---

### Task A4 — Implement Custom Autoscaling with KEDA

**Problem Statement**  
A batch processing service should scale its worker pods based on the length of an AWS SQS queue (or a Kafka topic). When the queue is empty, scale to 0. When messages arrive, scale up to a maximum of 20 workers. Use KEDA to implement event-driven autoscaling.

**Expected Outcome**
- KEDA installed in the cluster.
- A `ScaledObject` targeting the worker `Deployment` with `minReplicaCount: 0` and `maxReplicaCount: 20`.
- With an empty queue, `kubectl get pods` shows 0 worker pods.
- Publishing 100 messages to the queue causes KEDA to scale workers up within 30 seconds.
- Queue draining causes scale-down to 0.

**Hints**
- Install KEDA via Helm: `helm repo add kedacore https://kedacore.github.io/charts`.
- The `ScaledObject` references a `trigger` of type `aws-sqs-queue` or `kafka`.
- AWS credentials for SQS can be provided via a `TriggerAuthentication` resource using a `Secret` or IRSA (IAM Roles for Service Accounts).
- `targetQueueLength` in the trigger controls how many messages per worker pod.

**Skills Tested**
- Event-driven autoscaling concepts
- KEDA architecture and CRDs
- Scale-to-zero patterns
- External system integration (SQS/Kafka)

---

### Task A5 — Secure a Cluster with OPA/Gatekeeper Admission Policies

**Problem Statement**  
Enforce two cluster-wide security policies using OPA Gatekeeper:
1. All pods must have resource limits set.
2. Container images must only come from the approved registry `registry.company.internal/`.

Any pod violating either rule must be rejected at admission time with a descriptive error.

**Expected Outcome**
- Gatekeeper installed and the `validating-webhook-configuration` is active.
- A `ConstraintTemplate` and `Constraint` for each rule.
- `kubectl apply` of a pod with no limits is rejected: `denied by require-resource-limits`.
- `kubectl apply` of a pod using `docker.io/nginx` is rejected: `denied by approved-registries`.
- Compliant pods are admitted normally.

**Hints**
- Install Gatekeeper via Helm or its official manifest.
- `ConstraintTemplate` defines the Rego policy logic; `Constraint` instantiates it with parameters.
- In Rego, iterate over `input.review.object.spec.containers` to check each container.
- Use `kubectl get constraints` and `kubectl describe <constraint>` to see violations on existing resources.
- Test in `dryrun` enforcement mode first before switching to `deny`.

**Skills Tested**
- Policy-as-Code on Kubernetes
- OPA Rego policy writing
- Admission webhook concepts
- Shift-left security practices

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Debug a CrashLoopBackOff Pod

**Problem Statement**  
A pod is stuck in `CrashLoopBackOff`. You don't have the Dockerfile or application source. Diagnose the root cause and fix it using only `kubectl` commands.

**Expected Outcome**
- Root cause is identified (e.g., missing env var, wrong command, OOM kill, readiness probe misconfiguration).
- Pod reaches `Running` state after the fix.
- A brief post-mortem summary of the diagnosis steps is written in a comment in the manifest.

**Hints**
- Start with `kubectl describe pod <name>` — look at `Events`, `Last State`, and `Exit Code`.
- Exit code `1` = application error; `137` = OOM killed (memory limit too low); `132` = illegal instruction.
- Use `kubectl logs <pod> --previous` to see logs from the last crashed container.
- If the container exits too fast to exec into, override the command with `command: ["sleep", "3600"]` temporarily to inspect.
- Check for missing `ConfigMap`/`Secret` mounts that cause `CreateContainerConfigError`.

**Skills Tested**
- Systematic pod debugging methodology
- Understanding exit codes and pod lifecycle states
- `kubectl describe`, `kubectl logs --previous`, `kubectl exec`
- Temporary override techniques for debugging

---

### Task T2 — Fix a Service That Isn't Routing Traffic to Pods

**Problem Statement**  
A `ClusterIP` service exists and DNS resolves correctly, but `curl http://<service-name>` from another pod returns `Connection refused`. The pods are `Running` and the application is listening on port `8080`. Find and fix all misconfigurations.

**Expected Outcome**
- `curl http://<service-name>` returns a valid HTTP response.
- The fix requires editing the service or deployment manifest — no application changes.
- You document all misconfigurations found.

**Hints**
- Check `kubectl get endpoints <svc-name>` — if it shows `<none>`, the selector isn't matching any pods.
- Compare `spec.selector` in the `Service` to the `metadata.labels` on the pods.
- Check `spec.ports[].targetPort` in the `Service` — it must match the actual port the container listens on.
- Use `kubectl exec <pod> -- ss -tlnp` or `netstat` to confirm what port the app is actually listening on.
- Also check `spec.ports[].protocol` — TCP vs UDP mismatch silently breaks routing.

**Skills Tested**
- Service endpoint debugging
- Label selector troubleshooting
- Port mapping (port vs targetPort vs containerPort)
- Network debugging with `ss`, `curl`, `wget` inside pods

---

### Task T3 — Diagnose and Fix an OOMKilled Container

**Problem Statement**  
A memory-intensive data processing pod keeps getting killed with `OOMKilled` (exit code 137). The team claims the app "only needs 256Mi" but the kills keep happening. Diagnose actual memory usage and set correct limits to stabilize the pod.

**Expected Outcome**
- You determine the pod's real memory usage pattern using `kubectl top`.
- Memory `limits` are updated to a value that prevents OOMKill without over-provisioning.
- Pod runs stably for 10+ minutes under normal load.
- You document the before/after resource spec and explain how to use VPA (Vertical Pod Autoscaler) for future right-sizing.

**Hints**
- `kubectl top pod <name>` shows live memory usage — but watch it under load, not at idle.
- OOMKill happens when the container exceeds its `limits.memory` — the kernel kills it.
- Check if the app has JVM heap settings (e.g., `-Xmx`) that conflict with the container limit.
- Vertical Pod Autoscaler (VPA) in recommendation mode can suggest right-sized requests/limits.
- Set `requests < limits` to allow bursting but cap the ceiling.

**Skills Tested**
- OOM diagnosis and remediation
- `kubectl top` and metrics interpretation
- Request vs limit distinction
- VPA awareness

---

### Task T4 — Troubleshoot a Failing Readiness Probe

**Problem Statement**  
A newly deployed service has all pods in `Running` state but `kubectl get endpoints` shows no endpoints — meaning the `Service` never routes traffic to it. All pods show `0/1 Ready`. The application starts fine but takes 45 seconds to warm up.

**Expected Outcome**
- Root cause identified: readiness probe is checking too aggressively before the app is ready.
- Probe tuned with appropriate `initialDelaySeconds`, `periodSeconds`, and `failureThreshold`.
- Pods show `1/1 Ready` and endpoints are populated within 60 seconds of pod start.

**Hints**
- `kubectl describe pod <name>` shows readiness probe failures in Events.
- Readiness probe controls whether a pod receives traffic (endpoints); liveness probe controls whether a pod is restarted.
- For slow-starting apps, add `startupProbe` with a generous `failureThreshold * periodSeconds` window — this prevents liveness from killing the pod before it's ready.
- `initialDelaySeconds` is a simpler but less precise alternative to `startupProbe`.

**Skills Tested**
- Liveness vs Readiness vs Startup probe distinctions
- Probe tuning for slow-starting applications
- Endpoint population mechanics
- `kubectl describe` event interpretation

---

### Task T5 — Recover a Node That Is in NotReady State

**Problem Statement**  
One of your worker nodes shows `NotReady` in `kubectl get nodes`. Pods on this node are being evicted and rescheduled. Diagnose the node issue, recover it if possible, and ensure workloads are safely redistributed.

**Expected Outcome**
- Root cause of `NotReady` is identified (e.g., kubelet stopped, disk pressure, network partition, certificate expiry).
- Node is either recovered or cordoned and drained gracefully.
- All previously evicted pods are running on healthy nodes.
- A runbook for this failure scenario is written.

**Hints**
- `kubectl describe node <name>` shows `Conditions` (MemoryPressure, DiskPressure, PIDPressure, Ready).
- SSH to the node and check `systemctl status kubelet` and `journalctl -u kubelet -n 50`.
- Common causes: full disk (`/var/lib/docker` or `/var/log`), kubelet certificate expired, CNI plugin crash.
- `kubectl cordon <node>` prevents new scheduling; `kubectl drain <node>` evicts existing pods.
- After fix, `kubectl uncordon <node>` returns it to service.

**Skills Tested**
- Node lifecycle management
- Kubelet troubleshooting
- `kubectl cordon`, `drain`, `uncordon`
- Cluster operational runbook thinking

---

## Cluster Design & Architecture Tasks

---

### Task D1 — Design a Highly Available Multi-Zone Deployment

**Problem Statement**  
Design and implement a deployment strategy for a stateless web application that must tolerate the loss of any single availability zone. The application must have at least one replica per zone and must not schedule multiple replicas on the same node.

**Expected Outcome**
- A `Deployment` with `topologySpreadConstraints` ensuring pods are spread across zones.
- `podAntiAffinity` rules preventing two replicas from landing on the same node.
- Manually simulating a zone failure (cordoning all nodes in one zone) causes the remaining pods to still serve traffic.
- `kubectl get pods -o wide` confirms spread across zones.

**Hints**
- Use `topologySpreadConstraints` with `topologyKey: topology.kubernetes.io/zone` and `maxSkew: 1`.
- Combine with `podAntiAffinity` using `requiredDuringSchedulingIgnoredDuringExecution` for hard node anti-affinity.
- Node labels `topology.kubernetes.io/zone` must exist — check with `kubectl get nodes --show-labels`.
- Understand the difference between `requiredDuring...` (hard) and `preferredDuring...` (soft) affinity rules.

**Skills Tested**
- Topology spread constraints
- Pod affinity and anti-affinity
- High availability design
- Zone-aware scheduling

---

### Task D2 — Build a GitOps Pipeline with ArgoCD

**Problem Statement**  
Set up ArgoCD to manage the deployment of a Helm chart stored in a Git repository. Any change merged to the `main` branch should automatically sync to the `production` namespace in the cluster within 3 minutes. Changes to feature branches should trigger sync to a `preview` namespace.

**Expected Outcome**
- ArgoCD installed and UI accessible.
- An `Application` resource pointing to the Git repo and Helm chart.
- Merging a change to `main` (e.g., bumping the image tag) causes ArgoCD to auto-sync and the new pod is running within 3 minutes.
- ArgoCD UI shows `Synced` and `Healthy` status.

**Hints**
- Install ArgoCD via its official manifests or Helm chart.
- Set `spec.syncPolicy.automated.prune: true` and `selfHeal: true` for full GitOps automation.
- Use `spec.source.helm.valueFiles` to point to environment-specific values files in the repo.
- Create a `preview` `Application` with `targetRevision` pointing to the feature branch name.
- Use `argocd app sync` CLI to manually trigger sync during testing.

**Skills Tested**
- GitOps principles (Git as the source of truth)
- ArgoCD Application and AppProject resources
- Automated sync and self-healing
- Helm integration in GitOps workflows

---

### Task D3 — Implement Cluster Autoscaler for Cost-Optimized Scaling

**Problem Statement**  
Your cluster runs on AWS EKS with a single node group. Deploy the Cluster Autoscaler so that nodes are automatically added when pods are `Pending` due to insufficient capacity and removed when nodes are underutilized for 10 minutes. Demonstrate both scale-out and scale-in.

**Expected Outcome**
- Cluster Autoscaler deployed with correct IAM permissions (using IRSA).
- Deploying a resource-heavy workload causes a new EC2 node to join the cluster within 3 minutes.
- Scaling down the workload causes the extra node to terminate after the cool-down period.
- `kubectl logs -n kube-system -l app=cluster-autoscaler` shows scale-up and scale-down events.

**Hints**
- Cluster Autoscaler needs IAM permissions to describe and modify Auto Scaling Groups — use IRSA.
- Set `--scale-down-utilization-threshold=0.5` to trigger scale-down at 50% node utilization.
- Add annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "true"` to non-critical pods.
- Node groups must have the tag `k8s.io/cluster-autoscaler/<cluster-name>: owned`.
- Cluster Autoscaler and HPA can work together — understand the interaction.

**Skills Tested**
- Cluster Autoscaler setup and IAM (IRSA)
- Node group tagging and annotation conventions
- Scale-out and scale-in mechanics
- Cost optimization mindset

---

## Quick Reference — Key kubectl Commands

| Command | Purpose |
|---|---|
| `kubectl get pods -A -o wide` | List all pods across namespaces with node info |
| `kubectl describe pod <name> -n <ns>` | Full pod details including events |
| `kubectl logs <pod> -c <container> --previous` | Logs from last crashed container |
| `kubectl exec -it <pod> -- /bin/sh` | Interactive shell inside a container |
| `kubectl top pod / node` | Live CPU/memory usage (requires Metrics Server) |
| `kubectl rollout status deploy/<name>` | Wait for deployment to complete |
| `kubectl rollout undo deploy/<name>` | Roll back to previous revision |
| `kubectl rollout history deploy/<name>` | View revision history |
| `kubectl scale deploy/<name> --replicas=5` | Manually scale a deployment |
| `kubectl auth can-i <verb> <resource> --as=<user>` | Check RBAC permissions |
| `kubectl get endpoints <svc>` | Verify service endpoint population |
| `kubectl cordon / uncordon <node>` | Mark node unschedulable / restore it |
| `kubectl drain <node> --ignore-daemonsets` | Evict pods from a node safely |
| `kubectl apply --dry-run=client -o yaml` | Preview manifest without applying |
| `kubectl diff -f <file>` | Show diff between live and new config |
| `kubectl get events --sort-by='.lastTimestamp'` | View cluster events chronologically |
| `kubectl explain <resource>.<field>` | Inline API field documentation |

---

## Interview Cheat Sheet

- **"What is the difference between a Deployment and a StatefulSet?"**  
  Deployments are for stateless apps — pods are interchangeable. StatefulSets give each pod a stable hostname (`pod-0`, `pod-1`) and a dedicated PVC, required for databases and clustered apps.

- **"How does Kubernetes networking work?"**  
  Every pod gets a unique IP. Pods communicate directly (no NAT within the cluster). Services provide stable DNS names and virtual IPs (ClusterIP) with kube-proxy managing iptables/IPVS rules to load-balance to pod IPs.

- **"What is the role of the scheduler?"**  
  The `kube-scheduler` watches for unscheduled pods and selects a node based on resource availability, taints/tolerations, affinity rules, and topology constraints.

- **"Explain liveness vs readiness vs startup probes."**  
  Liveness = is the container alive? (restart if fails). Readiness = is the container ready for traffic? (remove from endpoints if fails). Startup = is the container done initializing? (holds off liveness/readiness checks).

- **"What is a ConfigMap vs a Secret?"**  
  Both store key-value data for pods. Secrets are base64-encoded (not encrypted by default) and are intended for sensitive data. Use RBAC and envelope encryption (KMS) to secure Secrets properly.

- **"How does HPA work?"**  
  HPA queries the Metrics Server (or custom metrics) every 15 seconds, computes `desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)`, and adjusts the replica count within `minReplicas`/`maxReplicas` bounds.

- **"What is a taint and toleration?"**  
  Taints repel pods from nodes unless the pod has a matching toleration. Used to reserve nodes for specific workloads (e.g., GPU nodes, production nodes).

- **"Explain RBAC in Kubernetes."**  
  Role/ClusterRole defines what can be done (verbs on resources). RoleBinding/ClusterRoleBinding assigns that role to a user, group, or ServiceAccount. Always use namespaced Roles unless cluster-wide access is genuinely required.

---

## Study Path

```
Week 1  → B1, B2, B3, B4             (Core objects, services, updates)
Week 2  → I1, I2, I3                 (RBAC, HPA, StatefulSets)
Week 3  → I4, I5, T1, T2             (Helm, NetworkPolicy, debugging)
Week 4  → T3, T4, T5, A1             (OOM, probes, nodes, quotas)
Week 5  → A2, A3, A4, A5             (PDB, Istio, KEDA, OPA)
Week 6  → D1, D2, D3                 (HA design, GitOps, autoscaling)
```

---

*Run every task on a local cluster (kind, minikube, or k3s) or on a free-tier cloud managed cluster. The hands-on muscle memory matters more than memorizing docs.*
