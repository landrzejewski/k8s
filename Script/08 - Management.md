# Kubernetes Operations and Advanced Configuration Patterns

Managing a production Kubernetes cluster involves not only deploying applications but also ensuring nodes stay healthy, applications update safely, and advanced configurations extend cluster capabilities. This article covers four key areas:

1. **Preparing for and Executing Node Maintenance**
2. **Application Update Strategies in Kubernetes**
3. **Custom Resource Definitions (CRDs)**
4. **Downward API: Exposing Pod Metadata to Containers**

---

## 1. Maintaining Cluster Machines with Maintenance Windows

A “maintenance window” for a Kubernetes node typically means draining workloads off the node, performing updates (kernel patches, OS upgrades, or hardware repairs), and then returning the node to service. Properly cordoning, draining, and uncordoning ensures minimal disruption.

### 1.1 Cordoning a Node

**Cordoning** marks a node as unschedulable, preventing new Pods from landing there:

```bash
kubectl cordon <node-name>
# Example:
kubectl cordon worker-node-01
```

* After `cordon`, any Pod creation or scaling will not choose this node.
* Existing Pods remain running until explicitly evicted.

### 1.2 Draining a Node

**Draining** evicts all eligible Pods from a node so that it can be taken offline:

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force
# Example:
kubectl drain worker-node-01 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

* `--ignore-daemonsets` skips DaemonSet-managed Pods (they will be re-created when the node returns).
* `--delete-emptydir-data` allows eviction of Pods using `emptyDir` volumes (data will be lost).
* By default, drain respects Pod Disruption Budgets (PDBs). If pods cannot be evicted due to PDB constraints, drain will hang or error.

#### Handling Pods Controlled by StatefulSets or PDBs

Pods managed by StatefulSets or protected by PDBs may refuse eviction if it violates availability constraints. To drain such a node:

1. **Check PodDisruptionBudgets**:

   ```bash
   kubectl get pdb -n <namespace>
   ```
2. **Temporarily relax PDBs** or scale down StatefulSets so that drain can proceed (and ensure applications tolerate brief scale‐downs).

### 1.3 Tainting for Maintenance

As an alternative to `cordon + drain`, you can **taint** a node with a “NoExecute” taint, which automatically evicts any Pod that does not tolerate the taint:

```bash
kubectl taint nodes worker-node-01 maintenance=true:NoExecute
```

* With `NoExecute`, Pods that lack a matching `toleration` are evicted immediately.
* Taints provide more granularity (e.g., you could allow certain critical Pods by giving them a toleration for `maintenance=true`).

Example Pod toleration for maintenance:

```yaml
tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 600
```

* `tolerationSeconds` indicates how long the Pod may stay on the node after the taint is applied.

### 1.4 Performing Maintenance

After draining or taint‐draining, perform your OS upgrades, kernel patches, or hardware checks. At this point, the node runs no application workloads (aside from DaemonSets), minimizing the blast radius.

### 1.5 Uncordoning/Removing Taints

Once maintenance is complete, re‐enable scheduling:

```bash
kubectl uncordon worker-node-01
# Or remove the maintenance taint:
kubectl taint nodes worker-node-01 maintenance:NoExecute-
```

* After `uncordon`, the scheduler may place new or restarted Pods on the node as workload scales up.

### 1.6 Automating Maintenance Windows

In larger clusters, consider:

* **Automation Tools**: Use tools (e.g., Kubernetes Operators, cluster‐management frameworks) to schedule rolling drains across a node group, ensuring you never take down too many nodes at once.
* **PodDisruptionBudget Configuration**: For critical workloads, set PDBs so that only a safe number of Pods can be evicted at once.
* **Node Pools**: If on managed services (GKE, EKS), leverage node pools (node groups) and perform rolling updates of the pool. The control plane will automatically cordon and drain one node at a time before upgrading.

---

## 2. Application Update Policies and Patterns

Updating running applications with zero or minimal downtime is essential. Kubernetes natively supports several update strategies through Deployments, but more advanced patterns (blue/green, canary, A/B testing, shadowing) often require additional tooling or manual configuration.

### 2.1 Deployment Update Strategies

A **Deployment** controls ReplicaSets. Two built‐in strategies exist:

#### 2.1.1 Recreate Strategy

* **Behavior**: Terminates all existing Pods before creating new Pods.
* **Use case**: Simple applications where brief downtime is acceptable, or when Pods cannot coexist gracefully (e.g., major schema changes).
* **Configuration** (default is `RollingUpdate`, so explicitly set):

  ```yaml
  strategy:
    type: Recreate
  ```

#### 2.1.2 RollingUpdate Strategy

* **Behavior**: Gradually replaces Pods with new versions, maintaining a specified minimum availability.
* **Control**: `maxSurge` (number of extra Pods above desired during update) and `maxUnavailable` (how many existing Pods can be unavailable during the update).

  ```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Allows 1 extra Pod during update
      maxUnavailable: 0   # Ensure 0 Pods are down at once
  ```
* **Example**: If you have `replicas: 3`, then with `maxSurge=1` and `maxUnavailable=1`, at most 4 Pods may run temporarily, and at least 2 remain available during the transition.

Rolling updates are the most common and provide near‐zero‐downtime for stateless services.

### 2.2 Blue/Green Deployment

