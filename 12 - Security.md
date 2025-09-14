
## Understanding Cloud-Native Security Architecture

### The Evolution from Traditional to Cloud-Native Security

Traditional infrastructure security relied heavily on perimeter defense, where a strong boundary protected internal resources. This model assumed that threats came primarily from outside the network, and once inside the perimeter, entities could be trusted. However, modern cloud-native environments fundamentally challenge these assumptions.

In Kubernetes environments, workloads are ephemeral, constantly scaling and moving across nodes. Network boundaries become fluid, with services communicating across namespaces, clusters, and even cloud providers. This dynamic nature requires a fundamentally different approach to security.

### Zero-Trust Security Model in Kubernetes

The zero-trust model operates on the principle of "never trust, always verify." In Kubernetes, this means every component, whether internal or external, must authenticate and be authorized for every action. This approach significantly reduces the blast radius of potential security breaches.

Key principles of zero-trust in Kubernetes include:

- Every API request requires authentication, regardless of origin
- Authorization decisions are made for each action, not just at connection time
- Network policies enforce microsegmentation between workloads
- Secrets and sensitive data are encrypted both in transit and at rest
- Regular rotation of credentials and certificates
- Continuous monitoring and auditing of all activities

### Defense in Depth Strategy

Defense in depth creates multiple layers of security controls, ensuring that if one layer fails, others continue to provide protection. In Kubernetes, these layers include:

Infrastructure Layer: The underlying compute, network, and storage resources that host the cluster. This includes the security of the cloud provider or on-premises data center.

Cluster Layer: The Kubernetes control plane components, including the API server, etcd, controller manager, and scheduler. Securing these components is critical as they form the brain of the cluster.

Container Layer: The container runtime, images, and registries. This includes vulnerability scanning, image signing, and runtime protection.

Application Layer: The actual workloads running in pods, including their configurations, secrets management, and inter-service communication.

Data Layer: The persistent storage, databases, and data flows within the cluster, ensuring data protection through encryption and access controls.

## The Four Phases of Cloud-Native Security

### Development Phase: Building Security Into Applications

Security in the development phase focuses on creating applications with security as a core design principle rather than an afterthought. This phase establishes the foundation for all subsequent security measures.

Secure Coding Practices: Developers must understand common vulnerabilities and implement secure coding patterns. This includes input validation, proper error handling, secure session management, and protection against injection attacks. Code should undergo security-focused reviews where reviewers specifically look for potential vulnerabilities.

Threat Modeling: Before writing code, teams should identify potential threats to their applications. This involves mapping out data flows, identifying trust boundaries, and understanding what assets need protection. Questions to consider include: What data does the application handle? Who should have access? What would happen if this component were compromised?

Dependency Management: Modern applications rely on numerous third-party libraries and frameworks. Each dependency represents a potential security risk. Teams must maintain an inventory of all dependencies, regularly update them, and monitor for known vulnerabilities. Tools for Software Composition Analysis (SCA) can automate much of this process.

Security Testing Integration: Security testing should be integrated into the development pipeline. This includes static application security testing (SAST) to analyze source code, dynamic application security testing (DAST) to test running applications, and interactive application security testing (IAST) that combines both approaches.

### Distribution Phase: Securing the Supply Chain

The distribution phase encompasses everything between code completion and deployment, focusing heavily on maintaining the integrity and security of artifacts.

Container Image Security: Container images should be built from minimal base images to reduce attack surface. Each layer in a container image should be scanned for vulnerabilities. Organizations should maintain a registry of approved base images and regularly update them. Multi-stage builds help ensure that build tools and dependencies don't make it into production images.

Image Signing and Verification: Digital signatures ensure that images haven't been tampered with between build and deployment. Using tools like Notary or Cosign, teams can sign images after successful builds and security scans. Kubernetes admission controllers can then verify these signatures before allowing pods to run.

Registry Security: Container registries should implement strong access controls, use TLS for all communications, and maintain audit logs of all activities. Private registries should be used for proprietary applications, with public registries only used for verified, official images.

Supply Chain Attestation: Beyond just signing images, teams should maintain attestations about how images were built, what security scans were performed, and who approved them for production use. This creates an auditable trail of the entire build and distribution process.

