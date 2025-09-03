# 03 - Kubernetes Basics

## Cluster Architecture

Kubernetes stands as a comprehensive container orchestration platform designed to manage containerized applications across clusters of machines. The platform provides robust infrastructure for running distributed systems with built-in resilience, managing scaling operations, handling failover scenarios, implementing deployment patterns, and orchestrating numerous other operational concerns. Operating through a declarative model, Kubernetes allows users to define their desired application state, then continuously reconciles actual state with this specification.

The architecture of a Kubernetes cluster follows a clear two-tier design pattern, separating control plane components from worker nodes. This architectural separation enables efficient workload management and seamless scalability, with the control plane orchestrating global cluster decisions while worker nodes handle the execution of containerized applications.

### Control Plane Components

The control plane functions as the cluster's central nervous system, containing the essential services responsible for coordinating all cluster-wide operations. Development environments can run the control plane on a single node, while production deployments typically distribute these components across multiple nodes to guarantee high availability and fault tolerance.

#### kube-apiserver

The kube-apiserver stands at the heart of the control plane, serving as the singular entry point for all cluster interactions. This component exposes the REST API that enables communication between clients, administrative tools, and internal cluster components. Every operation within the cluster, whether originating from administrators, applications, or automated processes, must pass through this centralized API gateway.

#### etcd

Serving as the cluster's memory, etcd provides a distributed key-value store that maintains the complete record of cluster state, configuration parameters, and metadata. This component guarantees data consistency throughout the cluster and functions as the authoritative source for all cluster state information.

#### kube-scheduler

The kube-scheduler takes responsibility for intelligent workload placement across the cluster. When new Pods require scheduling, this component evaluates resource requirements, placement constraints, and current cluster conditions to identify the optimal worker nodes for each workload.

#### kube-controller-manager

Operating as a unified collection of control loops, the kube-controller-manager continuously monitors cluster state and initiates corrective actions to align actual conditions with desired configurations. These controllers manage diverse responsibilities including node lifecycle, replication enforcement, endpoint management, and service account provisioning.

### Worker Node Components

Worker nodes constitute the cluster's execution layer, providing the computational resources where containerized applications actually run. These nodes maintain constant communication with the control plane while operating essential services that enable container execution and network connectivity.

#### Container Runtime

The container runtime delivers the core functionality needed to execute containers on each node. This component manages the complete container lifecycle, from pulling container images through creating instances and maintaining execution according to control plane specifications.

#### kubelet

Serving as the primary agent on each worker node, the kubelet ensures that containers run successfully within Pods by implementing PodSpecs received from the control plane. The kubelet exclusively manages containers created through Kubernetes, ignoring any containers started through other mechanisms. Its responsibilities include registering nodes with the API server, transmitting regular status updates about node and pod health, and executing pod lifecycle operations. After the scheduler assigns Pods to nodes, the kubelet receives these assignments and coordinates with the container runtime to create, monitor, and maintain containers according to their specifications.

#### kube-proxy

The kube-proxy component runs on every node as a network proxy, implementing the networking aspects of the Kubernetes Service abstraction. It establishes and maintains network rules that enable communication with Pods from both internal cluster sessions and external connections. Where possible, kube-proxy leverages the operating system's packet filtering capabilities; otherwise, it handles traffic forwarding directly. This component proves essential for enabling service discovery and implementing load balancing throughout the cluster.

### Cluster Add-ons

Add-ons extend Kubernetes functionality through pods and services that implement additional cluster features beyond core capabilities.

#### DNS

The DNS add-on delivers cluster-wide name resolution services, enabling services and pods to discover one another through DNS names instead of IP addresses. This capability proves essential in dynamic environments where IP addresses change frequently.

#### Web UI (Dashboard)

The Dashboard provides an intuitive web-based interface for cluster management, enabling users to deploy applications, investigate issues, and manage resources through a graphical environment.

#### Container Resource Monitoring