**Blue/Green** involves two separate but identical environments (blue and green). One is live, while the other is idle or used for staging. To deploy a new version:

1. Deploy new version to the **green** environment (e.g., a new Service selector, separate Deployment).
2. Run smoke tests against green.
3. Switch Service endpoint from **blue** to **green** (by updating Service selectors or swapping labels).
4. Once traffic flows to green, you can decommission blue or keep it as the previous version for quick rollback.

#### Kubernetes Implementation

* **Two Deployments** (blue and green), each with its own label:

  ```yaml
  # Blue Deployment (current)
  metadata:
    name: web-blue
  spec:
    selector:
      matchLabels:
        app: web
        version: blue
    template:
      metadata:
        labels:
          app: web
          version: blue
  ```

  ```yaml
  # Green Deployment (new)
  metadata:
    name: web-green
  spec:
    selector:
      matchLabels:
        app: web
        version: green
    template:
      metadata:
        labels:
          app: web
          version: green
  ```

* **Single Service** routing by label `app: web` and `version: blue` (initially):

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: web-svc
  spec:
    selector:
      app: web
      version: blue
    ports:
      - port: 80
        targetPort: 8080
  ```

* **Switch Traffic**: Update Service to point to `version: green`:

  ```bash
  kubectl patch svc web-svc -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
  ```

* **Rollback**: If green fails, patch Service back to `version: blue`.

### 2.3 Canary Deployment

**Canary** releases send a small percentage of traffic to a new version before ramping up to 100%. The goal is to validate behavior under real traffic.

#### Implementation Options

1. **Manual Pod Count Adjustment**

    * Initially create a small Deployment (e.g., 1 canary Pod) alongside existing Deployment.
    * Use a Service selector that matches both `version: stable` and `version: canary`, but rely on weights externally (e.g., ingress annotations) to send, say, 5% of traffic to canary Pods.

2. **Ingress/Service Mesh Traffic Splitting**

    * **Istio Example**: Use a VirtualService to split traffic by percentage:

      ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: web-virtualservice
      spec:
        hosts:
          - web.example.com
        http:
          - route:
              - destination:
                  host: web-svc
                  subset: stable
                weight: 95
              - destination:
                  host: web-svc
                  subset: canary
                weight: 5
      ```
    * Define `DestinationRule` subsets for stable and canary:

      ```yaml
      apiVersion: networking.istio.io/v1alpha3
      kind: DestinationRule
      metadata:
        name: web-destinationrule
      spec:
        host: web-svc
        subsets:
          - name: stable
            labels:
              version: stable
          - name: canary
            labels:
              version: canary
      ```
    * Deploy the canary version with label `version: canary`.

3. **Ingress Controller Annotations** (NGINX, Traefik, etc.)

    * Some controllers support weighted Service backends. For example, with NGINX Ingress Controller you might annotate as:

      ```yaml
      metadata:
        annotations:
          nginx.ingress.kubernetes.io/canary: "true"
          nginx.ingress.kubernetes.io/canary-weight: "5"
      ```
    * This sends 5% of traffic to the canary Ingress.

### 2.4 A/B Testing

**A/B testing** is similar to canary but targets distinct user segments (e.g., by header, cookie, or URL path). Implementation often relies on an Ingress or service mesh rule:

```yaml
# Example using NGINX Ingress with header-based routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/traffic-split: |
      stable=90,canary=10
    nginx.ingress.kubernetes.io/traffic-shaping-method: "cookie"
spec:
  rules:
    - host: web.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

* NGINX Ingress uses a cookie to split traffic 90/10 between stable and canary.
* You can also route based on a header (e.g., `X-Group: beta` to canary, else to stable).

### 2.5 Shadow (Mirroring) Traffic

**Shadow** or **traffic mirroring** sends a copy of live production traffic to a new version of the service without impacting user experience. Useful for load testing or validating changes on real requests.

* **Istio Example**:

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: web-virtualservice
  spec:
    hosts:
      - web-svc
    http:
      - match:
          - uri:
              prefix: /
        route:
          - destination:
              host: web-svc
              subset: stable
        mirror:
          host: web-svc
          subset: canary
        mirrorPercent: 100
  ```

    * All incoming requests route to `subset: stable`.
    * A copy is mirrored to `subset: canary`. Canary’s response is discarded.

* **NGINX Ingress** does not natively support mirroring; you need a service mesh or custom Lua scripts.

---

## 3. Custom Resource Definitions (CRDs)

Kubernetes’s extensibility is largely built on **Custom Resource Definitions (CRDs)**, which allow you to define new API objects (custom resources) that your own controllers or operators can manage.

### 3.1 What Is a CRD?

* A CRD is a “schema” that tells the Kubernetes API server to treat objects of a new `Kind` as first‐class API resources.
* Once a CRD is applied, you can `kubectl get` and `kubectl apply` resources of that custom kind.
* CRDs unlock the ability to build application‐specific abstractions (e.g., `BackupJob`, `CertificateRequest`, `IngressRoute`, etc.) without modifying Kubernetes core code.

### 3.2 Defining a CRD

A simple CRD manifest looks like:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgets.example.com
spec:
  group: example.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                size:
                  type: integer
                color:
                  type: string
      subresources:
        status: {}          # Enables /status subresource
      additionalPrinterColumns:
        - name: Size
          type: integer
          jsonPath: .spec.size
  scope: Namespaced
  names:
    plural: widgets
    singular: widget
    kind: Widget
    shortNames:
      - wd
