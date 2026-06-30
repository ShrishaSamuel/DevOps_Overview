# Istio — Zero to Hero

> A complete, beginner-to-advanced guide to Istio, the service mesh for managing, securing, and observing microservices.

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

### 1.1 What is Istio?

**Istio** is an open-source **service mesh** that provides a uniform way to **connect, secure, control, and observe** microservices. It layers transparently onto a distributed application (typically on Kubernetes) and manages service-to-service communication — handling traffic routing, mutual TLS encryption, authorization, retries, and telemetry — **without requiring changes to application code**. Istio is a CNCF Graduated project, originally created by Google, IBM, and Lyft.

### 1.2 What is a service mesh?

A **service mesh** is a dedicated infrastructure layer for handling service-to-service communication. As microservices multiply, cross-cutting concerns — retries, timeouts, encryption, observability, traffic shifting — get duplicated in every service. A service mesh **extracts these concerns out of the application** and into the platform.

```
   Without a mesh                        With a service mesh
   ──────────────                        ───────────────────
   Each service implements:             Sidecar proxies handle:
   - retries/timeouts                   - retries/timeouts
   - TLS/mTLS                           - mTLS encryption
   - metrics/tracing       →            - metrics/tracing
   - routing/load balancing             - routing/load balancing
   (duplicated everywhere)              (uniform, app stays simple)
```

### 1.3 The sidecar pattern

Istio's classic model injects a **sidecar proxy** (Envoy) next to each application container in the same pod. All inbound/outbound traffic is transparently redirected through the proxy.

```
   Pod
   ┌──────────────────────────────┐
   │  App container                │
   │      │ all traffic            │
   │      ▼                        │
   │  Envoy sidecar proxy ─────────┼──► to other services' sidecars
   └──────────────────────────────┘
```

The app talks to `localhost`; the sidecar does mTLS, routing, retries, and reports telemetry. The app is unaware the mesh exists.

> **Ambient mode** (newer) removes per-pod sidecars in favor of a shared per-node **ztunnel** + optional **waypoint** proxies, reducing overhead — see Advanced.

### 1.4 Architecture: data plane and control plane

```
   ┌─────────────────────── Control Plane ───────────────────────┐
   │                        istiod                                │
   │  - Pilot: config & service discovery → pushes to proxies     │
   │  - Citadel: certificate authority (mTLS identities)          │
   │  - Galley/config: validates & distributes configuration      │
   └───────────────────────────┬──────────────────────────────────┘
                               │ xDS API (config, certs)
   ┌───────────────────────────▼──────────────────────────────────┐
   │                       Data Plane                              │
   │   Envoy sidecars (one per pod) carry the actual traffic       │
   │   svcA[envoy] ⇄ mTLS ⇄ [envoy]svcB ⇄ mTLS ⇄ [envoy]svcC       │
   └───────────────────────────────────────────────────────────────┘
```

- **Data plane:** The **Envoy** proxies that intercept and handle all service traffic.
- **Control plane:** **istiod** — a single binary combining Pilot (config/discovery), Citadel (CA for mTLS), and config validation — which configures the proxies via the **xDS** API.

### 1.5 The three pillars Istio addresses

| Pillar | What Istio provides |
|--------|---------------------|
| **Traffic management** | Routing, canary/blue-green, retries, timeouts, fault injection, mirroring |
| **Security** | Automatic mTLS, identity (SPIFFE), authn/authz policies |
| **Observability** | Metrics, distributed tracing, access logs, service graph — out of the box |

### 1.6 Why use Istio?

- **Zero-trust security** via automatic mutual TLS between services.
- **Advanced traffic control** (canary releases, A/B testing) without app changes.
- **Uniform observability** (golden metrics, tracing) across all services.
- **Resilience** (retries, timeouts, circuit breaking, outlier detection).
- **Policy enforcement** (access control, rate limiting) centrally.

### 1.7 When to use Istio (and when not)

**Use Istio when:** you run many microservices on Kubernetes and need consistent security (mTLS), sophisticated traffic management, and observability without modifying every service.

