## Introduction

Kubernetes is an open-source platform designed for automating the deployment, scaling, and management of containerized 
applications. At its core, a Kubernetes cluster comprises a **control plane** and a set of **worker nodes**, each fulfilling 
specific roles to ensure the cluster’s desired state is maintained, applications run reliably, and resources are utilized efficiently. 

---

## 1. Cluster Architecture Overview

A Kubernetes cluster follows a **hub-and-spoke** design pattern, in which the control plane acts as the “hub” responsible 
for orchestrating the overall cluster state, while the worker nodes are the “spokes” that actually run applications in containers. 
Every cluster requires at least one worker node to host Pods (the smallest deployable units in Kubernetes) and at least one 
control-plane instance to manage and schedule work.

* **Control Plane**
  The control plane is responsible for maintaining the desired state of the cluster. It exposes the Kubernetes API, stores 
* configuration data (e.g., deployed applications, node information) in a distributed key-value store, and runs controllers 
* that constantly reconcile the current cluster state with user-defined desired state definitions. In production environments, 
* control-plane components are typically distributed across multiple machines to ensure high availability and fault tolerance. ([Kubernetes][1])

* **Worker Nodes (Nodes)**
  A “node” is a physical or virtual machine provisioned in the cluster. Each node is managed by the control plane and contains 
* the services necessary to run Pods. Nodes host application containers, communicate status back to the control plane, and 
* execute instructions from control-plane components. Multiple nodes coexist within a cluster, enabling workloads to scale 
* across machines. ([Kubernetes][1], [Kubernetes][2])

---

## 2. Control-Plane Components

In a Kubernetes cluster, the primary control-plane components are:

1. **etcd**
   `etcd` is a distributed, highly available key-value store that persistently records the entire cluster state, including 
2. desired configuration (e.g., Deployments, Services, ConfigMaps). It emphasizes consistency over availability 
3. (per the CAP theorem), ensuring the control plane always has an up-to-date, single source of truth for scheduling 
4. decisions and cluster operations. ([Wikipedia][3])

2. **kube-apiserver**
   The API server (`kube-apiserver`) is the central management entity that exposes the Kubernetes API. All read and write operations (requests from both internal components and end users) pass through the API server via RESTful HTTP(S). It performs API schema validation, verifies authentication and authorization, and persists configuration changes to `etcd`. ([Wikipedia][3])

3. **kube-controller-manager**
   The `kube-controller-manager` runs a set of controllers, each responsible for a specific control loop. Controllers watch the API server for changes in object definitions and make necessary adjustments to reconcile the actual state of the cluster with the desired state. Key controllers include:

    * **Node Controller**: Monitors node health and reacts when nodes fail (e.g., evicting pods or rescheduling them on healthy nodes).
    * **ReplicationController / ReplicaSet Controller**: Ensures the specified number of Pod replicas are running.
    * **Deployment Controller**: Manages rollout and rollback of Deployment updates.
    * **DaemonSet Controller**: Ensures a copy of a Pod is running on selected nodes.
    * **Job Controller**: Tracks Jobs (one-off tasks) and ensures they complete successfully.
    * **Lease Controller**: Implements leader election among controllers by creating and renewing Lease objects (in the `coordination.k8s.io` API group), which act as distributed locks for high-availability scenarios. ([Kubernetes][4], [Wikipedia][3])

4. **kube-scheduler**
   The `kube-scheduler` watches for newly created Pods that have no assigned node and selects an appropriate node according to resource requirements (CPU, memory), affinity/anti-affinity rules, taints and tolerations, and custom policies. Once a node is chosen, the scheduler updates the Pod’s specification to assign it to that node and informs the API server. ([Wikipedia][3])

5. **cloud-controller-manager** (optional/add-on)
   When running Kubernetes in a cloud environment, the `cloud-controller-manager` integrates cloud-specific logic into Kubernetes (e.g., provisioning load balancers, adjusting node groups). By decoupling cloud-provider code from core controllers, it enables cloud providers to iterate independently of Kubernetes’ main release cycle. Key sub-controllers include:

    * **Node Controller** (cloud sub-controller): Checks cloud-provider details to determine if a node has been deleted in the cloud API.
    * **Route Controller**: Manages network routes in some cloud environments.
    * **Service Controller**: Integrates with cloud load balancers for Service resources of type `LoadBalancer`.
    * **PersistentVolume Controller**: Creates PersistentVolumes in the cloud when a PersistentVolumeClaim is created. ([Kubernetes][5])

