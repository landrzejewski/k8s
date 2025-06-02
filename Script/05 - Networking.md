## Understanding Kubernetes Services

Kubernetes **Services** are an abstraction that defines a logical set of Pods and a policy to access them—often referred to as a microservice’s “front door.” Because Pods are ephemeral (they can be created, destroyed, or rescheduled at any time), Services provide stable networking endpoints that allow clients (other Pods, external users, or external systems) to reliably reach those Pods even as the underlying Pods come and go. This article covers:

1. The Role of Services in Kubernetes
2. How Services Select Pods
3. Service Types and Use Cases

    * ClusterIP
    * NodePort
    * LoadBalancer
    * ExternalName
    * Headless Services
4. Service Discovery and DNS
5. Service Endpoints and `kube-proxy`
6. Advanced Features

    * Session Affinity
    * Annotations (for external integrations)
    * Service Topology and External Traffic Policy
7. Example Service Manifests
8. Best Practices

---

## 1. The Role of Services in Kubernetes

When you deploy an application, you typically run it as one or more Pods managed by a controller (Deployment, StatefulSet, etc.). Each Pod receives its own IP address on the cluster’s Pod network. However, if a Pod crashes, is evicted, or is rescheduled onto another node, its IP changes. If your application depended on directly referencing Pod IPs, any change would break connectivity.

A **Service** solves this by providing:

* **A stable virtual IP (ClusterIP)** that persists as long as the Service object exists.
* **A DNS name** (within the cluster DNS) that resolves to the Service’s IP.
* **Load-balanced routing** to one or more Pods (selected by label) behind the scenes.

In effect, consumers reference the Service IP (or DNS name) instead of individual Pod IPs. Kubernetes handles routing traffic to the currently available Pods that match the Service’s selector.

> **Key Benefits**
>
> * **Decoupling:** Clients only need to know “service name,” not Pod IPs.
> * **Load Balancing:** Traffic can be distributed evenly (by default) across all matching Pods.
> * **Resiliency:** When Pods fail or scale, the Service endpoints list adjusts automatically, keeping traffic flowing.

---

## 2. How Services Select Pods

At its core, a Service defines:

* **`selector`**: A set of key/value label matches. Any Pod with labels matching that selector is considered “part of” the Service’s endpoint set.
* **`ports`**: The port(s) that the Service listens on (inside the cluster). Optionally, a different `targetPort` for each container port on the Pods.
* **`type`**: The networking mode (ClusterIP, NodePort, LoadBalancer, etc.).

For example, if you have Pods labeled `app=web` and `tier=frontend`, you might create:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: web
    tier: frontend
  ports:
    - port: 80
      targetPort: 8080
```

* The Service listens on port 80.
* Any Pod with labels `app=web` and `tier=frontend` is automatically “selected” as an endpoint; traffic to the Service’s IP on port 80 is forwarded to port 8080 on those Pods.

> **Important**: If a Pod’s labels no longer match (for instance, you change a Pod’s `app` label in a Deployment), the Service removes that Pod from its endpoints list automatically.

---

## 3. Service Types and Use Cases

Kubernetes Services offer several types, each serving different networking needs:

### 3.1 ClusterIP (Default)

* **Definition**: Provides an internal-only IP within the cluster.
* **Use Case**: When you want Pods within the cluster to communicate with each other (e.g., a front-end Deployment talks to a `backend` Service).
* **Behavior**:

    * A ClusterIP Service allocates a virtual IP (often in the `10.0.0.0/16` range, depending on your cluster’s configuration).
    * Other Pods reference it via that IP (or its DNS name, e.g., `my-service.default.svc.cluster.local`).
    * The Service does not expose any external ports outside the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
```

### 3.2 NodePort

* **Definition**: Exposes the Service on a static port (the “NodePort”) on each Node’s IP.
* **Use Case**: When you need basic external access without a cloud load balancer. For example, to expose your Service on `<NodeIP>:<NodePort>` and allow external clients to connect directly.
* **Behavior**:

    1. Kubernetes allocates (or you specify) a port from the **NodePort range** (default: 30000–32767).
    2. The Service still gets a ClusterIP internally.
    3. Each Node in the cluster opens that same NodePort on all its network interfaces. Traffic to any Node’s external IP at that port is proxied to the Service’s Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80        # Service port
      targetPort: 8080
      nodePort: 30080 # Static NodePort (must be in 30000–32767)
```

* Now, if you have three nodes (`10.1.1.10`, `10.1.1.11`, `10.1.1.12`), traffic to `10.1.1.10:30080`, `10.1.1.11:30080`, or `10.1.1.12:30080` will be routed to one of the `frontend` Pods.

### 3.3 LoadBalancer

* **Definition**: Creates a cloud-provider load balancer (if supported) with a public IP, forwarding to your Service’s NodePort/ClusterIP.
* **Use Case**: When running in a supported cloud environment (GKE, EKS, AKS, etc.) and you want a managed load balancer that distributes traffic across all healthy Pods.
* **Behavior**:

    1. Kubernetes allocates or provisions a Load Balancer from your cloud provider.
    2. That load balancer receives an external IP (or DNS name) accessible from the Internet (or your VPC).
    3. It forwards traffic to the Service’s NodePort on each node.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

After applying this, `kubectl get svc frontend-lb` will show an `EXTERNAL-IP`. Clients from outside can connect directly to that IP (or its DNS alias), and the cloud provider’s LB automatically spreads traffic across nodes and healthy Pods.

### 3.4 ExternalName

* **Definition**: Maps a Service to a DNS name (outside the cluster) by returning a CNAME record. There is no proxying—clients receive a DNS response pointing to the external name.
* **Use Case**: When you want Pods inside Kubernetes to reach an external resource via the same Service name, without changing their code. E.g., connecting to a managed database or an external API.
* **Behavior**:

    * The Service’s `spec.externalName` (e.g., `mydb.example.com`) is returned as a CNAME record for any DNS query to `my-service.default.svc.cluster.local`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.net
```

* Any Pod querying `external-db` gets back a CNAME to `db.example.net` and queries that domain—bypassing cluster proxying entirely.

### 3.5 Headless Service

* **Definition**: A Service with `clusterIP: None` (no virtual IP).
* **Use Case**: When you need direct, client‐side load balancing or service discovery (for example, in a StatefulSet for a database cluster), rather than cluster‐side proxying.
* **Behavior**:

    1. Kubernetes does *not* allocate a ClusterIP.
    2. Instead, it creates DNS “A” records for each Pod IP matching the selector.
    3. Clients can do DNS lookups (e.g., a multi‐address SRV) and receive the list of Pod IPs, then choose how to connect—useful for StatefulSets or custom client‐side logic.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
    - port: 9092
      targetPort: 9092
