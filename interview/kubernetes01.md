### **Part 1: The Core Architecture & Fundamentals**

*Goal: To verify you understand the "brain" of the cluster and didn't just memorize commands.*

**1. Q: Explain the workflow of a Pod creation request from the moment I run `kubectl apply` to when the container starts running.**

* **Answer:**
1. **API Server:** The request goes to the API Server (the gatekeeper), which validates the request and persists the desired state in **etcd**.
2. **Scheduler:** The Scheduler notices "unbound" pods (pods with no node assigned) via the API Server. It runs an algorithm (filtering and scoring) to find the best Node and updates the Pod definition with that NodeName.
3. **Kubelet:** The Kubelet on the specific Node detects a new Pod assignment (watching the API Server).
4. **CRI/Runtime:** The Kubelet instructs the Container Runtime (e.g., containerd) to pull the image and start the container.
5. **Status Update:** Kubelet updates the Pod status back to the API Server.


* *Interviewer Note:* If you forget **etcd** or the **Scheduler**, it’s a red flag.

**2. Q: What is the role of the Kube-Proxy? How does it actually work?**

* **Answer:** Kube-proxy runs on every node and maintains network rules (using iptables or IPVS) to allow network communication to your Pods from network sessions inside or outside of your cluster. It is responsible for implementing the virtual IP mechanism for **Services**.
* *Follow-up:* "Does it handle Pod-to-Pod traffic directly?" No, the CNI plugin (Calico/Flannel) handles the routing; Kube-proxy handles the *Service* abstraction.

**3. Q: Why is etcd considered the most critical component? How do you back it up?**

* **Answer:** etcd is the consistent, highly-available key-value store for all cluster data. If etcd is lost, the cluster state is lost. You back it up using `etcdctl snapshot save`.
* *Scenario:* "What happens if the Master node goes down?" The existing workloads continue running (if nodes are healthy), but you cannot deploy new pods or change the state until the control plane is restored.

**4. Q: Explain the difference between a Liveness, Readiness, and Startup probe.**

* **Answer:**
* **Liveness:** "Is the app dead?" If this fails, Kubelet restarts the container.
* **Readiness:** "Is the app ready to serve traffic?" If this fails, the Pod is removed from the Service endpoints (Load Balancer stops sending traffic).
* **Startup:** "Has the app finished booting?" Used for slow-starting legacy apps. It disables other probes until it succeeds.


* *Real-world Scenario:* "I have an app that takes 2 minutes to load data into memory. Which probe do I need?" (Answer: Startup Probe, or a Readiness probe with a long initial delay).

**5. Q: What is an "Init Container" and give me a real-world use case.**

* **Answer:** Init containers run to completion *before* the main application container starts.
* *Use Case:* Waiting for a Database to be up (using netcat in a loop) before starting the API server, or fetching secrets/configuration files from a remote vault and placing them on a shared volume.

---

### **Part 2: Workloads, Scheduling & Scaling (High Priority)**

*Goal: To test if you can manage resources efficiently and handle traffic spikes.*

**6. Q: You have a Deployment and you want to update the image. Explain the "Rolling Update" strategy.**

* **Answer:** It updates Pods in a rolling fashion with zero downtime. It uses `maxSurge` (how many extra pods can be created above the desired count) and `maxUnavailable` (how many pods can be unavailable during the update).
* *Scenario:* "If I set `maxUnavailable` to 0, what happens?" K8s must create a new Pod and wait for it to be Ready before it terminates an old one. This is safer but requires more resource quota.

**7. Q: What is the difference between a DaemonSet and a StatefulSet?**

* **Answer:**
* **DaemonSet:** Ensures a copy of the Pod runs on *every* (or selected) Node. Used for logging agents (Fluentd) or monitoring (node-exporter).
* **StatefulSet:** Used for stateful apps (DBs, Kafka). Guarantees stable network IDs (`web-0`, `web-1`), stable storage (PVCs usually stick to the ID), and ordered deployment/termination.



**8. Q: Explain the concept of "Quality of Service" (QoS) classes in Kubernetes.**