### Deployment Phase: Controlling What Runs Where

The deployment phase enforces policies about what can be deployed and ensures that deployments meet security requirements.

Admission Control: Kubernetes admission controllers act as gatekeepers, evaluating requests to create or modify resources. They can enforce policies such as requiring specific labels, preventing privileged containers, or ensuring resource limits are set. Both validating admission webhooks (which accept or reject requests) and mutating admission webhooks (which can modify requests) play crucial roles.

Policy as Code: Tools like Open Policy Agent (OPA) allow teams to define security policies as code. These policies can enforce standards such as requiring health checks, mandating non-root users, or blocking deprecated API versions. Policy as code ensures consistency and allows policies to be version controlled and tested.

Deployment Authorization: Not everyone who can access a cluster should be able to deploy to production namespaces. RBAC policies should carefully control who can create and modify resources in different namespaces. Critical namespaces might require additional approval workflows.

Configuration Validation: Before deployment, configurations should be validated against security best practices. This includes checking for hard-coded secrets, overly permissive security contexts, or missing network policies. Tools can scan YAML files and Helm charts for security issues before they reach the cluster.

### Runtime Phase: Protecting Running Workloads

The runtime phase encompasses all security measures for actively running workloads and the ongoing operation of the cluster.

Runtime Protection: Container runtime security involves monitoring process behavior, file system changes, and network connections. Anomaly detection can identify when containers behave unexpectedly, potentially indicating a compromise. Runtime protection might block certain system calls, prevent writing to specific directories, or alert on unexpected network connections.

Resource Isolation: Proper resource limits prevent single workloads from consuming all available resources and affecting other applications. This includes CPU and memory limits, but also PID limits, ephemeral storage limits, and constraints on other resources.

Security Monitoring: Continuous monitoring of security events is essential. This includes monitoring API audit logs for suspicious activities, tracking failed authentication attempts, and watching for privilege escalations. Security Information and Event Management (SIEM) systems can aggregate and analyze these logs.

Incident Response: When security incidents occur, teams need clear procedures for response. This includes isolating affected workloads, collecting forensic data, and restoring services. Regular incident response drills ensure teams are prepared for real events.

## Pod Security Standards in Detail

### Understanding Pod Security Levels

Kubernetes Pod Security Standards provide a framework for defining security policies at different levels of strictiveness. These standards recognize that different workloads have different security requirements and that overly restrictive policies can impede legitimate functionality.

### Privileged Profile: Maximum Permissions

The Privileged profile provides unrestricted access and is essentially equivalent to root access on the host. This profile is necessary for certain system-level components and infrastructure workloads.

Use Cases for Privileged Profile:

- Container Network Interface (CNI) plugins that need to configure host networking
- Storage drivers that need to mount volumes on the host
- Monitoring agents that need to access host metrics
- System components that manage node-level resources

While necessary in some cases, privileged workloads should be minimized and carefully audited. They should run in isolated namespaces with additional monitoring.

### Baseline Profile: Preventing Known Risks

The Baseline profile prevents known privilege escalations while maintaining broad compatibility. It's designed to be adoptable by most applications without significant modifications.

Key Restrictions in Baseline:

- No Host Namespaces: Containers cannot share the host's network, PID, or IPC namespaces, preventing them from seeing host processes or network interfaces
- No Privileged Containers: Containers cannot run with privileged access, limiting their ability to perform administrative operations
- Limited Capabilities: Containers can only use a safe set of Linux capabilities, preventing them from bypassing kernel security mechanisms
- No Host Path Volumes: Containers cannot mount arbitrary host directories, preventing access to sensitive host files
- Controlled Security Profiles: AppArmor and SELinux profiles must be from an approved list

The Baseline profile works well for most application workloads, web services, and databases that don't require special host access.

### Restricted Profile: Maximum Security

The Restricted profile enforces current Pod hardening best practices, providing the strongest security guarantees at the potential cost of application compatibility.

Additional Restrictions in Restricted:

