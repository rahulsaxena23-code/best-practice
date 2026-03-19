# EKS Pod Best Practices
### Solution Architect Recommendations — Spring Boot Microservices on AWS EKS

---

## Table of Contents

1. [Resource Management](#1-resource-management)
2. [Health Checks (Probes)](#2-health-checks-probes)
3. [Pod Disruption Budget](#3-pod-disruption-budget-pdb)
4. [Horizontal Pod Autoscaler](#4-horizontal-pod-autoscaler-hpa)
5. [Security — Never Run as Root](#5-security--never-run-as-root)
6. [Secret Management](#6-secret-management--never-hardcode)
7. [Network Policy — Zero Trust](#7-network-policy--zero-trust)
8. [Pod Anti-Affinity](#8-pod-anti-affinity--spread-across-nodes)
9. [Graceful Shutdown](#9-graceful-shutdown)
10. [Image Best Practices](#10-image-best-practices)
11. [Namespace Isolation](#11-namespace-isolation)
12. [Observability](#12-observability)
13. [Deployment Strategy](#13-deployment-strategy)
14. [Complete Checklist](#complete-best-practices-checklist)
15. [Priority Order](#priority-order)

---

## 1. Resource Management

**Always set requests and limits on every pod:**

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**Why it matters:**

```
No requests set → scheduler places pod on any node
                → node runs out of memory
                → pod gets OOMKilled

No limits set   → one bad pod consumes all node resources
                → all other pods on that node starve
```

**Golden rule:**

| Setting | Value |
|---|---|
| `requests` | What pod needs at normal load |
| `limits` | Maximum pod can ever consume |
| Starting point | limits = 2x requests |

---

## 2. Health Checks (Probes)

**Always define all three probes:**

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**What each probe does:**

| Probe | Purpose | Failure Action |
|---|---|---|
| Liveness | Is pod alive? | Restart pod |
| Readiness | Is pod ready for traffic? | Remove from load balancer |
| Startup | Has pod finished starting? | Give more time before liveness kicks in |

> ⚠️ **Never skip readiness probe** — without it, pod receives traffic before
> Spring Boot finishes loading → users see 503 errors.

---

## 3. Pod Disruption Budget (PDB)

**Ensure minimum pods always running during deployments:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-spring-boot-app
```

**Without PDB vs With PDB:**

```
Without PDB:
Rolling update starts
→ All old pods terminate at once
→ Zero pods running for 30 seconds
→ Users see downtime

With PDB:
Rolling update starts
→ Kubernetes ensures min 2 pods always running
→ Zero downtime rolling deployment
```

---

## 4. Horizontal Pod Autoscaler (HPA)

**Scale pods based on CPU and memory:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-spring-boot-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**Scaling rules explained:**

| Setting | Value | Reason |
|---|---|---|
| `minReplicas` | 2 | Never go below 2 — HA guarantee |
| `maxReplicas` | 20 | Cap to control costs |
| CPU target | 70% | Scale up before hitting 100% |
| Memory target | 80% | Prevent OOMKill before it happens |

---

## 5. Security — Never Run as Root

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

**Why this matters:**

```
Running as root → if pod is compromised
               → attacker has root on node
               → entire cluster at risk

Running as non-root → blast radius contained
                    → attacker limited to pod only
```

---

## 6. Secret Management — Never Hardcode

**❌ Wrong — hardcoded in manifest:**

```yaml
env:
  - name: DB_PASSWORD
    value: "mypassword123"
```

**❌ Wrong — still in plain YAML:**

```yaml
env:
  - name: DB_PASSWORD
    value: "${DB_PASSWORD}"
```

**✅ Correct — Secrets Manager + IRSA:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/myapp-role
```

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: aurora-secret
        key: password
```

**Credential flow:**

```
AWS Secrets Manager (source of truth)
        ↓
External Secrets Operator syncs to K8s
        ↓
K8s Secret (in-cluster, encrypted)
        ↓
Pod reads via secretKeyRef
        ↓
Never touches Git, never in plain YAML
```

---

## 7. Network Policy — Zero Trust

**Step 1 — Default deny everything:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Step 2 — Explicitly allow only what is needed:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-spring-boot-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: api-gateway
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: aurora-endpoint
      ports:
        - port: 5432
    - to:
        - namespaceSelector: {}
      ports:
        - port: 53    # allow DNS resolution
```

---

## 8. Pod Anti-Affinity — Spread Across Nodes

**Never put all pods on the same node:**

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - my-spring-boot-app
        topologyKey: kubernetes.io/hostname
```

**Without vs With anti-affinity:**

```
Without anti-affinity:
All 3 pods land on Node-1
Node-1 goes down → All 3 pods gone → Complete outage

With anti-affinity:
Pod-1 → Node-1
Pod-2 → Node-2
Pod-3 → Node-3
Node-1 goes down → only Pod-1 lost → Pod-2 and Pod-3 still serving
```

---

## 9. Graceful Shutdown

**Pod spec configuration:**

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 10"]
```

**Spring Boot configuration:**

```yaml
# application.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**Shutdown sequence:**

```
K8s sends SIGTERM
     ↓
preStop sleep 10s   ← load balancer stops sending traffic
     ↓
Spring starts graceful shutdown
     ↓
Finishes in-flight requests (up to 30s)
     ↓
Pod exits cleanly
     ↓
No dropped requests
```

---

## 10. Image Best Practices

```dockerfile
# Use specific version — never latest
FROM eclipse-temurin:17-jre-alpine

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy artifact
COPY --chown=appuser:appgroup app.jar /app/app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

**Image rules:**

| Rule | Detail |
|---|---|
| ✅ Pin exact version tags | Use `17.0.9` not `17` or `latest` |
| ✅ Use slim/alpine base | Smaller attack surface |
| ✅ Non-root user | Set in Dockerfile |
| ✅ Scan images | Use Trivy or ECR scanning |
| ✅ Store in ECR | Not DockerHub |
| ❌ Never use latest | Unpredictable in production |

---

## 11. Namespace Isolation

**Create namespaces per environment:**

```bash
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development
```

**Resource quotas per namespace:**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

**Namespace strategy:**

| Namespace | Quotas | PDB Required | Audit |
|---|---|---|---|
| `production` | Strict | ✅ Yes | ✅ Yes |
| `staging` | Moderate | ⚠️ Optional | ⚠️ Optional |
| `development` | Loose | ❌ No | ❌ No |

---

## 12. Observability

**Expose metrics from Spring Boot:**

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

**Pod annotations for Prometheus scraping:**

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/actuator/prometheus"
```

**Observability stack:**

```
Pod → /actuator/prometheus
  ↓
Prometheus scrapes every 15s
  ↓
Grafana dashboards
  ↓
Alerts → PagerDuty / Slack
```

**Critical alerts to configure:**

| Metric | Threshold | Action |
|---|---|---|
| JVM heap usage | > 85% | Scale up / investigate |
| HikariCP pool wait | > 1s | DB connection issue |
| HTTP error rate | > 1% | Investigate logs |
| Pod restarts | > 3 in 5 mins | Check liveness probe |
| CPU throttling | > 50% | Increase limits |

---

## 13. Deployment Strategy

**Rolling update with zero downtime:**

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # create 1 extra pod during update
      maxUnavailable: 0  # never remove pod before new one is ready
```

**How it works:**

```
maxUnavailable: 0 → zero downtime guaranteed
maxSurge: 1       → briefly run 4 pods during rollout
                  → new pod becomes ready
                  → old pod removed
                  → repeat until all pods updated
```

---

## Complete Best Practices Checklist

### Resources
- [ ] CPU requests set on all containers
- [ ] Memory requests set on all containers
- [ ] CPU limits set on all containers
- [ ] Memory limits set on all containers
- [ ] Limits are ~2x requests as starting point

### Availability
- [ ] Liveness probe configured
- [ ] Readiness probe configured
- [ ] Startup probe configured
- [ ] PodDisruptionBudget created
- [ ] minReplicas ≥ 2 in HPA
- [ ] maxReplicas set to control costs
- [ ] Pod anti-affinity across nodes configured
- [ ] terminationGracePeriodSeconds: 60 set
- [ ] preStop sleep hook configured
- [ ] Graceful shutdown enabled in Spring Boot

### Security
- [ ] runAsNonRoot: true
- [ ] readOnlyRootFilesystem: true
- [ ] allowPrivilegeEscalation: false
- [ ] All capabilities dropped
- [ ] No hardcoded secrets in YAML or code
- [ ] IRSA role attached to Service Account
- [ ] Credentials fetched from Secrets Manager only
- [ ] NetworkPolicy — default deny applied
- [ ] Only required ports explicitly allowed
- [ ] Image scanned for vulnerabilities (Trivy / ECR)
- [ ] No latest tag used anywhere
- [ ] Images stored in ECR

### Operations
- [ ] Namespaces created per environment
- [ ] Resource quotas set per namespace
- [ ] Prometheus metrics exposed
- [ ] Grafana dashboard created
- [ ] Alerts configured for key metrics
- [ ] Image tag pinned to exact version
- [ ] RollingUpdate strategy with maxUnavailable: 0
- [ ] maxSurge: 1 configured

---

## Priority Order

| Priority | Practice | Risk if Skipped |
|---|---|---|
| 🔴 P1 — Do Today | Resource limits | Node crash, OOMKill |
| 🔴 P1 — Do Today | Readiness probe | Traffic to unready pods, 503 errors |
| 🔴 P1 — Do Today | No hardcoded secrets | Credential leak, security breach |
| 🔴 P1 — Do Today | runAsNonRoot | Full node compromise if pod breached |
| 🟡 P2 — This Sprint | PodDisruptionBudget | Downtime during node drain or deploy |
| 🟡 P2 — This Sprint | HPA | Cannot handle traffic spikes |
| 🟡 P2 — This Sprint | Pod anti-affinity | Single node failure = full outage |
| 🟡 P2 — This Sprint | Graceful shutdown | Dropped requests on every deployment |
| 🟢 P3 — Next Sprint | NetworkPolicy | Lateral movement risk if pod breached |
| 🟢 P3 — Next Sprint | Full observability | Blind to production issues |
| 🟢 P3 — Next Sprint | Namespace quotas | Runaway resource consumption |

---

## Quick Reference — Full Pod Spec Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-spring-boot-app
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: my-spring-boot-app
  template:
    metadata:
      labels:
        app: my-spring-boot-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: myapp-sa
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - my-spring-boot-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: my-spring-boot-app
          image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0.5
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: aurora-secret
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: aurora-secret
                  key: password
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
```

---

*Document version 1.0 · Solution Architecture Team · March 2026*