* **Answer:**
* **Guaranteed:** Requests == Limits for CPU/RAM. (Top priority, last to be killed).
* **Burstable:** Requests < Limits. (Middle priority).
* **BestEffort:** No requests or limits set. (First to be killed if the node runs out of resources).



**9. Q: Why did my Pod get "OOMKilled"? How is that different from CPU throttling?**

* **Answer:**
* **OOMKilled:** The container tried to use more RAM than its *Limit*. RAM is not compressible; the kernel kills the process.
* **CPU Throttling:** The container tried to use more CPU than its *Limit*. CPU is compressible; the OS just slows the process down (throttles it), it doesn't kill it.



**10. Q: How does the Horizontal Pod Autoscaler (HPA) work?**

* **Answer:** HPA queries the Metrics Server (or custom metrics adapter) for resource usage (e.g., CPU %). If usage > target, it calculates the desired replica count and updates the Deployment/ReplicaSet.
* *Gotcha:* "Does HPA work well with Java apps?" Sometimes no, because JVM memory usage might not correlate linearly with load. Custom metrics (requests per second) are often better.

**11. Q: Explain Taints and Tolerations vs. Node Affinity.**

* **Answer:**
* **Taints/Tolerations:** "Repel" pods. A Node says "I have a Taint (e.g., GPU), don't schedule on me unless you Tolerate it."
* **Affinity:** "Attract" pods. A Pod says "I prefer to run on a Node with Label X."


* *Scenario:* "How do I ensure only my Critical Payment App runs on specific high-security nodes?" Taint the nodes `class=secure:NoSchedule` and add a toleration to the Payment App.

---

### **Part 3: Networking & Services**

*Goal: This is where most 2+ YOE engineers struggle. I need to see you understand the traffic flow.*

**12. Q: Explain the difference between ClusterIP, NodePort, and LoadBalancer.**

* **Answer:**
* **ClusterIP:** Default. Exposes service on an internal IP. Only reachable from within the cluster.
* **NodePort:** Exposes service on a static port on *each Node's IP*.
* **LoadBalancer:** Provisions an external Cloud Load Balancer (AWS ELB, GCP LB) that points to the NodePorts.



**13. Q: How does Service Discovery work within the cluster?**

* **Answer:** Kubernetes uses CoreDNS. When a Service `my-service` is created in namespace `default`, a DNS record `my-service.default.svc.cluster.local` is created. Pods can reach it using that name.

**14. Q: What is an Ingress Controller and why do I need it if I have a LoadBalancer service?**

* **Answer:** A LoadBalancer Service (L4) gives you one IP per service, which is expensive. An Ingress (L7) sits behind *one* LoadBalancer and routes traffic to multiple internal Services based on Hostname (`api.example.com`) or Path (`/app`). It handles SSL termination as well.

**15. Q: By default, can Pod A in Namespace X talk to Pod B in Namespace Y? How do I stop it?**

* **Answer:** Yes, by default, traffic is open (flat network). To stop it, you use **Network Policies**.
* *Key Concept:* A "Default Deny" policy is best practice to block all traffic, then you allowlist specific routes.

---

### **Part 4: Storage & Configuration**

**16. Q: What is the difference between a ConfigMap and a Secret? Are Secrets actually secure?**

* **Answer:** ConfigMaps are for non-sensitive data (config files). Secrets are for sensitive data (passwords).
* *The "Senior" Detail:* By default, Secrets are just base64 encoded, not encrypted. Anyone with API access can decode them. To be secure, you must enable **Encryption at Rest** in etcd or use external vaults (HashiCorp Vault, AWS Secrets Manager).

**17. Q: Explain the lifecycle of a Persistent Volume (PV) and Persistent Volume Claim (PVC).**

* **Answer:**
1. Admin creates a **StorageClass** (defines the type of storage, e.g., gp2).
2. User creates a **PVC** requesting specific size/access mode.
3. K8s dynamically provisions a **PV** (actual disk) and binds it to the PVC.
4. Pod mounts the PVC.



---

### **Part 5: Security & Operations (RBAC, Helm, CI/CD)**