- Must Run as Non-Root: All containers must run with a non-root user ID, preventing many types of privilege escalation
- No Privilege Escalation: The allowPrivilegeEscalation flag must be false, preventing processes from gaining additional privileges
- Capability Dropping: All containers must drop ALL capabilities, only adding back NET_BIND_SERVICE if needed for binding to ports below 1024
- Volume Type Restrictions: Only certain volume types (configMap, secret, emptyDir, etc.) are allowed, preventing access to host resources
- Seccomp Profiles: Containers must use either the RuntimeDefault or Localhost seccomp profiles, limiting available system calls

### Implementing Pod Security Admission

Pod Security Admission is built into Kubernetes and provides namespace-level enforcement of Pod Security Standards.

Enforcement Modes:

Enforce Mode: Violations cause pod creation to fail. This provides hard enforcement but requires that all workloads comply before enabling.

Audit Mode: Violations are logged but pods are still created. This allows teams to understand what would break before enforcing policies.

Warn Mode: Violations trigger warnings to users but don't block pod creation. This helps educate users about security requirements.

## ServiceAccounts: Identity for Workloads

### ServiceAccount Architecture and Purpose

ServiceAccounts provide identity for processes running in pods, enabling them to interact with the Kubernetes API and other services. Unlike user accounts, which represent human operators, ServiceAccounts are designed for programmatic access.

Every ServiceAccount consists of several components:

- A Kubernetes API object that defines the account
- An automatically generated token for API authentication
- Optionally, image pull secrets for accessing private registries
- RBAC bindings that grant permissions

### Token Management and Security

Modern Kubernetes uses projected service account tokens, which offer significant security improvements over legacy tokens:

Time-Limited Tokens: Tokens automatically expire and are refreshed by the kubelet. This limits the impact of token compromise.

Audience-Scoped Tokens: Tokens can be restricted to specific audiences, preventing them from being used with unintended services.

Bound Object References: Tokens are bound to specific pods and secrets, making them invalid if those objects are deleted.

### ServiceAccount Best Practices

Avoid Default ServiceAccount: The default ServiceAccount in each namespace should not be used for applications. Create dedicated ServiceAccounts for each application or component.

Least Privilege Principle: Grant only the minimum permissions required. Start with no permissions and add them incrementally as needed.

Disable Auto-Mounting: For pods that don't need API access, disable automatic token mounting to reduce attack surface.

Regular Rotation: Although tokens auto-rotate, ServiceAccounts themselves should be periodically reviewed and recreated if their permissions have changed.

## RBAC: Fine-Grained Access Control

### RBAC Components Deep Dive

Roles and ClusterRoles define sets of permissions. They specify what actions (verbs) can be performed on which resources. Roles are namespace-scoped, while ClusterRoles are cluster-wide.

Subjects are the entities that permissions are granted to. These can be Users (for humans), ServiceAccounts (for pods), or Groups (collections of users or ServiceAccounts).

RoleBindings and ClusterRoleBindings connect subjects to roles. A RoleBinding grants permissions within a namespace, while a ClusterRoleBinding grants cluster-wide permissions.

### Permission Aggregation

RBAC permissions are additive - there are no "deny" rules. If a subject has multiple bindings, they get the union of all permissions. This means:

- Adding a new binding can only increase permissions, never decrease them
- To remove permissions, you must modify or delete existing bindings
- There's no way to create exceptions or override higher-level permissions

### Common RBAC Patterns

Read-Only Access: Granting get, list, and watch verbs allows subjects to view resources without modifying them.

Namespace Admin: Granting all verbs on all resources within a namespace gives full control over that namespace.

Resource-Specific Access: Granting specific verbs on specific resource types allows fine-grained control.

Cross-Namespace Access: Using ClusterRoles with RoleBindings allows reusing permission sets across namespaces.

### RBAC Security Considerations

Several RBAC permissions can lead to privilege escalation:

Creating Workloads: The ability to create pods, deployments, or other workloads implicitly grants access to any secrets or configMaps in the namespace.

Modifying RBAC: The ability to create or modify roles and bindings allows subjects to grant themselves additional permissions.

Impersonation: The impersonate verb allows subjects to perform actions as other users or ServiceAccounts.

## Network Security with NetworkPolicies

### NetworkPolicy Fundamentals

NetworkPolicies provide network segmentation within Kubernetes clusters. They act as a firewall between pods, controlling which pods can communicate with each other.

