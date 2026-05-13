<p><a target="_blank" href="https://app.eraser.io/workspace/qlSLY5FH7pOvatMmg920" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

A practical guide to the platforms and patterns that run modern distributed systems in production.

---

# Part 1: Kubernetes
## 1.1 What It Is
**Kubernetes (K8s)** is an open-source container orchestration platform originally built by Google (based on their internal Borg system), now maintained by the CNCF. It automates the **deployment, scaling, networking, and lifecycle management** of containerized applications across a cluster of machines.

You declare _what_ you want ("3 replicas of this app, exposed on port 80, with 1 GB memory"), and Kubernetes figures out _how_ to make it happen — and keeps it that way (the **reconciliation loop**).

## 1.2 Architecture
```
┌──────────────────────────── Control Plane ────────────────────────────┐
│                                                                       │
│   ┌──────────────┐   ┌────────────────┐   ┌────────────────┐          │
│   │  API Server  │──►│   etcd         │   │  Scheduler     │          │
│   │  (REST)      │   │  (key-value    │   │  (assigns pods │          │
│   └──────┬───────┘   │   state store) │   │   to nodes)    │          │
│          │           └────────────────┘   └────────────────┘          │
│          ▼                                                            │
│   ┌──────────────────────────┐    ┌──────────────────────┐            │
│   │ Controller Manager       │    │ Cloud Controller Mgr │            │
│   │ (deployment, replicaset, │    │ (cloud-specific)     │            │
│   │  node, endpoint, ...)    │    └──────────────────────┘            │
│   └──────────────────────────┘                                        │
└───────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────── Worker Nodes ────────────────┐
│  ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │  Node 1  │ │  Node 2  │ │  Node 3  │      │
│  │          │ │          │ │          │      │
│  │ kubelet  │ │ kubelet  │ │ kubelet  │      │
│  │ kube-proxy│kube-proxy │ │kube-proxy│      │
│  │ runtime  │ │ runtime  │ │ runtime  │      │
│  │          │ │          │ │          │      │
│  │ [Pods]   │ │ [Pods]   │ │ [Pods]   │      │
│  └──────────┘ └──────────┘ └──────────┘      │
└──────────────────────────────────────────────┘
```
### Control Plane components
- **API Server** — the front door; everything talks to it via REST.
- **etcd** — distributed key-value store; the single source of truth for cluster state.
- **Scheduler** — decides which node a new pod should run on.
- **Controller Manager** — runs control loops (Deployment controller, ReplicaSet controller, Node controller, etc.) that drive actual state toward desired state.
- **Cloud Controller Manager** — integrates with cloud APIs (load balancers, volumes).
### Node components
- **kubelet** — agent that talks to the API server and runs containers on the node.
- **kube-proxy** — handles network rules so services can be reached.
- **Container runtime** — containerd, CRI-O (Docker is no longer used directly).
## 1.3 Core Objects
| Object | Purpose |
| ----- | ----- |
| **Pod** | Smallest deployable unit; one or more co-located containers sharing network/storage |
| **ReplicaSet** | Ensures N copies of a pod are running |
| **Deployment** | Manages ReplicaSets; supports rolling updates and rollbacks |
| **StatefulSet** | Like Deployment, but for stateful apps (stable IDs, ordered startup, persistent volumes) |
| **DaemonSet** | One pod per node (logging, monitoring agents) |
| **Job / CronJob** | Run-to-completion or scheduled tasks |
| **Service** | Stable virtual IP + DNS for a set of pods (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP/HTTPS routing rules to services (path/host based) |
| **ConfigMap / Secret** | Configuration and secrets injected into pods |
| **PersistentVolume / PVC** | Storage abstraction |
| **Namespace** | Logical isolation within a cluster |
| **HPA / VPA** | Horizontal/Vertical Pod Autoscaler |
## 1.4 Example Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  labels: { app: orders-api }
spec:
  replicas: 3
  selector:
    matchLabels: { app: orders-api }
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      containers:
      - name: orders-api
        image: myregistry/orders-api:1.4.0
        ports: [{ containerPort: 8080 }]
        resources:
          requests: { cpu: "200m", memory: "256Mi" }
          limits:   { cpu: "500m", memory: "512Mi" }
        livenessProbe:
          httpGet: { path: /healthz, port: 8080 }
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet: { path: /ready, port: 8080 }
        env:
        - name: DB_URL
          valueFrom: { secretKeyRef: { name: db-secret, key: url } }
---
apiVersion: v1
kind: Service
metadata: { name: orders-api }
spec:
  selector: { app: orders-api }
  ports: [{ port: 80, targetPort: 8080 }]
  type: ClusterIP
```
## 1.5 Why Kubernetes Matters
- **Declarative** — you describe the end state; K8s reconciles toward it.
- **Self-healing** — restarts failed containers, reschedules pods from dead nodes.
- **Auto-scaling** — HPA scales replicas on CPU/memory/custom metrics.
- **Service discovery & load balancing** built in.
- **Rolling deployments and rollbacks** built in.
- **Portable** — runs the same way on AWS, GCP, Azure, on-prem.
- **Massive ecosystem** — Helm, Operators, service meshes, GitOps tools (Argo CD, Flux).
## 1.6 Trade-offs
- Steep learning curve; complex to operate.
- Heavy for small teams — managed K8s (EKS, GKE, AKS) is almost always preferred.
- Misconfiguration risks (RBAC, network policies, resource limits).
- Not the right tool for everything — small apps may do better with serverless or PaaS.
---

# Part 2: Helm Charts
## 2.1 What It Is
**Helm** is the **package manager for Kubernetes** — like `apt`, `npm`, or `pip` but for K8s manifests. A **Helm chart** is a versioned, parameterized bundle of YAML templates.

Without Helm, deploying an app might mean managing dozens of YAML files across environments. With Helm, you get one templated package with values you can override per environment.

## 2.2 Chart Structure
```
my-app/
Chart.yaml          ← chart metadata (name, version, appVersion)
values.yaml         ← default configuration values
templates/
  deployment.yaml   ← Go templates with {{ .Values.* }}
  service.yaml
  ingress.yaml
  configmap.yaml
  _helpers.tpl      ← template helpers
charts/             ← dependencies (subcharts)
```
## 2.3 Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```
```yaml
replicaCount: 3
image:
  repository: myregistry/orders-api
  tag: 1.4.0
resources:
  requests: { cpu: 200m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```
## 2.4 Common Commands
```bash
helm install orders ./my-app -f values-prod.yaml     # install
helm upgrade orders ./my-app -f values-prod.yaml     # upgrade
helm rollback orders 2                               # rollback to revision 2
helm uninstall orders                                # remove
helm repo add bitnami https://charts.bitnami.com     # use third-party charts
helm install pg bitnami/postgresql                   # install one
```
## 2.5 Why Helm
- **Reusable, templated manifests** — one chart, many environments.
- **Versioning and rollbacks** — `helm rollback`  is one command.
- **Dependency management** — charts can depend on other charts.
- **Huge ecosystem** — most popular OSS tools ship Helm charts.
- **Standard packaging format** for Kubernetes.
## 2.6 Trade-offs
- YAML templating with Go templates is **ugly and error-prone**.
- Hard to debug rendering issues.
- Alternatives: **Kustomize** (overlay-based, no templating), **Carvel/ytt**, **Jsonnet**, **CDK8s**, or generating manifests from Pulumi/Terraform.
---

# Part 3: Service Mesh
## 3.1 What It Is
A **service mesh** is an infrastructure layer that handles **service-to-service communication** in a microservices architecture — transparently to the application code. It moves cross-cutting concerns (mTLS, retries, traffic shaping, observability) **out of every service and into a dedicated layer**.

Implementations: **Istio**, **Linkerd**, **Consul Connect**, **Cilium Service Mesh**, **AWS App Mesh**.

## 3.2 Architecture
A service mesh is usually composed of:

- **Data plane** — sidecar proxies (typically **Envoy**) deployed alongside every service pod. All traffic to/from the service flows through the proxy.
- **Control plane** — central component (Istio's `istiod` , Linkerd's controller) that distributes config to all proxies.
```
┌──────────────────────┐         ┌──────────────────────┐
│  Pod A               │         │  Pod B               │
│ ┌────────┐ ┌───────┐ │ mTLS    │ ┌───────┐ ┌────────┐ │
│ │  app   │►│ envoy │─┼─────────┼►│ envoy │►│  app   │ │
│ └────────┘ └───────┘ │         │ └───────┘ └────────┘ │
└──────────────────────┘         └──────────────────────┘
            ▲                                ▲
            └────── control plane ───────────┘
                  (policy, certs, config)
```
## 3.3 What It Provides
- **mTLS everywhere** — automatic mutual TLS between all services.
- **Traffic management** — canary, blue-green, weighted routing, retries, timeouts, circuit breaking.
- **Observability** — distributed tracing, metrics, access logs without code changes.
- **Authorization policies** — "service A can call service B but only on path /foo".
- **Fault injection** — inject latency or errors to test resilience.
- **Multi-cluster** — connect services across clusters/regions.
## 3.4 Trade-offs
- **Complexity** — Istio in particular is famously complex; Linkerd is simpler.
- **Latency** — every hop now goes through 2 proxies (small but real overhead).
- **Resource cost** — each pod has a sidecar consuming CPU/memory.
- **Operational burden** — another system to upgrade, debug, secure.
### When to use
- You have **dozens or hundreds of services** with complex traffic, security, and observability needs.
- You want **mTLS by default** without app changes.
- You need **advanced traffic management** for safe rollouts.
### When not to use
- A handful of services — the complexity isn't worth it.
- You can solve it with an **API gateway** + library-based resilience (Resilience4j).
---

# Part 4: Sidecar Pattern
## 4.1 What It Is
A **sidecar** is a helper container deployed in the **same pod** as the main application container. They share the pod's network and (optionally) storage. The sidecar augments the main app **without modifying it**.

```
┌────────────────────────── Pod ─────────────────────────┐
│  ┌──────────────┐    ┌──────────────────────────────┐  │
│  │   App        │    │   Sidecar                    │  │
│  │  container   │    │  (proxy / log forwarder /    │  │
│  │              │    │   secrets agent / etc.)      │  │
│  └──────────────┘    └──────────────────────────────┘  │
│       shares localhost network, optionally volumes     │
└────────────────────────────────────────────────────────┘
```
## 4.2 Common Uses
- **Service mesh proxies** (Envoy in Istio/Linkerd).
- **Log shippers** (Fluent Bit, Filebeat) reading app logs and shipping to ELK/Loki.
- **Metrics exporters** (Prometheus exporters that scrape the app and expose metrics).
- **Secrets agents** (HashiCorp Vault Agent injecting secrets to a shared volume).
- **Config sync** (e.g., a sidecar that pulls config from a Git repo).
- **TLS termination / mutual TLS**.
## 4.3 Pros
- **Language-agnostic** — works for any app, no SDK required.
- **Separation of concerns** — operational features stay out of the app code.
- **Reusable** — same sidecar across many services.
- **Independent lifecycle** in code, but tied to the pod at runtime.
## 4.4 Cons
- **Resource overhead** per pod (CPU, memory, startup time).
- **Complexity** — pod becomes multi-container; harder to debug.
- **Versioning** — sidecar must stay compatible with main app.
---

# Part 5: Ambassador Pattern
## 5.1 What It Is
A specialized sidecar that acts as a **proxy to the outside world** on behalf of the main application. The app talks to `localhost`; the ambassador handles all the messy details of communicating with external services.

```
┌─────────────────── Pod ───────────────────┐         ┌─────────────────┐
│  ┌──────────┐    ┌─────────────────────┐  │         │ External        │
│  │  App     │───►│  Ambassador         │──┼────────►│ Service (DB,    │
│  │ (calls   │    │  (TLS, retries,     │  │         │ Redis, API,     │
│  │ localhost)│   │   service discovery,│  │         │ third-party)    │
│  └──────────┘    │   circuit breaker)  │  │         └─────────────────┘
│                  └─────────────────────┘  │
└───────────────────────────────────────────┘
```
## 5.2 Examples
- **Database proxy** — Cloud SQL Proxy, AWS RDS Proxy as a sidecar; app connects to `localhost:5432` , proxy handles auth and TLS to the cloud DB.
- **Service discovery proxy** — app connects to `localhost` , ambassador resolves and load-balances to actual backends.
- **Resilience proxy** — Envoy as ambassador adding retries, timeouts, circuit breakers without changing the app.
- **Cloud SDK shim** — ambassador speaks the cloud-specific protocol; app only knows generic HTTP.
## 5.3 Why
- **Simplifies the app** — no need to know about TLS, retries, service discovery, secrets.
- **Easier testing** — point ambassador to a mock backend.
- **Polyglot-friendly** — same ambassador for Java, Go, Python services.
- **Migration help** — swap backends without app changes.
It's essentially a **special case of the sidecar pattern**, focused on **outbound** traffic to a single external dependency.

---

# Part 6: Adapter Pattern
## 6.1 What It Is
A sidecar that **standardizes / normalizes the interface** between the main app and the rest of the system. While ambassador handles outbound, adapter typically **transforms what the app exposes** so it looks "standard" from outside.

```
┌────────────── Pod ──────────────┐         ┌─────────────────┐
│  ┌────────┐    ┌─────────────┐  │         │ Prometheus,     │
│  │  App   │───►│  Adapter    │──┼────────►│ Logging system, │
│  │ (legacy │   │ (translates │  │         │ Monitoring tool │
│  │ format)│    │  to standard│  │         └─────────────────┘
│  └────────┘    │   format)   │  │
│                └─────────────┘  │
└─────────────────────────────────┘
```
## 6.2 Examples
- **Metrics adapter** — app exposes JVM/JMX metrics; adapter sidecar converts them to Prometheus format on `/metrics` .
- **Log adapter** — app writes plain text logs; adapter parses and emits structured JSON to stdout.
- **Health check adapter** — translate app-specific health endpoint into a Kubernetes-compatible `/healthz` .
- **Protocol bridging** — app speaks HTTP; adapter exposes it as gRPC (or vice versa).
## 6.3 Why
- **Standardize legacy or third-party apps** to fit your platform's conventions.
- **No app changes** required.
- **Centralizes platform integration code** in a reusable sidecar.
### Sidecar / Ambassador / Adapter — quick distinction
| Pattern | Direction | Purpose |
| ----- | ----- | ----- |
| **Sidecar** | Either | General helper container |
| **Ambassador** | Outbound | Proxy to external services |
| **Adapter** | Inbound to platform | Normalize what app exposes |
---

# Part 7: Init Containers
## 7.1 What It Is
**Init containers** are special containers in a pod that **run to completion before the main app containers start**. They run sequentially; if any fails, the pod is restarted from scratch.

```yaml
apiVersion: v1
kind: Pod
metadata: { name: web }
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  - name: run-migrations
    image: myregistry/migrations:1.0
    command: ['./migrate']
  containers:
  - name: web
    image: myregistry/web:2.3
    ports: [{ containerPort: 8080 }]
```
## 7.2 Common Uses
- **Wait for dependencies** — wait until DB / Kafka / config service is ready.
- **Run database migrations** before the app starts.
- **Clone Git repos** or fetch config into a shared volume.
- **Set file permissions** on mounted volumes.
- **Generate certificates / secrets** for the main container to use.
- **Pre-warm caches** or download large model files (ML).
## 7.3 Why Use Them
- Keep the main image **clean and single-purpose**.
- Use **different images** for setup vs runtime (e.g., a tools-heavy migration image vs a slim app image).
- **Stronger security** — setup container can have privileges the runtime container doesn't.
- **Sequential, ordered startup** logic is easier than baking it into the entrypoint script.
## 7.4 Note vs Sidecars
- **Init container** — runs **before** main container, exits, then app starts.
- **Sidecar** — runs **alongside** the main container for the pod's lifetime.
(Kubernetes 1.28+ also added "native sidecar containers" implemented as restartable init containers — bridging the two concepts.)

---

# Part 8: Deployment Strategies
When releasing a new version, you need to balance **risk, downtime, infrastructure cost, and rollback speed**. The main strategies:

| Strategy | Downtime | Infra cost | Rollback speed | Risk |
| ----- | ----- | ----- | ----- | ----- |
| Recreate | Yes | Low | Slow | High |
| Rolling | None | Low | Medium | Medium |
| Blue-Green | None | 2× | Instant | Low |
| Canary | None | Low (+ small extra) | Fast | Lowest |
---

# Part 9: Blue-Green Deployment
## 9.1 What It Is
Run **two identical production environments** — "Blue" (current) and "Green" (new). Deploy the new version to Green; once verified, **flip the load balancer** to send all traffic to Green. Blue stays as a hot standby for instant rollback.

```
┌──────────────────────────────┐
   │     Load Balancer            │
   │      (router/ingress)        │
   └─────────────┬────────────────┘
                 │
        ┌────────┴────────┐
        ▼ 100%            ▼ 0%
  ┌───────────┐     ┌───────────┐
  │   BLUE    │     │   GREEN   │
  │ (v1, live)│     │ (v2, new) │
  └───────────┘     └───────────┘
After cutover:
        ▼ 0%              ▼ 100%
  ┌───────────┐     ┌───────────┐
  │   BLUE    │     │   GREEN   │
  │ (standby) │     │  (live)   │
  └───────────┘     └───────────┘
```
## 9.2 Pros
- **Instant rollback** — flip the LB back to Blue.
- **Zero downtime** — switch is atomic.
- **Easy to test** new version in production setup before going live.
- **Predictable** — no in-place upgrades, no partial states.
## 9.3 Cons
- **Doubles infrastructure** during deployment.
- **Database schema changes are tricky** — both versions may need to coexist (use **expand/contract migrations**).
- **All-or-nothing** — if the new version is bad, every user hits the bug at the same moment until you flip back.
- **Long-lived sessions / WebSockets** are awkward to drain.
## 9.4 When to Use
- Critical apps where instant rollback matters more than infra cost.
- Stateless services with backwards-compatible DB changes.
- Releases that need full smoke-testing in production environment.
---

# Part 10: Canary Deployment
## 10.1 What It Is
Roll out the new version to a **small subset of users (or traffic)** first — the "canary." Monitor metrics (errors, latency, business KPIs). If healthy, gradually increase traffic; if not, roll back before most users are affected.

```
┌──────────────────────┐
   │    Load Balancer     │
   └─────────┬────────────┘
             │
     ┌───────┼─────────┐
     ▼ 95%             ▼ 5%
  ┌───────┐        ┌─────────┐
  │ v1    │        │ v2      │
  │ (10   │        │ (canary │
  │  pods)│        │ 1 pod)  │
  └───────┘        └─────────┘
Gradually: 5% → 25% → 50% → 100%
```
## 10.2 How It's Done
- **K8s native** — multiple deployments + service routing (manual ratio control).
- **Service mesh (Istio, Linkerd)** — declarative `VirtualService`  with weighted routing.
- **Ingress controllers** (NGINX, Traefik) with header/weight-based routing.
- **Progressive delivery tools** — **Argo Rollouts**, **Flagger** — automate the canary with metrics-based gating.
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: orders }
spec:
  hosts: [orders]
  http:
  - route:
    - destination: { host: orders, subset: v1 }
      weight: 90
    - destination: { host: orders, subset: v2 }
      weight: 10