**Reconsider when:** you have only a few services (the operational complexity may outweigh benefits), a very small team, or constrained resources — a simpler ingress + libraries may suffice. Istio adds real complexity and overhead; adopt it when the scale justifies it. Lighter meshes (Linkerd) are an alternative.

---

## 2. Installation & Setup

### 2.1 Prerequisites

- A running Kubernetes cluster and `kubectl`. For local practice: kind, minikube, or k3d (give it enough CPU/RAM).

### 2.2 Install istioctl

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*/
export PATH=$PWD/bin:$PATH
istioctl version
```

### 2.3 Install Istio onto the cluster

```bash
# Install with a configuration profile (demo = full features for learning)
istioctl install --set profile=demo -y

# Verify
kubectl get pods -n istio-system
istioctl verify-install
```

**Expected:** `istiod` and ingress/egress gateway pods are `Running` in `istio-system`.

**Profiles:** `default` (production baseline), `demo` (all features, for learning), `minimal` (control plane only), `ambient` (sidecar-less mode).

### 2.4 Enable sidecar injection

```bash
# Label a namespace so new pods get sidecars automatically
kubectl label namespace default istio-injection=enabled
kubectl get namespace -L istio-injection
```

Now deploy (or redeploy) your apps — each pod gets an Envoy sidecar.

```bash
kubectl apply -f myapp.yaml
kubectl get pod                    # READY shows 2/2 (app + sidecar)
```

### 2.5 Install observability addons

```bash
kubectl apply -f samples/addons/      # Kiali, Prometheus, Grafana, Jaeger
istioctl dashboard kiali              # open the service graph UI
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Sidecar injection

Istio works by injecting the Envoy proxy. Two ways:
- **Automatic:** label the namespace `istio-injection=enabled`; new pods get sidecars.
- **Manual:** `istioctl kube-inject -f deploy.yaml | kubectl apply -f -`.

A pod shows `2/2` ready when the app + sidecar are running.

#### 3.1.2 The Envoy proxy

**Envoy** is the high-performance C++ proxy that forms Istio's data plane. Each sidecar:
- Intercepts all pod traffic (via iptables rules).
- Applies routing, load balancing, retries, timeouts.
- Performs mTLS encryption/decryption.
- Emits metrics, logs, and traces.

#### 3.1.3 istiod (the control plane)

**istiod** configures all the Envoy proxies. It:
- Translates high-level Istio config (VirtualService, etc.) into Envoy config.
- Provides service discovery.
- Acts as the certificate authority issuing identities for mTLS.

#### 3.1.4 Gateways (ingress)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway          # use the built-in ingress gateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "myapp.example.com"
```

A **Gateway** manages inbound (or outbound) traffic at the mesh edge — like an ingress, but it only configures L4-L6; routing rules come from a VirtualService bound to it.

#### 3.1.5 The Bookinfo sample

Istio ships a **Bookinfo** demo app (product page + reviews v1/v2/v3 + ratings + details) used throughout the docs to demonstrate traffic routing, canaries, and observability. It's the canonical learning playground.

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

### 3.2 INTERMEDIATE

#### 3.2.1 VirtualService (routing rules)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination:
            host: reviews
            subset: v2          # internal users → v2
    - route:
        - destination:
            host: reviews
            subset: v1          # everyone else → v1
```

A **VirtualService** defines **how** requests are routed to a service — by header, URI, weight, etc. It's the core traffic-management resource.

#### 3.2.2 DestinationRule (subsets and policies)

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
    - name: v3
      labels: { version: v3 }
```

A **DestinationRule** defines **what happens after routing**: named **subsets** (versions), load balancing, connection pools, and outlier detection. VirtualService routes *to* subsets defined here.

#### 3.2.3 Canary deployments (weighted routing)

```yaml
http:
  - route:
      - destination: { host: reviews, subset: v1 }
        weight: 90
      - destination: { host: reviews, subset: v2 }
        weight: 10          # send 10% of traffic to the new version