By default, Kubernetes allows all pods to communicate freely. Once a NetworkPolicy selects a pod, that pod's traffic is restricted to what the policies explicitly allow. This follows a "default deny" model once policies are in place.

### NetworkPolicy Specifications

NetworkPolicies consist of several key components:

Pod Selector: Determines which pods the policy applies to. Uses label selectors to identify pods.

Policy Types: Specifies whether the policy affects Ingress (incoming), Egress (outgoing), or both types of traffic.

Ingress Rules: Define allowed incoming connections, including source pods, namespaces, or IP blocks, and allowed ports.

Egress Rules: Define allowed outgoing connections, including destination pods, namespaces, or IP blocks, and allowed ports.

### Advanced NetworkPolicy Patterns

Namespace Isolation: Policies can isolate entire namespaces, allowing communication only from specific namespaces.

IP-Based Policies: Policies can allow or deny traffic based on IP addresses, useful for controlling external access.

Port-Specific Rules: Policies can restrict communication to specific ports and protocols.

Combined Selectors: Using both pod and namespace selectors creates precise traffic rules.

### NetworkPolicy Limitations

NetworkPolicies have several limitations to consider:

- They require CNI plugin support (not all plugins implement NetworkPolicies)
- They operate at Layer 3/4 and cannot inspect application-layer protocols
- They cannot enforce encryption or authentication
- DNS traffic often needs special consideration

## Secrets Management Security

### Understanding Kubernetes Secrets

Kubernetes Secrets store sensitive data such as passwords, tokens, and keys. While they provide better security than hard-coding credentials, they require careful handling to maintain security.

Secrets are stored in etcd and can be encrypted at rest. They're only sent to nodes that have pods requiring them, and they're stored in tmpfs on nodes, never written to disk.

### Secret Security Best Practices

Encryption at Rest: Enable encryption at rest for etcd. Without this, secrets are merely base64 encoded, providing no real security.

RBAC Restrictions: Limit who can create, view, and list secrets. The ability to list secrets effectively grants read access to their contents.

Avoid Secret Sprawl: Regularly audit and remove unused secrets. Each secret increases the attack surface.

External Secret Management: Consider using external secret management systems like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault for enhanced security features.

Rotation Policies: Implement regular rotation of secrets, especially for critical credentials.

### Secret Usage Patterns

Environment Variables: Simple but visible in process listings and crash dumps.

Volume Mounts: More secure as files can have restricted permissions, but require application changes to read from files.

Image Pull Secrets: Special secrets for authenticating with private container registries.

## Practical Workshop Exercises

### Workshop: ServiceAccount Management

This section demonstrates ServiceAccount creation and token management with comprehensive examples.

```bash
# First, examine the default ServiceAccount that exists in every namespace
kubectl get sa default -o yaml
kubectl describe sa default

# Check current cluster authentication info
kubectl config view --minify
kubectl auth whoami

# Create a new namespace for security demonstrations
kubectl create namespace security-workshop

# Create a custom ServiceAccount with annotations
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: security-workshop
  annotations:
    description: "ServiceAccount for application workloads"
    owner: "platform-team"
    purpose: "workshop-demo"
EOF

# Create a long-lived token for the ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: app-service-account-token
  namespace: security-workshop
  annotations:
    kubernetes.io/service-account.name: app-service-account
type: kubernetes.io/service-account-token
EOF

# Retrieve and decode the token
TOKEN=$(kubectl get secret app-service-account-token -n security-workshop -o jsonpath='{.data.token}' | base64 -d)
echo "Token length: ${#TOKEN}"

# Test API access with the token
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
echo "API Server: $APISERVER"

# Test unauthenticated access (should fail)
curl -k $APISERVER/api/v1/namespaces/security-workshop/pods

# Test with ServiceAccount token (should get 403 Forbidden - authenticated but not authorized)
curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/security-workshop/pods

# Create multiple ServiceAccounts for different purposes
kubectl create sa monitoring-sa -n security-workshop
kubectl create sa deployment-sa -n security-workshop
kubectl create sa readonly-sa -n security-workshop

# List all ServiceAccounts
kubectl get sa -n security-workshop
```

