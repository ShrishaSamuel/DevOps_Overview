# Kubernetes — Zero to Hero

> A complete, beginner-to-advanced guide to Kubernetes (K8s) container orchestration for DevOps engineers.

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

### 1.1 What is Kubernetes?

**Kubernetes** (often abbreviated **K8s** — "K", eight letters, "s") is an open-source **container orchestration platform** originally developed by Google (based on their internal "Borg" system) and now maintained by the **Cloud Native Computing Foundation (CNCF)**. It automates the deployment, scaling, networking, and management of containerized applications across a cluster of machines.

Docker lets you run a container on one host. Kubernetes answers: *How do I run hundreds of containers across many hosts reliably, scale them on demand, recover from failures, roll out updates with zero downtime, and connect them together?*

### 1.2 Why Kubernetes? The problems it solves

| Problem | Kubernetes solution |
|---------|--------------------|
| A container crashes | Self-healing: restarts/replaces it automatically |
| Traffic spikes | Horizontal autoscaling adds replicas |
| Deploying updates without downtime | Rolling updates and rollbacks |
| Distributing load | Built-in service discovery and load balancing |
| Scheduling across many servers | Smart placement based on resources/constraints |
| Managing config and secrets | ConfigMaps and Secrets |
| Storage for stateful apps | Persistent Volumes |

### 1.3 Declarative model

Kubernetes is **declarative**: you describe the **desired state** (e.g., "I want 3 replicas of this app") in YAML manifests, and Kubernetes continuously works to make the **actual state** match. This is achieved by **controllers** running **reconciliation loops**.

```
        ┌──────────────────────────────────────┐
        │  Desired State (your YAML manifests)  │
        └───────────────────┬──────────────────┘
                            │ apply
                            ▼
        ┌──────────────────────────────────────┐
        │     Control Plane reconciles          │
        │  (observe → diff → act → repeat)      │
        └───────────────────┬──────────────────┘
                            ▼
        ┌──────────────────────────────────────┐
        │   Actual State (running containers)   │
        └──────────────────────────────────────┘
```

### 1.4 Cluster architecture

A Kubernetes cluster has a **control plane** (the brain) and **worker nodes** (where workloads run).

```
                        CONTROL PLANE (master)
   +-------------------------------------------------------------+
   |  kube-apiserver   <── all communication goes through here   |
   |  etcd             <── cluster state key-value store         |
   |  kube-scheduler   <── decides which node runs a pod         |
   |  controller-mgr   <── runs controllers (reconcile loops)    |
   |  cloud-controller <── integrates with cloud provider        |
   +-------------------------------------------------------------+
                            │ (API)
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                    ▼
   WORKER NODE 1       WORKER NODE 2        WORKER NODE 3
   +-----------+       +-----------+        +-----------+
   | kubelet   |       | kubelet   |        | kubelet   |  agent talking to API
   | kube-proxy|       | kube-proxy|        | kube-proxy|  networking rules
   | container |       | container |        | container |  runtime (containerd)
   |  runtime  |       |  runtime  |        |  runtime  |
   |  [Pods]   |       |  [Pods]   |        |  [Pods]   |
   +-----------+       +-----------+        +-----------+
```

**Control plane components:**
- **kube-apiserver** — the front door; all components and users talk to it via the REST API.
- **etcd** — consistent, distributed key-value store holding all cluster state.
- **kube-scheduler** — assigns newly created Pods to nodes based on resources and constraints.
- **kube-controller-manager** — runs controllers (Node, ReplicaSet, Deployment, Job, etc.).
- **cloud-controller-manager** — integrates with the underlying cloud (load balancers, volumes).

**Worker node components:**
- **kubelet** — the node agent; ensures containers described by Pods are running and healthy.
- **kube-proxy** — maintains network rules enabling Service networking.
- **container runtime** — runs containers (containerd, CRI-O).

### 1.5 When to use Kubernetes

- Running microservices at scale with many containers.
- Needing high availability, self-healing, and autoscaling.
- Multi-cloud or hybrid deployments needing portability.
- Complex deployments with rolling updates and canaries.

When **not** to use it: a single small app, very small teams without ops capacity, or simple workloads where managed PaaS / Docker Compose / serverless is sufficient. Kubernetes adds significant operational complexity.

---

## 2. Installation & Setup

### 2.1 Local clusters for learning