```

* **`group: example.com`**: API group for the CRD.
* **`versions`**: List of API versions; `v1alpha1` is served and stored. Must define an OpenAPI v3 schema.
* **`scope: Namespaced`**: Resource lives in a namespace (vs. `Cluster` for cluster‐scoped).
* **`names`**: Specifies how to reference the resource (`kubectl get widgets`, kind=`Widget`).
* **`subresources.status`**: Enables `.status` so controllers can `kubectl patch` status separately.
* **`additionalPrinterColumns`**: Custom columns when you do `kubectl get widgets`.

### 3.3 Using a CRD

1. **Apply the CRD**:

   ```bash
   kubectl apply -f widget-crd.yaml
   ```

2. **Create a Custom Resource (CR)**:

   ```yaml
   apiVersion: example.com/v1alpha1
   kind: Widget
   metadata:
     name: my-widget
   spec:
     size: 3
     color: "blue"
   ```

   ```bash
   kubectl apply -f my-widget.yaml
   ```

3. **Inspect the CR**:

   ```bash
   kubectl get widgets
   # NAME        AGE
   # my-widget   10s
   kubectl describe widget my-widget
   ```

4. **Build a Controller or Operator**:

    * A custom controller watches `Widget` objects (via client libraries such as [client-go](https://github.com/kubernetes/client-go), [kubebuilder](https://book.kubebuilder.io/), [Operator SDK](https://sdk.operatorframework.io/) ) and takes actions (e.g., create Deployments, Services) to reconcile the real cluster state with the `Widget` spec.
    * The controller updates `status` subresource to report progress (e.g., `status.phase: Ready`).

### 3.4 Versioning and Conversion

* As your CRD evolves, you may need multiple versions (`v1alpha1`, `v1beta1`, `v1`). Kubernetes can use **conversion webhooks** or `spec.conversion` strategies to convert between versions.
* Eventually, graduate your CRD to a stable version (`v1`) and set `storage: true` on that version.

---

## 4. Downward API: Exposing Pod and Container Metadata

The **Downward API** lets a container retrieve information about its own Pod (labels, annotations, namespace, name, resource requests, etc.) and expose that to the application via environment variables or files. This is useful when applications need to be aware of their context without external configuration.

### 4.1 Key Use Cases

* **Pod Name / Namespace**: Include Pod identity in logs or metrics (e.g., tags in Prometheus).
* **Labels/Annotations**: Application behavior can adapt based on labels (e.g., `featureX=enabled`).
* **Resource Requests**: Application can read its own CPU/memory requests to size internal caches.
* **Pod IP**: For building self‐registration or peer discovery.

### 4.2 Exposing via Environment Variables

Example Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-demo
  namespace: production
  labels:
    app: metrics-proxy
    tier: backend
  annotations:
    log-level: debug
spec:
  containers:
    - name: app
      image: myapp:latest
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LABEL_APP
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['app']
        - name: ANNOT_LOG_LEVEL
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['log-level']
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: requests.cpu
              divisor: "1"
        - name: MEMORY_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: requests.memory
              divisor: "1Mi"
      resources:
        requests:
          cpu: "250m"
          memory: "512Mi"
```

* **`metadata.name/namespace`**: Direct Pod identity.
* **`metadata.labels[...]`**: Read specific label value.
* **`metadata.annotations[...]`**: Read specific annotation.
* **`resourceFieldRef`**: Read resource requests (CPU as millicores, memory in MiB).

Within the container, environment variables appear as:

```
POD_NAME=downward-demo
POD_NAMESPACE=production
LABEL_APP=metrics-proxy
ANNOT_LOG_LEVEL=debug
CPU_REQUEST=250m
MEMORY_REQUEST=512Mi
```

### 4.3 Exposing via Volume (Files)

The Downward API can also project metadata into files:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-volume-demo
  labels:
    app: logger
    version: v1
