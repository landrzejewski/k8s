# 09 - Storage

## Understanding Kubernetes Storage Architecture

### The Fundamental Challenge of Container Storage

Containers are designed to be ephemeral and stateless by nature. When a container starts, it receives a fresh filesystem layer, and when it terminates, all changes made to that filesystem disappear. This design principle works perfectly for stateless applications but presents significant challenges for applications that need to persist data, share information between containers, or maintain state across restarts.

Consider a database container: without persistent storage, every restart would result in complete data loss. Similarly, when multiple containers need to collaborate by sharing files, they require a common storage mechanism that transcends individual container boundaries. Kubernetes addresses these challenges through a comprehensive storage architecture that provides multiple abstraction layers, each serving specific purposes.

### The Storage Abstraction Hierarchy

Kubernetes implements a sophisticated storage model that separates the concerns of different stakeholders in the cluster. This separation allows infrastructure teams to manage storage resources while application developers focus on their storage requirements without needing deep knowledge of the underlying infrastructure.

The storage hierarchy consists of four primary abstractions:

1. Volumes: These represent the most basic storage unit in Kubernetes. A Volume is essentially a directory that is accessible to containers within a Pod. Unlike the ephemeral filesystem of a container, a Volume's lifecycle is tied to the Pod itself, meaning data persists across container restarts within the same Pod instance. However, when the Pod is deleted, standard volumes are also removed.
    
2. PersistentVolumes (PV): These are cluster-wide resources that represent actual storage capacity in your infrastructure. A PersistentVolume might correspond to a physical disk, a partition, a network storage system, or cloud-based block storage. Cluster administrators typically provision these resources, either manually (static provisioning) or through automated systems (dynamic provisioning). The key characteristic of a PV is that it exists independently of any Pod or application using it.
    
3. PersistentVolumeClaims (PVC): These are requests for storage made by users or applications. A PVC expresses storage needs in terms of capacity and access patterns without requiring knowledge of how that storage is actually provided. Think of a PVC as a "storage voucher" that applications can redeem for actual storage resources. The beauty of this abstraction is that developers can write applications that request "10GB of storage with read-write access" without knowing whether that storage comes from local SSDs, network-attached storage, or cloud providers.
    
4. StorageClasses: These define different tiers or types of storage available in the cluster. A StorageClass might represent "fast SSD storage," "replicated network storage," or "archive storage." StorageClasses enable dynamic provisioning, where PersistentVolumes are created automatically when PersistentVolumeClaims request them, eliminating the need for manual PV pre-provisioning.
    

### Ephemeral Storage: Temporary by Design

Ephemeral storage serves temporary needs and is ideal for use cases where data persistence isn't required beyond the Pod's lifecycle. This type of storage is perfect for scratch space, caches, temporary file processing, and inter-container communication within a Pod.

The most common ephemeral volume type is emptyDir, which creates an empty directory when a Pod is scheduled to a node. All containers in the Pod can read and write to this directory, making it an excellent mechanism for sharing data between containers. The storage backing an emptyDir can be the node's local filesystem or, for performance-critical applications that can afford to lose data on node failure, memory-backed tmpfs.

Other ephemeral volume types serve specialized purposes:

- configMap volumes inject configuration data from ConfigMap resources into Pods
- secret volumes provide sensitive data like passwords or certificates to containers
- downwardAPI volumes expose Pod and container metadata to the running application

### Persistent Storage: Data That Survives

Persistent storage maintains data independently of Pod lifecycles. This storage type is essential for stateful applications like databases, content management systems, message queues, and any application where data must survive Pod restarts, rescheduling, or failures.

The persistence guarantee means that even if a Pod is deleted and recreated, moved to a different node, or experiences a crash, the data remains intact and can be reattached to the new Pod instance. This capability is fundamental for building reliable stateful services in Kubernetes.

## Volume Types in Detail

### EmptyDir Volumes: Shared Temporary Storage Within Pods

EmptyDir volumes are one of the most frequently used volume types due to their simplicity and versatility. When Kubernetes creates a Pod with an emptyDir volume, it allocates an empty directory on the node where the Pod runs. This directory is accessible to all containers within the Pod, providing a shared filesystem namespace.

The emptyDir volume offers two storage media options:

1. Disk-based (default): The directory is created on the node's filesystem. This provides standard performance and capacity limited by available disk space. Data survives container crashes but is lost if the Pod is evicted or deleted.
    
2. Memory-based (tmpfs): By specifying `medium: Memory`, the emptyDir uses RAM instead of disk storage. This provides exceptional performance for I/O-intensive operations but consumes memory resources and loses data if the node reboots.
    

Common use cases for emptyDir volumes include:

- Sharing files between containers in a Pod (such as a web server and a log processor)
- Providing scratch space for computations that generate large intermediate files
- Buffering data before writing to slower persistent storage
- Implementing cache layers that can be rebuilt if lost

Important considerations when using emptyDir:

- Data is lost when the Pod is removed from the node
- Storage capacity is limited by the node's available resources
- No data migration occurs if the Pod is rescheduled to a different node
- Memory-based emptyDir counts against the container's memory limits