### Workshop: RBAC Configuration

This section demonstrates comprehensive RBAC setup with progressive permission grants.

```bash
# Create a Role with specific permissions
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-workshop
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

# Create a Role for ConfigMap management
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: security-workshop
  name: configmap-manager
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Create RoleBindings
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: security-workshop
subjects:
- kind: ServiceAccount
  name: readonly-sa
  namespace: security-workshop
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Bind configmap-manager role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: manage-configmaps
  namespace: security-workshop
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: security-workshop
roleRef:
  kind: Role
  name: configmap-manager
  apiGroup: rbac.authorization.k8s.io
EOF

# Test permissions using impersonation
echo "Testing readonly-sa permissions:"
kubectl auth can-i get pods --as=system:serviceaccount:security-workshop:readonly-sa -n security-workshop
kubectl auth can-i create pods --as=system:serviceaccount:security-workshop:readonly-sa -n security-workshop
kubectl auth can-i get configmaps --as=system:serviceaccount:security-workshop:readonly-sa -n security-workshop

echo "Testing app-service-account permissions:"
kubectl auth can-i get configmaps --as=system:serviceaccount:security-workshop:app-service-account -n security-workshop
kubectl auth can-i create configmaps --as=system:serviceaccount:security-workshop:app-service-account -n security-workshop

# Create a ClusterRole for node information
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes"]
  verbs: ["get", "list"]
EOF

# Create ClusterRoleBinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: security-workshop
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify cluster-level permissions
kubectl auth can-i get nodes --as=system:serviceaccount:security-workshop:monitoring-sa

# Create an aggregated ClusterRole
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.authorization.k8s.io/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["services/endpoints"]
  verbs: ["get", "list"]
EOF

# Create the parent aggregated role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-aggregated
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-monitoring: "true"
rules: []
EOF

# Deploy a pod with custom ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: security-workshop
spec:
  serviceAccountName: app-service-account
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/rbac-test-pod -n security-workshop --timeout=60s

# Test in-pod permissions
kubectl exec -it rbac-test-pod -n security-workshop -- kubectl get configmaps
kubectl exec -it rbac-test-pod -n security-workshop -- kubectl get pods
```

### Workshop: Network Security Implementation

This section demonstrates NetworkPolicy configuration with practical examples.

```bash
# Create test namespaces with labels
kubectl create namespace prod-frontend
kubectl create namespace prod-backend
kubectl create namespace prod-database
kubectl label namespace prod-frontend environment=production tier=frontend
kubectl label namespace prod-backend environment=production tier=backend
kubectl label namespace prod-database environment=production tier=database

# Deploy test applications
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
  namespace: prod-frontend
  labels:
    app: frontend
    version: v1
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
  namespace: prod-backend
  labels:
    app: backend
    version: v1
spec:
  containers:
  - name: httpd
    image: httpd:alpine
    ports:
    - containerPort: 80
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: prod-database
  labels:
    app: postgres
    version: v13
spec:
  containers:
  - name: postgres
    image: postgres:13-alpine
    env:
    - name: POSTGRES_PASSWORD
      value: testpassword
    ports:
    - containerPort: 5432
EOF

# Create a test client pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-client
  namespace: prod-frontend
  labels:
    app: client
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sleep", "3600"]
EOF

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod/frontend-app -n prod-frontend --timeout=60s
kubectl wait --for=condition=ready pod/backend-api -n prod-backend --timeout=60s
kubectl wait --for=condition=ready pod/test-client -n prod-frontend --timeout=60s

# Test initial connectivity (should work if CNI supports it)
kubectl exec -n prod-frontend test-client -- wget -qO- --timeout=2 frontend-app.prod-frontend.svc.cluster.local 2>/dev/null || echo "Connection failed or CNI doesn't support NetworkPolicies"

# Implement default deny policies
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod-frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod-backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod-database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# Create specific allow policies
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-traffic
  namespace: prod-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: prod-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-backend
  namespace: prod-database
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Test connectivity after policies
echo "Testing frontend connectivity:"
kubectl exec -n prod-frontend test-client -- wget -qO- --timeout=2 frontend-app.prod-frontend.svc.cluster.local 2>/dev/null || echo "Connection blocked by NetworkPolicy or CNI doesn't support it"

# Create egress policies
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: prod-frontend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
EOF
```