```bash
# --- Option A: Minikube ---
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker
minikube status

# --- Option B: kind (Kubernetes in Docker) ---
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind create cluster --name dev

# --- Option C: k3s (lightweight, great for edge/VMs) ---
curl -sfL https://get.k3s.io | sh -
```

### 2.2 Install kubectl (the CLI)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 2.3 Verify your cluster

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A          # all pods across all namespaces
```

**Expected:** At least one node in `Ready` status; system pods running in `kube-system`.

### 2.4 Quality-of-life setup

```bash
# Autocompletion and alias
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# Helpful tools
# kubectx/kubens — switch contexts/namespaces
# k9s — terminal UI for clusters
# stern — multi-pod log tailing
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Pods

A **Pod** is the smallest deployable unit in Kubernetes — one or more containers that share network (same IP) and storage. Usually one container per Pod.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod
kubectl exec -it nginx-pod -- bash
kubectl delete pod nginx-pod
```

> Pods are **ephemeral** — you rarely create them directly. Instead you use controllers (Deployments) that manage Pods for you.

#### 3.1.2 Labels and selectors

**Labels** are key-value tags on objects; **selectors** filter by them. They're how Kubernetes connects Services to Pods, Deployments to ReplicaSets, etc.

```bash
kubectl get pods -l app=nginx
kubectl label pod nginx-pod env=prod
kubectl get pods --show-labels
```

#### 3.1.3 kubectl basics

```bash
kubectl get <resource>            # list (pods, deployments, services, nodes...)
kubectl get pods -o wide          # more detail (node, IP)
kubectl get pods -o yaml          # full YAML
kubectl describe <resource> <name>  # detailed status + events
kubectl apply -f file.yaml        # create/update from manifest (declarative)
kubectl delete -f file.yaml
kubectl logs <pod>                # container logs
kubectl exec -it <pod> -- sh      # exec into a pod
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl explain pod.spec          # built-in schema docs
```

#### 3.1.4 Namespaces

**Namespaces** partition a cluster into virtual sub-clusters for isolation and organization.

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl apply -f app.yaml -n dev
kubectl get pods -n dev
kubectl config set-context --current --namespace=dev   # default namespace
```

Default namespaces: `default`, `kube-system` (control plane), `kube-public`, `kube-node-lease`.

### 3.2 INTERMEDIATE

#### 3.2.1 ReplicaSets and Deployments

A **Deployment** manages **ReplicaSets**, which keep a desired number of identical Pods running. Deployments add rolling updates and rollbacks.

```yaml
# deployment.yaml
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
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get rs                            # replicasets
kubectl scale deployment web --replicas=5
kubectl set image deployment/web web=nginx:1.27-alpine   # rolling update
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web       # roll back
```

#### 3.2.2 Services

A **Service** gives a stable network endpoint (IP + DNS name) to a set of Pods (which come and go). It load-balances across matching Pods.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web          # matches Pod labels
  ports:
    - port: 80        # service port
      targetPort: 80  # container port
  type: ClusterIP     # default
```

**Service types:**

| Type | Exposure | Use case |
|------|----------|----------|
| `ClusterIP` | Internal only (default) | Pod-to-Pod communication |
| `NodePort` | Each node's IP on a static port (30000–32767) | Dev/testing external access |
| `LoadBalancer` | Cloud load balancer with external IP | Production external access |
| `ExternalName` | Maps to a DNS name | Aliasing external services |

```bash
kubectl apply -f service.yaml
kubectl get svc
kubectl describe svc web-svc
kubectl port-forward svc/web-svc 8080:80   # access locally
minikube service web-svc                    # open NodePort in browser (minikube)
```

DNS: a Service is reachable at `<service>.<namespace>.svc.cluster.local`.

#### 3.2.3 ConfigMaps and Secrets

Separate configuration from images.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
---
# secret.yaml (values base64-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=    # echo -n 'password' | base64
```

Consume them in a Pod:

```yaml
spec:
  containers:
    - name: app
      image: myapp
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret
      # or mount as files:
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

```bash
kubectl create configmap app-config --from-literal=KEY=value
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=password
```

> Secrets are base64-encoded, **not encrypted** by default. Enable encryption at rest and use external secret managers for production.

#### 3.2.4 Storage: Volumes, PV, PVC

- **PersistentVolume (PV)** — a piece of storage in the cluster (provisioned by admin or dynamically).
- **PersistentVolumeClaim (PVC)** — a request for storage by a user/Pod.
- **StorageClass** — defines dynamic provisioning ("how" to create PVs).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
```