---

## 3. Worker-Node Components

Each worker node runs a set of components crucial to Pod execution and networking:

1. **Container Runtime (containerd)**
   The container runtime is responsible for fetching container images, starting, stopping, and managing container lifecycles. Kubernetes interacts with container runtimes through the **Container Runtime Interface (CRI)**, which enables pluggable runtime implementations. Originally, Docker (via “dockershim”) was the default runtime, but since Kubernetes v1.24, `dockershim` was removed, and Kubernetes relies on CRI-compliant runtimes such as `containerd` or `CRI-O` for production. `containerd` is a widely adopted, lightweight container runtime that integrates directly with Kubernetes without requiring an intermediary shim. ([Reddit][6], [Wikipedia][3])

2. **kubelet**
   The `kubelet` is an agent that runs on each node. It:

    * Watches for Pods assigned to its node (via the API server).
    * Ensures that the specified containers are running and healthy (probes, restarts).
    * Monitors node-level health (resource usage, disk pressure, etc.) and reports status back to the API server.
    * Implements Pod lifecycle management: creating, deleting, and updating containers as dictated by the control plane.
    * Manages secrets and ConfigMaps by mounting them into Pods as volumes or environment variables.
    * Sends periodic heartbeats to the control plane, often implemented via updating a Lease object in the coordination API group (facilitating leader election and high-availability scenarios among controllers). ([Kubernetes][2], [Wikipedia][3])

3. **kube-proxy**
   `kube-proxy` is responsible for implementing Services (abstracting Pods behind a stable virtual IP address). It watches Service and Endpoints objects in the API server and sets up networking rules (e.g., iptables or IPVS) on each node to route traffic destined for a Service’s ClusterIP to one of its backend Pod IPs. In effect, `kube-proxy` acts as a cluster-level, TCP/UDP load balancer, ensuring Pods receive traffic even as endpoints scale up or down. ([Kubernetes][2], [Wikipedia][3])

4. **cAdvisor** (built into `kubelet`)
   Although not often highlighted as a standalone component, `cAdvisor` gathers resource usage and performance metrics for all running containers on the node. These metrics are used by the kubelet to enforce resource limits and by cluster monitoring stacks (e.g., Prometheus) to inform scaling decisions.

---

## 4. Control-Plane ↔ Node Communication

Communication between control-plane components (especially the API server) and nodes (via kubelet) follows a **hub-and-spoke** pattern:

* **Node ➔ Control Plane**

    * **HTTPS (Default)**: Each node’s `kubelet` frequently (every few seconds) sends status updates (heartbeats) and Pod resource usage metrics to the API server over a secure HTTPS port (usually port 443).
    * **Client Authentication**: Nodes use client certificates (via kubelet TLS bootstrapping) or service account tokens when pods inside the node (e.g., sidecars or in-cluster clients) connect to the API server. A `kubernetes` Service (in the `default` namespace) provides a virtual IP that forwards traffic to the API server’s HTTPS endpoint via `kube-proxy`. ([Kubernetes][7])

* **Control Plane ➔ Node**

    * **API Server ➔ Kubelet**: The API server initiates HTTPS connections to each node’s kubelet (by default on port 10250) for operations such as log retrieval (`kubectl logs`), port forwarding, and `exec`/`attach` requests. By default, the API server may not verify the kubelet’s serving certificate (unless the `--kubelet-certificate-authority` flag is configured), which can pose security risks if run over untrusted networks. SSH tunneling or Konnectivity (TCP-level proxy) can secure these communications over untrusted or public networks. ([Kubernetes][7], [Reddit][6])
    * **API Server ➔ Pods/Services**: For proxying connections (e.g., `kubectl proxy`), the API server also issues requests to nodes, pods, and services, which default to plain HTTP unless prefixed with `https:` in the API URL. However, certificates are not validated, so the integrity and authenticity cannot be guaranteed unless additional verification is enabled. ([Kubernetes][7])

---

## 5. Kubernetes Object Management: Imperative vs. Declarative

Kubernetes offers two principal paradigms for managing cluster objects: **imperative** and **declarative**.