Resource monitoring add-ons collect and centralize container metrics, offering visibility into resource consumption and performance patterns across the entire cluster.

#### Cluster-level Logging

Logging add-ons aggregate logs from all containers into centralized storage, simplifying application debugging and system health monitoring.



## Kubernetes Objects - The Building Blocks

### Understanding Kubernetes Objects

Kubernetes objects represent persistent entities that define your cluster's state. These fundamental building blocks describe running containerized applications, available resources, and behavioral policies. Each object serves as a "record of intent" – creating an object communicates to Kubernetes your desired cluster workload configuration.

### The Declarative Model

The declarative model underpinning Kubernetes allows users to specify desired state while the system handles the implementation details to achieve and maintain that state. This approach differs fundamentally from imperative systems that require explicit step-by-step instructions. The declarative model enables resilience and self-healing behaviors, as Kubernetes automatically responds to failures and changes to preserve the desired state.

### Object Specification and Status

Every Kubernetes object contains two essential nested fields that define its configuration and track its state.

The Spec Field captures the desired state you want for the object. When creating objects, you populate the spec with characteristics defining how the resource should exist in your cluster. This declaration of intent specifies configuration details including replica counts for deployments, container images, resource limits, and networking parameters.

The Status Field reflects the current observed state as determined by the Kubernetes system. The control plane continuously monitors actual object states and updates status fields accordingly. This ongoing reconciliation between spec and status drives Kubernetes' self-healing and auto-scaling behaviors.

### Required Object Fields

All Kubernetes objects must include specific metadata enabling identification and management within the cluster.

The apiVersion Field indicates which Kubernetes API version you're using for object creation. This versioning enables API evolution while preserving backward compatibility, with different versions potentially offering varied features for the same object types.

The kind Field specifies the object type being created, such as Pod, Service, Deployment, or ConfigMap. This designation determines which controller manages the object and defines expected behaviors.

The metadata Field holds identification data that uniquely distinguishes the object. This encompasses the object's name, optional namespace assignment, labels, annotations, and additional organizational information. Metadata provides the contextual framework Kubernetes needs for object management throughout its lifecycle.

The spec Field contains the desired state definition specific to each object type. Different kinds of objects have unique spec structures containing fields appropriate to their purpose and functionality.

## API Resources

The Kubernetes API resources form the essential infrastructure that enables application deployment, configuration, and management across cluster environments. These resources offer declarative interfaces for specifying application deployment patterns, configuration parameters, and operational behaviors.

### Core Resource Types

#### Pods

Pods represent the atomic unit of deployment in Kubernetes, serving as the smallest executable primitive. Each Pod encapsulates one or more containers that share common networking, storage, and lifecycle management. While containers within Pods communicate directly via localhost, Pods themselves remain ephemeral, subject to creation, destruction, and recreation based on cluster needs.

#### Deployments

Deployments extend Pod capabilities with production-ready features including automated replication and rolling update mechanisms. When applications require horizontal scaling or zero-downtime updates, Deployments orchestrate multiple Pod replicas and coordinate updates through gradual instance replacement, maintaining service availability throughout the update process.

#### ConfigMaps

ConfigMaps enable configuration externalization by storing configuration files, environment variables, and initialization parameters as Kubernetes resources. This architectural separation keeps applications portable across environments while allowing configuration modifications without container image rebuilds.

#### Services

Services solve networking challenges in dynamic container environments by establishing stable network endpoints for application access. As Pods undergo frequent creation and destruction, Services function as load balancers, distributing traffic across available Pod instances while preserving consistent access points despite underlying Pod changes.

#### Ingress and Gateway APIs

Ingress and Gateway APIs expand networking capabilities through reverse proxy functionality, enabling external cluster access. These resources manage SSL termination, path-based routing, and host-based routing, allowing multiple applications to share entry points while directing traffic to appropriate backend services.

#### Persistent Volumes