```

Gradually shift weights (90/10 → 50/50 → 0/100) to perform a **canary release** — no app changes, pure config.

#### 3.2.4 Resilience: timeouts, retries, circuit breaking

```yaml
# VirtualService: timeout + retries
http:
  - route:
      - destination: { host: ratings, subset: v1 }
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure
```

```yaml
# DestinationRule: circuit breaking via outlier detection
trafficPolicy:
  connectionPool:
    http: { http1MaxPendingRequests: 100, maxRequestsPerConnection: 10 }
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionTime: 30s        # eject unhealthy hosts
```

#### 3.2.5 Fault injection (chaos testing)

```yaml
http:
  - fault:
      delay:
        percentage: { value: 10 }
        fixedDelay: 5s            # add latency to 10% of requests
      abort:
        percentage: { value: 5 }
        httpStatus: 500           # return 500 for 5% of requests
    route:
      - destination: { host: ratings, subset: v1 }
```

Test how your system handles failures and latency — without breaking real dependencies.

#### 3.2.6 Mutual TLS (mTLS)

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT          # require mTLS for all service-to-service traffic
```

Istio **automatically** encrypts service-to-service traffic with mTLS and rotates certificates. Modes: `PERMISSIVE` (accept both plaintext and mTLS — for migration) and `STRICT` (mTLS only).

### 3.3 ADVANCED

#### 3.3.1 Authorization policies

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-policy
  namespace: default
spec:
  selector:
    matchLabels: { app: reviews }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/productpage"]
      to:
        - operation:
            methods: ["GET"]
```

**AuthorizationPolicy** enforces fine-grained, identity-based access control (who can call whom, which methods/paths) using the SPIFFE identities from mTLS — enabling **zero-trust** networking.

#### 3.3.2 Request authentication (JWT)

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
spec:
  selector:
    matchLabels: { app: productpage }
  jwtRules:
    - issuer: "https://accounts.example.com"
      jwksUri: "https://accounts.example.com/.well-known/jwks.json"
```

Validate end-user JWTs at the mesh layer; combine with AuthorizationPolicy to require valid tokens.

#### 3.3.3 Observability stack

Istio generates telemetry automatically:
- **Metrics** (Prometheus): the **golden signals** — request rate, error rate, latency — per service.
- **Distributed tracing** (Jaeger/Zipkin/Tempo): trace requests across services (apps should propagate trace headers).
- **Kiali:** a service-graph dashboard visualizing topology, traffic, health, and config.
- **Grafana:** dashboards for mesh and service metrics.
- **Access logs:** Envoy logs per request.

```bash
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
```

#### 3.3.4 Multi-cluster and mesh expansion

Istio can form a **single mesh across multiple clusters** (multi-primary or primary-remote) for HA and locality, and extend the mesh to **VMs/non-Kubernetes workloads** via WorkloadEntry/WorkloadGroup. East-west gateways connect clusters securely.

#### 3.3.5 Egress control

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts: ["api.external.com"]
  ports:
    - { number: 443, name: https, protocol: TLS }
  resolution: DNS
  location: MESH_EXTERNAL
```

By default, services can call external endpoints; for tighter security you can set the mesh to block egress and explicitly allow destinations via **ServiceEntry** + an **egress Gateway** (audit/control outbound traffic).

#### 3.3.6 Ambient mesh (sidecar-less)

Istio's **ambient mode** removes per-pod sidecars:
- **ztunnel** — a per-node L4 proxy providing mTLS and identity (the secure overlay).
- **waypoint proxy** — an optional per-namespace/service L7 proxy for advanced routing/policy.

```
   Sidecar mode:  one Envoy per pod (more overhead, full L7 everywhere)
   Ambient mode:  shared ztunnel per node (L4) + optional waypoints (L7)
                  → lower resource cost, incremental adoption