```yaml
# In a Pod/Deployment
    volumeMounts:
      - name: data
        mountPath: /var/lib/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-pvc
```

#### 3.2.5 Health probes

```yaml
    livenessProbe:        # restart container if this fails
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:       # remove from Service endpoints if this fails
      httpGet:
        path: /ready
        port: 80
      periodSeconds: 5
    startupProbe:         # for slow-starting apps
      httpGet:
        path: /healthz
        port: 80
      failureThreshold: 30
      periodSeconds: 10
```

- **Liveness:** is the app alive? Fail → restart the container.
- **Readiness:** is the app ready to serve? Fail → stop routing traffic to it.
- **Startup:** protects slow-starting apps from premature liveness failures.

### 3.3 ADVANCED

#### 3.3.1 StatefulSets

For stateful apps (databases) needing stable network identities and ordered, persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

Pods get stable names: `postgres-0`, `postgres-1`, `postgres-2`, each with its own PVC.

#### 3.3.2 DaemonSets, Jobs, CronJobs

```yaml
# DaemonSet: one pod per node (log collectors, monitoring agents)
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: node-exporter }
spec:
  selector: { matchLabels: { app: node-exporter } }
  template:
    metadata: { labels: { app: node-exporter } }
    spec:
      containers: [{ name: node-exporter, image: prom/node-exporter }]
---
# Job: run to completion (batch task)
apiVersion: batch/v1
kind: Job
metadata: { name: migrate }
spec:
  template:
    spec:
      restartPolicy: Never
      containers: [{ name: migrate, image: myapp, command: ["./migrate.sh"] }]
---
# CronJob: scheduled jobs
apiVersion: batch/v1
kind: CronJob
metadata: { name: backup }
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers: [{ name: backup, image: backup-tool }]
```

#### 3.3.3 Ingress

An **Ingress** manages external HTTP(S) access, routing by host/path to Services. Requires an **Ingress Controller** (nginx, Traefik, etc.).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
```

#### 3.3.4 Autoscaling

```yaml
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
kubectl autoscale deployment web --cpu-percent=70 --min=2 --max=10
kubectl get hpa
```

- **HPA** scales the number of Pods.
- **VPA** adjusts Pod resource requests.
- **Cluster Autoscaler** adds/removes nodes.

#### 3.3.5 RBAC (Role-Based Access Control)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

`Role`/`RoleBinding` are namespaced; `ClusterRole`/`ClusterRoleBinding` are cluster-wide.

#### 3.3.6 Scheduling controls

```yaml
spec:
  nodeSelector:
    disktype: ssd
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a"]
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

- **Taints & tolerations:** nodes repel pods unless the pod tolerates the taint.
- **Affinity/anti-affinity:** attract or repel pods relative to nodes/other pods.
- **Resource requests/limits:** drive scheduling and enforcement.

#### 3.3.7 Helm (the package manager)

**Helm** packages Kubernetes manifests into reusable, templated **charts**.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-release bitnami/nginx
helm list
helm upgrade my-release bitnami/nginx --set replicaCount=3
helm rollback my-release 1
helm uninstall my-release
helm create mychart                 # scaffold a new chart
helm template mychart               # render templates locally
```

#### 3.3.8 Observability and debugging

```bash
kubectl get pods -A
kubectl describe pod <pod>          # events at the bottom reveal scheduling/pull errors
kubectl logs <pod> --previous       # logs of crashed container
kubectl top nodes                   # resource usage (needs metrics-server)
kubectl top pods
kubectl get events --sort-by=.lastTimestamp
kubectl debug -it <pod> --image=busybox --target=<container>
```

Common Pod states: `Pending` (can't schedule), `ImagePullBackOff` (bad image/registry auth), `CrashLoopBackOff` (app keeps crashing), `OOMKilled` (out of memory), `Running`, `Completed`.

---

## 4. Hands-on Tasks

### Task 1: Deploy a Pod and inspect it

```bash
kubectl run nginx --image=nginx:1.27
kubectl get pods -o wide
kubectl describe pod nginx
kubectl logs nginx
```

**Expected:** Pod reaches `Running`; describe shows events and the assigned node.

### Task 2: Create a Deployment and scale it

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl get pods -l app=web
kubectl scale deployment web --replicas=5
kubectl get rs
```