```
## 10.3 Pros
- **Lowest blast radius** — bad release affects only a small % of users.
- **Real production validation** — catch issues that staging missed.
- **Automated promotion** with metrics gates (error rate, p99 latency).
- **Cheap** — only a small extra deployment.
## 10.4 Cons
- **More complex** to set up (routing, monitoring, automation).
- **Stateful interactions** can be tricky — a user hitting v2 once and v1 next call may see inconsistencies (sticky sessions help).
- **DB schema** must be backward compatible.
- Slower than a flip-the-switch deploy.
## 10.5 Variants
- **User-segment canary** — route by user ID, geography, header (internal users / beta users first).
- **Shadow traffic / dark launch** — duplicate real traffic to new version but discard responses; useful for performance testing.
---

# Part 11: Rolling Deployment
## 11.1 What It Is
Replace pods of the old version with the new version **incrementally**, a few at a time, until all are upgraded. **The default Kubernetes deployment strategy.**

```
Initial:  v1 v1 v1 v1 v1
Step 1:   v1 v1 v1 v1 v2     (one new added, then one old removed)
Step 2:   v1 v1 v1 v2 v2
Step 3:   v1 v1 v2 v2 v2
Step 4:   v1 v2 v2 v2 v2
Done:     v2 v2 v2 v2 v2
```
Controlled by:

- `**maxSurge**`  — how many extra pods (above desired) can exist during rollout.
- `**maxUnavailable**`  — how many pods can be down during rollout.
```yaml
spec:
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```
## 11.2 Pros
- **No extra infrastructure** required.
- **Zero downtime** if `maxUnavailable: 0`  and readiness probes are good.
- **Simple, built-in** to K8s — `kubectl rollout undo`  to roll back.
- **Gradual exposure** — somewhere between blue-green and canary in safety.
## 11.3 Cons
- **Both versions run simultaneously** during rollout — must be backwards compatible.
- **Rollback is not instant** — it's another rolling update in reverse.
- **No traffic shaping** — every user can hit either version randomly during the rollout (no 5%/95% control).
- **Slow for large fleets** — surge/unavailable settings throttle the pace.
- **Bad release affects everyone progressively** — unlike canary, there's no "stop at 5%."
## 11.4 When to Use
- The **default** for most stateless services.
- When you don't need fine-grained traffic control or instant rollback.
- When DB and API contracts are backward compatible.
---

# Part 12: Choosing a Deployment Strategy
| Situation | Best Strategy |
| ----- | ----- |
| Default for stateless service | **Rolling** |
| Critical service, instant rollback required | **Blue-Green** |
| Risky change, large user base | **Canary** |
| Internal tool, downtime acceptable | **Recreate** |
| Need to test in production safely | <p>**Canary**</p><p> or </p><p>**shadow traffic**</p> |
| Need to coordinate complex DB migration | <p>**Blue-Green**</p><p> (with expand/contract)</p> |
| Have a service mesh / progressive delivery tooling | <p>**Canary**</p><p> (automated)</p> |
### Universal rules
- **Always use readiness probes** — without them, K8s sends traffic to pods that aren't ready, and any rolling deploy looks like an outage.
- **Always make schema changes backward compatible** (expand/contract: add column → deploy app → migrate data → remove old column).
- **Always have a rollback plan** — automated if possible.
- **Always monitor SLIs during deploys** — error rate, latency, saturation.
---

# Part 13: How These Patterns Fit Together
A modern production deployment stack often looks like:

```
GitOps repo (Argo CD / Flux)
         │
         ▼