### NFS Volumes: Network-Attached Storage for Node Independence

Network File System (NFS) volumes provide true shared storage that is independent of any specific node in the cluster. Unlike HostPath volumes that tie Pods to specific nodes, NFS volumes allow Pods to access the same data regardless of which node they run on. This makes NFS ideal for applications that need to share data across multiple Pods or survive Pod rescheduling to different nodes.

NFS offers several advantages:

- Node Independence: Pods can be rescheduled to any node and still access their data
- True Shared Access: Multiple Pods can read and write to the same files simultaneously
- External Storage: Data lives outside the cluster, simplifying backup and disaster recovery
- Legacy Integration: Easy integration with existing NFS infrastructure

However, NFS also has considerations:

- Network latency affects performance compared to local storage
- Requires external NFS server setup and maintenance
- File locking and consistency models may differ from local filesystems
- Security requires careful configuration of export rules and permissions

### HostPath Volumes: Direct Access to Node Filesystem

HostPath volumes mount files or directories from the host node's filesystem directly into Pods. This volume type provides a way for Pods to access node-level resources, interact with the container runtime, or use storage that must persist beyond the Pod lifecycle but remain tied to a specific node.

HostPath supports several types that control behavior when the specified path doesn't exist:

- DirectoryOrCreate: Creates the directory if it doesn't exist
- Directory: The directory must already exist
- FileOrCreate: Creates an empty file if it doesn't exist
- File: The file must already exist
- Socket: A UNIX socket must exist at the path
- CharDevice: A character device must exist at the path
- BlockDevice: A block device must exist at the path

While HostPath volumes have legitimate uses, they introduce significant security and portability concerns:

Security risks:

- Pods gain access to sensitive host filesystem areas
- Privilege escalation possibilities if not carefully restricted
- Potential for container escape scenarios

Portability issues:

- Pods become tied to specific nodes
- Different nodes might have different directory structures
- Data doesn't follow Pods when they're rescheduled

Appropriate use cases for HostPath are limited and specific:

- Development and testing on single-node clusters
- Accessing node-specific resources like `/dev` devices
- Running system-level monitoring or logging agents
- Container runtime integration (accessing Docker socket)

Production clusters should avoid HostPath volumes except for these specific system-level use cases, preferring PersistentVolumes for application data.

## Workshop: Hands-On Storage Exploration

This workshop provides practical exercises to understand Kubernetes storage concepts through real examples. Each exercise builds upon previous concepts, demonstrating increasingly sophisticated storage patterns.

### Exercise 1: Working with EmptyDir Volumes

Let's start by creating a Pod with two containers that share data through an emptyDir volume. This example demonstrates inter-container communication and data sharing patterns.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
  labels:
    app: storage-demo
    type: emptydir
spec:
  containers:
  - name: writer-container
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Writer container starting at $(date)" > /shared-data/startup.log
      counter=0
      while true; do 
        counter=$((counter + 1))
        echo "$(date '+%Y-%m-%d %H:%M:%S'): Message $counter from writer" >> /shared-data/messages.log
        echo "Total messages written: $counter" > /shared-data/counter.txt
        sleep 10
      done
    volumeMounts:
    - name: cache-volume
      mountPath: /shared-data
  - name: reader-container
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Reader container starting, waiting for writer..." 
      sleep 5
      while true; do
        echo "========== Reader Report at $(date '+%Y-%m-%d %H:%M:%S') =========="
        if [ -f /shared-data/counter.txt ]; then
          cat /shared-data/counter.txt
          echo "Latest messages:"
          tail -3 /shared-data/messages.log 2>/dev/null || echo "No messages yet"
        else
          echo "Waiting for writer to create files..."
        fi
        echo ""
        sleep 15
      done
    volumeMounts:
    - name: cache-volume
      mountPath: /shared-data
  volumes:
  - name: cache-volume
    emptyDir: {}
```

Deploy and interact with the emptyDir demo:

```bash
# Create the Pod
kubectl apply -f emptydir-demo.yaml

# Wait for Pod to be running
kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=30s

# Watch the writer container logs
kubectl logs emptydir-demo -c writer-container --tail=5

# Watch the reader container logs  
kubectl logs emptydir-demo -c reader-container --tail=10

# Execute commands in the writer container to add custom data
kubectl exec emptydir-demo -c writer-container -- sh -c 'echo "Manual entry: Important data" >> /shared-data/messages.log'

# Verify the reader can see the manually added data
kubectl exec emptydir-demo -c reader-container -- tail -5 /shared-data/messages.log

# Examine the shared directory structure
kubectl exec emptydir-demo -c writer-container -- ls -la /shared-data/

# Check disk usage of the emptyDir
kubectl exec emptydir-demo -c writer-container -- df -h /shared-data