### Workshop: Pod Security Contexts

This section demonstrates comprehensive pod security configurations.

```bash
# Create a namespace with Pod Security Standards labels
kubectl create namespace secure-apps
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Deploy a secured pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-webapp
  namespace: secure-apps
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]
  containers:
  - name: webapp
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
      requests:
        memory: "64Mi"
        cpu: "50m"
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
  - name: nginx-conf
    configMap:
      name: nginx-config
      optional: true
EOF

# Create a pod that violates security standards (should fail in restricted namespace)
kubectl create namespace restricted-apps
kubectl label namespace restricted-apps \
  pod-security.kubernetes.io/enforce=restricted

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: restricted-apps
spec:
  containers:
  - name: privileged
    image: nginx:alpine
    securityContext:
      privileged: true
EOF

# The above should fail. Now create a compliant pod:
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: restricted-apps
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}
EOF

# Verify security contexts
kubectl get pod compliant-pod -n restricted-apps -o jsonpath='{.spec.securityContext}' | python3 -m json.tool
kubectl exec compliant-pod -n restricted-apps -- id
kubectl exec compliant-pod -n restricted-apps -- ps aux
```

### Workshop: Security Auditing and Monitoring

This section demonstrates security monitoring and auditing capabilities.

```bash
# Check RBAC permissions for all ServiceAccounts
cat <<'SCRIPT' > audit-rbac.sh
#!/bin/bash
NAMESPACE=${1:-default}
echo "Auditing RBAC permissions in namespace: $NAMESPACE"
echo "========================================="

for sa in $(kubectl get sa -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  echo ""
  echo "ServiceAccount: $sa"
  echo "Permissions:"
  kubectl auth can-i --list --as=system:serviceaccount:$NAMESPACE:$sa -n $NAMESPACE 2>/dev/null | head -20
  echo "---"
done
SCRIPT

chmod +x audit-rbac.sh
./audit-rbac.sh security-workshop

# Find overprivileged ServiceAccounts
echo "Checking for ServiceAccounts with cluster-admin privileges:"
kubectl get clusterrolebindings -o json | \
  python3 -c "import sys, json; crbs = json.load(sys.stdin)['items']; [print(f\"ClusterRoleBinding: {crb['metadata']['name']}, ServiceAccount: {subject['namespace']}/{subject['name']}\") for crb in crbs if crb['roleRef']['name'] == 'cluster-admin' for subject in crb.get('subjects', []) if subject['kind'] == 'ServiceAccount']"

# Check for pods running as root
echo "Pods running as root or privileged:"
kubectl get pods --all-namespaces -o json | \
  python3 -c "import sys, json; pods = json.load(sys.stdin)['items']; [print(f\"{pod['metadata']['namespace']}/{pod['metadata']['name']}: privileged={container.get('securityContext', {}).get('privileged', False)}, runAsUser={container.get('securityContext', {}).get('runAsUser', 'not set')}\") for pod in pods for container in pod['spec']['containers'] if container.get('securityContext', {}).get('privileged', False) or container.get('securityContext', {}).get('runAsUser', None) == 0]"

# List all Secrets and their types
echo "Secret inventory:"
kubectl get secrets --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.type,AGE:.metadata.creationTimestamp

# Check for unused ServiceAccounts
cat <<'SCRIPT' > find-unused-sa.sh
#!/bin/bash
NAMESPACE=${1:-default}
echo "Checking for unused ServiceAccounts in namespace: $NAMESPACE"

# Get all ServiceAccounts
for sa in $(kubectl get sa -n $NAMESPACE -o jsonpath='{.items[*].metadata.name}'); do
  # Check if any pods are using this ServiceAccount
  pod_count=$(kubectl get pods -n $NAMESPACE -o json | grep -c "\"serviceAccountName\": \"$sa\"")
  if [ "$pod_count" -eq 0 ] && [ "$sa" != "default" ]; then
    echo "Unused ServiceAccount: $sa"
  fi
done
SCRIPT

chmod +x find-unused-sa.sh
./find-unused-sa.sh security-workshop

# Monitor API access patterns (requires API audit logs to be enabled)
echo "Recent authentication failures (if audit logs are available):"
kubectl get events --all-namespaces --field-selector reason=FailedAuthentication 2>/dev/null || echo "Audit logs not available or no recent failures"

# Check NetworkPolicy coverage
echo "Namespaces without NetworkPolicies:"
for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
  policy_count=$(kubectl get networkpolicies -n $ns --no-headers 2>/dev/null | wc -l)
  if [ "$policy_count" -eq 0 ]; then
    echo "  - $ns (no network policies)"
  fi
done

# Security compliance check
cat <<'SCRIPT' > security-check.sh
#!/bin/bash
echo "Kubernetes Security Compliance Check"
echo "===================================="

# Check if Pod Security Standards are enforced
echo ""
echo "1. Pod Security Standards enforcement:"
kubectl get namespaces -o json | \
  python3 -c "import sys, json; nss = json.load(sys.stdin)['items']; [print(f\"  {ns['metadata']['name']}: enforce={ns['metadata'].get('labels', {}).get('pod-security.kubernetes.io/enforce', 'not set')}\") for ns in nss]" | head -10

# Check for default deny NetworkPolicies
echo ""
echo "2. Default deny NetworkPolicies:"
for ns in kube-system default; do
  if kubectl get networkpolicy -n $ns default-deny-all 2>/dev/null >/dev/null; then
    echo "  $ns: default-deny policy exists"
  else
    echo "  $ns: WARNING - no default-deny policy"
  fi
done

# Check etcd encryption (requires cluster admin)
echo ""
echo "3. Encryption at rest:"
kubectl get secrets -n kube-system -o json 2>/dev/null | grep -q "k8s:enc:aescbc" && echo "  Secrets appear to be encrypted" || echo "  Unable to verify encryption status"

echo ""
echo "Security check complete"
SCRIPT

chmod +x security-check.sh
./security-check.sh
```