**Expected:** 5 Pods running, managed by a ReplicaSet.

### Task 3: Expose with a Service

```bash
kubectl expose deployment web --port=80 --type=NodePort
kubectl get svc web
kubectl port-forward svc/web 8080:80 &
curl -s http://localhost:8080 | head -5
```

**Expected:** nginx welcome page via the Service.

### Task 4: Perform a rolling update and rollback

```bash
kubectl set image deployment/web nginx=nginx:1.27-alpine
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

**Expected:** Update rolls out gradually; rollback restores the previous image.

### Task 5: Use a ConfigMap and Secret

```bash
kubectl create configmap demo-config --from-literal=GREETING=hello
kubectl create secret generic demo-secret --from-literal=TOKEN=abc123
kubectl get configmap demo-config -o yaml
kubectl get secret demo-secret -o jsonpath='{.data.TOKEN}' | base64 -d; echo
```

**Expected:** ConfigMap shows the value; decoded secret prints `abc123`.

### Task 6: Add liveness and readiness probes

Apply a Deployment with the probes from section 3.2.5, then:

```bash
kubectl describe pod <pod> | grep -A3 Liveness
```

**Expected:** Probe configuration listed; unhealthy containers get restarted.

### Task 7: Persist data with a PVC

```bash
kubectl apply -f pvc.yaml      # from section 3.2.4
kubectl get pvc
```

**Expected:** PVC `Bound` to a PV (dynamic provisioning).

### Task 8: Schedule a CronJob

```bash
kubectl create cronjob hello --image=busybox --schedule="*/1 * * * *" -- echo "hello from cron"
kubectl get cronjob
sleep 70
kubectl get jobs
kubectl logs job/<job-name>
```

**Expected:** A Job is created each minute; logs print the message.

### Task 9: Inspect a CrashLoopBackOff

```bash
kubectl run crasher --image=busybox -- /bin/sh -c "exit 1"
kubectl get pod crasher          # CrashLoopBackOff
kubectl describe pod crasher
kubectl logs crasher --previous
kubectl delete pod crasher
```

**Expected:** Understand how to diagnose crashing pods.

### Task 10: Install an app with Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install demo bitnami/nginx
helm list
kubectl get pods
helm uninstall demo
```

**Expected:** Helm deploys nginx; resources appear; uninstall cleans up.

---

## 5. Projects

### Project 1 (Beginner): Deploy a Stateless Web App

**Goal:** Deploy a containerized web app with a Deployment + Service, exposed locally.

Steps:
1. Containerize a simple app (or use nginx) and push to a registry.
2. Write a Deployment (3 replicas, resource requests/limits, probes).
3. Write a ClusterIP Service.
4. Access it via `port-forward`.
5. Perform a rolling update and a rollback.

**Skills:** Pods, Deployments, Services, rollouts, probes.

### Project 2 (Intermediate): Multi-Tier App with Config, Secrets, Storage, Ingress

**Goal:** Deploy frontend + backend + database with proper configuration and external routing.

Architecture:
```
Ingress → frontend Service → frontend Pods
       → backend Service  → backend Pods → Postgres (StatefulSet + PVC)
ConfigMap + Secret feed the backend.
```

Steps:
1. Deploy Postgres as a StatefulSet with a `volumeClaimTemplate`.
2. Store DB credentials in a Secret; app settings in a ConfigMap.
3. Deploy backend (Deployment) consuming the Secret/ConfigMap, with probes.
4. Deploy frontend (Deployment + Service).
5. Add an Ingress routing `/` to frontend and `/api` to backend.
6. Verify end-to-end and demonstrate self-healing by deleting a Pod.

**Skills:** StatefulSets, PVCs, ConfigMaps/Secrets, Ingress, multi-tier networking.

### Project 3 (Advanced): Production-Ready Platform with Autoscaling, Monitoring & GitOps

**Goal:** Operate an app like production.