```

Benefits: lower overhead, no pod restarts to enroll, pay for L7 only where needed.

#### 3.3.7 Performance and resource considerations

- Sidecars add **latency** (typically a few ms) and **CPU/memory** per pod.
- Tune **concurrency**, scope config with **Sidecar** resources (limit what each proxy knows), and use **PeerAuthentication** scoping.
- Use the `default` profile (not `demo`) in production; right-size istiod.
- Ambient mode reduces the data-plane footprint significantly.

#### 3.3.8 Gateway API and the future

Istio increasingly supports the **Kubernetes Gateway API** (the successor to Ingress) for traffic configuration, providing a standardized, portable way to define gateways and routes alongside Istio's own CRDs.

---

## 4. Hands-on Tasks

### Task 1: Install Istio and verify

Install with the `demo` profile; confirm `istiod` and gateways are running.

**Expected:** `istio-system` pods are `Running`; `istioctl verify-install` passes.

### Task 2: Enable injection and deploy Bookinfo

Label `default` with `istio-injection=enabled`; deploy Bookinfo.

**Expected:** Each pod is `2/2` (app + sidecar).

### Task 3: Expose via a Gateway

Apply the Bookinfo Gateway + VirtualService; access the product page.

**Expected:** The app is reachable through the ingress gateway.

### Task 4: Route all traffic to v1

Apply DestinationRules (subsets) and a VirtualService sending reviews to `v1`.

**Expected:** The reviews section consistently shows the v1 version.

### Task 5: Header-based routing

Route a specific user (header `end-user: jason`) to `v2`, others to `v1`.

**Expected:** That user sees v2; everyone else sees v1.

### Task 6: Canary with weights

Split reviews traffic 80/20 between v1 and v3.

**Expected:** Roughly 20% of requests hit v3.

### Task 7: Inject a fault

Add a 5s delay to a percentage of ratings requests.

**Expected:** Affected requests are delayed; observe app behavior/timeouts.

### Task 8: Enable STRICT mTLS

Apply a `PeerAuthentication` with `mode: STRICT` in the namespace.

**Expected:** Plaintext traffic is rejected; service-to-service is encrypted.

### Task 9: Add an AuthorizationPolicy

Allow only `productpage` to call `reviews` via GET; deny others.

**Expected:** Unauthorized callers receive `403`.

### Task 10: Explore the service graph

Generate traffic and open **Kiali**.

**Expected:** Kiali shows the topology, traffic flow, mTLS status, and health.

---

## 5. Projects

### Project 1 (Beginner): Mesh-Enable an App with Observability

**Goal:** Get a microservice app into the mesh and observe it.

Steps:
1. Install Istio (`demo`) and enable injection on a namespace.
2. Deploy Bookinfo (or your own multi-service app) with sidecars.
3. Expose it via a **Gateway** + **VirtualService**.
4. Install addons; explore **Kiali**, **Grafana**, **Jaeger**.
5. Generate load and read the golden metrics + traces.

**Skills:** injection, gateways, basic routing, observability.

### Project 2 (Intermediate): Progressive Delivery + Resilience

**Goal:** Safely roll out a new version with resilience controls.

Steps:
1. Define **DestinationRule** subsets for v1/v2/v3.
2. Perform a **canary** by shifting weights gradually; optionally mirror traffic.
3. Add **timeouts, retries, and circuit breaking** (outlier detection).
4. Use **fault injection** to test failure handling.
5. Watch metrics in Grafana/Kiali to validate each step; roll back via config.

**Skills:** canary/weighted routing, traffic mirroring, resilience, fault injection.

### Project 3 (Advanced): Zero-Trust, Multi-Service Platform

**Goal:** A secure, governed, observable mesh for production.

Steps:
1. Enforce **STRICT mTLS** mesh-wide; verify certificate rotation.
2. Implement **AuthorizationPolicies** (least-privilege service-to-service) and **RequestAuthentication** (JWT for end users).
3. Control **egress** with ServiceEntry + egress gateway; default-deny external calls.
4. Add ingress with the **Gateway API**, TLS, and rate limiting.
5. Set up full **observability** (Prometheus/Grafana/Jaeger/Kiali) with SLOs/alerts.
6. Evaluate **ambient mode** to cut overhead; scope proxies with `Sidecar` resources.
7. (Optional) Extend to **multi-cluster** for HA.

**Skills:** zero-trust security, authz/authn, egress control, Gateway API, observability/SLOs, ambient mode, multi-cluster.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Start small** — mesh one namespace/app first; expand gradually.
- **Use the `default` profile in production** (not `demo`).
- **Migrate mTLS via PERMISSIVE → STRICT** to avoid breaking traffic.
- **Pair VirtualService with DestinationRule** (subsets must exist before routing to them).
- **Adopt least-privilege AuthorizationPolicies** for zero-trust.
- **Set timeouts, retries, and circuit breakers** for resilience.
- **Use canary/weighted routing** for safe releases; mirror traffic to test.
- **Scope proxy config** with `Sidecar` resources to reduce memory/CPU.
- **Leverage built-in observability** (Kiali/Grafana/Jaeger); propagate trace headers in apps.
- **Control egress** explicitly for security-sensitive environments.
- **Evaluate ambient mode** to lower overhead.
- **Keep Istio and Envoy updated**; plan upgrades with canary control-plane revisions.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Adopting Istio with few services | Complexity > benefit | Use simpler ingress/libraries until scale justifies |
| Forgetting namespace injection label | No sidecars; mesh features absent | `istio-injection=enabled` + redeploy |
| VirtualService routing to undefined subsets | Routing failures | Define subsets in a DestinationRule first |
| Jumping straight to STRICT mTLS | Breaks plaintext callers | Start PERMISSIVE, then STRICT |
| Using the `demo` profile in prod | Excess resource use, insecure defaults | Use `default`/tuned profile |
| Ignoring sidecar resource overhead | Cluster pressure | Right-size, scope with `Sidecar`, or use ambient |
| No timeouts/retries config | Cascading failures | Configure resilience policies |
| Apps not propagating trace headers | Broken distributed traces | Forward B3/W3C trace headers |
| Unbounded egress | Data exfiltration risk | ServiceEntry + egress gateway, default-deny |
| Big-bang rollout | Wide blast radius | Incremental adoption + canary control plane |

---

## 7. Interview Questions

### Beginner

1. **What is a service mesh?**
   *A dedicated infrastructure layer that manages service-to-service communication (routing, security, resilience, observability) outside of application code.*

2. **What is Istio?**
   *An open-source service mesh that connects, secures, controls, and observes microservices, typically on Kubernetes, without changing app code.*

3. **What is the sidecar pattern?**
   *Running a proxy (Envoy) container alongside each app container in the same pod to intercept and manage all its traffic.*

4. **What proxy does Istio use in its data plane?**
   *Envoy.*

5. **What is istiod?**
   *Istio's control plane component that configures the proxies, provides service discovery, and acts as the certificate authority for mTLS.*

### Intermediate

6. **What is the difference between the data plane and control plane?**
   *The data plane (Envoy sidecars) carries the actual traffic; the control plane (istiod) configures and manages those proxies.*

7. **What do VirtualService and DestinationRule do?**
   *A VirtualService defines how traffic is routed to a service (by header/URI/weight); a DestinationRule defines subsets (versions) and policies (load balancing, outlier detection) applied after routing.*

8. **How do you perform a canary release in Istio?**
   *Define version subsets in a DestinationRule and use weighted routing in a VirtualService, gradually shifting traffic from the old to the new version.*

9. **How does Istio provide mTLS?**
   *istiod issues SPIFFE identities/certificates to each workload; sidecars automatically encrypt and authenticate service-to-service traffic, with PERMISSIVE or STRICT modes.*

10. **What resilience features does Istio offer?**
    *Timeouts, retries, circuit breaking (outlier detection), connection pool limits, and fault injection — all configured declaratively.*

### Advanced

11. **How does Istio implement zero-trust security?**
    *Through automatic mTLS (strong workload identity) plus AuthorizationPolicies that allow only explicitly permitted, identity-based service-to-service calls, defaulting to deny.*

12. **What is ambient mode and how does it differ from sidecars?**
    *Ambient mode removes per-pod sidecars, using a shared per-node ztunnel for L4/mTLS and optional waypoint proxies for L7, reducing resource overhead and enabling incremental adoption.*

13. **How do you control egress traffic in Istio?**
    *Define external destinations with ServiceEntry and route them through an egress Gateway, optionally setting the mesh to block undeclared external traffic for security.*

14. **How does Istio enable observability without code changes?**
    *Envoy sidecars automatically emit metrics (golden signals to Prometheus), access logs, and trace spans; tools like Kiali, Grafana, and Jaeger visualize them (apps need only propagate trace headers).*

15. **What are the trade-offs of adopting Istio?**
    *It adds operational complexity, per-pod latency, and CPU/memory overhead in exchange for uniform security, traffic control, and observability — justified at microservice scale but often overkill for small systems.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Istio is a:
- A) Container runtime  B) Service mesh  C) CI tool  D) Database

**Q2.** Istio's data plane proxy is:
- A) NGINX  B) HAProxy  C) Envoy  D) Traefik

**Q3.** The control plane component is called:
- A) Pilot only  B) istiod  C) Citadel only  D) Galley only

**Q4.** Which resource defines traffic routing rules?
- A) DestinationRule  B) VirtualService  C) Gateway  D) ServiceEntry

**Q5.** Which defines version subsets and policies?
- A) VirtualService  B) DestinationRule  C) PeerAuthentication  D) Sidecar

**Q6.** Automatic encryption between services is via:
- A) TLS offload  B) mTLS  C) IPsec  D) SSH

**Q7.** Which enables a canary release?
- A) Fault injection  B) Weighted routing  C) mTLS  D) Egress

**Q8.** Which enforces identity-based access control?
- A) AuthorizationPolicy  B) Gateway  C) DestinationRule  D) ServiceEntry

**Q9.** Sidecar-less Istio mode is called:
- A) minimal  B) demo  C) ambient  D) remote

**Q10.** The mTLS mode that accepts both plaintext and mTLS is:
- A) STRICT  B) PERMISSIVE  C) DISABLE  D) AUTO

### Short Answer

**S1.** What label enables automatic sidecar injection on a namespace?

**S2.** Name the per-node proxy used in ambient mode.

**S3.** Which sample app is used throughout Istio docs?

**S4.** Which CRD lets you call/allow external services?

**S5.** Name the service-graph visualization tool bundled with Istio.

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-B, Q5-B, Q6-B, Q7-B, Q8-A, Q9-C, Q10-B

**Short Answer:**
- **S1.** `istio-injection=enabled`.
- **S2.** ztunnel.
- **S3.** Bookinfo.
- **S4.** ServiceEntry.
- **S5.** Kiali.

---

## 9. Further Resources

### Official Documentation
- Istio docs — https://istio.io/latest/docs/
- Getting started — https://istio.io/latest/docs/setup/getting-started/
- Traffic management — https://istio.io/latest/docs/concepts/traffic-management/
- Security — https://istio.io/latest/docs/concepts/security/
- Ambient mesh — https://istio.io/latest/docs/ambient/

### Learning
- Istio by Example — https://istiobyexample.dev/
- Bookinfo tutorial — https://istio.io/latest/docs/examples/bookinfo/
- Envoy docs — https://www.envoyproxy.io/docs

### Tools
- istioctl, Kiali, Grafana, Jaeger, Prometheus
- Kubernetes Gateway API
- Linkerd (alternative service mesh)

### Books
- *Istio in Action* — Christian Posta & Rinor Maloku (Manning)
- *Istio: Up and Running* — Lee Calcote & Zack Butcher (O'Reilly)
- *Mastering Service Mesh* — Anjali Khatri & Vikram Khatri

### Certifications
- Istio Certified Associate (ICA) — CNCF/Linux Foundation
- CKA/CKAD (Kubernetes foundation)

---

*End of Istio — Zero to Hero.*