### Workshop: Secrets Management

This section demonstrates secure secret handling practices.

```bash
# Create different types of secrets
kubectl create namespace secrets-demo

# Create a generic secret from literal values
kubectl create secret generic app-credentials \
  --from-literal=username=appuser \
  --from-literal=password='S3cur3P@ssw0rd!' \
  -n secrets-demo

# Create a secret from files
echo -n 'admin' > ./username.txt
echo -n 'secretpassword' > ./password.txt
kubectl create secret generic file-secret \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt \
  -n secrets-demo
rm ./username.txt ./password.txt

# Create a TLS secret
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"
kubectl create secret tls myapp-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n secrets-demo
rm tls.crt tls.key

# Create a Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com \
  -n secrets-demo

# Use secrets in pods - as environment variables
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
  namespace: secrets-demo
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-credentials
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-credentials
          key: password
EOF

# Use secrets as volume mounts
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
  namespace: secrets-demo
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: app-credentials
      defaultMode: 0400
      items:
      - key: username
        path: app-username
      - key: password
        path: app-password
EOF

# Wait for pods to be ready
kubectl wait --for=condition=ready pod/secret-env-pod -n secrets-demo --timeout=60s
kubectl wait --for=condition=ready pod/secret-volume-pod -n secrets-demo --timeout=60s

# Verify secret access
echo "Checking environment variables:"
kubectl exec secret-env-pod -n secrets-demo -- printenv | grep SECRET_

echo "Checking volume mounts:"
kubectl exec secret-volume-pod -n secrets-demo -- ls -la /etc/secrets/
kubectl exec secret-volume-pod -n secrets-demo -- cat /etc/secrets/app-username

# Demonstrate secret rotation
echo "Rotating secret..."
kubectl create secret generic app-credentials \
  --from-literal=username=appuser \
  --from-literal=password='N3wS3cur3P@ss!' \
  --dry-run=client -o yaml | kubectl apply -f - -n secrets-demo

# Verify the update propagated (may take up to a minute for volume mounts)
sleep 5
kubectl exec secret-volume-pod -n secrets-demo -- cat /etc/secrets/app-password

# Clean up sensitive operations
kubectl delete pod secret-env-pod secret-volume-pod -n secrets-demo
```