# Clean up
kubectl delete pod emptydir-demo
```

### Exercise 2: Memory-backed EmptyDir for High Performance

This example demonstrates using RAM-backed storage for performance-critical operations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-emptydir-demo
  labels:
    app: storage-demo
    type: memory-emptydir
spec:
  containers:
  - name: performance-test
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Starting performance test with memory-backed storage"
      
      # Write test with small files
      echo "Writing 1000 small files..."
      start_time=$(date +%s)
      for i in $(seq 1 1000); do
        echo "Test data $i" > /cache/file_$i.txt
      done
      end_time=$(date +%s)
      echo "Write time: $((end_time - start_time)) seconds"
      
      # Read test
      echo "Reading all files..."
      start_time=$(date +%s)
      for i in $(seq 1 1000); do
        cat /cache/file_$i.txt > /dev/null
      done
      end_time=$(date +%s)
      echo "Read time: $((end_time - start_time)) seconds"
      
      # Keep container running for inspection
      echo "Test complete. Container will stay running for inspection."
      sleep 3600
    resources:
      requests:
        memory: "128Mi"
      limits:
        memory: "256Mi"
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
```

Deploy and test memory-backed storage:

```bash
# Create the Pod with memory-backed emptyDir
kubectl apply -f memory-emptydir-demo.yaml

# Wait for Pod to complete initial tests
kubectl wait --for=condition=Ready pod/memory-emptydir-demo --timeout=30s
sleep 10

# Check the performance test results
kubectl logs memory-emptydir-demo

# Verify the storage is memory-backed (should show tmpfs)
kubectl exec memory-emptydir-demo -- df -h /cache

# Check memory usage
kubectl exec memory-emptydir-demo -- cat /proc/meminfo | grep -E "MemFree|MemAvailable"

# Clean up
kubectl delete pod memory-emptydir-demo
```

### Exercise 3: HostPath Volume for Node-level Access

This example shows appropriate use of HostPath for system-level operations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
  labels:
    app: storage-demo
    type: hostpath
spec:
  containers:
  - name: node-inspector
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Pod $(hostname) started on node at $(date)" > /host-tmp/pod-marker.txt
      echo "Examining host temporary directory..."
      
      # Create a subdirectory for this Pod
      mkdir -p /host-tmp/pod-$(hostname)
      
      # Write some data
      echo "This file was created by Pod $(hostname)" > /host-tmp/pod-$(hostname)/data.txt
      
      # Monitor the directory
      while true; do
        echo "$(date '+%Y-%m-%d %H:%M:%S'): Files in host temp directory:"
        ls -la /host-tmp/ | head -20
        echo "Disk usage: $(du -sh /host-tmp 2>/dev/null | cut -f1)"
        echo "---"
        sleep 30
      done
    volumeMounts:
    - name: host-tmp
      mountPath: /host-tmp
  volumes:
  - name: host-tmp
    hostPath:
      path: /tmp/k8s-hostpath-demo
      type: DirectoryOrCreate
```

Deploy and explore HostPath behavior:

```bash
# Create the Pod with HostPath volume
kubectl apply -f hostpath-demo.yaml

# Wait for Pod to be ready
kubectl wait --for=condition=Ready pod/hostpath-demo --timeout=30s

# Check which node the Pod is running on
kubectl get pod hostpath-demo -o wide

# View the files created on the host
kubectl exec hostpath-demo -- ls -la /host-tmp/

# Create additional data
kubectl exec hostpath-demo -- sh -c 'echo "Additional data at $(date)" >> /host-tmp/persistent.log'

# Delete the Pod
kubectl delete pod hostpath-demo

# Recreate the Pod
kubectl apply -f hostpath-demo.yaml

# Wait for the new Pod to be ready
kubectl wait --for=condition=Ready pod/hostpath-demo --timeout=30s

# Verify data persisted on the node (may fail if Pod scheduled to different node)
kubectl exec hostpath-demo -- cat /host-tmp/persistent.log 2>/dev/null || echo "Pod likely scheduled to different node"

# Clean up
kubectl delete pod hostpath-demo
```

## NFS Storage: Node-Independent Persistent Storage

### Setting Up NFS for Kubernetes

NFS provides a network-based storage solution that allows multiple nodes and Pods to access the same files simultaneously. This section demonstrates how to use NFS for truly portable persistent storage that survives Pod rescheduling across nodes.

### Quick NFS Server Setup (For Testing)

If you don't have an existing NFS server, here's how to quickly set one up for testing. In production, use a dedicated NFS server or managed NFS service.

```yaml
# nfs-server-deployment.yaml
# Simple NFS server for testing (NOT for production use)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-server-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: itsthenetwork/nfs-server-alpine:12
        env:
        - name: SHARED_DIRECTORY
          value: "/exports"
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
        - name: storage
          mountPath: /exports
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: nfs-server-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
  namespace: default
spec:
  selector:
    app: nfs-server
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  clusterIP: None  # Headless service
```

Deploy the test NFS server:

```bash
# Deploy NFS server (for testing only)
kubectl apply -f nfs-server-deployment.yaml

# Wait for NFS server to be ready
kubectl wait --for=condition=Available deployment/nfs-server --timeout=60s

# Get the NFS server's IP address
kubectl get pod -l app=nfs-server -o wide