Helm chart  ─►  Kubernetes manifests
                       │
                       ▼
┌────────────────── Pod ──────────────────┐
│ Init Container: wait-for-db, migrate    │
│ Main Container: app                     │
│ Sidecar: Envoy proxy (service mesh)     │
│ Sidecar: log shipper, metrics exporter  │
│ Ambassador: cloud-sql-proxy             │
└─────────────────────────────────────────┘
         │
         ▼
Service mesh (Istio/Linkerd):
  - mTLS, retries, circuit breaking, tracing
  - Canary routing for new releases
         │
         ▼
Argo Rollouts / Flagger:
  - Progressive delivery (canary, blue-green)
  - Automatic metric-based promotion or rollback
```
---

# TL;DR
- **Kubernetes** — declarative orchestration platform; the de-facto standard for running containers at scale.
- **Helm** — package manager for K8s; templated, versioned, parameterized YAML bundles.
- **Service mesh** — moves cross-cutting concerns (mTLS, retries, observability, traffic shaping) out of app code into sidecar proxies.
- **Sidecar / Ambassador / Adapter** — variations of "helper container in the same pod": general helper, outbound proxy, interface normalizer.
- **Init containers** — run-to-completion setup before the main app starts (migrations, dependency waits, fetching config).
- **Blue-Green** — two full environments, instant cutover, instant rollback, 2× cost.
- **Canary** — gradual rollout to a small slice of traffic first; lowest blast radius, best with a mesh + automation.
- **Rolling** — K8s default; replace pods incrementally with controlled surge/unavailable; cheap, safe, but no traffic shaping.
- Pick the deployment strategy based on **risk tolerance, infra budget, rollback speed, and traffic-shaping needs** — and combine these patterns for a robust production platform.




<!--- Eraser file: https://app.eraser.io/workspace/qlSLY5FH7pOvatMmg920 --->