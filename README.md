# Infrastructure DevOps Intern Assessment

This repository documents my end-to-end setup, deployment, debugging, and reasoning for the **Infrastructure DevOps Intern Assessment**. The focus of this exercise was ownership of production-like infrastructure, clear decision-making, debugging, security awareness, and cost consciousness.

---

## Environment

* **Cloud Provider:** Hetzner Cloud
* **OS:** Ubuntu 24.04
* **Cluster Type:** Single-node Kubernetes (k3s)
* **Access:** SSH with full sudo

---

## Part 1: VM & Kubernetes Cluster Setup

### 1. Docker Installation

Docker Engine was installed and configured so the user can run Docker commands without sudo.

**Validation Commands:**

```bash
docker version
docker ps
```

Docker runs as the container runtime for Kubernetes workloads and local container testing.

---

### 2. k3s Installation

A single-node **k3s** cluster was installed and configured. `kubectl` access was enabled for the user.

**Validation Commands:**

```bash
kubectl get nodes
kubectl get pods -A
```

**Why k3s for an early-stage startup:**

* Lightweight and fast to bootstrap
* Low operational overhead
* Minimal resource usage
* Ideal for MVPs and early production workloads

**Limitations:**

* Single-node setup is not highly available
* Limited fault tolerance
* Not suitable for large-scale production without expansion

---

## Part 2: Application Deployment (Helm)

### Helm Setup

```bash
helm repo add open-webui https://helm.openwebui.com/
helm repo update
```

### Deployment

```bash
kubectl create namespace openwebui
helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP \
  --dry-run

helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP
```

**Validation:**

```bash
kubectl get all -n openwebui
```

---

## Part 3: Authentication Configuration (OIDC)

### OIDC Values File (`values-oidc.yaml`)

```yaml
oidc:
  clientId: "test"
  clientSecret: ""
  issuer: "https://<YOUR_DOMAIN>/auth/realms/hyperplane/.well-known/openid-configuration"
  scopes:
    - openid
    - profile
    - email
```

### Apply Configuration

```bash
helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --values values-oidc.yaml
```

---

## Part 4: Debugging (Intentional Failure)

### Observed Behavior

After enabling OIDC, the application pods remained in a **Running** state and did not crash. However, authentication failed at runtime when users attempted to log in.

### Root Cause

Open WebUI does **not validate the OIDC issuer during container startup**. Validation occurs only during authentication requests. Because the issuer URL was invalid/unreachable, authentication failed even though the pod was healthy.

### Diagnosis Steps

* Verified pod status using `kubectl get pods`
* Checked logs using `kubectl logs`
* Confirmed no CrashLoopBackOff or startup errors
* Identified this as a **functional failure**, not an infrastructure failure

### Fix

* Configure a valid OIDC issuer
* Store `clientSecret` securely using Kubernetes Secrets
* Trust the OIDC provider’s CA certificate
* Restart pods to apply the corrected configuration

This reflects real-world production behavior where services may appear healthy but fail functionally.

---

## Part 5: Ownership Questions

### 1. Production Readiness

**Top 5 Risks:**

1. Single Node Failure
2. No High Availability
3. Limited Security Configuration
4. No Proper Monitoring and Alerting
5. No Backup Strategy

**First 2 Fixes:**

1. Add High Availability and Scaling 

2.Improve Security and Observability 

---

### 2. Failure Scenario (10x Traffic Spike at 2 AM)

**What breaks first:**

* CPU and memory exhaustion on the single node

**Recovery Steps:**

* Restart affected pods
* Scale workloads if possible
* Reduce load temporarily

**Next-Day Improvements:**

* Add horizontal pod autoscaling
* Introduce a multi-node cluster

---

### 3. Security & Secrets

* Secrets managed using Kubernetes Secrets or external secret managers
* Never store secrets, tokens, or private keys in Git
* Rotate credentials, tokens, and certificates regularly

---

### 4. Backups & Recovery

* Backup application data, Redis data, and configuration
* Daily backups with retention policy
* Regular restore testing in a staging environment

---

### 5. Cost Ownership (Hetzner)

* Use right-sized VMs
* Monitor resource usage
* Avoid over-provisioning early
* Move away from k3s when high availability, scaling, or compliance is required

---


## Validation Output

```bash
kubectl get nodes

root@raginiagrawal1230:~# kubectl get nodes
NAME                STATUS   ROLES           AGE   VERSION
raginiagrawal1230   Ready    control-plane   10h   v1.34.3+k3s1


kubectl get all -n openwebui

root@raginiagrawal1230:~# kubectl get all -n openwebui
NAME                                       READY   STATUS    RESTARTS   AGE
pod/open-webui-0                           1/1     Running   0          10h
pod/open-webui-ollama-5d99896fd7-9n4vx     1/1     Running   0          10h
pod/open-webui-pipelines-7d8757f9c-wvsww   1/1     Running   0          10h
pod/open-webui-redis-c47dbfbcd-rhzn2       1/1     Running   0          10h

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/open-webui             ClusterIP   10.43.185.237   <none>        80/TCP      10h
service/open-webui-ollama      ClusterIP   10.43.153.140   <none>        11434/TCP   10h
service/open-webui-pipelines   ClusterIP   10.43.112.4     <none>        9099/TCP    10h
service/open-webui-redis       ClusterIP   10.43.234.164   <none>        6379/TCP    10h

NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/open-webui-ollama      1/1     1            1           10h
deployment.apps/open-webui-pipelines   1/1     1            1           10h
deployment.apps/open-webui-redis       1/1     1            1           10h

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/open-webui-ollama-5d99896fd7     1         1         1       10h
replicaset.apps/open-webui-pipelines-7d8757f9c   1         1         1       10h
replicaset.apps/open-webui-redis-c47dbfbcd       1         1         1       10h

NAME                          READY   AGE
statefulset.apps/open-webui   1/1     10h

```

---

## Final Note

This assessment reflects real-world production ownership, focusing on reliability, security, and thoughtful decision-making rather than artificial success or failure states.