spec:
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
          - path: "podname"
            fieldRef:
              fieldPath: metadata.name
          - path: "cpu"
            resourceFieldRef:
              containerName: app
              resource: requests.cpu
              divisor: "1"
          - path: "memory"
            resourceFieldRef:
              containerName: app
              resource: requests.memory
              divisor: "1Mi"
  containers:
    - name: app
      image: busybox
      command:
        - "sh"
        - "-c"
        - |
          echo "Pod name: $(cat /etc/podinfo/podname)"
          echo "Labels: $(cat /etc/podinfo/labels)"
          echo "Annotations: $(cat /etc/podinfo/annotations)"
          echo "CPU Request: $(cat /etc/podinfo/cpu)m"
          echo "Memory Request: $(cat /etc/podinfo/memory)Mi"
          sleep 3600
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
```

* Files under `/etc/podinfo/` contain JSON‐encoded metadata. For example, `/etc/podinfo/labels` might contain:

  ```json
  {"app":"logger","version":"v1"}
  ```

* The container can parse or log these values as needed.

### 4.4 Common Use Cases

1. **Logging and Metrics**: A sidecar logger can tag logs with Pod labels (e.g., service, version).
2. **Configuration by Labels**: If an application checks the `featureX` label and toggles behavior accordingly without a full ConfigMap or flag.
3. **Self‐Registration**: A Pod registers itself in a service registry using its own `podIP` and labels.
4. **Resource‐Aware Behavior**: A container that adjusts thread pool size based on allocated CPU.

# Collecting Logs in Kubernetes and Centralized Monitoring of Cluster Resources

Managing logs and metrics in a Kubernetes (K8s) environment is critical for troubleshooting, performance tuning, and ensuring overall cluster health. Because containers are ephemeral and distributed across many nodes, both logs and metrics need to be aggregated, centralized, and made easily searchable or visualizable. This article covers:

1. **Log Collection in Kubernetes**

    * Challenges of containerized logging
    * Architectural patterns (DaemonSets, sidecar containers)
    * Overview of popular log‐aggregation tools: the ELK Stack (Elasticsearch, Logstash/Beats, Kibana) and Graylog
    * Comparison of ELK vs. Graylog

2. **Centralized Resource Monitoring in Kubernetes**

    * Prometheus: metrics collection, exporters, and scraping
    * Alertmanager: defining and delivering alerts
    * Grafana: dashboards, data sources, and visualization
    * Example reference architecture (kube-prometheus stack)

---

## 1. Log Collection in Kubernetes

### 1.1 The Challenge of Containerized Logging

Containers in a K8s cluster present several unique logging challenges:

* **Ephemeral Pods**: Pods can be created and destroyed frequently; their logs need to be collected before the Pod disappears.
* **Multi-Node Architecture**: Containers run on many nodes; logs must be aggregated centrally.
* **Standardized Interfaces**: By default, containers write logs to `stdout`/`stderr`. K8s writes these to files under `/var/log/containers/…`. To avoid manual `kubectl logs`, logs should be shipped to a central store.
* **Structured vs. Unstructured Logs**: Some applications emit structured JSON logs, which are easier to parse; others produce plain text. The aggregation pipeline must handle both.

A common pattern is to use a **DaemonSet**–deployed log forwarder on every node. These forwarders tail container log files and forward them to a log‐aggregation backend (e.g., Elasticsearch or Graylog). Sidecar containers (per‐Pod log collectors) are also possible, but DaemonSets scale automatically and avoid modifying each Pod spec.

### 1.2 Architectural Patterns for Logging

#### 1.2.1 Node-Level Log Forwarder (DaemonSet)

1. **Log Files on Host**

    * Kubelet captures each container’s `stdout`/`stderr` and writes them to files:

      ```
      /var/log/pods/<pod-uid>/<container-name>/0.log
      /var/log/containers/<pod_name>_<namespace>_<container_name>-<container_id>.log
      ```
    * A node-level daemon (e.g., Fluentd or Fluent Bit) can `tail` these host files.

2. **DaemonSet Deployment**

    * A DaemonSet ensures one pod per node. The Pod mounts the host’s `/var/log/containers` and `/var/log/pods` directories:

      ```yaml
      apiVersion: apps/v1
      kind: DaemonSet
      metadata:
        name: fluentd
        namespace: logging
      spec:
        selector:
          matchLabels:
            k8s-app: fluentd-logging
        template:
          metadata:
            labels:
              k8s-app: fluentd-logging
          spec:
            tolerations:
              - operator: "Exists"               # Tolerate all taints so that log collector runs on all nodes
            containers:
              - name: fluentd
                image: fluent/fluentd-kubernetes-daemonset:v1.14.0
                env:
                  - name: FLUENT_ELASTICSEARCH_HOST
                    value: "elasticsearch.logging.svc.cluster.local"
                volumeMounts:
                  - name: varlog
                    mountPath: /var/log
                  - name: varlibdockercontainers
                    mountPath: /var/lib/docker/containers
                    readOnly: true
            volumes:
              - name: varlog
                hostPath:
                  path: /var/log
              - name: varlibdockercontainers
                hostPath:
                  path: /var/lib/docker/containers
      ```
    * Fluentd (or Fluent Bit) reads each container’s log file, applies parsing/filtering, and forwards to a backend (Elasticsearch, Graylog, etc.).

#### 1.2.2 Sidecar Container Pattern

* In certain cases, you may run a sidecar container alongside your application container in the same Pod, sharing a `volumeMount` (e.g., an `emptyDir`) where the application writes logs. The sidecar runs a log forwarder that reads logs from that shared mount and ships them.
* **Pros**:

    * Application logs can be written to a standardized location (e.g., `/var/log/app.log`).
    * Parsing/forwarding logic is encapsulated per‐Pod.
* **Cons**:

    * Requires modifying each Pod spec to add the sidecar.
    * Higher resource overhead (one log agent per Pod).
    * Less flexible when scaling or deploying new applications.

In most production setups, the DaemonSet pattern is preferred for cluster‐wide log aggregation.

---

### 1.3 ELK Stack (Elasticsearch, Logstash/Beats, Kibana)

The **ELK Stack** is one of the most popular open‐source solutions for log aggregation, search, and visualization:

1. **Elasticsearch**

    * A distributed, RESTful search and analytics engine.
    * Stores log documents as JSON records and provides powerful full-text search, filtering, and aggregations.
    * Typically run as a StatefulSet or Deployment with persistent storage via PersistentVolumeClaims (PVs).

2. **Logstash** (or **Beats**)

    * **Logstash**: A log processor that ingests, transforms, and forwards data to Elasticsearch. It uses a pipeline of inputs (e.g., file, syslog), filters (grok, date parsing), and outputs (Elasticsearch).
    * **Beats** (e.g., Filebeat, Metricbeat, Packetbeat): Lightweight shippers that run on every node, read local files (Filebeat) or metrics (Metricbeat), and ship data directly to Elasticsearch.
    * In K8s, you typically use **Fluentd** or **Fluent Bit** instead of Logstash (lighter, easier integration). But if you prefer the Elastic Beats ecosystem, you can run Filebeat as a DaemonSet.

3. **Kibana**

    * A web‐based UI to visualize logs stored in Elasticsearch.
    * Enables creating dashboards, graphs, and running ad-hoc queries.

#### 1.3.1 Deploying the ELK Stack in Kubernetes

* **Elasticsearch Cluster**:

  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: elasticsearch
    namespace: logging
  spec:
    serviceName: "elasticsearch"
    replicas: 3
    selector:
      matchLabels:
        app: elasticsearch
    template:
      metadata:
        labels:
          app: elasticsearch
      spec:
        containers:
          - name: elasticsearch
            image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
            resources:
              limits:
                memory: "2Gi"
                cpu: "1000m"
              requests:
                memory: "1Gi"
                cpu: "500m"
            env:
              - name: cluster.name
                value: "k8s-logs"
              - name: node.name
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: discovery.seed_hosts
                value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
              - name: cluster.initial_master_nodes
                value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
            ports:
              - containerPort: 9200
                name: rest
              - containerPort: 9300
                name: inter-node
            volumeMounts:
              - name: storage
                mountPath: /usr/share/elasticsearch/data
    volumeClaimTemplates:
      - metadata:
          name: storage
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
  ```