1. **Imperative Management**
   In the imperative approach, users issue direct commands to create, update, or delete resources by specifying resource specifications on the command line. For example:

   ```bash
   kubectl create deployment nginx --image=nginx:1.21
   kubectl scale deployment nginx --replicas=3
   kubectl delete svc my-service
   ```

   Here, each command explicitly tells the API server what action to perform in sequence. This approach is convenient for quick one-off tasks, experimentation, or scripting simple workflows. ([Kubernetes][8])

2. **Declarative Management**
   The declarative approach involves defining the desired state of resources in one or more **YAML manifest files**, which are then applied to the cluster. For instance:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:1.21
           ports:
           - containerPort: 80
   ```

   Applying this manifest via `kubectl apply -f deployment.yaml` instructs Kubernetes to create or update the Deployment, reconciling differences between the current cluster state and the defined desired state. Subsequent modifications to the YAML (e.g., changing `replicas` to 5) can be reapplied, causing Kubernetes to converge toward the updated configuration. This approach aligns with Infrastructure-as-Code (IaC) best practices, enabling version control, review processes, and reproducible cluster configurations. ([Kubernetes][8])

### Choosing Between Imperative and Declarative

* **Imperative** is often used for:

    * Ad-hoc tasks or quick experiments.
    * Environments without a strong CI/CD pipeline or where resources need to be created on the fly.
    * Scripting simple automation where writing full YAML files is cumbersome.

* **Declarative** is preferred for:

    * Production workloads requiring consistency, reproducibility, and auditability.
    * Teams practicing GitOps, where the Git repository of manifests is the single source of truth.
    * Complex deployments that benefit from version control, peer reviews, and rollback capabilities.

---

## 6. Object Manifests: Structure and Format (YAML)

Kubernetes resources are defined in **YAML** (or JSON) manifests, which follow a prescribed schema to describe an object’s metadata, specification of the desired state, and, in some cases, its current status. Although JSON is supported, YAML is more human-readable and thus more commonly used.

A typical Kubernetes manifest includes the following top-level fields:

```yaml
apiVersion: <group>/<version>       # e.g., apps/v1, v1, batch/v1
kind: <ResourceKind>                # e.g., Deployment, Service, Pod
metadata:
  name: <resource-name>             # Unique name within the namespace
  namespace: <namespace>            # Defaults to "default" if omitted
  labels:                           # Key-value pairs for organization and selection
    key1: value1
    key2: value2
  annotations:                      # Arbitrary metadata, often for tooling
    annotation-key: annotation-value
spec:
  # Spec varies by resource kind. For Deployments, spec includes:
  replicas: <integer>               # Number of Pod replicas desired
  selector:                         # Label selector for matching Pods
    matchLabels:
      app: nginx
  template:                         # Pod template describing Pod metadata and spec
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: <container-name>      # Name of container inside Pod
        image: <image-name:tag>     # Container image (e.g., nginx:1.21)
        ports:
        - containerPort: <port>     # Port inside the container
        env:                        # Environment variables
        - name: ENV_VAR
          value: "value"
        volumeMounts:               # Optional: mounting volumes inside the container
        - name: config-volume
          mountPath: /etc/nginx/conf.d
      volumes:                      # Pod-level volumes (e.g., ConfigMaps, Secrets)
      - name: config-volume
        configMap:
          name: nginx-config