**18. Q: Explain RBAC. What is the difference between a Role and a ClusterRole?**

* **Answer:**
* **Role:** Namespaced permissions (e.g., can read Pods in "dev" namespace).
* **ClusterRole:** Cluster-wide permissions (e.g., can read Nodes) or for resources that don't belong to a namespace.
* **Binding:** You need a RoleBinding to attach a User/Group to a Role.



**19. Q: Why use Helm? What is `helm upgrade --install`?**

* **Answer:** Helm is a package manager. It allows you to template YAML manifests so you can reuse them across environments (Dev/QA/Prod) just by changing a `values.yaml` file.
* *Command:* `upgrade --install` is idempotent: it installs the chart if it's missing, or upgrades it if it exists.

**20. Q: How would you handle a CI/CD pipeline for Kubernetes?**

* **Answer:**
1. Code Commit -> CI (Jenkins/GitLab) runs tests.
2. CI builds Docker Image -> Pushes to Registry (ECR/DockerHub) tagged with commit SHA.
3. CD (ArgoCD or `kubectl set image`) updates the Deployment manifest to use the new image tag.


* *Green Flag:* Mentioning **GitOps** (ArgoCD/Flux) where the cluster state automatically syncs with a Git repo.



---

### **Part 6: The "Scenario & Debugging" Gauntlet**

*These are the questions that determine if you get the job.*

**Scenario A: The "CrashLoopBackOff"**
**21. Q: You deploy a Pod and it goes into `CrashLoopBackOff`. Walk me through how you debug it.**

* **Answer:**
1. `kubectl get pods` to confirm status.
2. `kubectl describe pod <name>`: Check "Events" at the bottom. Is it a Liveness probe failure? OOMKilled?
3. `kubectl logs <name> --previous`: **Crucial step.** Check the logs of the *crashed* instance to see the application error (e.g., "Database connection failed").
4. If logs are empty, it might be a configuration issue (wrong command/entrypoint).



**Scenario B: The "Pending" Pod**
**22. Q: A Pod is stuck in `Pending` state for an hour. Why?**

* **Answer:**
1. **Insufficient Resources:** Cluster is full (CPU/Memory).
2. **Taints/Affinity:** The pod cannot tolerate any available nodes.
3. **PVC Issues:** The storage volume cannot be provisioned or mounted.
4. *Check:* `kubectl describe pod` will tell you exactly why the Scheduler couldn't place it.



**Scenario C: The "ImagePullBackOff"**
**23. Q: You see `ImagePullBackOff` or `ErrImagePull`. usage causes?**

* **Answer:**
1. Image name/tag is wrong.
2. Image doesn't exist in the registry.
3. **Authentication:** The cluster doesn't have the `imagePullSecret` to access the private registry.



**Scenario D: Connectivity**
**24. Q: App A cannot talk to App B. How do you troubleshoot?**

* **Answer:**
1. Check if App B's Service exists and has **Endpoints** (`kubectl get endpoints <service_name>`). If no endpoints, the Selector doesn't match the Pod labels.
2. Check DNS: Can I resolve the service name? (`kubectl exec -it podA -- nslookup serviceB`).
3. Check Network Policies (is traffic blocked?).
4. Check Application Logs (is App B actually listening on port 8080?).


Here are the detailed, interview-ready answers for the remaining questions (25–50).

As an interviewer, when I ask these, I am looking for "conceptual connections." Don't just give the dictionary definition; tell me how it affects your day-to-day operations.

---

### **Detailed Conceptual & Operational Answers (25–50)**

#### **25. Q: What is a "Headless Service"? When do you use it?**

* **Concept:** A Headless Service is a Kubernetes Service where `ClusterIP` is set to `None`.
* **Detailed Answer:**
* Normally, a Service gives you a single virtual IP (VIP) that load balances across all backing Pods. You talk to the VIP, and Kube-proxy routes you to a random Pod.
* In a **Headless Service**, Kubernetes does *not* assign a VIP. Instead, the DNS query for the service returns **A records for every single Pod** behind that service.
* **Use Case:** This is critical for **StatefulSets** (like Cassandra, MongoDB, or Kafka) where the application needs to know exactly which node it is connecting to (e.g., "I need to talk specifically to `mongo-0` to elect a leader," not just "any random mongo pod").