* **Kibana Deployment**:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: kibana
    namespace: logging
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: kibana
    template:
      metadata:
        labels:
          app: kibana
      spec:
        containers:
          - name: kibana
            image: docker.elastic.co/kibana/kibana:7.17.0
            env:
              - name: ELASTICSEARCH_URL
                value: http://elasticsearch.logging.svc.cluster.local:9200
            ports:
              - containerPort: 5601
  ```
* **Log Forwarder (Fluentd)**:

    * As shown in **Section 1.2.1**, deploy Fluentd as a DaemonSet.
    * Fluentd’s config (`fluent.conf`) includes an output plugin for Elasticsearch:

      ```xml
      <match **>
        @type elasticsearch
        @id out_elasticsearch
        host elasticsearch.logging.svc.cluster.local
        port 9200
        logstash_format true
        logstash_prefix kubernetes-logs
        include_tag_key true
        type_name _doc
        flush_interval 5s
      </match>
      ```
    * Fluentd reads each log line from container log files, applies filters (e.g., parse JSON vs. plain text), and writes to Elasticsearch index named `kubernetes-logs-YYYY.MM.DD`.
    * Kibana is then pointed at `kubernetes-logs-*` index pattern to visualize logs.

#### 1.3.2 Advantages & Challenges

* **Pros**:

    * Highly scalable and mature—Elasticsearch can store terabytes of logs and handle complex queries.
    * Kibana’s rich UI for building dashboards, creating alerts (Watcher), and exploring logs.
    * Beats ecosystem for light resource usage on nodes (e.g., Filebeat).

* **Cons**:

    * Elasticsearch is memory‐ and CPU-intensive—requires careful resource planning.
    * Operational complexity: scaling Elasticsearch, managing shards, backups, and upgrades can be challenging.
    * Licensing: the Elastic Stack’s advanced features (e.g., security, alerting) require a commercial license (though the basic features remain open source).

---

### 1.4 Graylog

**Graylog** is another widely used centralized log management platform. It consists of:

1. **Graylog Server (Backend)**

    * Receives logs (via GELF, Beats, or Syslog), parses them, and stores them in Elasticsearch (or OpenSearch).
    * Provides a web UI for searching, dashboards, and alerting.
    * Supports pipelines for log transformation and enrichment.

2. **MongoDB**

    * Stores Graylog’s metadata and configuration (e.g., index sets, user information).

3. **Elasticsearch (or OpenSearch)**

    * Serves as the primary storage for log events.

4. **Sidecar (Collector)**

    * **Graylog Sidecar**: A lightweight agent that can run on each node, managing log shippers (e.g., Filebeat, NXLog, Winlogbeat) and their configurations (coming from Graylog’s UI).

#### 1.4.1 Deploying Graylog in Kubernetes

* **Elasticsearch & MongoDB**:

    * Typically deployed as StatefulSets (similar to ELK) with persistent volumes.
* **Graylog Server**:

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: graylog
    namespace: logging
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: graylog
    template:
      metadata:
        labels:
          app: graylog
      spec:
        containers:
          - name: graylog
            image: graylog/graylog:4.3
            env:
              - name: GRAYLOG_PASSWORD_SECRET
                valueFrom:
                  secretKeyRef:
                    name: graylog-secret
                    key: password_secret
              - name: GRAYLOG_ROOT_PASSWORD_SHA2
                valueFrom:
                  secretKeyRef:
                    name: graylog-secret
                    key: root_password_sha2
              - name: GRAYLOG_HTTP_EXTERNAL_URI
                value: "http://graylog.logging.svc.cluster.local:9000/"
            ports:
              - containerPort: 9000    # Web UI
              - containerPort: 12201   # GELF TCP input
            volumeMounts:
              - name: graylog-data
                mountPath: /usr/share/graylog/data
        volumes:
          - name: graylog-data
            emptyDir: {}
  ```

    * **Secrets** hold `GRAYLOG_PASSWORD_SECRET` and `GRAYLOG_ROOT_PASSWORD_SHA2` (SHA-256 of admin password).