# Note the Pod IP for use in NFS volume definitions
NFS_SERVER_IP=$(kubectl get pod -l app=nfs-server -o jsonpath='{.items[0].status.podIP}')
echo "NFS Server IP: $NFS_SERVER_IP"
```

### Using NFS Volumes Directly in Pods

The simplest way to use NFS storage is to mount it directly in a Pod specification:

```yaml
# nfs-direct-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-direct-pod
  labels:
    app: nfs-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Pod $(hostname) writing to NFS at $(date)" > /nfs-data/$(hostname).txt
      counter=0
      while true; do
        counter=$((counter + 1))
        echo "$(date '+%Y-%m-%d %H:%M:%S'): Message $counter from $(hostname)" >> /nfs-data/shared-log.txt
        echo "Files in NFS share:"
        ls -la /nfs-data/
        sleep 30
      done
    volumeMounts:
    - name: nfs-volume
      mountPath: /nfs-data
  volumes:
  - name: nfs-volume
    nfs:
      server: "10.244.0.5"  # Replace with your NFS server IP
      path: "/"
      readOnly: false
```

Before deploying, update the NFS server IP:

```bash
# Replace the NFS server IP in the Pod definition
sed "s/10.244.0.5/$NFS_SERVER_IP/g" nfs-direct-pod.yaml | kubectl apply -f -

# Check Pod status
kubectl get pod nfs-direct-pod

# View the files created on NFS
kubectl exec nfs-direct-pod -- ls -la /nfs-data/

# Check the log file
kubectl exec nfs-direct-pod -- tail -5 /nfs-data/shared-log.txt
```

### NFS with PersistentVolumes and PersistentVolumeClaims

For production use, it's better to abstract NFS details using PV/PVC:

```yaml
# nfs-pv-pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    type: nfs
    environment: shared
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteMany  # NFS supports multiple readers/writers
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  nfs:
    server: "10.244.0.5"  # Replace with your NFS server IP
    path: "/"
    readOnly: false
  mountOptions:
  - hard
  - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  labels:
    app: nfs-demo
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-storage
  selector:
    matchLabels:
      type: nfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-app-deployment
  labels:
    app: nfs-app
spec:
  replicas: 3  # Multiple replicas can share the same NFS storage
  selector:
    matchLabels:
      app: nfs-app
  template:
    metadata:
      labels:
        app: nfs-app
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
        - |
          HOSTNAME=$(hostname)
          echo "Pod $HOSTNAME started at $(date)" >> /shared-data/pods-log.txt
          
          # Create a pod-specific file
          echo "This is pod $HOSTNAME" > /shared-data/pod-$HOSTNAME.txt
          
          # Main loop - read and write shared data
          while true; do
            # Write to shared log
            echo "$(date '+%Y-%m-%d %H:%M:%S'): Pod $HOSTNAME is active" >> /shared-data/activity.log
            
            # Read shared data
            echo "=== Pod $HOSTNAME reading shared data ==="
            echo "Active pods:"
            ls /shared-data/pod-*.txt 2>/dev/null | sed 's/.*pod-/  - /'
            
            echo "Last 3 activity entries:"
            tail -3 /shared-data/activity.log 2>/dev/null || echo "No activity yet"
            
            echo "---"
            sleep 20
          done
        volumeMounts:
        - name: nfs-storage
          mountPath: /shared-data
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
          limits:
            cpu: "100m"
            memory: "64Mi"
      volumes:
      - name: nfs-storage
        persistentVolumeClaim:
          claimName: nfs-pvc
```

Deploy and test the NFS PV/PVC setup:

```bash
# Update NFS server IP and create resources
sed "s/10.244.0.5/$NFS_SERVER_IP/g" nfs-pv-pvc.yaml | kubectl apply -f -

# Check PV and PVC binding
kubectl get pv nfs-pv
kubectl get pvc nfs-pvc

# Wait for deployment to be ready
kubectl wait --for=condition=Available deployment/nfs-app-deployment --timeout=60s

# Check all pods are running
kubectl get pods -l app=nfs-app

# Verify all pods can write to shared storage
kubectl exec deployment/nfs-app-deployment -- ls -la /shared-data/

# Check the shared activity log
kubectl exec deployment/nfs-app-deployment -- tail -10 /shared-data/activity.log

# Watch pods writing to shared storage in real-time
kubectl logs -l app=nfs-app --tail=5 -f
```

### Demonstrating Node Independence with NFS

This example shows how NFS storage remains accessible even when Pods are rescheduled to different nodes:

```yaml
# nfs-node-independence-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-writer
  labels:
    app: nfs-demo
    role: writer
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      # Write initial data
      echo "Important data written at $(date)" > /nfs-share/important-data.txt
      echo "Initial node: $(cat /etc/hostname)" >> /nfs-share/important-data.txt
      
      # Get node information
      echo "Running on node: $NODE_NAME" >> /nfs-share/important-data.txt
      
      # Write continuous data
      counter=0
      while true; do
        counter=$((counter + 1))
        echo "Entry $counter at $(date) from node $NODE_NAME" >> /nfs-share/continuous-log.txt
        sleep 10
      done
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    volumeMounts:
    - name: nfs-volume
      mountPath: /nfs-share
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-pvc
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node1  # Prefer first node initially
---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-reader
  labels:
    app: nfs-demo
    role: reader