Steps:
1. Package the app as a **Helm chart** with values for dev/staging/prod.
2. Configure **HPA** to scale on CPU/memory; load-test to trigger scaling.
3. Add **resource requests/limits**, **PodDisruptionBudgets**, and **anti-affinity** for HA.
4. Apply **RBAC** least-privilege roles and **NetworkPolicies** to restrict traffic.
5. Install **Prometheus + Grafana** (e.g., kube-prometheus-stack) for monitoring/alerting.
6. Implement **GitOps** with Argo CD / Flux: cluster state synced from a Git repo.
7. Demonstrate a controlled rollout (canary/blue-green) and an automatic rollback.

**Skills:** Helm, autoscaling, HA, RBAC, network policies, observability, GitOps.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Always set resource requests and limits** — prevents noisy neighbors and OOM/scheduling issues.
- **Use Deployments, not bare Pods**, so failures self-heal.
- **Define liveness/readiness probes** for reliable rollouts and traffic routing.
- **Externalize config** with ConfigMaps/Secrets; never bake config into images.
- **Use namespaces** to separate environments/teams; apply RBAC least privilege.
- **Pin image tags / use digests** — avoid `latest` for reproducible deploys.
- **Manage manifests with Helm/Kustomize** and store them in Git (GitOps).
- **Apply NetworkPolicies** to restrict pod-to-pod traffic (default-deny).
- **Use PodDisruptionBudgets** and multiple replicas for high availability.
- **Monitor** with metrics-server, Prometheus/Grafana; centralize logs.
- **Encrypt secrets at rest** and integrate an external secret manager.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| No resource limits | Node exhaustion, OOMKills | Set requests/limits |
| Using `latest` tag | Unpredictable/irreproducible deploys | Pin versions/digests |
| Missing readiness probe | Traffic to not-ready pods → errors | Add readinessProbe |
| Treating Pods as durable | Data/identity lost on reschedule | Use StatefulSets + PVCs |
| Storing secrets in ConfigMaps/images | Credentials leak | Use Secrets + encryption/external managers |
| One giant namespace | No isolation, RBAC sprawl | Namespace per env/team |
| Ignoring `kubectl describe` events | Slow debugging | Read events first |
| No NetworkPolicy | Flat, open network | Default-deny + explicit allows |
| Editing live resources by hand | Drift from Git | GitOps / declarative apply |
| Underestimating complexity | Operational burden | Use managed K8s (EKS/GKE/AKS) when possible |

---

## 7. Interview Questions

### Beginner

1. **What is a Pod?**
   *The smallest deployable unit — one or more containers sharing network and storage, scheduled together.*

2. **What is the difference between a Deployment and a Pod?**
   *A Pod is a single instance; a Deployment manages a set of identical Pods (via ReplicaSets) with self-healing, scaling, and rolling updates.*

3. **What does a Service do?**
   *Provides a stable IP/DNS endpoint and load-balances traffic across a dynamic set of Pods selected by labels.*

4. **What is `kubectl apply` vs `kubectl create`?**
   *`apply` is declarative (creates or updates to match the manifest, idempotent); `create` is imperative (fails if the object exists).*

5. **What are labels and selectors?**
   *Labels are key-value tags on objects; selectors filter objects by labels — the mechanism that connects Services to Pods, etc.*

### Intermediate

6. **Explain the difference between liveness and readiness probes.**
   *Liveness determines if a container should be restarted; readiness determines if it should receive traffic. A failing readiness probe removes the pod from Service endpoints without restarting it.*

7. **Compare the Service types.**
   *ClusterIP (internal), NodePort (static port on each node), LoadBalancer (external cloud LB), ExternalName (DNS alias).*

8. **What is the difference between a Deployment and a StatefulSet?**
   *Deployments are for stateless, interchangeable Pods; StatefulSets provide stable network identities, ordered deployment/scaling, and per-Pod persistent storage for stateful apps.*

9. **How do ConfigMaps and Secrets differ?**
   *Both externalize config; Secrets are intended for sensitive data (base64-encoded, can be encrypted at rest, RBAC-restricted), ConfigMaps for non-sensitive config.*

10. **What happens when you run `kubectl apply` on a Deployment image change?**
    *A new ReplicaSet is created and Pods are rolled out gradually (rolling update) while old ones are scaled down, enabling zero-downtime updates and easy rollback.*

### Advanced

11. **Walk through the reconciliation loop.**
    *Controllers watch desired state (via the API server/etcd), compare it to actual state, and take actions to converge them, repeating continuously. This is the core of Kubernetes' self-healing.*