* **Sidecar DaemonSet (optional)**:

    * Deploy the Graylog Sidecar DaemonSet on each node to manage Filebeat or Fluent Bit configurations.

#### 1.4.2 Ingesting Logs

* **GELF Input**: Applications or log forwarders can send logs in GELF format directly to Graylog’s TCP/UDP input on port 12201.
* **Beats Output**: Configure Filebeat on each node (via Sidecar) to send logs to Graylog using the `graylog` output plugin.
* **Syslog**: Graylog can accept syslog messages from pods or host.

#### 1.4.3 Searching and Alerting

* Graylog’s web UI provides:

    * **Streams**: Logical flows of log events (e.g., “nginx access logs”, “application errors”).
    * **Pipelines**: DSL to parse, enrich, or drop messages before indexing.
    * **Alert Conditions**: Define threshold‐based alerts (e.g., “> 100 errors in 5 minutes”) and notifications (email, Slack, etc.).

#### 1.4.4 Advantages & Challenges

* **Pros**:

    * Intuitive UI for searching logs, building dashboards, and creating alerts.
    * Flexible pipelines for log processing.
    * Can scale Elasticsearch independently for storage needs.
* **Cons**:

    * Requires managing an additional component (MongoDB).
    * Slightly steeper learning curve around pipelines compared to Kibana.
    * Relies on Elasticsearch for storage, so still inherits some operational complexity.

---

### 1.5 Comparison: ELK vs. Graylog

| Feature                | ELK Stack                                                                                     | Graylog                                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **UI & Dashboards**    | Kibana: highly customizable, rich visualizations; requires learning Kibana.                   | Graylog UI: focused on log streams, alerts, and pipelines; easier for “out‐of‐the‐box” log management. |
| **Architecture**       | Elasticsearch (storage), Logstash/Beats/Fluentd (processing), Kibana (UI).                    | Graylog (processing + UI), MongoDB (metadata), Elasticsearch (storage), Sidecar (agent management).    |
| **Log Processing**     | Logstash pipelines, Beats modules, Fluentd plugins.                                           | Graylog pipelines with a DSL; Sidecar can manage Beats/Fluentd.                                        |
| **Alerting**           | Watcher (commercial) or ElastAlert community plugin.                                          | Built‐in alerting based on stream conditions.                                                          |
| **Scalability**        | Highly scalable Elasticsearch cluster; can be complex to tune shard allocation.               | Scalable via Elasticsearch; Graylog nodes can be clustered.                                            |
| **Resource Overhead**  | Logstash is resource‐heavy; Beats/Fluentd lighter. Elasticsearch requires significant memory. | MongoDB adds extra overhead; Graylog nodes moderate, but Elasticsearch still heavy.                    |
| **Ease of Deployment** | Many Helm charts and operators exist (e.g., Elastic Cloud on Kubernetes).                     | Official Helm chart or manual manifests; less “official” support than Elastic.                         |
| **Cost & Licensing**   | Basic features open source; advanced features (security, alerting) require license.           | Open source core; enterprise features under commercial license.                                        |
| **Learning Curve**     | Moderate to steep (Elasticsearch query language, Kibana DSL, Logstash config).                | Moderate (Graylog’s pipeline language, stream/alert concepts).                                         |

---

## 2. Central Monitoring of Cluster Resources

While logs give insight into application behavior and failures, **metrics** provide quantitative data about resource usage, performance, and capacity. A typical K8s monitoring stack comprises:

* **Prometheus**: Time-series database and scraping engine.
* **AlertManager**: Handles alert notifications (email, Slack, PagerDuty).
* **Grafana**: Visualization layer—dashboards, charts, alerts.

### 2.1 Prometheus Architecture

Prometheus operates on a **pull model**: it periodically scrapes metrics from instrumented targets over HTTP. Key components:

1. **Prometheus Server**

    * **Scrapes**: Configured endpoints (e.g., `/metrics`) on services, nodes, or exporters.
    * **Stores**: Time-series data locally (on disk) with built-in retention policies.
    * **Query Language**: PromQL allows ad-hoc querying and alert rule definitions.

2. **Exporters**

    * **node\_exporter**: Runs on each node (DaemonSet), exporting host‐level metrics (CPU, memory, disk I/O, network).
    * **kube-state-metrics**: Exposes cluster‐state metrics (Deployments, DaemonSets, ReplicaSets, Pod status) from the Kubernetes API.
    * **cAdvisor** (built into Kubelet): Exposes container‐level metrics (resource usage per container).
    * **Application custom exporters**: Many applications (Redis, MySQL, NGINX) have native Prometheus exporters.