```

If there are three Pods (`kafka-0`, `kafka-1`, `kafka-2`) with label `app=kafka`, DNS entries become:

```
kafka-0.kafka.default.svc.cluster.local → Pod IP of kafka-0
kafka-1.kafka.default.svc.cluster.local → Pod IP of kafka-1
kafka-2.kafka.default.svc.cluster.local → Pod IP of kafka-2
```

No single “service IP” exists; clients perform round‐robin at the DNS level or run their own logic (for example, to connect to a particular broker in a Kafka cluster).

> **When to Use Headless**
>
> * Stateful applications (databases, message queues, etc.) where each Pod has a unique identity and clients need to connect directly to a specific Pod.
> * Custom load‐balancing logic (e.g., clients read the full list of endpoints and choose one).

---

## 4. Service Discovery and DNS

Kubernetes includes a built‐in DNS server (CoreDNS) that automatically creates DNS entries for Services. By default, each Service in namespace `N` named `my-service` gains:

```
my-service.N.svc.cluster.local → ClusterIP
```

Additionally:

* **A and SRV Records**

    * For a **ClusterIP**, DNS returns a single A record with the Service’s IP.
    * For a **Headless Service**, DNS returns multiple A records—one for each Pod IP matching the selector.
    * Kubernetes also creates SRV records for each port, allowing clients to discover port numbers dynamically.

* **Pod DNS**
  Each Pod gets its own DNS name:

  ```
  <pod-ip-address>.<namespace>.pod.cluster.local
  ```

  but Pods rarely use their own DNS names; most communication goes via Services.

* **Environment Variables** (Legacy)
  In older Kubernetes versions, when a Pod started, the kubelet injected environment variables for each Service (e.g., `MY_SERVICE_SERVICE_HOST`, `MY_SERVICE_SERVICE_PORT`). This method is largely deprecated; DNS is the preferred mechanism.

---

## 5. Service Endpoints and `kube-proxy`

### 5.1 Endpoints Object

Under the hood, when you create a Service with a label selector, Kubernetes automatically creates an **Endpoints** object with the same name as the Service. This Endpoints object lists the IP addresses and ports of all Pods currently matching the selector. For example, if you have:

```bash
kubectl get svc frontend-svc
# NAME           TYPE        CLUSTER-IP   PORT(S)   AGE
# frontend-svc   ClusterIP   10.0.0.42    80/TCP    15m
```

Then:

```bash
kubectl get endpoints frontend-svc
# NAME           ENDPOINTS                    AGE
# frontend-svc   10.1.1.12:8080,10.1.2.15:8080 15m
```

* Whenever a Pod is created, deleted, or has its labels changed, the Endpoints list is updated dynamically.
* If no Pod matches the selector, the Endpoints list is empty (no traffic is sent to anywhere).

### 5.2 `kube-proxy` and Traffic Routing

Each Kubernetes node runs a **kube-proxy** process, which watches the Kubernetes API for Service and Endpoints changes. It ensures that:

1. Traffic sent to the Service’s IP (ClusterIP) or NodePort is forwarded to one of the Pod IPs in the Endpoints list.
2. Traffic load-balances across all available endpoints. By default, kube-proxy uses **iptables** or **IPVS** to implement:

    * **Round-robin** (iptables mode) or **more efficient** load-balancing algorithms (IPVS mode).

**Flow (ClusterIP Service)**:

* A client Pod on Node A sends a TCP packet to the Service’s ClusterIP (e.g., `10.0.0.42:80`).
* **kube-proxy** on Node A sees the destination IP, rewrites the packet’s destination to one of the Pod IPs (e.g., `10.1.1.12:8080`), and forwards it.
* The packet reaches the target Pod; the response goes back through the same kube-proxy, which rewrites the source/destination so the client sees the Service IP as the endpoint.

**Flow (NodePort Service)**:

* A request from outside `NodeA:30080` hits Node A’s IP on port 30080.
* kube-proxy on Node A intercepts that port, picks an available Pod IP (could be on Node A or Node B), and routes the traffic accordingly.

> **Note on External Traffic Policy**
>
> * **`Cluster` (default)**: kube-proxy can load-balance external NodePort/LB traffic to any Pod in the cluster, regardless of which node the Pod runs on.
> * **`Local`**: If you set `externalTrafficPolicy: Local`, kube-proxy only routes traffic to Pods on the same node. This preserves the original client IP (useful for logging or client-based decisions), but if no Pod exists locally, traffic to that node is dropped.

---

## 6. Advanced Service Features

### 6.1 Session Affinity

By default, each request to a Service can be routed to any Pod in the endpoint list. In some scenarios—such as stateful applications without sticky sessions at the application layer—you may want to guarantee that all requests from a particular client IP go to the same Pod. Kubernetes supports **session affinity** via:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # (default 3 hours)
```

* When `ClientIP` is enabled, kube-proxy inserts an `iptables` rule that tracks the source IP and binds it to a single endpoint for the duration of the timeout.
* After `timeoutSeconds` of inactivity, subsequent requests from that client IP may go to a different Pod.

Use session affinity only when necessary—for example, a legacy application that cannot maintain state in an external store. Always weigh the trade-off: affinity can lead to uneven load distribution if some clients generate heavy traffic.

### 6.2 Service Annotations (Integrations)

Kubernetes allows adding annotations to Services that external controllers or Ingress controllers can interpret. Common annotations include:

* **Ingress or Load Balancer Customization**

    * Specify health check paths
    * Set SSL certificates
    * Adjust idle timeouts or algorithm selection