Persistent Volumes address data persistence needs by providing storage resources independent of Pod lifecycles. Unlike ephemeral Pod storage, Persistent Volumes preserve critical application data through Pod restarts, rescheduling, and updates, supporting stateful applications including databases and file storage systems.

## Labels and Selectors - Organizing Resources

### The Purpose of Labels

Labels consist of key-value pairs attached to Kubernetes objects, serving as identifying attributes meaningful to users while carrying no inherent significance to the core system. These labels represent one of Kubernetes' most powerful organizational tools, enabling users to project their organizational structures onto system objects through loose coupling. Unlike rigid hierarchical structures in traditional infrastructure, labels offer flexible, multi-dimensional organization reflecting real-world complexity.

Labels facilitate efficient queries and watches, making them particularly valuable for user interfaces and command-line operations. They differ fundamentally from annotations: labels identify and select objects, while annotations store non-identifying metadata.

### Multi-Dimensional Organization

Labels reveal their true potential through simultaneous representation of multiple organizational dimensions. Service deployments and batch processing pipelines often exhibit multi-dimensional characteristics spanning partitions, release tracks, tiers, and micro-services. Traditional hierarchical representations cannot adequately capture these cross-cutting relationships.

Typical label dimensions encompass release tracks (stable, canary), environments (development, qa, production), tiers (frontend, backend, cache), customer partitions, and temporal tracks (daily, weekly). Combining these dimensions creates sophisticated organizational schemes reflecting actual operational requirements rather than infrastructure constraints.

### Label Selectors

Label selectors provide the core grouping mechanism in Kubernetes, enabling identification and operation on object sets. The system implements two selector types: equality-based and set-based, each addressing different use cases with varying expressiveness levels.

Equality-Based Selectors employ straightforward equality and inequality operators. They support three operations: equals (= or ==) and not-equals (!=). These selectors handle most basic filtering requirements effectively. Multiple requirements combine through AND logic, requiring all conditions to match for object selection.

Set-Based Selectors offer enhanced filtering capabilities using set membership operations. They support "in", "notin", and "exists" operators, enabling complex selection logic. The "in" operator selects resources with label values within specified sets. The "notin" operator selects resources with label values outside specified sets. The "exists" operator selects resources possessing specific label keys, regardless of values.

## Namespaces - Virtual Clusters

### The Concept of Namespaces

Namespaces establish isolation boundaries for resource groups within single physical clusters, effectively creating virtual cluster environments. They provide naming scope, requiring unique resource names within namespaces while permitting duplication across different namespaces. This abstraction enables multi-tenancy and logical resource separation without managing multiple physical clusters.

Namespaces target environments supporting numerous users across multiple teams or projects. They deliver organizational benefits alongside technical capabilities including resource isolation, access control boundaries, and resource quota enforcement.

### Namespace Scope and Boundaries

The Kubernetes system distinguishes between namespaced and cluster-scoped resources. Namespaced resources encompass most workload-related objects including Pods, Services, Deployments, and ConfigMaps. These resources exist within specific namespace contexts, isolated from identical resources in other namespaces.

Cluster-scoped resources operate beyond namespace boundaries, affecting the entire cluster. These include Nodes, PersistentVolumes, StorageClasses, and ClusterRoles. Such resources remain visible and accessible cluster-wide, independent of namespace context.

### Initial System Namespaces

Kubernetes initializes with four default namespaces, each fulfilling specific operational purposes. The "default" namespace enables immediate cluster usage without namespace creation. The "kube-system" namespace contains objects created by Kubernetes itself. The "kube-public" namespace remains readable by all clients, including unauthenticated ones. The "kube-node-lease" namespace maintains Lease objects associated with nodes for health monitoring.

## Options for Running Kubernetes Applications

Kubernetes provides multiple deployment approaches, each optimized for specific application characteristics and operational requirements:

### Pod (Basic Unit)