3. **Service Discovery**

    * Prometheus discovers scrape targets via K8s service discovery (based on Pod labels, annotations, or `Endpoints` objects).
    * Configuration example in `prometheus.yml`:

      ```yaml
      scrape_configs:
        - job_name: 'kubernetes-node'
          kubernetes_sd_configs:
            - role: node
          relabel_configs:
            - source_labels: [__meta_kubernetes_node_label_role]
              regex: "worker"
              action: keep
            - source_labels: [__meta_kubernetes_node_name]
              target_label: instance
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              regex: "true"
              action: keep
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
              regex: "(.+)"
              target_label: __metrics_path__
            - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
              regex: "([^:]+)(?::\\d+)?;([0-9]+)"
              replacement: "$1:$2"
              target_label: __address__
      ```
    * Pods wishing to be scraped can annotate:

      ```yaml
      metadata:
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/port: "8080"
          prometheus.io/path: "/metrics"
      ```

### 2.2 AlertManager

**AlertManager** ingests alerts generated by Prometheus based on **alerting rules** and dispatches notifications to various receivers.

1. **Defining Alerting Rules**

    * In Prometheus’s `rules.yml`:

      ```yaml
      groups:
        - name: node-alerts
          rules:
            - alert: NodeMemoryHigh
              expr: node_memory_Active_bytes / node_memory_MemTotal_bytes * 100 > 90
              for: 5m
              labels:
                severity: warning
              annotations:
                summary: "Node memory usage > 90%"
                description: "Node {{ $labels.instance }} memory usage is above 90% for more than 5 minutes."
        - name: pod-down
          rules:
            - alert: PodCrashLooping
              expr: increase(kube_pod_container_status_restarts_total[10m]) > 5
              for: 0m
              labels:
                severity: critical
              annotations:
                summary: "Pod {{ $labels.pod }} is crash looping"
      ```
    * Prometheus evaluates these every evaluation interval (default 1m). If an expression is true for the specified `for` duration, it sends an alert to AlertManager.

2. **Configuring AlertManager**

    * AlertManager’s `alertmanager.yml`:

      ```yaml
      global:
        resolve_timeout: 5m
      route:
        group_by: ['alertname', 'priority']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 4h
        receiver: 'team-email'
        routes:
          - match:
              severity: critical
            receiver: 'pagerduty'
      receivers:
        - name: 'team-email'
          email_configs:
            - to: 'devops@example.com'
              from: 'alertmanager@example.com'
              smarthost: 'smtp.example.com:587'
        - name: 'pagerduty'
          pagerduty_configs:
            - service_key: '<YOUR_SERVICE_KEY>'
              severity: 'critical'
      inhibit_rules:
        - source_match:
            severity: 'critical'
          target_match:
            severity: 'warning'
          equal: ['alertname', 'instance']
      ```
    * **Routes** let you send critical alerts to PagerDuty and warnings to email.
    * **Inhibition** prevents redundant notifications (if a critical alert is firing, suppress related warnings).

3. **Integrating with Prometheus**

    * In `prometheus.yml`:

      ```yaml
      alerting:
        alertmanagers:
          - static_configs:
              - targets:
                - 'alertmanager.logging.svc.cluster.local:9093'
      ```

### 2.3 Grafana

**Grafana** is the de facto standard for visualizing Prometheus metrics:

1. **Data Source Configuration**

    * In Grafana’s UI (or via a ConfigMap/CRD for automated setup), add Prometheus as a data source (URL: `http://prometheus.logging.svc.cluster.local:9090`).
2. **Dashboards**

    * Create or import community dashboards (e.g., “Kubernetes Cluster Monitoring” by the Prometheus community).
    * Key dashboards include:

        * **Cluster Overview**: Node CPU/memory, cluster CPU/memory, Pod status breakdown.
        * **Node Exporter**: Detailed node metrics (disk I/O, network traffic, CPU per core).
        * **Kube-State-Metrics**: Deployments replicas, daemonset status, statefulset usage.
        * **Application-Specific Dashboards**: Custom metrics exposed by applications.
3. **Alerting in Grafana**

    * Grafana (v8+) supports built-in alerting. You can define alert rules (in a dashboard panel) and send notifications (Slack, email).
    * Alternatively, continue to use Prometheus → AlertManager for alert “source of truth” and use Grafana’s “Notification Channels” to route alerts to teams.

#### 2.3.1 Deploying Grafana

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:8.5
          env:
            - name: GF_SECURITY_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: grafana-admin-secret
                  key: admin-password
          ports:
            - containerPort: 3000
```

* Expose via a Service (`NodePort`/`LoadBalancer`/Ingress) so users can access the Grafana UI at `grafana.monitoring.svc.cluster.local:3000`.

---

### 2.4 Reference Architecture: kube-prometheus Stack

For a production-ready setup, consider deploying the **kube-prometheus** stack (maintained by the Prometheus community), which includes:

* **Prometheus Operator**: Simplifies management of Prometheus clusters, ServiceMonitors, and alert rules.
* **Alertmanager Operator**: Manages AlertManager deployment and configuration.
* **Grafana Operator**: Manages Grafana instances, data sources, dashboards.
* **Node Exporter DaemonSet**, **kube-state-metrics Deployment**, **Prometheus** StatefulSet(s), **AlertManager** StatefulSet.

You can install kube-prometheus via the [Prometheus Community Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) or by applying the manifests from the [kube-prometheus GitHub repo](https://github.com/prometheus-operator/kube-prometheus).

Key benefits:

* **CRD-Driven Configuration**: Define `ServiceMonitor` CRDs to specify which Pods/Services to scrape metrics from (instead of hand-editing `prometheus.yml`).
* **Automated TLS**: Integrates with cert-manager to generate certificates for alertmanager, Prometheus, or Grafana.
* **RBAC & Security**: Operators generate all necessary RBAC (Roles, RoleBindings) automatically.
* **Default Dashboards & Alerts**: Out-of-the-box dashboards for cluster health, node metrics, kube-apiserver, and more, plus recommended alert rules.

---

## 3. Putting It All Together

A typical logging and monitoring architecture in Kubernetes might look like this:

```
      +-----------------------------+
      |        Kubernetes           |
      |     Control Plane Nodes     |
      +-----------------------------+
                   ▲  ▲