```

Key points about manifest structure:

* **`apiVersion`** and **`kind`** identify the API endpoint for the object (e.g., `apps/v1` + `Deployment` maps to `/apis/apps/v1/deployments`).
* **`metadata`** provides identity (name, namespace) and organizational metadata (labels, annotations).
* **`spec`** describes the user’s desired state (what Kubernetes should ensure).
* **`status`** is generally omitted by users; it’s populated by Kubernetes controllers to reflect the current observed state (e.g., number of ready replicas).

Manifests can be applied individually or grouped together in multi-document YAML files separated by `---`. For example, a Service manifest and Deployment manifest can co-exist in a single file:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

([Kubernetes][8], [Wikipedia][3])

---

## 7. Controllers, Leases, and Self-Healing

The **controller pattern** underpins Kubernetes’ self-healing capabilities. A controller constantly compares the **actual state** (observed via API server and node reports) with the **desired state** (specified in the manifests) and takes corrective actions to reconcile discrepancies. Examples include:

* **ReplicaSet / Deployment Controller**
  Ensures the specified number of replicas are running. If a Pod crashes or its node becomes unreachable, the controller creates a new Pod on an available node. ([Kubernetes][4], [Wikipedia][3])

* **Node Controller & Leases**
  Each `kubelet` periodically renews a **Lease** object in the `coordination.k8s.io` API group. This heartbeat mechanism enables the control plane to detect node failures. If the node controller notices a Lease has not been renewed within a specified timeframe, it marks the node as `NotReady` and signals schedulers and other controllers to evict or reschedule Pods from that node. ([Kubernetes][4], [Wikipedia][3])

* **Horizontal Pod Autoscaler (HPA)**
  Observes resource metrics (CPU/memory) from `metrics-server` or Prometheus and scales Deployments up or down by adjusting replica counts accordingly.

* **StatefulSet, DaemonSet, and Job Controllers**
  Manage stateful applications (ensuring unique identities/persistent storage), background tasks on every node, and batch workloads, respectively.

Kubernetes’ self-healing and reconciliation loops ensure that, even in the face of failures (e.g., node crashes, container failures), the cluster consistently recovers to satisfy the desired state. ([Kubernetes][4], [Wikipedia][9])

---

## 8. Container Runtime Interface (CRI) and containerd

The **Container Runtime Interface (CRI)** provides a standardized API that allows Kubernetes to communicate with various container runtimes. Notable components include:

* **Docker Shim** (deprecated)
  Until Kubernetes v1.23, the Docker Engine included a shim that translated CRI calls into Docker Engine API calls. However, maintaining this shim proved burdensome, leading to its deprecation and removal in v1.24.

* **containerd**
  A high-performance, industry-standard container runtime that communicates directly via CRI. It handles:

    * Pulling images from registries.
    * Managing container lifecycle (creation, start, stop, deletion).
    * Managing container snapshots and storage.
    * Providing namespaces to isolate container resources.
      Kubernetes’ `kubelet` invokes `containerd` APIs to perform actions on containers requested by controllers and Pods. ([Wikipedia][3], [Reddit][6])

* **CRI-O**
  Another CRI-compliant runtime, primarily maintained by Red Hat and optimized for Kubernetes. While similar in purpose to `containerd`, CRI-O is designed to strictly implement CRI without additional features.

As Kubernetes no longer relies on Docker’s shim, cluster operators must explicitly configure container runtimes. For example, on many cloud-hosted managed Kubernetes services (e.g., Google Kubernetes Engine, Amazon EKS, Azure AKS), `containerd` is the default runtime. ([Reddit][6], [Wikipedia][3])

---

## 9. kubelet Detailed Functions

The **`kubelet`** on each node is the linchpin between the control plane and the container runtime. Its responsibilities include:

* **Pod Lifecycle Management**

    * Observes the API server for Pod definitions assigned to its node.
    * Creates containers via CRI (e.g., sending gRPC calls to `containerd`) according to Pod specifications (image, resource limits, volumes, environment variables).
    * Monitors container health via **liveness** and **readiness probes**. If a probe fails, the kubelet restarts or marks the Pod accordingly.

* **Node Status Reporting**

    * Aggregates node resource usage (CPU, memory, disk, network) via `cAdvisor` and updates the node’s `status` subresource in the API server.
    * Periodically renews a **Lease** object (in the `coordination.k8s.io` API group) to indicate node health and readiness. ([Kubernetes][2], [Kubernetes][4])

* **Volume Management**

    * Manages mounting of **Volumes** (e.g., PersistentVolumes, ConfigMaps, Secrets) into Pods.
    * Coordinates with CSI (Container Storage Interface) drivers to provision, attach, and mount external storage (e.g., cloud block storage, network file systems).

* **Node-Level Admission Checks**

    * Enforces resource limits and requests.
    * Checks for disk pressure, kernel memory availability, and other conditions that can preemptively evict Pods if necessary.

* **Static Pods** (optional)

    * Kubelet can run “static Pods” directly from a file on the node’s filesystem. These Pods are not visible to the API server initially but are reconciled by the kubelet. This feature is mainly used for bootstrapping the control plane itself on the first node. ([Kubernetes][2], [Wikipedia][3])

---

## 10. kube-proxy Networking and Service Abstraction

`kube-proxy` ensures that Pods can reach Services via a stable ClusterIP, independent of the actual Pod IP addresses behind the Service. It operates by:

1. **Watching the API Server** for Service and Endpoints objects.

2. **Maintaining Networking Rules** on each node:

    * **iptables mode**: Implements a series of iptables rules that match packets destined for a Service’s ClusterIP and port, then redirect them to a backend Pod IP chosen randomly (round-robin) among healthy endpoints.
    * **IPVS mode**: Uses Linux IPVS (IP Virtual Server) for higher performance and advanced load-balancing algorithms. IPVS mode is often preferred for large clusters due to improved scalability.

3. **Handling External Traffic** (e.g., `NodePort`, `LoadBalancer` Services) by opening ports on the node’s network interface or provisioning cloud load balancers (in combination with cloud-controller-manager).

By abstracting the complexities of Pod IP changes (e.g., when Pods are rescheduled), `kube-proxy` provides a consistent virtual IP and port for accessing applications. ([Kubernetes][2], [Wikipedia][3])

---

## 11. Kubernetes Object Model and Self-Healing

Every Kubernetes object represents a **persistent entity** in the API server. The cluster’s actual state emerges from the set of objects defined by users (or controllers). The key components of this model include:

* **Kubernetes Objects**
  Objects are persistent records of the cluster’s desired state, such as Deployments, ReplicaSets, Services, Pods, ConfigMaps, Secrets, and more. Each object has:

    1. **`metadata`**: Unique identifier (`name`, `namespace`), labels, and annotations.
    2. **`spec`**: Desired state (e.g., number of replicas, container images, resource limits).
    3. **`status`**: Current observed state (managed by controllers), including conditions (e.g., “Ready”, “Progressing”), counts, and other runtime information. ([Kubernetes][8], [Wikipedia][3])

* **Controllers & Reconciliation Loops**
  Controllers continuously watch for changes in objects’ `spec` (desired state) and compare it with the actual state (e.g., Pods running on nodes, node health). If discrepancies exist, controllers take action to converge the actual state toward the desired state. For instance, if a Pod managed by a Deployment fails or is evicted, the Deployment controller creates a new Pod to maintain the specified replica count.

* **Self-Healing**

    * When nodes fail (Lease expires or heartbeat missing), the Node controller marks the node as `NotReady`. The scheduler, responding to that event, refrains from generating new Pods for that node. Existing Pods are eventually evicted (after a grace period) and rescheduled onto healthy nodes.
    * When a container crashes or a readiness probe fails, the kubelet restarts the container according to Pod restart policies (e.g., `Always`, `OnFailure`, `Never`). The Pod’s status is updated accordingly, and controllers coordinate to ensure the overall workload remains healthy. ([Kubernetes][4], [Wikipedia][3])

---

## 12. Kubernetes Releases and Support Windows

Kubernetes follows a **time-based release cycle**, producing a new minor version approximately **every three to four months** (roughly three major releases per year). Each release is versioned using semantic versioning: `[major].[minor].[patch]`. As of May 2025, the most recent releases include v1.33, v1.32, and v1.31, with v1.33 being the latest stable minor release. ([Kubernetes][10], [Kubernetes][11])

* **Release Cadence**

    * **Minor Releases**: Introduce new features, enhancements, API changes (while maintaining backward compatibility), and improvements. They occur approximately every three months (e.g., v1.32 released in February 2025, v1.33 in May 2025).
    * **Patch Releases**: Address bug fixes, security patches, and minor feature updates within a minor version (e.g., v1.32.1, v1.32.2, …). Patch releases typically occur every one to two weeks, depending on severity. ([Kubernetes][11], [Kubernetes][10])

* **Supported Versions & EOL Policy**
  The Kubernetes project maintains **release branches for the three most recent minor releases** (e.g., if v1.33 is current, v1.32 and v1.31 are actively supported; v1.30 enters End of Life).

    * **Patch Support Duration**:

        * Versions **v1.19 and newer** receive approximately **one year** of patch support, which includes security and bug fixes.
        * Versions **v1.18 and older** historically received around **nine months** of patch support (prior to the policy change). ([Kubernetes][10], [Komodor][12])
    * **Actual Support Window**:

        * Total support spans roughly **14 months** from initial release (12 months of active support + 2-month grace period for upgrade planning).
        * After 14 months, a version is considered end-of-life (EOL) and no longer receives official patches. Administrators must upgrade to a supported version to continue receiving security updates.
    * **Managed Kubernetes Services** (e.g., EKS, AKS, GKE) often extend support windows with additional constraints (e.g., Amazon EKS provides extended support for up to 26 months in total, combining standard and extended windows). ([Dokumentacja AWS][13], [Komodor][12])

* **Version Skew Policy**
  To ensure compatibility between control-plane components and worker nodes, Kubernetes enforces a version skew policy:

    * **kubelet** on a node may be at most **one minor version older or newer** than the control plane. For example, a node running `v1.32.x` can communicate reliably with a control plane running `v1.33.x`, but running `v1.31.x` against a `v1.33.x` control plane is unsupported.
    * This policy facilitates gradual cluster upgrades, allowing administrators to upgrade control-plane components first, then rolling out upgrades to kubelets on nodes in a controlled fashion. ([Kubernetes][10], [Kubernetes][11])

* **Release Documentation & Change Logs**

    * Each minor version has an associated **release notes** page detailing new features, deprecations, and known issues.
    * **Deprecation Policy**: Features marked as deprecated are typically removed in the subsequent two minor releases, giving administrators time to migrate.
    * **Release Leads & SIG Release**: Community-driven teams coordinate the release cycle, maintain CHANGELOGs, and ensure proper testing (CI/CD, e2e tests) for each milestone. ([Kubernetes][11], [Kubernetes Contributors][14])

---

## Conclusion

The architecture of a Kubernetes cluster is modular and highly orchestrated, designed to deliver scalable, resilient, and self-healing container orchestration. By clearly separating the **control plane** (etcd, API server, controller-manager, scheduler, cloud-controller-manager) from the **worker nodes** (kubelet, container runtime, kube-proxy), Kubernetes delivers both consistency (via declarative API-driven object management) and flexibility (via pluggable runtimes and networking implementations). Controllers and Leases ensure high availability and automatic failure recovery, maintaining the cluster’s desired state.

Kubernetes supports both imperative and declarative management paradigms, enabling rapid experimentation and robust infrastructure-as-code practices. Object manifests, written in YAML, provide a standard format for defining resources, while the API server and controllers work in constant reconciliation loops to achieve and maintain the declared state.

Finally, understanding Kubernetes’ release cadence—new minor versions every three to four months, each supported for approximately one year of patches—helps cluster administrators plan upgrades effectively, maintain compatibility, and ensure ongoing security. As the platform evolves, staying current with release announcements and deprecation notices is critical to leveraging new features while minimizing operational risk. ([Kubernetes][10], [Komodor][12], [Dokumentacja AWS][13])

[1]: https://kubernetes.io/docs/concepts/architecture/?utm_source=chatgpt.com "Cluster Architecture | Kubernetes"
[2]: https://kubernetes.io/docs/concepts/architecture/nodes/?utm_source=chatgpt.com "Nodes - Kubernetes"
[3]: https://en.wikipedia.org/wiki/Kubernetes?utm_source=chatgpt.com "Kubernetes"
[4]: https://kubernetes.io/docs/concepts/architecture/leases/?utm_source=chatgpt.com "Leases - Kubernetes"
[5]: https://kubernetes.io/docs/concepts/architecture/cloud-controller/?utm_source=chatgpt.com "Cloud Controller Manager - Kubernetes"
[6]: https://www.reddit.com/r/kubernetes/comments/10n7a9t/how_does_control_plane_kubelet_communication_work/?utm_source=chatgpt.com "How does control plane <-> kubelet communication work? - Reddit"
[7]: https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/?utm_source=chatgpt.com "Communication between Nodes and the Control Plane - Kubernetes"
[8]: https://kubernetes.io/docs/concepts/overview/working-with-objects/?utm_source=chatgpt.com "Objects In Kubernetes"
[9]: https://en.wikipedia.org/wiki/Windows_Server_2019?utm_source=chatgpt.com "Windows Server 2019"
[10]: https://kubernetes.io/releases/?utm_source=chatgpt.com "Releases - Kubernetes"
[11]: https://kubernetes.io/releases/release/?utm_source=chatgpt.com "Kubernetes Release Cycle"
[12]: https://komodor.com/learn/kubernetes-eol-understanding-the-k8s-release-cycle-and-how-to-prepare-for-eol/?utm_source=chatgpt.com "Kubernetes EOL: Understanding the K8s Release Cycle - Komodor"
[13]: https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html?utm_source=chatgpt.com "Understand the Kubernetes version lifecycle on EKS - Amazon EKS"
[14]: https://www.kubernetes.dev/resources/release/?utm_source=chatgpt.com "Kubernetes v1.34 Release Information"
