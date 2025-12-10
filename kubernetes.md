

---

# 📘 **TOPIC 7 — KUBERNETES CHEAT SHEET (Junior → Intermediate Level)**

---

# 🔹 **1. What Kubernetes Is**

* Orchestrates containers across multiple machines
* Ensures desired state (pods running, correct versions, scaling)
* Handles networking, storage, updates, and self-healing

---

# 🔹 **2. Pods**

* Smallest deployable unit
* Can contain 1+ containers
* Containers share:

  * Network namespace (same IP, localhost)
  * Volumes
* Kubernetes does NOT run containers directly → Pods provide metadata, IPs, probes, restart policy

**Interview line:**

> “Pods wrap containers with networking, storage, and lifecycle control.”

---

# 🔹 **3. ReplicaSets**

* Ensure a **specific number** of Pods are always running
* Recreate Pods if they crash
* Usually created **via Deployments**, not manually

---

# 🔹 **4. Deployments**

* Manage ReplicaSets and Pods
* Provide rolling updates + rollbacks
* Used for stateless apps

### Rolling update behavior:

* New ReplicaSet created
* Old RS scaled down
* New RS scaled up
* Zero downtime

### Rollback:

```
kubectl rollout undo deployment/<name>
```

---

# 🔹 **5. Services**

Provide stable networking for Pods (Pods change IPs!).

### ClusterIP

* Default
* Internal access only
* `service-name:port`

### NodePort

* Exposes service on `<NodeIP>:<NodePort>`
* Ports 30000–32767
* Not ideal for production

### LoadBalancer

* Cloud LB + public IP
* Best for production external access

---

# 🔹 **6. Ingress**

* HTTP/HTTPS routing to multiple services
* Uses hostname/path rules
* Allows many services behind **one** LoadBalancer
* Requires an **Ingress Controller** (NGINX, ALB, Traefik)

**Example:**
`example.com/api → backend-service`

**Interview line:**

> “Ingress defines rules; the Ingress Controller implements them.”

---

# 🔹 **7. ConfigMaps & Secrets**

### ConfigMaps

* Store non-sensitive config (strings, files, env values)

### Secrets

* Store sensitive data (passwords, tokens)
* Base64 encoded, **not** encrypted
* Should be encrypted using KMS/Vault/etc.

---

# 🔹 **8. Kubernetes Storage**

### Volume

* Exists only during Pod lifecycle
* Used to share data between containers in a Pod

### PersistentVolume (PV)

* Actual storage in cluster
* Backed by cloud disks/NFS/local storage
* Lives independent of Pods

### PersistentVolumeClaim (PVC)

* Pod’s request for storage
* PVC binds to PV
* Pod mounts PVC

**Flow:**
**Pod → PVC → PV → Storage**

---

# 🔹 **9. Probes**

### Readiness Probe

* Is the app ready for traffic?
* Failure → removed from Service endpoints

### Liveness Probe

* Is the app dead/stuck?
* Failure → container restart

### Startup Probe

* Give slow apps time before other probes begin

---

# 🔹 **10. Autoscaling**

### HPA (Horizontal Pod Autoscaler)

* Adds/removes Pods
* **Uses CPU by default**
* Needs Metrics Server

### VPA (Vertical Pod Autoscaler)

* Adjusts CPU/memory of Pods
* May restart Pods
* Cannot manage same resource as HPA simultaneously

### Cluster Autoscaler

* Adds/removes nodes
* Works at infrastructure level

---

# 🔹 **11. Troubleshooting Pods**

### CrashLoopBackOff

* App crash
* Wrong CMD/ENTRYPOINT
* Missing ENV vars
* Config errors

**Debug:**
`kubectl logs`, `kubectl exec`, `kubectl describe`

---

### ImagePullBackOff

* Wrong image/tag
* No registry access

---

### Pending

* Not enough resources
* PVC not bound
* NodeSelector mismatch
* Taints/tolerations issue

---

### App unreachable

* App listening only on `127.0.0.1`
* Wrong Service port
* Readiness probe failing
* Ingress misconfigured

---

# 🔥 **Top Kubernetes Interview Lines to Memorize**

* “Pods are temporary; Services provide stable networking.”
* “Ingress is the routing rule; the controller actually processes traffic.”
* “HPA scales Pods horizontally based on CPU by default.”
* “PVC requests storage; PV provides it.”
* “CrashLoopBackOff means the container keeps crashing after start.”

---