12. **How does Pod scheduling work?**
    *The scheduler filters nodes (resource requests, taints, node selectors/affinity, etc.), scores the feasible ones, and binds the Pod to the best node; kubelet then runs it.*

13. **How would you debug a Pod stuck in `Pending`?**
    *`kubectl describe pod` — check events for insufficient resources, unschedulable due to taints/affinity, unbound PVC, or no available nodes; then adjust requests, tolerations, or capacity.*

14. **Explain taints, tolerations, and affinity.**
    *Taints repel Pods from nodes unless the Pod has a matching toleration; affinity/anti-affinity attract or repel Pods relative to nodes or other Pods for placement control and HA.*

15. **How do you achieve zero-downtime deployments and safe rollouts?**
    *Multiple replicas, readiness probes, rolling update strategy (maxSurge/maxUnavailable), PodDisruptionBudgets, and rollback via `kubectl rollout undo`; advanced: canary/blue-green with Ingress or service meshes.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** What is the smallest deployable unit in Kubernetes?
- A) Container  B) Pod  C) Node  D) Deployment

**Q2.** Which component stores all cluster state?
- A) kubelet  B) etcd  C) kube-proxy  D) scheduler

**Q3.** Which Service type exposes an app via a cloud load balancer?
- A) ClusterIP  B) NodePort  C) LoadBalancer  D) ExternalName

**Q4.** Which controller manages stateful apps with stable identities?
- A) Deployment  B) DaemonSet  C) StatefulSet  D) Job

**Q5.** A failing readiness probe will:
- A) Restart the container  B) Remove the pod from Service endpoints  C) Delete the pod  D) Drain the node

**Q6.** Which object provides a stable endpoint for a set of Pods?
- A) Ingress  B) Service  C) ConfigMap  D) Namespace

**Q7.** What runs one Pod on every node?
- A) Deployment  B) ReplicaSet  C) DaemonSet  D) StatefulSet

**Q8.** Which scales the number of Pods based on metrics?
- A) VPA  B) HPA  C) Cluster Autoscaler  D) PDB

**Q9.** Which command shows recent cluster events for a pod?
- A) `kubectl logs`  B) `kubectl describe pod`  C) `kubectl get nodes`  D) `kubectl top`

**Q10.** Helm packages are called:
- A) Bundles  B) Charts  C) Stacks  D) Manifests

### Short Answer

**S1.** Write a command to scale a Deployment named `api` to 4 replicas.

**S2.** What command rolls back a Deployment named `web` to its previous version?

**S3.** How do you decode a base64 Secret value stored under key `PASSWORD` in secret `db`?

**S4.** Which probe protects slow-starting applications?

**S5.** What is the fully qualified DNS name of a Service `api` in namespace `prod`?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-C, Q4-C, Q5-B, Q6-B, Q7-C, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `kubectl scale deployment api --replicas=4`
- **S2.** `kubectl rollout undo deployment/web`
- **S3.** `kubectl get secret db -o jsonpath='{.data.PASSWORD}' | base64 -d`
- **S4.** The **startup** probe.
- **S5.** `api.prod.svc.cluster.local`

---

## 9. Further Resources

### Official Documentation
- Kubernetes Docs — https://kubernetes.io/docs/
- Kubernetes API reference — https://kubernetes.io/docs/reference/
- kubectl cheat sheet — https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- Helm Docs — https://helm.sh/docs/

### Interactive Learning
- Kubernetes the Hard Way — https://github.com/kelseyhightower/kubernetes-the-hard-way
- KillerCoda K8s scenarios — https://killercoda.com/kubernetes
- Play with Kubernetes — https://labs.play-with-k8s.com
- Katacoda-style labs / kube.academy

### Tools
- k9s (TUI), kubectx/kubens, stern (log tailing), kustomize
- Lens (IDE for K8s)
- Argo CD / Flux (GitOps), kube-prometheus-stack (monitoring)
- Trivy / kube-bench / kube-hunter (security)

### Books
- *Kubernetes Up & Running* — Burns, Beda, Hightower
- *The Kubernetes Book* — Nigel Poulton
- *Programming Kubernetes* — Hausenblas & Schimanski

### Certifications
- CKA — Certified Kubernetes Administrator
- CKAD — Certified Kubernetes Application Developer
- CKS — Certified Kubernetes Security Specialist

---

*End of Kubernetes — Zero to Hero.*