spec:
  containers:
  - name: reader
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Reader pod started on node: $NODE_NAME"
      
      while true; do
        echo "=== Reader on node $NODE_NAME at $(date) ==="
        
        if [ -f /nfs-share/important-data.txt ]; then
          echo "Important data content:"
          cat /nfs-share/important-data.txt
        else
          echo "Waiting for data..."
        fi
        
        if [ -f /nfs-share/continuous-log.txt ]; then
          echo "Latest log entries:"
          tail -3 /nfs-share/continuous-log.txt
        fi
        
        echo "---"
        sleep 15
      done
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    volumeMounts:
    - name: nfs-volume
      mountPath: /nfs-share
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-pvc
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node2  # Prefer different node
```

Test node independence:

```bash
# Deploy writer and reader pods
kubectl apply -f nfs-node-independence-demo.yaml

# Check which nodes pods are running on
kubectl get pods -l app=nfs-demo -o wide

# Verify writer is writing data
kubectl logs nfs-writer --tail=5

# Verify reader can read the data (potentially from different node)
kubectl logs nfs-reader --tail=10

# Delete writer pod to simulate failure
kubectl delete pod nfs-writer

# Create a new writer pod (may schedule to different node)
kubectl apply -f nfs-node-independence-demo.yaml

# Verify new writer pod can still access previous data
kubectl exec nfs-writer -- cat /nfs-share/important-data.txt