Pods form the foundation of all Kubernetes applications, representing the smallest deployable units capable of containing one or more containers. While direct Pod creation remains possible, higher-level controllers typically manage Pods for enhanced functionality.

### Deployment (Recommended for Stateless Apps)

Stateless applications that don't maintain persistent data or user sessions benefit most from Deployments. These controllers enhance Pod functionality through:

- Automated horizontal scaling
- Zero-downtime rolling updates
- Self-healing through automatic Pod replacement

### StatefulSet (For Stateful Apps)

Applications requiring persistent data storage, stable network identities, or ordered deployment sequences need StatefulSet controllers. StatefulSets deliver:

- Guaranteed persistent storage
- Stable, predictable network identities
- Ordered, controlled deployment and scaling

### Helm (Package Management)

Helm simplifies application deployment through package management, offering pre-configured templates with managed dependencies.



## Working with kubectl

The kubectl command-line tool provides the primary interface for Kubernetes cluster interaction, delivering comprehensive functionality for managing cluster operations and application deployments.

kubectl functions through a command-line interface supporting extensive Kubernetes operations, from basic resource queries through complex deployment configurations. The tool communicates with the Kubernetes API server to execute operations throughout the cluster.

## Workshop: Essential kubectl Commands

These fundamental Kubernetes commands serve as discovery and exploration tools, revealing cluster structure, available resources, and providing assistance when needed. Consider them your information gathering toolkit – they read cluster state without modifying anything, providing crucial insights about available resources, configurations, and possible actions.

### Cluster Information Commands

#### `kubectl cluster-info`

Purpose: Reveals cluster connection details and running services

```bash
# Show basic cluster information
kubectl cluster-info

# Show detailed cluster information with additional services
kubectl cluster-info dump
```

Example Output:

```
Kubernetes control plane is running at https://kubernetes.docker.internal:6443
CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```



#### `kubectl get nodes`

Purpose: Displays all cluster nodes with status and basic details

```bash
# List all nodes
kubectl get nodes

# Show detailed node information
kubectl get nodes -o wide

# Show node labels
kubectl get nodes --show-labels

# Get specific node details
kubectl describe node node1
```

Example Output:

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   45d   v1.28.2
```



### API Discovery Commands

#### `kubectl explain`

Purpose: Provides documentation for Kubernetes resources and fields

```bash
# Explain a resource type
kubectl explain pod

# Explain specific fields (use dot notation)
kubectl explain pod.spec

# Explain nested fields
kubectl explain pod.spec.containers

# Get complete field documentation
kubectl explain pod --recursive
```

Example Usage:

```bash
# Understand deployment structure
kubectl explain deployment.spec.template.spec.containers

# Learn about service types
kubectl explain service.spec.type
```



#### `kubectl api-resources`

Purpose: Enumerates all available cluster resource types

```bash
# List all API resources
kubectl api-resources

# Filter by API group
kubectl api-resources --api-group=apps

# Show only namespaced resources
kubectl api-resources --namespaced=true

# Show resources with short names
kubectl api-resources -o wide
```

Example Output:

```
NAME                SHORTNAMES   APIVERSION        NAMESPACED   KIND
pods                po           v1                true         Pod
services            svc          v1                true         Service
deployments         deploy       apps/v1           true         Deployment
```



#### `kubectl api-versions`

Purpose: Displays all API versions supported by your cluster

```bash
# List all API versions
kubectl api-versions

# Check if specific version is supported
kubectl api-versions | grep apps
```

Example Output:

```
admissionregistration.k8s.io/v1
apps/v1
autoscaling/v1
autoscaling/v2
batch/v1
```



#### `kubectl -h` Help System

Purpose: Navigate kubectl help documentation

```bash
# Browse main kubectl help
kubectl -h | less

# Get help for specific commands
kubectl get -h | less
kubectl create -h | less
kubectl apply -h | less

# Get help for resource types
kubectl create deployment -h | less
```