* **ExternalDNS Integration**

    * If using [ExternalDNS](https://github.com/kubernetes-sigs/external-dns), annotate a Service with `external-dns.alpha.kubernetes.io/hostname: example.com`. ExternalDNS then creates DNS A records in your cloud DNS provider (Route 53, Cloud DNS, etc.) pointing to the LoadBalancer IP.

* **Service Mesh or CNI Plugins**

    * Annotate Services to configure sidecar injection (e.g., Istio’s `sidecar.istio.io/inject: "true"`).
    * Use annotations to opt in/out of network policies or external traffic settings.

Always consult your cloud provider or network plugin documentation for specific annotation keys and values. Annotations do not alter built-in Service behavior but provide metadata for controllers listening to Service events.

### 6.3 Service Topology and External Traffic Policy

For Services exposed via `NodePort` or `LoadBalancer`, you can control how traffic from outside the cluster is routed to local Pods:

```yaml
spec:
  externalTrafficPolicy: Local
```

* **`Cluster` (default)**: External traffic can reach any Pod in the cluster, even if a Pod is on a different node than the one receiving the traffic. kube-proxy does an additional hop through the network.
* **`Local`**: Traffic is only forwarded to Pods on the same node. If a Node has no matching Pods (i.e., the Deployment scaled down all local Pods), that Node’s IP on the Service port rejects traffic (or the Load Balancer marks that node unhealthy). This preserves client source IP and can be beneficial when you need the real client IP for logging or when you want to minimize cross-node hops.

---

## 7. Example Service Manifests

Below are several examples to illustrate different Service types and configurations:

### 7.1 Basic ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-backend
  labels:
    app: api
spec:
  type: ClusterIP              # This is implicit if you omit `type`
  selector:
    app: api
  ports:
    - port: 5000               # Service port
      targetPort: 5000         # Pod containerPort
      protocol: TCP
```

* Internal-only, accessible to Pods in the same namespace.
* DNS: `api-backend.default.svc.cluster.local`.

### 7.2 NodePort Service with External Traffic Policy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  annotations:
    external-dns.alpha.kubernetes.io/hostname: www.example.com
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 31000        # Must be in 30000–32767 (cluster default)
      protocol: TCP
  externalTrafficPolicy: Local
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

* Exposed on `NodeIP:31000` on every node.
* `externalTrafficPolicy: Local` preserves source IP.
* `sessionAffinity` ensures clients “stick” to the same Pod.
* `external-dns` annotation tells ExternalDNS to create a DNS record `www.example.com` → `<Node External IPs>` (with health checks).

### 7.3 LoadBalancer Service (Cloud)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "6379"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-west-2:123456789012:certificate/abcdefg
spec:
  type: LoadBalancer
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
      protocol: TCP
```

* In AWS EKS, this creates an ELB listening on port 6379, forwarding TLS‐encrypted traffic to Redis Pods.
* Annotations configure health checks and SSL termination. (Annotation keys differ per cloud provider.)

### 7.4 ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  type: ExternalName
  externalName: db.legacy.example.net
```

* Any Pod querying `legacy-db.default.svc.cluster.local` receives a CNAME to `db.legacy.example.net`.
* There is no Kubernetes proxy or Endpoints list; the DNS server returns the external name directly.

---

## 8. Best Practices

1. **Use ClusterIP for Intra-Cluster Communication**

    * Unless you specifically need external exposure, keep Services as `ClusterIP`. This avoids inadvertently exposing internal services.

2. **Limit NodePort Usage**

    * Rely on `LoadBalancer` Services (in cloud environments) or Ingress Controllers rather than NodePorts, unless you have no other option. NodePorts expose your nodes directly and can conflict with firewall rules or cloud provider limitations.

3. **Prefer Headless Services for StatefulSets**

    * When deploying databases or clustered applications (e.g., Redis, Kafka, MongoDB), use a Headless Service so clients can perform DNS lookups of each Pod’s IP. This allows StatefulSets to maintain stable identities.

4. **Annotate Services for External Integrations**

    * If using a cloud load balancer, use provider‐specific annotations to configure health checks, port mappings, or SSL certificates.
    * If using ExternalDNS or Cert-Manager, annotate Services to keep DNS and certificates in sync automatically.

5. **Control External Traffic Policy**

    * For Services exposed externally, pick `Cluster` or `Local` based on whether you need to preserve client IPs.
    * If you use `Local`, ensure you have Pod replicas on every node behind the Service front, or the Service might drop traffic on nodes without local Pods.

6. **Leverage Session Affinity Sparingly**

    * Use `sessionAffinity: ClientIP` only when absolutely necessary (e.g., legacy apps without stateless architecture). Otherwise, load balancing across pods is more even and reliable.

7. **Monitor Endpoints Health**

    * Keep an eye on `kubectl get endpoints <svc-name>` to verify that the expected Pods appear. If `endpoints` is empty, the Service has no backends and will drop traffic.
    * Use `kubectl describe service <svc-name>` to see event messages indicating issues (e.g., no matching pods found).

8. **Namespace Segmentation**

    * Deploy Services in the same namespace as their frontend/backends.
    * Use network policies (in conjunction with Service selectors) to restrict which Pods can call which Services, improving security.

## Kubernetes Ingress: Managing External HTTP and HTTPS Access

An **Ingress** is a Kubernetes API object that provides a flexible way to expose HTTP and HTTPS routes from outside the cluster to services within the cluster. Unlike a Service of type `NodePort` or `LoadBalancer`—which allocates a port on every node or provisions a cloud load balancer—Ingress consolidates and manages routing rules, hostnames, and TLS termination in one place, making it simpler to host multiple web applications behind a single external IP([Kubernetes][1]).

---

### 1. What Is an Ingress?

* **Definition**: An Ingress is a collection of rules that define how external HTTP(S) traffic should be routed to backend services based on hostnames and paths. It does **not** itself serve traffic; instead, it requires an **Ingress Controller**—a Kubernetes component that watches Ingress resources and configures a proxy (such as NGINX, HAProxy, or Traefik) to implement the routing rules.
* **Purpose**:

    * Consolidate traffic management: Instead of creating multiple Services or LoadBalancers, you define all HTTP(S) routing in one object.
    * Support virtual hosting: Route `foo.example.com` to one Service and `bar.example.com` to another, even sharing a single external IP.
    * Centralize TLS: Terminate TLS at the Ingress level and offload certificates, freeing backend pods from managing TLS themselves.

Ingress was promoted to **stable** in Kubernetes v1.19 and is commonly used for HTTP/HTTPS workloads in most clusters([Kubernetes][1]).

---

### 2. Ingress Components

An Ingress setup involves two main pieces:

1. **Ingress Resource**:

    * A YAML object that defines a set of routing rules. These rules map hostnames and URL paths to Kubernetes Services. For example, you can specify that traffic to `shop.example.com/cart` goes to the `cart-service`, while `shop.example.com/checkout` goes to `checkout-service`.
    * Can also include TLS configuration to specify the secret containing TLS certificates and keys to use for one or more hostnames.

2. **Ingress Controller**:

    * A deployment (or DaemonSet) running inside the cluster responsible for:

        1. Watching the Kubernetes API for new or updated Ingress resources.
        2. Translating those rules into configurations for an underlying proxy (e.g., NGINX, Envoy, HAProxy).
        3. Setting up health checks, load balancing, and TLS termination based on Ingress annotations or CRDs.
    * Multiple implementations exist (NGINX Ingress Controller, Traefik, Istio’s ingress gateway, etc.). You must install at least one Ingress Controller; otherwise, Ingress resources have no effect. The controller listens on a Service (often of type `LoadBalancer` or `NodePort`) to accept incoming traffic and enforce the defined rules([Kubernetes][2]).

---

### 3. Anatomy of an Ingress Resource

Below is a basic Ingress manifest illustrating host- and path-based routing, including TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls-secret
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /cart
            pathType: Prefix
            backend:
              service:
                name: cart-service
                port:
                  number: 80
          - path: /checkout
            pathType: Prefix
            backend:
              service:
                name: checkout-service
                port:
                  number: 80
```

* **`spec.ingressClassName`**: Specifies which Ingress Controller should handle this resource (e.g., `nginx`). Without this, the cluster’s default Ingress Class is used.
* **`spec.tls`**:

    * **`hosts`**: A list of hostnames for which TLS should be enabled.
    * **`secretName`**: The name of the Kubernetes Secret that contains the TLS certificate (`tls.crt`) and private key (`tls.key`) for those hostnames.
* **`spec.rules`**:

    * Each rule has a **`host`** (e.g., `shop.example.com`) and an **`http.paths`** block mapping path prefixes to a backend Service and port.
    * **`pathType: Prefix`** ensures that any request whose URL path begins with `/cart` is routed to the `cart-service`.

When applied, an Ingress Controller (e.g., the NGINX Ingress Controller) updates its internal proxy configuration so that:

1. Requests arriving at its external endpoint for `shop.example.com:443` match the TLS certificate in `shop-tls-secret`.
2. Depending on the URL path:

    * `/cart/*` is forwarded to `cart-service:80`.
    * `/checkout/*` is forwarded to `checkout-service:80`.

All other hostnames or paths not matching any rule result in a `404 Not Found` or a default backend (if configured)([Kubernetes][1]).

---

### 4. Ingress Controller Implementations

The Ingress resource is just the “desired state”; you need a controller to realize it. Here are common options:

* **NGINX Ingress Controller**: Uses NGINX as the reverse proxy. Supports annotations to control timeouts, rewrites, SSL settings, and more.
* **Traefik**: A dynamic, Kubernetes-native proxy that directly reads Ingress objects to configure routing.
* **Istio Ingress Gateway**: Part of the Istio service mesh, uses Envoy for advanced traffic management.
* **HAProxy Ingress Controller**: Leverages HAProxy’s high performance for Ingress routing.
* **Cloud-Specific Controllers** (e.g., GKE’s GCE Ingress Controller for Google Cloud Load Balancers, AWS ALB Ingress Controller). These provision native cloud load balancers (e.g., ALB, ELB) and configure them based on Ingress rules.

Refer to the [Ingress Controllers documentation](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) for a comprehensive list and comparisons([Kubernetes][2]).

---

### 5. TLS Termination and Certificates

Ingress can terminate TLS (HTTPS) at the controller, acting as a TLS reverse proxy. Key points:

1. **Creating the TLS Secret**

    * Create a Secret of type `kubernetes.io/tls` containing:

        * `tls.crt`: PEM-encoded certificate (can bundle multiple certificates in a chain).
        * `tls.key`: PEM-encoded private key.
    * Example:

      ```bash
      kubectl create secret tls shop-tls-secret \
        --cert=shop.crt \
        --key=shop.key
      ```

2. **Reference in Ingress**

    * Under `spec.tls`, list the hostname (`shop.example.com`) and the `secretName` (`shop-tls-secret`).
    * If multiple hosts share the same certificate, list them all under `hosts` in the same TLS block.

3. **Fallback HTTPS Port**

    * When a request arrives on port 443, the Ingress Controller uses the matching certificate to decrypt the traffic, then forwards the plaintext HTTP request to the backend Service.
    * If no TLS block matches the requested `Host` header, a client may see a TLS handshake error (or a default certificate), depending on the controller’s configuration.

4. **Annotations for SSL Settings**

    * Controllers often allow setting strict TLS versions, ciphers, HTTP → HTTPS redirects, and HSTS via annotations. For example, with NGINX Ingress:

      ```yaml
      metadata:
        annotations:
          nginx.ingress.kubernetes.io/ssl-redirect: "true"
          nginx.ingress.kubernetes.io/hsts-max-age: "15724800"
      ```

By centralizing TLS at the Ingress, backend Pods only need to serve HTTP, simplifying their configuration and offloading TLS management to the Ingress Controller([Kubernetes][1]).

---

### 6. Advanced Ingress Rules

#### 6.1 Path vs. Exact Matching

* **`pathType: Prefix`**: Matches all requests where the URL path begins with the specified value. For example, `/images` matches `/images/cat.jpg` and `/images/subdir/puppy.png`.
* **`pathType: Exact`**: Only matches if the path is exactly equal (e.g., `/images` matches only `/images`).
* **`pathType: ImplementationSpecific`**: Delegates matching behavior to the Ingress Controller (most default to prefix-like matching).

#### 6.2 Rewrites and Redirects

Many Ingress Controllers support **rewriting** or **redirecting** paths. For instance, if your backend expects requests at root (`/`), but you want users to see `/app` in the URL, you can rewrite:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

This configuration causes requests to `example.com/app/foo` to be forwarded as `example.com/foo` to `myapp-service`([Kubernetes][1]).

#### 6.3 Default Backend

If an Ingress does not have a rule matching a request’s hostname or path, traffic is usually routed to a **default backend**—a catch-all Service that returns a 404 or a custom error page. You can specify a default backend in the Ingress Controller’s main ConfigMap or by adding a single rule with an empty host and path `/` directing to a dummy Service.

---

### 7. Exposing Ingress to the Outside

The Ingress Controller itself must be exposed so that external traffic reaches it. Common methods include:

1. **LoadBalancer Service**

    * Deploy the Ingress Controller as a Deployment (or DaemonSet), then front it with a Service of type `LoadBalancer`. This allocates a cloud load balancer (e.g., an AWS ELB or GCP GLB) with a public IP.
    * Ingress rules (hostnames, TLS, paths) are managed by the controller; external clients connect to the load balancer IP or DNS name.

2. **NodePort Service**

    * If you lack a cloud load balancer, expose the Ingress Controller via a Service of type `NodePort`. This opens a high port (30000–32767) on every node.
    * You must then configure external DNS or users to point to `NodeIP:NodePort` for HTTP/HTTPS access.

3. **HostPort or HostNetwork**

    * For bare-metal clusters or advanced setups, the Ingress Controller Pods can bind directly to ports 80/443 on the host (`hostNetwork: true`) or use `hostPort: 80/443`. This makes the Host’s IP directly serve as the Ingress endpoint.
    * Be careful: This can lead to port conflicts if multiple services try to use the same host port.

Once the Ingress Controller is reachable, creating an Ingress Resource immediately makes routing rules active—no changes to other Services are required. Clients simply connect to the controller’s IP/DNS on standard HTTP/HTTPS ports, and the controller directs traffic based on the Ingress Resource definitions([Kubernetes][1], [Kubernetes][2]).

---

### 8. Example Workflow

1. **Deploy Backend Services**

    * Create two Deployments and corresponding Services (both `ClusterIP`):

        1. `cart-service` listening on port 80.
        2. `checkout-service` listening on port 80.

2. **Install an Ingress Controller**

    * For instance, install the NGINX Ingress Controller via Helm or a manifest. This creates:

        * A Deployment (or DaemonSet) running the NGINX Controller Pods.
        * A Service (type `LoadBalancer`) named `ingress-nginx` with an external IP.

3. **Create a TLS Secret**

   ```bash
   kubectl create secret tls shop-tls-secret \
     --cert=shop.example.com.crt \
     --key=shop.example.com.key
   ```

4. **Create the Ingress Resource**

   ```bash
   kubectl apply -f example-ingress.yaml
   ```

   (Use the manifest shown earlier in Section 3.)

5. **Test Routing**

    * Confirm that `kubectl get ingress example-ingress` shows an address (external IP).
    * Ensure DNS for `shop.example.com` points to that IP.
    * In a browser or via `curl`, visit:

        * `https://shop.example.com/cart` → should reach `cart-service`.
        * `https://shop.example.com/checkout` → should reach `checkout-service`.

6. **Monitor and Troubleshoot**

    * Use `kubectl describe ingress example-ingress` to see events (e.g., certificate errors, backend Service not found).
    * Check the Ingress Controller’s logs (`kubectl logs deployment/ingress-nginx-controller`) for errors or misconfigurations.

---

### 9. Best Practices

1. **Specify `ingressClassName` Explicitly**

    * In multi-controller clusters, setting `spec.ingressClassName` ensures the correct controller processes your Ingress. Otherwise, a default class may inadvertently capture or ignore your Ingress.

2. **Use Secure Defaults**

    * Enable automatic HTTP → HTTPS redirection by default, using annotations like `nginx.ingress.kubernetes.io/ssl-redirect: "true"`.
    * Set HSTS headers (`nginx.ingress.kubernetes.io/hsts: "true"`) to enforce secure connections.

3. **Limit Path Rewrites**

    * Keep rewrites minimal and predictable. Complex rewrites can cause confusion between client-visible URLs and backend paths.

4. **Leverage Annotations for Fine-Grained Control**

    * Each controller supports custom annotations for timeouts, buffering, rate limiting, or custom error pages. For instance, NGINX Ingress supports:

      ```yaml
      nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "120"
      ```
    * Always consult your chosen controller’s documentation for available annotations.

5. **Monitor Health of Backends**

    * Ensure that Services referenced by Ingress have healthy endpoints (`kubectl get endpoints <service-name>`). If no endpoints exist, Ingress returns 502/503.
    * Use readiness probes on Pods so that the controller only targets ready backends.

6. **Rate Limit and Security**

    * For public-facing applications, consider adding rate‐limiting annotations or external Web Application Firewalls (WAFs) (e.g., ModSecurity with NGINX Ingress).
    * Use authentication annotations (e.g., BasicAuth or OAuth) where supported.

7. **Version Control Ingress Manifests**

    * Store Ingress YAML files alongside other manifests in Git. Changes to hostnames, paths, or TLS configurations should be auditable.

8. **Leverage Gateway API for Advanced Use Cases**

    * If you require advanced traffic shaping (header-based routing, traffic weight distribution, multi‐cluster routing), evaluate the **Gateway API** as a more expressive alternative to Ingress. It provides protocol‐aware configuration aligned with organizational roles (infrastructure, cluster operator, developer).

[1]: https://kubernetes.io/docs/concepts/services-networking/ingress/?utm_source=chatgpt.com "Ingress - Kubernetes"
[2]: https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/?utm_source=chatgpt.com "Ingress Controllers - Kubernetes"

Kubernetes’ **Gateway API** is an evolution of the Ingress API, designed to provide a more expressive, extensible, and role-oriented framework for configuring L3–L7 traffic routing. Rather than a single monolithic Ingress resource, Gateway API introduces multiple CRDs (Custom Resource Definitions) that map naturally to organizational roles—separating infrastructure-level concerns (e.g., load balancer provisioning), cluster-wide policies (e.g., access controls), and application-level routing rules. Below is a detailed overview of Gateway API’s design principles, core kinds, lifecycle, and common use cases. ([Kubernetes][1], [Kubernetes][2])

## Gateway API

Gateway API was crafted around four key principles that address limitations of Ingress:

1. **Role-Oriented**

    * **Infrastructure Provider**: Manages global infrastructure, tunnels, multi-cluster concerns, or cloud provider integrations.
    * **Cluster Operator**: Controls cluster-wide policies, security, and network-level configurations.
    * **Application Developer**: Defines application-specific routing (e.g., HTTP paths, hostname mappings).
      By decoupling these contexts into separate API kinds, each stakeholder can operate within their scope without stepping on another’s domain. ([Kubernetes][1])

2. **Portable**

    * Gateway API is defined as Kubernetes CRDs (`GatewayClass`, `Gateway`, `HTTPRoute`, etc.) and is supported by multiple implementations—NGINX, Traefik, Contour, Istio, GKE, AWS ALB, and others.
    * A single set of YAML manifests can function across environments, irrespective of the underlying load-balancer implementation. ([Kubernetes][1])

3. **Expressive**

    * Compared to Ingress (where advanced routing required annotations), Gateway API supports protocol-aware matching (e.g., HTTP headers, gRPC services, TCP/UDP), traffic splitting/weighting, and rich TLS configurations out of the box.
    * This expressiveness eliminates “annotation hacks” and surfaces capabilities in a structured, versioned API. ([Kubernetes][1], [Kubernetes][3])

4. **Extensible**

    * Gateway API defines extension points at various layers (e.g., `Listener`, `RouteBinding`, `RouteParentRef`) that allow custom CRDs to integrate seamlessly.
    * Operators can introduce new route kinds (like `GRPCRoute` or `TLSRoute`) or attach policy CRDs (e.g., `BackendTLSPolicy`) without altering the core API. ([Kubernetes][4])

Gateway API revolves around three stable API kinds and their interrelationships:

1. **GatewayClass (`gateway.networking.k8s.io/v1beta1`)**

    * **Role**: Defined by the Infrastructure Provider (e.g., cloud vendor or CNI plugin).
    * **Purpose**: Acts as a “blueprint” or “class” for Gateways—identifying the controller responsible and providing cluster-wide default configurations (e.g., which load-balancer implementation to use).
    * **Spec Highlights**:

        * `controller`: A unique string (e.g., `example.com/aws-alb`) that ties this class to a specific controller’s implementation.
        * `parametersRef` (optional): Points to a CRD (e.g., `GatewayClassParameters`) containing provider-specific settings (e.g., AWS ALB tags, GCP ILB flags).

   **Example**:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: GatewayClass
   metadata:
     name: example-nginx-gatewayclass
   spec:
     controller: example.com/nginx-gateway-controller
   ```

   Once created, only a controller whose name matches `spec.controller` can claim Gateways referring to this class. ([Kubernetes][1])

2. **Gateway (`gateway.networking.k8s.io/v1beta1`)**

    * **Role**: Managed by the Cluster Operator (or Infrastructure Provider, via the controller).
    * **Purpose**: Defines an instance of “traffic-handling infrastructure”—for example, a cloud load balancer or an NGINX proxy—within a specific namespace or cluster. Each Gateway manifests as one or more actual dataplane endpoints (e.g., LB IPs, ports).
    * **Spec Highlights**:

        * `gatewayClassName` (required): The name of a `GatewayClass` that this Gateway belongs to.
        * `listeners`: A list of `Listener` specs, each defining:

            * `protocol` (HTTP, HTTPS, TCP, TLS, UDP).
            * `port` (integer 1–65535).
            * For `HTTPS` or `TLS`, a `tls` block specifying the certificate reference (a Kubernetes Secret).
            * An optional `allowedRoutes` filter specifying which Routes can bind (e.g., by namespace or label selector).
        * `addresses` (optional): Allows specifying IPs (or hostname) where the Gateway is reachable (useful for static IPs in bare-metal environments).

   **Example**:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: Gateway
   metadata:
     name: example-gateway
     namespace: cluster-system
   spec:
     gatewayClassName: example-nginx-gatewayclass
     listeners:
       - name: http
         protocol: HTTP
         port: 80
         allowedRoutes:
           namespaces:
             from: All
       - name: https
         protocol: HTTPS
         port: 443
         tls:
           certificateRefs:
             - name: shop-tls-secret
         allowedRoutes:
           namespaces:
             from: All
   ```

   In this example, the controller responsible for `example-nginx-gatewayclass` will provision (or configure) one or more NGINX instances listening on ports 80 and 443, using `shop-tls-secret` for TLS. ([Kubernetes][1], [Kubernetes][5])

3. **Routes** (e.g., `HTTPRoute`, `TCPRoute`, `TLSRoute`, `GRPCRoute`)

    * **Role**: Defined by Application Developers to describe application-level routing rules (e.g., “all traffic to `shop.example.com/cart` goes to `cart-service:80`”).
    * **Purpose**: Associates backend Services (or other endpoints) with a specific Gateway namespace or listener. Routes select a `parentRef` or `parentRefs` (one or more Gateways or additional Routes, enabling chaining).
    * **Core Route Kinds** (stable):

        1. **HTTPRoute** (`gateway.networking.k8s.io/v1beta1`): HTTP-specific rules (hosts, path matching, header matching, traffic splitting).
        2. **TCPRoute**: Routes raw TCP connections by port to backend services.
        3. **TLSRoute**: Allows SNI-based routing for TLS (layer 4) protocols like HTTPS, routing based on server names.
        4. **GRPCRoute**: For gRPC services, allows matching on gRPC methods or service names.

   **HTTPRoute Example**:

   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: HTTPRoute
   metadata:
     name: shop-router
     namespace: default
   spec:
     parentRefs:
       - name: example-gateway
         sectionName: http
     hostnames:
       - shop.example.com
     rules:
       - matches:
           - path:
               type: PathPrefix
               value: /cart
         backendRefs:
           - name: cart-service
             port: 80
             weight: 80
           - name: discount-service
             port: 80
             weight: 20
       - matches:
           - path:
               type: PathPrefix
               value: /checkout
         backendRefs:
           - name: checkout-service
             port: 80
   ```

    * `parentRefs`: Binds this route to the `http` listener on `example-gateway`.
    * `hostnames`: Restricts routing to traffic whose `Host` header is `shop.example.com`.
    * `matches`/`path`: Matches on URL path prefixes (`/cart`, `/checkout`).
    * `backendRefs`: Points at Service resources (in the same namespace unless fully qualified). `weight` enables traffic splitting (e.g., 80% to `cart-service`, 20% to `discount-service`). ([Kubernetes][1], [Kubernetes][3])

A **Gateway** object may accept multiple route kinds. For example, you can attach both an `HTTPRoute` and a `TCPRoute` to the same Gateway listener, provided the listener’s `protocol` matches each route’s `kind`.

The relationship between these core kinds can be visualized as:

```
[GatewayClass]
       ↑
       | (className)
  [Gateway] ─────────► [Listener (HTTP:80, HTTPS:443, TCP:443) ── allowedRoutes ──┐
       ↑                                                  ▲                   │
       │ (parentRef)                                       │ (binds)           │
  [HTTPRoute] ──┐                                         │                   │
               ├── [matches: hosts, paths, headers, splits]───► [Service(s)]    │
               │                                                             ┌► [Service(s)] (for TCPRoute)
  [GRPCRoute] ─┤                                                             │
               └── [matches: host, method, service name, splits]             │
                                                                         │
  [TLSRoute] ───────────────────────────────────────────────────────────────┘
```

1. **GatewayClass → Gateway**: A Gateway must reference exactly one GatewayClass. The class’s `controller` field determines which controller will manage that Gateway.
2. **Gateway → Listener**: Each Gateway defines multiple `listeners` (protocol + port). Listeners may filter which Routes can bind using `allowedRoutes` (e.g., by namespace or label).
3. **Route (e.g., HTTPRoute) → Gateway**: Each Route has one or more `parentRefs` referencing Gateways (and, optionally, a `sectionName` to bind to a specific listener).
4. **Route → BackendRef**: Rules in the Route describe how to map traffic (HTTP matches, TCP port matches, TLS SNI, or gRPC method) to one or more `backendRefs`. Each `backendRef` typically points to a Kubernetes Service (or an ExternalService for cloud resources).
5. **Traffic Flow**: Traffic arrives at the Gateway’s dataplane (e.g., a cloud LB or NGINX pod) on the specified port. The Gateway Controller’s proxy inspects L7 metadata (Host header, path, TLS SNI, gRPC metadata) and routes to the matched Service’s endpoint(s).

By decoupling “where the traffic lands” (Gateway + Listener) from “how to route it” (Routes) and “what proxies implement it” (GatewayClass/controller), Gateway API allows each role to focus on their configuration slice:

* Infrastructure Providers define `GatewayClass` CRDs and controllers.
* Cluster Operators instantiate Gateways with cluster-specific parameters (e.g., node selector, IP pools, node affinity).
* Application Developers create Routes that bind to those Gateways without worrying how the underlying LB is provisioned.

---

## 4. Example: End-to-End Configuration

Below is a step-by-step example of turning on HTTP/HTTPS ingress for an ecommerce “shop” application:

### 4.1 Create a GatewayClass (Infrastructure Provider)

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: example-nginx-gatewayclass
spec:
  controller: example.com/nginx-gateway-controller
```

* The cluster already has an NGINX-based controller watching `GatewayClass` with `controller: example.com/nginx-gateway-controller`. ([Kubernetes][1])

### 4.2 Provision a Gateway (Cluster Operator)

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: shop-gateway
  namespace: cluster-system
spec:
  gatewayClassName: example-nginx-gatewayclass
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        certificateRefs:
          - name: shop-tls-secret
      allowedRoutes:
        namespaces:
          from: All
```

* The NGINX controller creates an NGINX deployment and a corresponding Service (often of type `LoadBalancer`) listening on ports 80 and 443.
* TLS termination on port 443 uses `shop-tls-secret`, a Kubernetes Secret in `cluster-system` namespace already containing valid cert/key for `shop.example.com`. ([Kubernetes][1], [Kubernetes][5])

### 4.3 Define Application Services (Development Team)

```yaml
# cart-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: cart-service
  namespace: shop
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: cart
---
# checkout-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout-service
  namespace: shop
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: checkout
```

* Two Deployments (`app: cart` and `app: checkout`) exist in the `shop` namespace, each exposing a container on port 8080. These Service objects direct intra-cluster traffic to the appropriate Pods. ([Kubernetes][1])

### 4.4 Create an HTTPRoute (Application Developer)

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: shop-route
  namespace: shop
spec:
  parentRefs:
    - name: shop-gateway
      sectionName: http
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /cart
      backendRefs:
        - name: cart-service
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /checkout
      backendRefs:
        - name: checkout-service
          port: 80
```

* Binds to `shop-gateway`’s `http` listener (port 80).
* Restricts routing to requests where `Host=shop.example.com`.
* `/cart/*` → `cart-service:80`; `/checkout/*` → `checkout-service:80`. ([Kubernetes][1], [Kubernetes][3])

### 4.5 Verify End-to-End

1. Ensure the Gateway is ready:

   ```bash
   kubectl get gateway -n cluster-system shop-gateway
   # Check `.status.addresses` field for external IP or hostname.
   ```
2. Verify HTTP routing:

   ```bash
   curl http://<gateway-address>/cart/items
   # Should return response from a Pod behind cart-service.
   curl http://<gateway-address>/checkout
   # Should hit checkout-service.
   ```
3. Verify HTTPS routing (requires DNS or `/etc/hosts` mapping `shop.example.com` to the Gateway IP):

   ```bash
   curl -k https://shop.example.com/checkout
   # TLS handshake uses shop-tls-secret, then proxies to checkout-service.
   ```
4. Check for metrics or logs in the Ingress Controller’s Pods to confirm rules are applied. ([Kubernetes][1])

---

## 5. Advanced Features

### 5.1 Traffic Splitting and Weighting

Within a single `HTTPRoute` rule, you can specify multiple `backendRefs` with weights, enabling simple A/B tests or gradual rollouts:

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: stable-service
        port: 80
        weight: 90
      - name: canary-service
        port: 80
        weight: 10
```

Traffic to `/` is split 90/10 between `stable-service` and `canary-service`. ([Kubernetes][1], [Kubernetes][3])

### 5.2 Header, Query, and Method Matching

`HTTPRoute` supports fine-grained matching based on:

* **Headers**: e.g., `matches: [{ headers: [{ name: "X-User-Type", value: "premium" }] }]`.
* **Query Params**: e.g., `matches: [{ queryParams: [{ name: "version", value: "beta" }] }]`.
* **HTTP Methods**: e.g., `matches: [{ method: GET }]`.

These capabilities enable advanced routing (e.g., sending only POST requests to a “write” backend). ([Kubernetes][1])

### 5.3 TLS Passthrough and SNI

A `TLSRoute` can match incoming TLS connections based on SNI without terminating TLS:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: TLSRoute
metadata:
  name: shop-tlsroute
  namespace: shop
spec:
  parentRefs:
    - name: shop-gateway
      sectionName: https
  hostnames:
    - api.shop.example.com
  rules:
    - matches:
        - sniHosts: ["api.shop.example.com"]
      backendRefs:
        - name: backend-tls-service
          port: 443
```

* The Gateway forwards encrypted traffic directly to `backend-tls-service:443` (e.g., a service that implements its own TLS).
* Useful for workloads requiring end-to-end encryption to backend Pods. ([Kubernetes][1])

### 5.4 gRPC Routing

`GRPCRoute` adds gRPC-specific matching (service and method names) atop HTTP/2 semantics. Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GRPCRoute
metadata:
  name: shopping-grpc-route
  namespace: shop
spec:
  parentRefs:
    - name: shop-gateway
      sectionName: http
  hostnames:
    - grpc.shop.example.com
  rules:
    - matches:
        - method:
            name: GetCart
      backendRefs:
        - name: shopping-grpc-svc
          port: 50051
    - matches:
        - method:
            name: Checkout
      backendRefs:
        - name: checkout-grpc-svc
          port: 50052
```

* Routes gRPC calls based on the RPC method name.
* Offers precise control over how different gRPC methods map to different backends. ([Kubernetes][1])

### 5.5 Backend TLS Policy

Starting in v1.2, Gateway API introduces `BackendTLSPolicy` to configure TLS from Gateway to backend services (end-to-end TLS). For instance:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: BackendTLSPolicy
metadata:
  name: service-tls-policy
  namespace: shop
spec:
  targetRef:
    group: ""
    kind: Service
    name: backend-service
  tls:
    certificateRefs:
      - name: backend-cert-secret
    mode: RequireTLS
    validation:
      caCertificates:
        secretName: backend-ca-secret
```

* `RequireTLS` ensures the Gateway uses TLS when connecting to `backend-service`.
* `caCertificates` verifies server certificates from backend Pods, enabling mutual TLS if needed. ([Kubernetes][4])

---

## 6. Lifecycle and Status

1. **GatewayClass Creation**

    * An administrator applies a `GatewayClass`. The controller implementing that class watches for new Gateways referencing it.
    * `GatewayClass.status.conditions` reflect whether the class is accepted or not.

2. **Gateway Creation**

    * Applying a `Gateway` triggers the associated controller (as per `gatewayClassName`).
    * The controller provisions or configures dataplane resources (e.g., cloud-provider LBs or an NGINX pod + Service).
    * Once ready, `Gateway.status.addresses` shows the external IP(s) or hostname(s).

3. **Route Creation**

    * Developers create `HTTPRoute`, `TCPRoute`, etc., in application namespaces.
    * Controllers evaluate `parentRefs` to see which Gateway(s) the route should attach to, respecting `allowedRoutes` filters.
    * If a route is accepted, `HTTPRoute.status.parents` is populated (indicating which Gateways/listeners the route is bound to and whether there are conflicts).

4. **Traffic Flow**

    * External clients send HTTP/TLS requests to the Gateway’s address.
    * The Gateway’s dataplane proxy matches incoming traffic against attached routes, forwarding connections to the appropriate backend Service.

5. **Updates**

    * Modifying a `Route` (e.g., adding a new path) or a `Gateway` (e.g., adding a new listener) triggers reconciliation.
    * The controller pushes updated configuration to the dataplane (e.g., NGINX reload, cloud LB reconfiguration).

6. **Deletion**

    * Deleting a `Route` removes it from the Gateway’s configuration; subsequent requests no longer match.
    * Deleting a `Gateway` (once no Routes reference it) causes the controller to tear down the underlying infrastructure (e.g., delete the cloud LB or scale down the proxy).

Throughout, status fields and `Conditions` on each kind (e.g., `Ready`, `RouteConflict`, `Accepted`) allow operators to diagnose misconfigurations (e.g., overlapping host rules, missing TLS secrets).

---

## 7. Migrating from Ingress

Kubernetes explicitly recommends moving from Ingress to Gateway API for future-proofing and advanced features. Key migration considerations:

* **Convert Ingress rules to `HTTPRoute`**

    * Host and path matches translate directly.
    * Annotations (e.g., rewrites, timeouts) often map to Route or Gateway annotations (NGINX-specific).
* **Define a `GatewayClass`/`Gateway` instead of relying on the default Ingress controller**

    * Allows choosing multiple Gateway classes in the same cluster (e.g., one for internal apps, another for external with WAF).
* **Leverage richer matching** (headers, methods, GRPC) that was impossible with Ingress alone.

Because Ingress is now considered feature-frozen and Gateway API is the extension point for future networking innovations, transitioning to Gateway API ensures alignment with Kubernetes’ road map. ([Kubernetes][6], [Kubernetes][2])

---

## 8. Best Practices

1. **Always Specify `ingressClassName` (or `gatewayClassName`)**

    * Avoid ambiguity in multi-controller clusters by explicitly pinning a `Gateway` to the intended controller.

2. **Use Namespaces to Isolate Routes**

    * `HTTPRoutes` in the same namespace can bind only to Gateways that allow routes from that namespace (via `allowedRoutes.namespaces.from: Selector` or `All`).
    * Use labels on `HTTPRoutes` and `Gateways` to enforce scoping.

3. **Keep TLS Secrets in the Same Namespace as `Gateway`**

    * Most controllers require the TLS `Secret` to reside in the same namespace as the `Gateway`, unless the implementation supports cross-namespace references.

4. **Use `BackendTLSPolicy` for End-to-End Security**

    * For sensitive workloads, ensure that after terminating TLS at the Gateway, connections to backend Pods remain encrypted and verified.

5. **Monitor `status.conditions` on Each Resource**

    * `GatewayClass`: Check if `Accepted` condition is `True`.
    * `Gateway`: Confirm `Ready` and that `status.addresses` is populated.
    * `HTTPRoute` (or other Routes): Inspect `ResolvedRefs` and `Accepted` conditions to ensure the route is bound correctly.

6. **Plan for Traffic Splits and Canary Releases**

    * Use `weight` in `backendRefs` to gradually shift traffic.
    * Keep stable and canary backends in the same `HTTPRoute` for easier rollback.

7. **Namespace and Label-Based Access Controls**

    * Leverage `allowedRoutes` in `Listener` to restrict which namespaces or labels of `Route` can bind, preventing unauthorized routes from hijacking Gateways.

8. **Extend with Custom Routes or Policies**

    * If you need functionality beyond HTTP/TCP/TLS—for example, Kafka or gRPC Gateway—the extensibility model allows introducing new CRDs (e.g., `KafkaRoute`).
    * Use `ReferenceGrant` to allow cross-namespace backendRefs or TLS Secret references. ([Kubernetes][1], [Kubernetes][3])

---

## 9. When to Use Gateway API

* **You require advanced L7 routing features** (e.g., header-based routing, traffic weighting, fine-grained TLS policies).
* **Your cluster serves multiple teams with separate infrastructure requirements** (e.g., some teams need ALB, others need NGINX Ingress). GatewayClass/Gateway isolates concerns.
* **You plan to adopt a service mesh**: Gateway API integrates natively with service meshes (e.g., Istio’s Gateway controller can serve both ingress and mesh traffic).
* **You need protocol-aware routing beyond HTTP**, such as gRPC methods, raw TCP, or UDP.
* **You want a future-proof API**: Ingress is effectively frozen, whereas Gateway API is actively evolving (v1.1, v1.2, etc.).

If you have a simple HTTP-only application in a single namespace and are comfortable with Ingress annotations, you may not need Gateway API immediately. However, for production-grade, multi-team, multi-protocol environments, Gateway API is the recommended path forward. ([Kubernetes][6], [Kubernetes][7])

[1]: https://kubernetes.io/docs/concepts/services-networking/gateway/?utm_source=chatgpt.com "Gateway API - Kubernetes"
[2]: https://kubernetes.io/blog/2022/07/13/gateway-api-graduates-to-beta/?utm_source=chatgpt.com "Kubernetes Gateway API Graduates to Beta"
[3]: https://kubernetes.io/blog/2021/04/22/evolving-kubernetes-networking-with-the-gateway-api/?utm_source=chatgpt.com "Evolving Kubernetes networking with the Gateway API"
[4]: https://kubernetes.io/blog/2023/11/28/gateway-api-ga/?utm_source=chatgpt.com "New Experimental Features in Gateway API v1.0 - Kubernetes"
[5]: https://kubernetes.io/docs/concepts/services-networking/?utm_source=chatgpt.com "Services, Load Balancing, and Networking - Kubernetes"
[6]: https://kubernetes.io/docs/concepts/services-networking/ingress/?utm_source=chatgpt.com "Ingress - Kubernetes"
[7]: https://kubernetes.io/blog/2024/05/09/gateway-api-v1-1/?utm_source=chatgpt.com "Gateway API v1.1: Service mesh, GRPCRoute, and a whole lot more"

## Network Policies

**Kubernetes Network Policies** allow you to define how groups of Pods are permitted to communicate with each other and with other network “entities” (such as namespaces or external IP blocks). By default, Pods within a Kubernetes cluster can talk to any other Pod (and any external service) unless you explicitly restrict that behavior. Network Policies give you a declarative, namespaced way to lock down those connections—enabling a “zero-trust” or “least-privilege” networking posture for your workloads. ([Kubernetes][1])

Below is a deep dive into Network Policies, covering:

1. **Why Network Policies Matter**
2. **How Network Policies Work**
3. **NetworkPolicy API Overview**
4. **Common Examples**

    * Allow All / Deny All
    * Restrict Ingress to a Namespace
    * Restrict Egress to a Service or CIDR
    * Pod-to-Pod Whitelisting with Labels
5. **Before You Begin: CNI Requirements**
6. **Debugging and Observability**
7. **Best Practices and Considerations**

---

## 1. Why Network Policies Matter

In a typical Kubernetes cluster, once a Pod is running, it can (by default) send or accept traffic to \_any\_ other Pod or external IP address. This “flat” networking model simplifies connectivity, but it also means that if one Pod becomes compromised, it could potentially attack or exfiltrate data from any other Pod. Network Policies let you close that gap by specifying exactly which Pods or IP blocks each Pod is allowed to talk to, and on which ports. ([Kubernetes][1])

Key security benefits include:

* **Microsegmentation**: Instead of trusting every Pod within a Namespace (or the entire cluster), you segment by workload. For instance, only your front-end Pods can reach your back-end database Pods, and nothing else in the cluster can connect to the database.
* **Egress Control**: Prevent Pods from calling arbitrary external endpoints. For example, you might allow only your logging agent Pod to talk to `logging.company.com` and disallow all other outbound traffic.
* **Compliance & Audit**: Many regulations (PCI-DSS, HIPAA, GDPR) require that workloads do not talk to resources they shouldn’t. Network Policies let you codify and document those constraints in YAML manifests.

Because Network Policies are namespace-scoped (you create them in a particular namespace), you can give each application or team its own set of policies without affecting other namespaces.

---

## 2. How Network Policies Work

A **Network Policy** is a namespaced Kubernetes object that defines:

* **Pod Selectors**: Which Pods \_the policy applies to\_.
* **Policy Types**: Whether the policy governs **Ingress** (incoming) traffic, **Egress** (outgoing) traffic, or both.
* **Rules**: For each type (Ingress/Egress), a list of “allow” rules specifying \_who\_ (other Pods, Namespaces, or IP blocks) may connect, on which ports and protocols.

> **Important**: Network Policies are “*whitelists*.” If you create a policy that selects Pod A for Ingress rules, any traffic not explicitly allowed by those rules is \_denied\_. If you have no NetworkPolicy selecting Pod A, Pod A remains fully open (all traffic allowed). ([Kubernetes][1])

### 2.1 Pod Selection and Policy Application

* A NetworkPolicy’s **`spec.podSelector`** determines which Pods within its namespace the policy applies to.
* Any Pod that matches that selector becomes subject to \_all\_ of the policy’s rules.
* If you omit `podSelector` (i.e., leave it empty: `{}`), the policy applies to \_all Pods\_ in that namespace.

Once a Pod is “selected,” Kubernetes enforces:

1. **Ingress**: Other endpoints (Pods, Namespaces, IP blocks) must match at least one Ingress rule to send traffic to the selected Pod. All other inbound connections are dropped.
2. **Egress**: The selected Pod may send to only those endpoints matching at least one Egress rule; all other outbound connections are dropped.

### 2.2 Rule Matching

Within each policy, you can specify multiple **Ingress** and/or **Egress** rules. Each rule consists of:

* **`from`** (for Ingress) or **`to`** (for Egress) \_peers\_:

    * **PodSelector**: Match Pods by labels (in the same namespace, unless combined with `namespaceSelector`).
    * **NamespaceSelector**: Match all Pods in certain namespaces (via labels on the namespace object).
    * **IPBlock**: Match any IP in a given CIDR range, optionally excluding specific sub-ranges.

* **`ports`**: A list of port + protocol (TCP/UDP) pairs. Only connections to those ports on the Pod (for Ingress) or from those ports on the Pod (for Egress) are allowed. If you omit `ports`, \_all\_ ports are allowed (for the matched peers).

Multiple `from` or `to` entries within one rule are combined with an \_OR\_. Traffic that matches any one of the rule’s peers (and any one of the rule’s ports) is allowed. However, to be allowed, traffic must match at least one rule. If no rules match, it is denied.

### 2.3 Default “Allow All” vs. “Deny All”

* **No NetworkPolicy** selects a Pod → That Pod remains wide-open (both Ingress and Egress allowed).
* Create one NetworkPolicy that selects Pod A but defines ZERO Ingress rules: → Pod A \_cannot\_ receive any traffic (Ingress denied).
* Similarly, a policy that defines zero Egress rules means selected Pods cannot initiate any outbound traffic.

---

## 3. NetworkPolicy API Overview

Below is a skeleton of a NetworkPolicy manifest. We’ll dissect each section:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-network-policy
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
        - namespaceSelector:
            matchLabels:
              environment: production
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/16
            except:
              - 10.0.5.0/24
      ports:
        - protocol: TCP
          port: 1234
```

### 3.1 `podSelector`

* A label selector targeting Pods **within the same namespace** as the policy.
* In the example above, any Pod with `role: db` in `my-namespace` is “selected” by this policy.

### 3.2 `policyTypes`

* A list containing one or both of `Ingress` and `Egress`.

    * If you include **only** `Ingress`, then only inbound traffic to the selected Pods is affected (outbound remains wide-open).
    * If you include **only** `Egress`, then only outbound traffic from selected Pods is affected (inbound remains wide-open).
    * If you specify both, both directions become restricted according to the rules you write.
* **If you omit** `policyTypes`, the API defaults to `Ingress` (for backward compatibility). ([Kubernetes][1])

### 3.3 `ingress` Rules

Each item in the `ingress` array is a separate rule. In our example:

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            role: frontend
      - namespaceSelector:
          matchLabels:
            environment: production
    ports:
      - protocol: TCP
        port: 5432
```

* **`from:`**

    1. Any Pod in the same namespace labeled `role: frontend`.
    2. \_OR\_ any Pod in a namespace labeled `environment: production` (regardless of that Pod’s labels).
* **`ports:`** Only allow inbound connections if the destination port on the DB Pod is TCP 5432.
* If a connection originates from another Pod or namespace not matching those selectors, or from a different port, it is dropped.

### 3.4 `egress` Rules

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 10.0.0.0/16
          except:
            - 10.0.5.0/24
    ports:
      - protocol: TCP
        port: 1234
```

* **`to:`** Means outbound connections from the selected Pods only succeed if the destination IP is in `10.0.0.0/16`, except any in `10.0.5.0/24`.
* **`ports:`** The destination port must be TCP 1234. All other egress traffic is dropped.

---

## 4. Common Examples

Below are some typical real-world patterns.

### 4.1 Allow All / Deny All in a Namespace

**Deny All Ingress** to a namespace, except Pods decorated with `allow: "true"`. First, label the Pods that are allowed:

```bash
kubectl label pod frontend-1 -n team-a allow=true
kubectl label pod frontend-2 -n team-a allow=true
```

Then create:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress-except-allowed
  namespace: team-a
spec:
  podSelector: {}             # select all Pods in namespace
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              allow: "true"
```

* Any Pod in **team-a** without `allow=true` label cannot receive Ingress traffic from anywhere (even from other Pods in `team-a`), because only Pods with `allow=true` can connect.
* Egress is still open; if you also want to block outbound, add `policyTypes: [Ingress, Egress]` and `egress: []` (empty list) to drop all outbound as well.

### 4.2 Restrict Ingress to a Namespace

Allow only traffic from Pods in the same namespace (`team-b`) to `role=db` Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-team-communication
  namespace: team-b
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: team-b
```

* Any Pod in namespace `team-b` can talk to `role=db` Pods on any port.
* Pods outside `team-b` (or without that namespace label) cannot connect to `role=db`.

### 4.3 Restrict Egress to an External Service or CIDR

Suppose your Pods must send logs only to `10.1.1.100/32` on TCP 514 (syslog). In namespace `logging`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-syslog-egress
  namespace: logging
spec:
  podSelector: {}  # apply to all Pods in logging namespace
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.1.1.100/32
      ports:
        - protocol: TCP
          port: 514
```

* Outbound connections to anything other than `10.1.1.100:514/TCP` are dropped.
* Ingress remains open (unless a separate policy exists).

### 4.4 Pod-to-Pod Whitelisting with Labels

Isolate a three-tier application (web → app → db) in namespace `prod` so that:

* **`web` Pods** can talk to **`app` Pods** only on port 8080.
* **`app` Pods** can talk to **`db` Pods** only on port 5432.
* No other Pod in the namespace can connect to **`db`** or **`app`**.

**Step 1: Deny all to `app` except from `web`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-deny-all-except-web
  namespace: prod
spec:
  podSelector:
    matchLabels:
      tier: app
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: web
      ports:
        - protocol: TCP
          port: 8080
```

**Step 2: Deny all to `db` except from `app`:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all-except-app
  namespace: prod
spec:
  podSelector:
    matchLabels:
      tier: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: app
      ports:
        - protocol: TCP
          port: 5432
```

* Because no policy selects `web` Pods at all, `web` can still initiate inbound or outbound to anywhere (unless you add a separate policy limiting `web`).
* `app` Pods can receive traffic only from `web` on port 8080; they can still open outbound (if no Egress policy).
* `db` Pods can receive traffic only from `app` on port 5432; they cannot accept connections from `web` or any other Pod.

---

## 5. Before You Begin: CNI Requirements

**Important**: Network Policies are enforced by the underlying Container Networking Interface (CNI) plugin. Not all CNIs support them; popular CNIs that do include:

* Calico
* Cilium
* Weave Net (in “Weave Net Plugin” mode)
* Kube-Router
* Antrea
* Canal (Calico + Flannel combination)

If you install a CNI that does \_not\_ implement NetworkPolicy enforcement, your policies will exist in the API server but have no effect—Pods remain fully open. Always verify your cluster’s CNI supports Network Policies before relying on them for security. ([Kubernetes][1])

---

## 6. Debugging and Observability

When you apply a NetworkPolicy and traffic stops flowing as expected, use these steps to debug:

1. **Check Endpoints & Pods**

    * Ensure both the client Pod and server Pod(s) are running and labeled correctly.
    * For example:

      ```bash
      kubectl get pods -l tier=app -n prod
      kubectl describe pod app-xyz-123 -n prod
      ```

2. **Verify NetworkPolicy Selection**

    * Describe the policy to confirm `podSelector` matches your Pods:

      ```bash
      kubectl describe networkpolicy app-deny-all-except-web -n prod
      ```
    * Check that `policyTypes` and `ingress`/`egress` match your intent.

3. **Confirm CNI Enforcement**

    * Many CNIs offer a “policy log” or “policy status” command. For instance, with Calico:

      ```bash
      calicoctl get networkpolicies -o wide
      ```
    * With Cilium:

      ```bash
      cilium policy get
      cilium status
      ```
    * These tools show which policies are active on each node.

4. **Test Connectivity**

    * **From inside a Pod**, use `curl`, `nc`, or `telnet` to attempt connections:

      ```bash
      kubectl exec -it <client-pod> -n prod -- sh
      # Inside the Pod:
      nc -vz db-service 5432
      ```
    * If connection is refused or times out, inspect policy logic.

5. **Check EndpointSlices/Endpoints**

    * If a Service front-end is involved, ensure its Endpoints exist:

      ```bash
      kubectl get endpoints db-service -n prod
      ```
    * If Endpoints is empty, no Pod matches the Service’s `selector`, so traffic is dropped at the Service layer even before NetworkPolicy is evaluated.

6. **Logging & Packet Captures** (Advanced)

    * Some CNIs let you generate logs for dropped packets or even capture traffic on the host’s veth interfaces to see exactly which rule is blocking a flow.
    * For example, with Calico, you can enable IP-level logging via BPF or IPset rule logs.

---

## 7. Best Practices and Considerations

1. **Start with “Deny All” and Gradually Open**

    * It’s often easier to create a catch-all “deny all” policy (`podSelector: {}` with no rules) for a namespace, then add specialized policies to allow only needed traffic. This prevents accidental wide-open holes.

   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: deny-all
     namespace: secure-ns
   spec:
     podSelector: {}
     policyTypes:
       - Ingress
       - Egress
     ingress: []  # no rules → deny all inbound
     egress: []   # no rules → deny all outbound
   ```

2. **Label Consistently**

    * Design a labeling scheme that reflects your application tiers (e.g., `app=frontend`, `app=db`, `environment=prod`).
    * Use those labels in both Pod specs and NetworkPolicy selectors to avoid mismatches.

3. **Namespace Isolation**

    * If you want to ensure Pods in namespace A cannot talk to Pods in namespace B, label each namespace (e.g., `name: team-a`, `name: team-b`) and write policies in each namespace’s default deny that restrict traffic to the same namespace only.

   ```yaml
   # In namespace "team-a":
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: namespace-isolation
   spec:
     podSelector: {}
     policyTypes:
       - Ingress
     ingress:
       - from:
           - namespaceSelector:
               matchLabels:
                 name: team-a
   ```

4. **Beware of Implicit “Allow” if No Policy**

    * A Pod with no NetworkPolicy selecting it remains fully reachable. When you roll out new workloads, double-check which policies might already exist. You may need to add a new policy to cover the new Pod.

5. **Combine PodSelector + NamespaceSelector**

    * When you want to allow traffic from specific Pods in a different namespace, chain both selectors:

      ```yaml
      ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                  environment: staging
              podSelector:
                matchLabels:
                  app: metrics
      ```

      This matches only Pods labeled `app=metrics` in namespaces labeled `environment=staging`.

6. **IPBlock for External Dependencies**

    * Use `ipBlock.cidr` when your Pods must call an external API (e.g., `api.stripe.com`) or a legacy data center.
    * If you need to exclude certain IP ranges (e.g., internal network of another team), use `except:` within `ipBlock`.

7. **Combine Multiple Policies**

    * You can create multiple NetworkPolicy objects targeting the same Pods. The **union** of all rules applies. For instance, one policy might allow database egress, another might allow DNS egress.
    * Be careful: a single deny rule (empty rule) in one policy effectively denies everything unless another policy explicitly allows it. In other words, Kubernetes takes the union of all “allow” rules across all NetworkPolicy objects selecting a Pod.

8. **Plan for Service Traffic**

    * If you use Services that cluster-internally forward traffic (e.g., `ClusterIP`), remember that traffic to a Service is actually DNAT’d to a backend Pod IP. Your NetworkPolicy rules see the source as the original Pod and the destination as the backend Pod. You may need to allow traffic from the Service’s “frontend” Pod address if using headless or special Service types.
    * For **NodePort** or **LoadBalancer** traffic, remember that connections may appear as if they come from node IPs or from kube-proxy SNAT addresses, depending on your CNI. You may need to allow those node CIDRs or `ipBlock`s.

9. **Monitor and Audit Regularly**

    * As your cluster evolves, old policies may become stale or too permissive. Regularly `kubectl get networkpolicy --all-namespaces` and review them.
    * Use CI/CD checks to prevent deployments that inadvertently remove critical labels (breaking selectors) without updating the related policies.

[1]: https://kubernetes.io/docs/concepts/services-networking/network-policies/?utm_source=chatgpt.com "Network Policies - Kubernetes"