# Check that both old and new data exist
kubectl exec nfs-reader -- ls -la /nfs-share/
```

### NFS Storage with StatefulSets

Using NFS with StatefulSets for shared configuration or data:

```yaml
# nfs-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: nfs-stateful-service
  labels:
    app: nfs-stateful
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nfs-stateful
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nfs-statefulset
spec:
  serviceName: nfs-stateful-service
  replicas: 3
  selector:
    matchLabels:
      app: nfs-stateful
  template:
    metadata:
      labels:
        app: nfs-stateful
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
        - |
          POD_NAME=$(hostname)
          POD_INDEX=${POD_NAME##*-}
          
          # Each pod creates its own directory on shared NFS
          mkdir -p /nfs-shared/pod-$POD_INDEX
          echo "Pod $POD_NAME initialized at $(date)" > /nfs-shared/pod-$POD_INDEX/info.txt
          
          # Also maintain shared data
          echo "Pod $POD_NAME joined at $(date)" >> /nfs-shared/cluster-members.log
          
          # Simulate work with both local and shared storage
          while true; do
            # Write to pod-specific local storage
            echo "$(date): Local data for $POD_NAME" >> /local-data/local.log
            
            # Write to shared NFS storage
            echo "$(date): Shared data from $POD_NAME" >> /nfs-shared/shared-activity.log
            
            # Read cluster state from NFS
            echo "Current cluster members:"
            ls -d /nfs-shared/pod-* 2>/dev/null | xargs -n1 basename
            
            sleep 30
          done
        volumeMounts:
        - name: local-storage
          mountPath: /local-data
        - name: nfs-shared
          mountPath: /nfs-shared
      volumes:
      - name: nfs-shared
        persistentVolumeClaim:
          claimName: nfs-pvc
  volumeClaimTemplates:
  - metadata:
      name: local-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Deploy and test StatefulSet with NFS:

```bash
# Deploy StatefulSet with NFS shared storage
kubectl apply -f nfs-statefulset.yaml

# Watch pods being created
kubectl get pods -l app=nfs-stateful -w

# Each pod has both local and shared storage
kubectl exec nfs-statefulset-0 -- ls -la /local-data/
kubectl exec nfs-statefulset-0 -- ls -la /nfs-shared/

# Verify all pods can see shared data
for i in 0 1 2; do
  echo "=== Pod nfs-statefulset-$i ==="
  kubectl exec nfs-statefulset-$i -- cat /nfs-shared/cluster-members.log
done

# Scale down and verify shared data persists
kubectl scale statefulset nfs-statefulset --replicas=1
kubectl exec nfs-statefulset-0 -- ls -la /nfs-shared/

# Scale back up
kubectl scale statefulset nfs-statefulset --replicas=3
kubectl exec nfs-statefulset-2 -- cat /nfs-shared/cluster-members.log
```

## The PersistentVolume and PersistentVolumeClaim System

### Understanding the Separation of Concerns

The PV/PVC system represents one of Kubernetes' most elegant design patterns. By separating storage provisioning (PV) from storage consumption (PVC), Kubernetes enables different teams to work independently while maintaining clear interfaces between infrastructure and applications.

This separation provides several key benefits:

1. Abstraction: Developers don't need to know storage implementation details
2. Portability: Applications can run in different environments without modification
3. Flexibility: Storage can be replaced or upgraded without application changes
4. Security: Sensitive storage credentials remain with administrators
5. Governance: Administrators control storage allocation and quotas

### The Lifecycle of Persistent Storage

Understanding how PVs and PVCs interact throughout their lifecycle is crucial for managing persistent storage effectively.

#### Provisioning Phase

Storage provisioning can occur through two methods:

Static Provisioning: Administrators create PersistentVolumes manually before applications need them. This approach works well when:

- Specific storage requirements exist (particular disks or storage arrays)
- Storage needs are well-understood and predictable
- Tight control over storage allocation is required

Dynamic Provisioning: PersistentVolumes are created automatically when PersistentVolumeClaims request them. This requires:

- A StorageClass that defines how to provision storage
- A storage provisioner that can create volumes on demand
- Proper permissions for the provisioner to create resources

#### Binding Phase

The binding process matches PVCs to PVs based on several criteria:

1. Capacity Matching: The PV must have at least the requested capacity. Kubernetes will select the smallest PV that satisfies the request to minimize waste.
    
2. Access Mode Compatibility: The PV must support the requested access modes. If a PVC requests ReadWriteMany, the PV must support it.
    
3. StorageClass Matching: Both PV and PVC must reference the same StorageClass (or both have none).
    
4. Selector Matching: If the PVC specifies a selector, only PVs matching the selector labels are considered.
    
5. Volume Mode Matching: Both must agree on whether to use filesystem or raw block mode.
    

The binding is exclusive and bidirectional. Once bound, a PV belongs to a specific PVC, and that PVC is satisfied by that specific PV. This one-to-one relationship ensures clear ownership and prevents conflicts.

#### Usage Phase

During the usage phase, Pods mount PVCs as volumes. The mounting process involves:

1. The Pod specification references a PVC
2. The kubelet on the node verifies the PVC is bound
3. The storage driver mounts the underlying storage to the node
4. The volume is made available to containers at specified mount paths

Important characteristics during usage:

- Multiple Pods can use the same PVC (if access modes permit)
- The PVC remains bound even if no Pods are using it
- Storage metrics and monitoring can track usage
- Volumes can be resized (if the StorageClass allows)

#### Reclamation Phase

When a PVC is deleted, the reclamation policy determines what happens to the PV:

Retain Policy: The PV is kept with data intact. Administrators must manually clean up:

- The PV enters "Released" state
- Data remains on the storage
- Manual intervention required to delete or recycle
- Useful for critical data requiring manual review

Delete Policy: The PV and underlying storage are automatically deleted:

- Storage resources are immediately freed
- Data is permanently lost
- Simplifies cleanup but requires confidence in backups
- Default for dynamically provisioned volumes

Recycle Policy (Deprecated): Previously performed basic cleanup and made PV available again. Now deprecated in favor of dynamic provisioning.

### Creating and Managing PersistentVolumes

Let's create a comprehensive example showing different PV configurations:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-example
  labels:
    type: local
    environment: development
    tier: standard
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual-storage
  hostPath:
    path: /mnt/k8s-pv-data/local-pv-example
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node1
          - node2
```

Key components explained:

capacity.storage: Defines the size of the storage. This is the amount of storage that will be available to PVCs that bind to this PV.

volumeMode: Specifies whether to provide a formatted filesystem (default) or raw block device. Most applications use Filesystem mode.

accessModes: Defines how the volume can be mounted:

- `ReadWriteOnce (RWO)`: Volume can be mounted read-write by a single node
- `ReadOnlyMany (ROX)`: Volume can be mounted read-only by multiple nodes
- `ReadWriteMany (RWX)`: Volume can be mounted read-write by multiple nodes
- `ReadWriteOncePod (RWOP)`: Volume can be mounted by a single Pod

persistentVolumeReclaimPolicy: Determines data handling after PVC deletion.

storageClassName: Links this PV to a specific storage class. PVCs must request the same class to bind.

nodeAffinity: (For local volumes) Restricts which nodes can access this storage, essential for local storage that physically exists on specific nodes.

### Creating and Managing PersistentVolumeClaims

PVCs express storage needs from the application perspective:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc-example
  labels:
    app: my-application
    component: database
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: manual-storage
  selector:
    matchLabels:
      environment: development
      tier: standard
```

PVC components explained:

accessModes: Must be compatible with the PV's access modes. The PVC can request a subset of what the PV offers.

resources.requests.storage: Minimum storage needed. Kubernetes will bind to a PV with at least this capacity.

storageClassName: Must match the PV's storageClassName for binding to occur.

selector: Optional field to bind only to PVs with specific labels, providing fine-grained control over storage selection.

### Complete PV/PVC Example with Application

Here's a complete example demonstrating the full storage stack:

```yaml
# PersistentVolume definition
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
  labels:
    type: local
    app: database
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-manual
  hostPath:
    path: /mnt/data/postgres
    type: DirectoryOrCreate
---
# PersistentVolumeClaim definition
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  labels:
    app: postgres
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-manual
  selector:
    matchLabels:
      app: database
---
# PostgreSQL Pod using the PVC
apiVersion: v1
kind: Pod
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  containers:
  - name: postgres
    image: postgres:13-alpine
    env:
    - name: POSTGRES_DB
      value: testdb
    - name: POSTGRES_USER
      value: testuser
    - name: POSTGRES_PASSWORD
      value: testpass123
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    ports:
    - containerPort: 5432
      name: postgres
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```

Deploy and test the complete stack:

```bash
# Create all resources
kubectl apply -f postgres-complete.yaml

# Check PV status
kubectl get pv data-pv -o wide

# Check PVC status and binding
kubectl get pvc postgres-pvc -o wide

# Wait for Pod to be ready
kubectl wait --for=condition=Ready pod/postgres --timeout=60s

# Connect to PostgreSQL and create data
kubectl exec -it postgres -- psql -U testuser -d testdb -c "CREATE TABLE test_data (id SERIAL PRIMARY KEY, message TEXT);"
kubectl exec -it postgres -- psql -U testuser -d testdb -c "INSERT INTO test_data (message) VALUES ('Persistent data test');"
kubectl exec -it postgres -- psql -U testuser -d testdb -c "SELECT * FROM test_data;"

# Delete the Pod (simulating failure)
kubectl delete pod postgres

# Recreate the Pod
kubectl apply -f postgres-complete.yaml

# Wait for Pod to be ready again
kubectl wait --for=condition=Ready pod/postgres --timeout=60s

# Verify data persisted
kubectl exec -it postgres -- psql -U testuser -d testdb -c "SELECT * FROM test_data;"

# Clean up
kubectl delete pod postgres
kubectl delete pvc postgres-pvc
kubectl delete pv data-pv
```

## StorageClasses and Dynamic Provisioning

### Understanding StorageClasses

StorageClasses revolutionize storage management by enabling on-demand storage provisioning. Instead of administrators pre-creating PersistentVolumes, StorageClasses define templates for creating storage dynamically when applications request it.

Each StorageClass represents a different "quality of service" level or type of storage. For example:

- `fast-ssd`: High-performance SSD storage for databases
- `standard`: Balanced performance and cost for general use
- `archive`: Low-cost, slower storage for backups

### StorageClass Components

A StorageClass definition includes several important components:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "10"
  encrypted: "true"
  fsType: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
mountOptions:
- debug
- noatime
```

provisioner: Identifies which volume plugin to use for creating PVs. Different provisioners support different storage backends:

- `kubernetes.io/aws-ebs`: Amazon EBS volumes
- `kubernetes.io/gce-pd`: Google Compute Engine persistent disks
- `kubernetes.io/azure-disk`: Azure managed disks
- `kubernetes.io/cinder`: OpenStack Cinder
- Custom CSI drivers for specialized storage systems

parameters: Provisioner-specific settings that control storage characteristics. These vary by provisioner but commonly include:

- Disk type (SSD vs HDD)
- IOPS settings
- Encryption options
- Replication factors
- Filesystem types

reclaimPolicy: Default reclaim policy for PVs created by this StorageClass. Can be overridden in individual PVCs.

allowVolumeExpansion: Whether PVCs can be resized after creation. Essential for production systems that may need to grow.

volumeBindingMode: Controls when volume binding and provisioning occurs:

- `Immediate`: PV is created and bound as soon as PVC is created
- `WaitForFirstConsumer`: PV creation is delayed until a Pod uses the PVC, ensuring the PV is created in the correct availability zone

mountOptions: Additional mount options passed to the filesystem mount command. Use carefully as invalid options can prevent mounting.

### Local Storage with StorageClass

For development or specific use cases, you can create a StorageClass for local storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

The `kubernetes.io/no-provisioner` indicates that PVs must be manually created but can still be associated with this StorageClass for organization.

### Dynamic Provisioning Example

Here's a complete example demonstrating dynamic provisioning:

```yaml
# StorageClass for dynamic provisioning (works with most cloud providers)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: dynamic-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: kubernetes.io/host-path  # For testing; use cloud-specific in production
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
# PVC requesting dynamic storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  labels:
    app: dynamic-demo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: dynamic-storage
---
# Application using dynamically provisioned storage
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-app
  labels:
    app: dynamic-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Application using dynamically provisioned storage"
      echo "Initial data written at $(date)" > /data/info.txt
      while true; do
        echo "$(date): Application running" >> /data/app.log
        echo "Storage usage:"
        df -h /data
        echo "Files in /data:"
        ls -la /data/
        sleep 60
      done
    volumeMounts:
    - name: app-storage
      mountPath: /data
  volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

## Advanced Storage Patterns

### StatefulSets with VolumeClaimTemplates

StatefulSets provide a powerful pattern for stateful applications that require stable storage. The volumeClaimTemplates feature automatically creates a PVC for each Pod replica:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: postgres-service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13-alpine
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: myuser
        - name: POSTGRES_PASSWORD
          value: mypass123
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "Pod $POD_NAME started at $(date)" > /var/lib/postgresql/data/pod-identity.txt
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Deploy and explore StatefulSet storage behavior:

```bash
# Deploy the StatefulSet
kubectl apply -f postgres-statefulset.yaml

# Watch Pods being created in order
kubectl get pods -l app=postgres -w

# Observe PVC creation for each Pod
kubectl get pvc -l app=postgres

# Each Pod gets its own persistent storage
kubectl exec postgres-statefulset-0 -- cat /var/lib/postgresql/data/pod-identity.txt
kubectl exec postgres-statefulset-1 -- cat /var/lib/postgresql/data/pod-identity.txt
kubectl exec postgres-statefulset-2 -- cat /var/lib/postgresql/data/pod-identity.txt

# Scale down and up to verify persistence
kubectl scale statefulset postgres-statefulset --replicas=1
kubectl get pods -l app=postgres
kubectl scale statefulset postgres-statefulset --replicas=3

# Data persists through scaling
kubectl exec postgres-statefulset-1 -- cat /var/lib/postgresql/data/pod-identity.txt

# Clean up
kubectl delete statefulset postgres-statefulset
kubectl delete service postgres-service
kubectl delete pvc -l app=postgres
```

### Shared Storage Patterns

When multiple Pods need to access the same data, you need storage that supports ReadWriteMany or ReadOnlyMany access modes:

```yaml
# Shared storage example (requires RWX-capable storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteMany  # Note: Not all storage providers support this
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-storage-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared-app
  template:
    metadata:
      labels:
        app: shared-app
    spec:
      containers:
      - name: app
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Pod $(hostname) started at $(date)" >> /shared/pods.log
          while true; do
            echo "$(date): Pod $(hostname) is running" >> /shared/activity.log
            echo "Current pods accessing storage:"
            cat /shared/pods.log
            sleep 30
          done
        volumeMounts:
        - name: shared-storage
          mountPath: /shared
      volumes:
      - name: shared-storage
        persistentVolumeClaim:
          claimName: shared-pvc
```

### Backup and Restore Patterns

Implementing backup strategies for persistent data:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: busybox:1.36
        command: ["/bin/sh", "-c"]
        args:
        - |
          echo "Starting backup at $(date)"
          SOURCE_DIR="/source-data"
          BACKUP_DIR="/backup-data"
          TIMESTAMP=$(date +%Y%m%d_%H%M%S)
          BACKUP_FILE="backup_${TIMESTAMP}.tar.gz"
          
          # Create compressed backup
          tar -czf "${BACKUP_DIR}/${BACKUP_FILE}" -C "${SOURCE_DIR}" .
          
          # Verify backup
          if [ -f "${BACKUP_DIR}/${BACKUP_FILE}" ]; then
            echo "Backup successful: ${BACKUP_FILE}"
            echo "Backup size: $(du -h ${BACKUP_DIR}/${BACKUP_FILE} | cut -f1)"
            
            # Keep only last 5 backups
            cd ${BACKUP_DIR}
            ls -t backup_*.tar.gz | tail -n +6 | xargs -r rm
            
            echo "Current backups:"
            ls -lh ${BACKUP_DIR}/backup_*.tar.gz
          else
            echo "Backup failed!"
            exit 1
          fi
        volumeMounts:
        - name: source-volume
          mountPath: /source-data
          readOnly: true
        - name: backup-volume
          mountPath: /backup-data
      restartPolicy: OnFailure
      volumes:
      - name: source-volume
        persistentVolumeClaim:
          claimName: app-data-pvc
      - name: backup-volume
        persistentVolumeClaim:
          claimName: backup-pvc
```

## Container Storage Interface (CSI)

### Understanding CSI Architecture

The Container Storage Interface (CSI) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes. Before CSI, adding support for new storage systems to Kubernetes required modifying the core Kubernetes code. CSI enables storage vendors to develop drivers independently of the Kubernetes release cycle.

CSI drivers typically consist of:

1. Controller Service: Handles volume provisioning, deletion, and attachment
2. Node Service: Handles volume mounting and unmounting on nodes
3. Identity Service: Provides driver information and capabilities

### CSI Driver Deployment Pattern

CSI drivers are deployed as containers within the cluster:

```yaml
# Example CSI driver components (simplified)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-node-driver
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-node-driver
  template:
    metadata:
      labels:
        app: csi-node-driver
    spec:
      containers:
      - name: node-driver-registrar
        image: k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.5.0
        args:
        - --csi-address=/csi/csi.sock
        - --kubelet-registration-path=/var/lib/kubelet/plugins/csi-driver/csi.sock
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: registration-dir
          mountPath: /registration
      - name: csi-driver
        image: vendor/csi-driver:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: pods-mount-dir
          mountPath: /var/lib/kubelet/pods
          mountPropagation: Bidirectional
      volumes:
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins/csi-driver
          type: DirectoryOrCreate
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry
          type: Directory
      - name: pods-mount-dir
        hostPath:
          path: /var/lib/kubelet/pods
          type: Directory
```

### Using CSI Volumes

Once a CSI driver is installed, you can use it through StorageClasses:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-storage
provisioner: vendor.io/csi-driver
parameters:
  storageType: "performance"
  replication: "3"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```