Reporting logs/metrics
                   |  |
  +----------------+  +-------------------+
  |                                      |
  |      Worker Node (Node 1)            |
  |  +-----------------------------+     |
  |  |  Application Pod (app-A)    |     |
  |  +-----------------------------+     |
  |  |     stdout/stderr → /var/   |     |
  |  | /var/lib/docker/containers/ |     |
  |  +-----------------------------+     |
  |  |  Fluentd (DaemonSet)        |     |<-- Sends logs to ES or Graylog
  |  |  Filebeat/Fluent Bit/etc.   |     |
  |  +-----------------------------+     |
  |  |  node_exporter (DaemonSet)   |    |
  |  +-----------------------------+     |<-- Exposes metrics on /metrics
  +--------------------------------------+     |
                                              |
  +----------------+  +--------------------+   |
  |                |  |                    |   |
  |  Worker Node 2 |  |  Worker Node 3     |   |
  |  ...           |  |  ...               |   |
  +----------------+  +--------------------+   |
                   ▲  ▲                       |
     Scraping metrics  |                        |
        by Prometheus  |                        |
                      |                        |
              +---------------------------------------------+
              |                  Monitoring Stack           |
              |                                             |
              |  +-------------+   +----------------------+  |
              |  | Prometheus  |   |   AlertManager       |  |
              |  +-------------+   +----------------------+  |
              |        ▲                   ▲                 |
              |        | Sends alerts     | Notifies Teams   |
              |        | (via Alert Rules)| (email, Slack)   |
              |        |                   |                 |
              |  +-------------+   +----------------------+  |
              |  |   Grafana   |   |     Elasticsearch    |  |
              |  +-------------+   +----------------------+  |
              |        ▲                   ▲                 |
              |        | Visualizes        | Stores Logs      |
              |        | Metrics/Dashboards | (via Fluentd)   |
              +---------------------------------------------+
```

* **On Each Node**:

    * **Fluentd/Fluent Bit** (DaemonSet) tails logs from `/var/log/containers` and forwards to Elasticsearch (or Graylog).
    * **Filebeat** (if using Beats) can be deployed as a DaemonSet to ship logs via Beats protocols.
    * **Node Exporter** (DaemonSet) exports Prometheus metrics about node resources.
    * **kube-state-metrics** runs as a Deployment to expose cluster state metrics (e.g., Deployment availability, Pod counts).

* **Prometheus** (Deployment/StatefulSet) scrapes:

    * **Node Exporter** on each Node.
    * **cAdvisor** (built into each Kubelet) for per-container metrics.
    * **kube-state-metrics** for cluster-level state.
    * **Application Metrics** via annotations or custom exporters.

* **AlertManager** receives alerts from Prometheus and routes them to configured channels (email, Slack, PagerDuty).

* **Grafana** connects to Prometheus as a data source and provides dashboards (e.g., Node CPU/memory usage, Pod restarts, API server latencies).

* **Elasticsearch / Graylog** stores logs from Fluentd/Filebeat.

    * **Kibana (with Elasticsearch)** or **Graylog UI** is used to search logs, drill down, create dashboards, and set up log‐based alerts.

---

## 4. Summary

1. **Log Collection**

    * Use a **DaemonSet** (Fluentd, Fluent Bit, or Filebeat) to tail container log files under `/var/log/containers` and forward them to a centralized backend.
    * The **ELK Stack** (Elasticsearch, Logstash/Beats, Kibana) is the canonical open-source solution; use Beats or Fluentd to ship logs to Elasticsearch, then visualize in Kibana.
    * **Graylog** offers a similar architecture (Graylog Server + MongoDB + Elasticsearch), with its own pipelines and built-in alerting.

2. **Central Monitoring**

    * **Prometheus** scrapes metrics from **node\_exporter**, **kube-state-metrics**, and application endpoints. It stores time-series data and evaluates alert rules.
    * **AlertManager** manages Prometheus alerts, routing them to Slack, email, PagerDuty, etc.
    * **Grafana** visualizes metrics via dashboards and can also send notifications.

3. **Reference Implementation**

    * A production-grade “kube-prometheus” stack (Prometheus Operator + ServiceMonitors + AlertManager + Grafana) simplifies deployments and best practices.
    * Logging can be deployed via the official “Elastic” or “Graylog” Helm charts/operators, ensuring persistence, clustering, and security.

By combining cluster-wide DaemonSet log forwarding (ELK or Graylog) with a robust Prometheus/AlertManager/Grafana monitoring stack, you achieve end-to-end observability: application logs, system metrics, alerts, and dashboards—all centralized and easily manageable.


