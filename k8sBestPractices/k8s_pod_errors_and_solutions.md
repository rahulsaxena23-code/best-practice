# Kubernetes Pod Errors & Solutions

> A comprehensive reference covering the most common Kubernetes pod failure states, their root causes, diagnostic commands, and step-by-step solutions.

---

## Table of Contents

1. [CrashLoopBackOff](#1-crashloopbackoff)
2. [ImagePullBackOff / ErrImagePull](#2-imagepullbackoff--errimagepull)
3. [Pending (Unschedulable)](#3-pending-unschedulable)
4. [OOMKilled (Out of Memory)](#4-oomkilled-out-of-memory)
5. [Error / Init:Error](#5-error--initerror)
6. [Init:CrashLoopBackOff](#6-initcrashloopbackoff)
7. [CreateContainerConfigError](#7-createcontainerconfigerror)
8. [RunContainerError / CreateContainerError](#8-runcontainererror--createcontainererror)
9. [PostStartHookError](#9-poststarthookerror)
10. [Evicted](#10-evicted)
11. [Terminating (Stuck)](#11-terminating-stuck)
12. [ContainerCannotRun](#12-containercannotrun)
13. [DeadlineExceeded / StartError](#13-deadlineexceeded--starterror)
14. [InvalidImageName](#14-invalidimagename)
15. [Liveness / Readiness Probe Failures](#15-liveness--readiness-probe-failures)
16. [Node Not Ready / NodeLost](#16-node-not-ready--nodelost)
17. [Insufficient CPU / Memory Resources](#17-insufficient-cpu--memory-resources)
18. [PodUnschedulable — Taints & Tolerations](#18-podunschedulable--taints--tolerations)
19. [Persistent Volume Claim (PVC) Pending](#19-persistent-volume-claim-pvc-pending)
20. [ServiceAccount / RBAC Errors](#20-serviceaccount--rbac-errors)
21. [Network Policy Blocking Traffic](#21-network-policy-blocking-traffic)
22. [ConfigMap / Secret Not Found](#22-configmap--secret-not-found)
23. [DNS Resolution Failures](#23-dns-resolution-failures)
24. [Pod Security Admission / PSP Violations](#24-pod-security-admission--psp-violations)

---

## Quick Diagnostic Commands

```bash
# Describe pod for events and conditions
kubectl describe pod <pod-name> -n <namespace>

# Tail logs from a running or crashed container
kubectl logs <pod-name> -n <namespace> --previous   # last crashed instance
kubectl logs <pod-name> -n <namespace> -f           # follow live

# Get all pod statuses in a namespace
kubectl get pods -n <namespace> -o wide

# Exec into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Get events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check node conditions
kubectl describe node <node-name>
```

---

## 1. CrashLoopBackOff

### What It Means
The container starts, crashes, Kubernetes restarts it, it crashes again — repeatedly. The backoff time increases exponentially (10 s → 20 s → 40 s … up to 5 min).

### Root Causes
- Application exits with a non-zero code (unhandled exception, missing env var, bad config)
- Missing dependency (DB not ready, file not found)
- Wrong entrypoint / command in the manifest
- Memory limit too low (OOMKilled triggers a restart loop)

### Diagnosis
```bash
kubectl describe pod <pod-name> -n <ns>
# Look for: Last State, Exit Code, Reason

kubectl logs <pod-name> -n <ns> --previous
# Read the actual application error
```

### Solution

**Step 1 — Read the logs:**
```bash
kubectl logs <pod-name> --previous
```

**Step 2 — Check exit code:**
```bash
kubectl describe pod <pod-name> | grep "Exit Code"
# Exit Code 1  = app error
# Exit Code 137 = OOMKilled
# Exit Code 139 = segfault
```

**Step 3 — Fix the root cause:**
```yaml
# Example: supply missing environment variable
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: url
```

**Step 4 — Add a sleep for debugging (temporary):**
```yaml
command: ["/bin/sh", "-c", "sleep 3600"]
```

---

## 2. ImagePullBackOff / ErrImagePull

### What It Means
Kubernetes cannot pull the container image from the registry.

### Root Causes
- Image name or tag typo
- Image does not exist in the registry
- Private registry with no pull secret
- Registry is unreachable from the cluster network
- Rate limit hit (Docker Hub)

### Diagnosis
```bash
kubectl describe pod <pod-name> | grep -A5 "Events"
# Look for: Failed to pull image, 401 Unauthorized, 404 Not Found
```

### Solution

**Fix image name/tag:**
```yaml
containers:
  - name: app
    image: myrepo/myapp:v1.2.3   # ensure tag exists
```

**Create and reference an image pull secret (private registry):**
```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n <namespace>
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

**Use `imagePullPolicy: IfNotPresent`** to avoid redundant pulls in dev:
```yaml
imagePullPolicy: IfNotPresent
```

---

## 3. Pending (Unschedulable)

### What It Means
The pod has been accepted by the API server but no node has been selected to run it.

### Root Causes
- Insufficient CPU/memory on all nodes
- No node matches `nodeSelector` or affinity rules
- Taint on every node without matching toleration
- PVC not bound
- Namespace resource quota exhausted

### Diagnosis
```bash
kubectl describe pod <pod-name> | grep -A10 "Events"
# Look for: 0/3 nodes available, insufficient cpu, didn't match node selector
```

### Solution

**Check node capacity:**
```bash
kubectl describe nodes | grep -A5 "Allocated resources"
```

**Relax resource requests:**
```yaml
resources:
  requests:
    cpu: "100m"       # reduce from higher value
    memory: "128Mi"
```

**Check and fix nodeSelector:**
```bash
kubectl get nodes --show-labels
```
```yaml
nodeSelector:
  kubernetes.io/os: linux   # must match existing label
```

**Check quota:**
```bash
kubectl describe resourcequota -n <namespace>
```

---

## 4. OOMKilled (Out of Memory)

### What It Means
The container used more memory than its `limits.memory` allowed and was killed by the Linux OOM killer.

### Root Causes
- Memory limit set too low
- Memory leak in the application
- Sudden traffic spike

### Diagnosis
```bash
kubectl describe pod <pod-name> | grep -A3 "Last State"
# Reason: OOMKilled

kubectl top pod <pod-name> -n <ns>
```

### Solution

**Increase memory limit:**
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"    # increase this
```

**Profile the app** — use heap dumps, profiling tools, or language-specific memory analyzers to find leaks.

**Use VPA (Vertical Pod Autoscaler)** for automatic right-sizing:
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
```

---

## 5. Error / Init:Error

### What It Means
An init container exited with a non-zero code, preventing the main container from starting.

### Root Causes
- Init container command fails (migration script, DB check)
- Missing dependency (config, secret, service not reachable)

### Diagnosis
```bash
kubectl describe pod <pod-name>
# Check init container status

kubectl logs <pod-name> -c <init-container-name>
```

### Solution

**Check init container logs:**
```bash
kubectl logs <pod-name> -c init-db-check --previous
```

**Fix the init container script or wait logic:**
```yaml
initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c',
      'until nc -z db-service 5432; do echo waiting; sleep 2; done']
```

---

## 6. Init:CrashLoopBackOff

### What It Means
An init container is crashing repeatedly, same as `CrashLoopBackOff` but for the init phase.

### Solution
Same approach as [CrashLoopBackOff](#1-crashloopbackoff) but target the init container:

```bash
kubectl logs <pod-name> -c <init-container-name> --previous
```

---

## 7. CreateContainerConfigError

### What It Means
Kubernetes cannot create the container configuration — usually a missing Secret or ConfigMap that is referenced in the pod spec.

### Root Causes
- Referenced Secret or ConfigMap does not exist
- Key inside Secret/ConfigMap does not exist
- Wrong namespace

### Diagnosis
```bash
kubectl describe pod <pod-name>
# Events: secret "my-secret" not found
```

### Solution

**Create the missing secret:**
```bash
kubectl create secret generic my-secret \
  --from-literal=password=mysecretpassword \
  -n <namespace>
```

**Create the missing ConfigMap:**
```bash
kubectl create configmap my-config \
  --from-literal=APP_ENV=production \
  -n <namespace>
```

**Verify all references exist:**
```bash
kubectl get secrets -n <ns>
kubectl get configmaps -n <ns>
```

---

## 8. RunContainerError / CreateContainerError

### What It Means
The container runtime (containerd/docker) failed to start the container.

### Root Causes
- Invalid command/entrypoint
- Volume mount path conflict
- Security context violation (read-only filesystem)
- Missing host path volume

### Diagnosis
```bash
kubectl describe pod <pod-name>
# Look for specific error in Events section
```

### Solution

**Fix entrypoint:**
```yaml
containers:
  - name: app
    image: myapp:latest
    command: ["/app/start.sh"]   # ensure file exists and is executable
    args: ["--config", "/etc/config.yaml"]
```

**Fix volume conflicts:**
```yaml
volumeMounts:
  - name: data
    mountPath: /data          # must not conflict with container's built-in paths
```

---

## 9. PostStartHookError

### What It Means
The `postStart` lifecycle hook command failed.

### Root Causes
- Command in `postStart` exits with non-zero code
- Command takes too long or blocks

### Solution

```yaml
lifecycle:
  postStart:
    exec:
      command: ["/bin/sh", "-c", "echo started > /tmp/ready || true"]
      # Add '|| true' to prevent hook failure from killing pod (use with caution)
```

Better approach — move startup logic to the container's own entrypoint or use a readiness probe instead.

---

## 10. Evicted

### What It Means
The kubelet evicted the pod to reclaim resources on the node.

### Root Causes
- Node disk pressure (ephemeral storage exceeded)
- Node memory pressure
- Pod's ephemeral storage request exceeded

### Diagnosis
```bash
kubectl describe pod <pod-name> | grep -A5 "Status\|Message"
# Message: The node was low on resource: ephemeral-storage

kubectl describe node <node-name> | grep -A5 "Conditions"
```

### Solution

**Set ephemeral storage limits:**
```yaml
resources:
  requests:
    ephemeral-storage: "100Mi"
  limits:
    ephemeral-storage: "500Mi"
```

**Clear disk on the node:**
```bash
# On the affected node
docker system prune -f         # if using Docker
crictl rmi --prune             # if using containerd

# Check what's consuming disk
du -sh /var/log/pods/*
```

**Use PVCs for persistent storage** instead of writing to ephemeral container filesystem.

---

## 11. Terminating (Stuck)

### What It Means
A pod is stuck in `Terminating` state and won't fully delete.

### Root Causes
- Finalizers not cleared
- Volume detach stuck
- Node is offline

### Diagnosis
```bash
kubectl describe pod <pod-name> | grep -i finalizer
kubectl get pod <pod-name> -o jsonpath='{.metadata.finalizers}'
```

### Solution

**Force delete (use with caution):**
```bash
kubectl delete pod <pod-name> --grace-period=0 --force -n <namespace>
```

**Remove finalizers:**
```bash
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge -n <namespace>
```

**If node is offline**, drain and cordon the node, then force-delete stuck pods.

---

## 12. ContainerCannotRun

### What It Means
The container runtime cannot start the container process (reported in `docker/containerd` events).

### Root Causes
- Binary/entrypoint not found inside the image
- Architecture mismatch (ARM image on x86 node)
- Permission denied on the executable

### Solution

**Verify the entrypoint exists in the image:**
```bash
docker run --rm --entrypoint /bin/sh myimage:tag -c "ls -la /app/start.sh"
```

**Fix file permissions in Dockerfile:**
```dockerfile
RUN chmod +x /app/start.sh
```

**Check architecture:**
```bash
docker inspect myimage:tag | grep Architecture
kubectl describe node <node> | grep "System Info" -A10
```

---

## 13. DeadlineExceeded / StartError

### What It Means
Pod failed to start within the `activeDeadlineSeconds` window, or the container start timed out.

### Solution

**Increase or remove active deadline:**
```yaml
spec:
  activeDeadlineSeconds: 600    # increase or remove this field
```

**Optimize image startup** — reduce image size, pre-pull images with a DaemonSet, or use init containers to pre-warm dependencies.

---

## 14. InvalidImageName

### What It Means
The image name in the manifest is syntactically invalid.

### Root Causes
- Uppercase letters in image name
- Missing tag and digest
- Invalid characters

### Solution
```yaml
# Wrong
image: MyApp/MyImage:Latest

# Correct
image: myapp/myimage:latest
```

Validate image names follow the format: `[registry/][namespace/]image[:tag][@digest]`

---

## 15. Liveness / Readiness Probe Failures

### What It Means
- **Liveness probe** fails → container is killed and restarted
- **Readiness probe** fails → pod is removed from Service endpoints (no traffic)

### Root Causes
- App takes longer to start than `initialDelaySeconds`
- Probe endpoint is wrong
- Probe thresholds are too strict

### Diagnosis
```bash
kubectl describe pod <pod-name>
# Events: Liveness probe failed: connection refused
```

### Solution

**Tune probe settings:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30      # give app time to start
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

startupProbe:                  # use startupProbe for slow-starting apps
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

---

## 16. Node Not Ready / NodeLost

### What It Means
The node running the pod is in `NotReady` state, causing pods to be in `Unknown` status.

### Diagnosis
```bash
kubectl get nodes
kubectl describe node <node-name>
# Conditions: MemoryPressure, DiskPressure, NetworkUnavailable, Ready
```

### Solution

```bash
# SSH to the node and check kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Restart kubelet
systemctl restart kubelet

# Drain the node for maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl cordon <node-name>
```

---

## 17. Insufficient CPU / Memory Resources

### What It Means
No node has enough allocatable CPU or memory to satisfy the pod's `requests`.

### Diagnosis
```bash
kubectl describe pod <pod-name>
# 0/3 nodes are available: 3 Insufficient cpu
```

### Solution

**Check actual node allocations:**
```bash
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl top nodes
```

**Horizontal scaling (add nodes)** or **reduce requests:**
```yaml
resources:
  requests:
    cpu: "50m"          # minimum needed, not maximum
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**Use Cluster Autoscaler** to automatically add nodes when resources are insufficient.

---

## 18. PodUnschedulable — Taints & Tolerations

### What It Means
Nodes have taints that repel the pod because no matching toleration is defined.

### Diagnosis
```bash
kubectl describe node <node> | grep Taint
# Taints: node-role.kubernetes.io/master:NoSchedule
```

### Solution

**Add toleration to pod spec:**
```yaml
tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

**Or remove taint from node (if appropriate):**
```bash
kubectl taint nodes <node-name> node-role.kubernetes.io/master:NoSchedule-
```

---

## 19. Persistent Volume Claim (PVC) Pending

### What It Means
The pod is stuck in `Pending` because its PVC is not bound to a Persistent Volume.

### Root Causes
- No PV matches the PVC's storage class, size, or access mode
- StorageClass provisioner not installed/configured
- Cloud provider volume quota exhausted

### Diagnosis
```bash
kubectl describe pvc <pvc-name> -n <ns>
# Events: no persistent volumes available for this claim
```

### Solution

**Check available PVs:**
```bash
kubectl get pv
kubectl get storageclass
```

**Create a matching PV manually (static provisioning):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/my-pv
```

**Use dynamic provisioning with a StorageClass:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

---

## 20. ServiceAccount / RBAC Errors

### What It Means
Pod fails to start or crashes because it lacks permissions to call the Kubernetes API.

### Root Causes
- ServiceAccount does not exist
- Missing Role/ClusterRole binding
- Token projection errors

### Diagnosis
```bash
kubectl logs <pod-name> | grep "forbidden\|unauthorized\|403\|401"
kubectl describe pod <pod-name> | grep "serviceAccount"
```

### Solution

**Create ServiceAccount:**
```bash
kubectl create serviceaccount my-app-sa -n <namespace>
```

**Bind a Role:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: my-ns
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: my-ns
roleRef:
  kind: Role
  name: my-app-role
  apiGroup: rbac.authorization.k8s.io
```

**Reference in pod:**
```yaml
spec:
  serviceAccountName: my-app-sa
```

---

## 21. Network Policy Blocking Traffic

### What It Means
A `NetworkPolicy` is dropping packets, causing the pod to fail health checks or be unable to reach dependencies.

### Diagnosis
```bash
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>

# Test connectivity from inside the pod
kubectl exec -it <pod-name> -- curl -v http://other-service:8080
```

### Solution

**Allow required ingress/egress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 5432
```

---

## 22. ConfigMap / Secret Not Found

### What It Means
Pod spec references a ConfigMap or Secret that doesn't exist in the namespace, causing `CreateContainerConfigError`.

### Solution

**List and compare:**
```bash
kubectl get configmap -n <ns>
kubectl get secret -n <ns>
```

**Create from file or literal:**
```bash
kubectl create configmap app-config \
  --from-file=config.yaml \
  -n <namespace>

kubectl create secret generic app-secret \
  --from-literal=API_KEY=abc123 \
  -n <namespace>
```

**Mount correctly:**
```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

---

## 23. DNS Resolution Failures

### What It Means
Pod cannot resolve service names, causing connection refused or timeout errors.

### Root Causes
- CoreDNS pods are down
- Incorrect `dnsPolicy`
- Pod in host network mode

### Diagnosis
```bash
# Test DNS from inside the pod
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -- cat /etc/resolv.conf

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Solution

**Restart CoreDNS if unhealthy:**
```bash
kubectl rollout restart deployment coredns -n kube-system
```

**Set correct dnsPolicy:**
```yaml
spec:
  dnsPolicy: ClusterFirst        # default for normal pods
  # dnsPolicy: ClusterFirstWithHostNet  # use when hostNetwork: true
```

**Custom DNS config:**
```yaml
spec:
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - my-namespace.svc.cluster.local
```

---

## 24. Pod Security Admission / PSP Violations

### What It Means
The pod spec violates the namespace's Pod Security Standards (`restricted`, `baseline`, `privileged`) and is rejected or mutated.

### Root Causes
- Container running as root
- Privileged container not allowed
- HostPath volumes not permitted
- seccompProfile not set

### Diagnosis
```bash
kubectl describe pod <pod-name>
# Warning: would violate PodSecurity "restricted:latest"

kubectl get ns <namespace> -o yaml | grep pod-security
```

### Solution

**Run as non-root:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

**Adjust namespace policy label if appropriate:**
```bash
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest
```

---

## Summary Table

| Error | Primary Cause | Quick Fix |
|---|---|---|
| CrashLoopBackOff | App crash / bad config | Check `--previous` logs |
| ImagePullBackOff | Bad image name / no pull secret | Fix tag, add `imagePullSecrets` |
| Pending | No resources / scheduling rules | Check node capacity, relax constraints |
| OOMKilled | Memory limit too low | Increase `limits.memory` |
| Init:Error | Init container fails | Fix init container command |
| CreateContainerConfigError | Missing Secret/ConfigMap | Create missing resource |
| RunContainerError | Bad entrypoint / volume | Fix command, mounts |
| Evicted | Disk/memory pressure | Add ephemeral-storage limits |
| Terminating (stuck) | Finalizers / offline node | Force delete, remove finalizers |
| Probe failure | Probe too strict / slow start | Tune `initialDelaySeconds` |
| Node NotReady | Node/kubelet issue | Restart kubelet, drain node |
| PVC Pending | No matching PV | Add PV or StorageClass |
| DNS failure | CoreDNS down | Restart CoreDNS pods |
| NetworkPolicy blocking | Missing allow rule | Add ingress/egress rules |
| PSP/PSA violation | Security context too permissive | Set `runAsNonRoot`, drop capabilities |

---

> **Tip:** Always start with `kubectl describe pod <name>` and `kubectl logs <name> --previous`. These two commands reveal the root cause in 80% of cases.