#### **26. Q: What exactly happens if the Master Node (Control Plane) fails?**

* **Concept:** Control Plane vs. Data Plane separation.
* **Detailed Answer:**
* The **Data Plane (Worker Nodes)** will continue running. Your existing applications/Pods will NOT crash; they will keep serving traffic.
* **What breaks:** The **Control Plane (API Server, Controller Manager, Scheduler)** is gone.
* You cannot use `kubectl` (API is down).
* You cannot deploy new Pods.
* **Self-healing stops:** If a Worker Node crashes while the Master is down, the Scheduler isn't there to notice and create replacement Pods elsewhere. The cluster state is "frozen" until the Master is restored.





#### **27. Q: What is the `kube-system` namespace?**

* **Detailed Answer:**
* It is the namespace reserved for Kubernetes internal system components.
* It typically contains:
* **CoreDNS:** For internal service discovery.
* **Kube-proxy:** For networking rules.
* **CNI Plugins:** (like Calico/Flannel pods).
* **Metrics Server:** For HPA.


* *Interviewer Note:* "I treat this as a 'No Fly Zone'. I don't deploy application workloads here to strictly separate user apps from system stability tools."



#### **28. Q: How do you limit the number of Pods or CPU/Memory a team can use in a namespace?**

* **Concept:** Governance and Multi-tenancy.
* **Detailed Answer:**
* We use a **ResourceQuota** object.
* It allows us to set hard limits on a Namespace, such as:
* `requests.cpu: "4"` (Total CPU reserved).
* `limits.memory: "8Gi"` (Total Memory limit).
* `count/pods: "10"` (Max number of pods).


* *Follow up:* "What happens if I try to deploy an 11th pod?" The API Server rejects the request with a `403 Forbidden` error saying quota is exceeded.



#### **29. Q: What is `kubectl drain` used for?**

* **Concept:** Node Maintenance.
* **Detailed Answer:**
* `kubectl drain <node-name>` is used before performing maintenance (like upgrading the kernel or replacing hardware).
* It does two things:
1. **Cordons** the node (marks it `SchedulingDisabled` so no *new* pods land there).
2. **Evicts** the existing pods safely (sends them a SIGTERM to shut down gracefully).


* *Pro Tip:* You usually need to add `--ignore-daemonsets` because DaemonSets (like logs) cannot be evicted (they must run on every node).



#### **30. Q: What is the "Sidecar" pattern? Give an example.**

* **Concept:** Multi-container Pods.
* **Detailed Answer:**
* A Sidecar is a secondary container running in the *same* Pod as the main application container.
* They share the same **Network Namespace** (same IP, same localhost) and can share **Storage Volumes**.
* **Example:**
* **Log Shipper:** Main app writes logs to a shared volume; Sidecar (Fluentd) reads those files and pushes them to Splunk/Elasticsearch.
* **Service Mesh:** Envoy proxy runs as a sidecar to handle mTLS and traffic routing for the main app.





#### **31. Q: Explain the concept of "Immutable Infrastructure" in the context of K8s.**

* **Detailed Answer:**
* It means we never modify a running container or node. If we need to change configuration or patch a bug, we **build a new image**, create a new version, and replace the old one.
* We do NOT SSH into a Pod and run `apt-get update`.
* *Why?* It prevents "Configuration Drift" (where servers become unique "snowflakes") and ensures that what we tested in Staging is exactly what runs in Prod.



#### **32. Q: Can you mount a ConfigMap as a volume? If I update the ConfigMap, does the file inside the Pod update?**

* **Detailed Answer:**
* Yes, you can mount a ConfigMap as a volume.
* **Does it update?** Yes, but with a delay (kubelet sync period, usually ~1 minute). Kubernetes updates the symlink to the file inside the container.
* *Crucial Caveat:* If your application reads the config file *only once at startup* (like many Java/Node apps), it won't see the change until you restart the Pod. You need a "hot reload" watcher in your app or just restart the pod.



#### **33. Q: Difference between a Job and a CronJob?**

* **Detailed Answer:**
* **Job:** A task that runs to completion (exit code 0) and then stops. If it fails, K8s can retry it (`backoffLimit`). Used for DB migrations or one-off scripts.
* **CronJob:** A wrapper around a Job that creates the Job object on a schedule (using cron syntax like `*/5 * * * *`).
* *Interviewer Note:* I look for knowledge of `concurrencyPolicy` (what happens if the previous job hasn't finished yet? Allow, Forbid, or Replace?).



#### **34. Q: What is a PDB (Pod Disruption Budget)? Why is it important?**

* **Concept:** High Availability during maintenance.
* **Detailed Answer:**
* PDBs limit how many Pods can be down simultaneously during **voluntary disruptions** (like `kubectl drain` or a cluster upgrade).
* **Example:** If I have a deployment with 3 replicas, and I set `minAvailable: 2`, Kubernetes will block a node drain if it would cause 2 pods to be down at the same time. It forces operations to happen sequentially to maintain uptime.



#### **35. Q: How do you safely store a database password in K8s? (Beyond just "Use Secrets")**

* **Detailed Answer:**
* Native Kubernetes Secrets are just base64 encoded strings stored in etcd. They are NOT encrypted by default.
* **Level 1 (Basic):** Use `kubectl create secret`. Restrict access via RBAC so only the app can read it.
* **Level 2 (Better):** Enable **Encryption at Rest** in etcd configuration so the raw data on the master node disk is encrypted.
* **Level 3 (Enterprise):** Don't store secrets in YAML at all. Use **External Secrets Operator** to sync secrets from AWS Secrets Manager / HashiCorp Vault directly into K8s memory.



#### **36. Q: How do I get the clean YAML of a running pod without all the status/timestamp junk?**

* **Detailed Answer:**
* Standard: `kubectl get pod <name> -o yaml`.
* *The "Pro" Answer:* "The standard output is noisy with `managedFields` and `status`. I usually pipe it to tools or use `kubectl neat` (a krew plugin) to clean it up. Or I manually delete the `status` block if I'm copying it for debugging."



#### **37. Q: Difference between `kubectl apply` and `kubectl create`?**

* **Detailed Answer:**
* **`kubectl create` (Imperative):** Tells the API "Create this object." If it already exists, it throws an error.
* **`kubectl apply` (Declarative):** Tells the API "Make the object look like this." If it doesn't exist, it creates it. If it exists, it calculates the "patch" (diff) and updates it. It stores the configuration in a `last-applied-configuration` annotation to manage these diffs.



#### **38. Q: How do you debug if a Service DNS name isn't resolving?**

* **Detailed Answer:**
1. First, verify it's actually DNS. Can I ping the IP directly?
2. Start a debug pod: `kubectl run -it --rm --image=busybox:1.28 dns-test -- sh`.
3. Run `nslookup my-service`.
4. If it fails, check **CoreDNS** pods in `kube-system`. Are they running? Check their logs (`kubectl logs -l k8s-app=kube-dns -n kube-system`).
5. Check `/etc/resolv.conf` inside the application pod to ensure it has the correct `nameserver` (usually the ClusterIP of kube-dns).



#### **39. Q: What is a "Static Pod"?**

* **Detailed Answer:**
* A Static Pod is managed directly by the **Kubelet** on a specific node, *not* by the API Server.
* You create them by placing a YAML file in a specific directory on the node (usually `/etc/kubernetes/manifests`).
* **Use Case:** This is how the Kubernetes Control Plane itself (etcd, api-server, scheduler) starts up. The Kubelet starts them before the API server even exists.



#### **40. Q: What is a "Container Runtime" (CRI)?**

* **Detailed Answer:**
* Kubernetes doesn't run containers itself; it instructs a Runtime to do it.
* Historically this was Docker. Now, Kubernetes uses the **CRI (Container Runtime Interface)** to talk to runtimes like **containerd** or **CRI-O**.
* *Relevance:* "We recently had to migrate from dockershim to containerd in version 1.24, which changed how we debug (using `crictl` instead of `docker ps`)."



#### **41. Q: What is etcd encryption?**

* **Detailed Answer:**
* It is a feature flag (`--encryption-provider-config`) enabled on the API Server.
* It ensures that resources marked as "secrets" are encrypted *before* being written to the etcd disk.
* This protects against physical theft of the etcd drive or unauthorized volume snapshots.



#### **42. Q: What is a "Service Account"?**

* **Detailed Answer:**
* Users (humans) have User Accounts (handled externally via OIDC/Google/AWS).
* **Processes (Pods)** use Service Accounts.
* When a Pod talks to the K8s API (e.g., an Ingress Controller needing to read Service endpoints), it authenticates using the mounted token of its assigned Service Account.



#### **43. Q: How do you update a Deployment without downtime?**

* **Detailed Answer:**
* By default, Deployments use the `RollingUpdate` strategy.
* I simply change the image tag in the manifest (`kubectl set image ...` or edit YAML) and apply it.
* Kubernetes brings up a new ReplicaSet, waits for the new Pods to pass their **Readiness Probes**, and only then terminates the old Pods.



#### **44. Q: What is "Self-Healing" in Kubernetes?**

* **Detailed Answer:**
* It is the loop driven by **Controllers** (like ReplicaSet or StatefulSet).
* The Controller constantly compares **Current State** (what's running) with **Desired State** (YAML).
* If a Pod crashes (Current State < Desired), the Controller commands the creation of a new one. It doesn't "fix" the dead pod; it replaces it.



#### **45. Q: What tool would you use to scan images for vulnerabilities?**

* **Detailed Answer:**
* **Trivy** is the industry standard open-source tool right now.
* I would implement it in the CI pipeline (Jenkins/GitHub Actions). If Trivy finds "High/Critical" CVEs, the build fails, and the image is never pushed to the registry.



#### **46. Q: What is "GitOps"?**

* **Detailed Answer:**
* GitOps is a methodology where the **Git repository** is the single source of truth for the cluster state.
* Instead of running `kubectl apply` manually (which is hidden/untrackable), we use a controller like **ArgoCD** or **Flux** inside the cluster.
* ArgoCD watches the Git repo. If I commit a change to `deployment.yaml`, ArgoCD detects the drift and automatically syncs the cluster to match Git.



#### **47. Q: How do you access a Service from outside without a Cloud LoadBalancer?**

* **Detailed Answer:**
* **NodePort:** Open a high port (30000+) on the worker node IP. (Good for demos, bad for prod).
* **Ingress with MetalLB:** If running on bare metal, MetalLB creates a software LoadBalancer implementation using ARP/BGP to assign IPs.
* **HostNetwork:** (Risky) The pod shares the host's network namespace directly.



#### **48. Q: What is "Namespace Isolation"? Does it isolate network traffic?**

* **Detailed Answer:**
* Namespaces isolate **Resources** (Service names, Secrets, ConfigMaps) and **Management** (RBAC, Quotas).
* **It does NOT isolate network traffic by default.** A pod in `dev` can ping a pod in `prod`.
* To isolate network traffic, you MUST apply **NetworkPolicies** (e.g., "Deny all ingress to `prod` unless from `prod`").



#### **49. Q: Why use `entrypoint` in Docker vs `command` in K8s?**

* **Detailed Answer:**
* It's a common confusion mapping:
* Docker `ENTRYPOINT` = K8s `command`
* Docker `CMD` = K8s `args`


* **Use Case:** If the Docker image has a script as an ENTRYPOINT (that does setup), I usually override the `args` in K8s to pass different parameters to that script.



#### **50. Q: How do you check cluster events? Why do they disappear?**

* **Detailed Answer:**
* `kubectl get events --sort-by=.metadata.creationTimestamp`
* Events are short-lived! They are stored in etcd but typically have a TTL (Time To Live) of **1 hour**.
* *Pro Tip:* For production, you should export events to a logging system (like ELK or Loki) so you can debug why a pod crashed 3 hours ago.
