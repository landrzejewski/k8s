# Cluster Monitoring, Backup, and Management

## Setting up Kubernetes Monitoring

Kubernetes monitoring is provided through the Metrics Server, which exposes a standard API for resource usage information.

### Installing Metrics Server

```yaml
# metrics-server.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
rules:
- apiGroups:
  - metrics.k8s.io
  resources:
  - pods
  - nodes
  verbs:
  - get
  - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - deployments
  - replicasets
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 10250
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          seccompProfile:
            type: RuntimeDefault
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```

Deploy Metrics Server:

```bash
# Apply the Metrics Server configuration
kubectl apply -f metrics-server.yaml

# Or use the official manifest (may need modification for some clusters)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for deployment to be ready
kubectl wait --for=condition=available --timeout=300s deployment/metrics-server -n kube-system

# Verify deployment
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server
```

### Verifying Metrics Server Installation

```bash
# Check Metrics Server logs
kubectl logs -n kube-system deployment/metrics-server

# Look for successful messages and no errors

# Wait a few minutes, then test metrics collection
kubectl top nodes

# View Pod resource usage across all namespaces
kubectl top pods --all-namespaces

# View resource usage for specific namespace
kubectl top pods -n kube-system
```

### Using kubectl top Commands

```bash
# Show node resource usage
kubectl top nodes

# Show detailed node resource usage
kubectl top nodes --use-protocol-buffers

# Show Pod resource usage with containers
kubectl top pods --containers=true

# Sort Pods by CPU usage
kubectl top pods --sort-by=cpu --all-namespaces

# Sort Pods by memory usage
kubectl top pods --sort-by=memory --all-namespaces

# Show resource usage for specific node
kubectl top pods --all-namespaces --field-selector spec.nodeName=<node-name>

# Show resource usage without headers
kubectl top nodes --no-headers

# Show resource usage with custom columns
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq '.'
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/pods" | jq '.'
```

---

## Advanced Monitoring Setup

### Deploying kube-state-metrics

kube-state-metrics provides additional cluster state information:

```yaml
# kube-state-metrics.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - serviceaccounts
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources:
  - cronjobs
  - jobs
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources:
  - horizontalpodautoscalers
  verbs: ["list", "watch"]
- apiGroups: ["authentication.k8s.io"]
  resources:
  - tokenreviews
  verbs: ["create"]
- apiGroups: ["authorization.k8s.io"]
  resources:
  - subjectaccessreviews
  verbs: ["create"]
- apiGroups: ["policy"]
  resources:
  - poddisruptionbudgets
  verbs: ["list", "watch"]
- apiGroups: ["certificates.k8s.io"]
  resources:
  - certificatesigningrequests
  verbs: ["list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources:
  - storageclasses
  - volumeattachments
  verbs: ["list", "watch"]
- apiGroups: ["admissionregistration.k8s.io"]
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs: ["list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources:
  - networkpolicies
  - ingresses
  verbs: ["list", "watch"]
- apiGroups: ["coordination.k8s.io"]
  resources:
  - leases
  verbs: ["list", "watch"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources:
  - clusterrolebindings
  - clusterroles
  - rolebindings
  - roles
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app.kubernetes.io/name: kube-state-metrics
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app.kubernetes.io/name: kube-state-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1
        ports:
        - name: http-metrics
          containerPort: 8080
        - name: telemetry
          containerPort: 8081
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
          limits:
            cpu: 200m
            memory: 150Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 65534
          seccompProfile:
            type: RuntimeDefault
```

Deploy kube-state-metrics:

```bash
# Apply kube-state-metrics
kubectl apply -f kube-state-metrics.yaml

# Verify deployment
kubectl get deployment kube-state-metrics -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-state-metrics

# Check metrics are being exposed
kubectl get svc kube-state-metrics -n kube-system
```

---

## Understanding and Managing Etcd

Etcd is the core database that stores all Kubernetes cluster state and configuration.

### Etcd Backup Job

Create a CronJob for automated etcd backups:

```yaml
# etcd-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: registry.k8s.io/etcd:3.5.9-0
            command:
            - /bin/sh
            - -c
            - |
              ETCDCTL_API=3 etcdctl \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
                --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
                snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db
              
              # Keep only last 7 days of backups
              find /backup -name "etcd-snapshot-*.db" -mtime +7 -delete
              
              echo "Backup completed successfully"
              
              # Verify backup
              ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-$(date +%Y%m%d)*.db
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup-storage
              mountPath: /backup
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
              limits:
                cpu: 500m
                memory: 500Mi
          restartPolicy: OnFailure
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup-storage
            hostPath:
              path: /var/backups/etcd
              type: DirectoryOrCreate
```

Deploy the backup CronJob:

```bash
# Apply the etcd backup CronJob
kubectl apply -f etcd-backup-cronjob.yaml

# Verify CronJob is created
kubectl get cronjobs -n kube-system

# Check CronJob details
kubectl describe cronjob etcd-backup -n kube-system

# Manually trigger backup job for testing
kubectl create job --from=cronjob/etcd-backup etcd-backup-manual -n kube-system

# Check backup job status
kubectl get jobs -n kube-system
kubectl logs job/etcd-backup-manual -n kube-system
```

### Manual Etcd Backup

Create a one-time backup job:

```yaml
# etcd-manual-backup.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: etcd-manual-backup
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: etcd-backup
        image: registry.k8s.io/etcd:3.5.9-0
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting etcd backup..."
          BACKUP_FILE="/backup/etcd-manual-backup-$(date +%Y%m%d-%H%M%S).db"
          
          ETCDCTL_API=3 etcdctl \
            --endpoints=https://127.0.0.1:2379 \
            --cacert=/etc/kubernetes/pki/etcd/ca.crt \
            --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
            --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
            snapshot save $BACKUP_FILE
          
          echo "Backup created: $BACKUP_FILE"
          
          # Verify backup
          ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE
          
          echo "Backup verification complete"
        volumeMounts:
        - name: etcd-certs
          mountPath: /etc/kubernetes/pki/etcd
          readOnly: true
        - name: backup-storage
          mountPath: /backup
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      restartPolicy: Never
      hostNetwork: true
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      volumes:
      - name: etcd-certs
        hostPath:
          path: /etc/kubernetes/pki/etcd
      - name: backup-storage
        hostPath:
          path: /var/backups/etcd
          type: DirectoryOrCreate
```

Run manual backup:

```bash
# Create manual backup
kubectl apply -f etcd-manual-backup.yaml

# Monitor backup progress
kubectl get jobs -n kube-system -w

# Check backup logs
kubectl logs job/etcd-manual-backup -n kube-system

# Clean up job after completion
kubectl delete job etcd-manual-backup -n kube-system
```

---

## Cluster Backup Strategy

### Comprehensive Backup Plan

Create backup for all critical cluster components:

```yaml
# cluster-backup-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cluster-backup
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cluster-backup
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting comprehensive cluster backup..."
          BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
          BACKUP_DIR="/backup/cluster-backup-$BACKUP_DATE"
          mkdir -p $BACKUP_DIR
          
          # Backup all namespaces
          echo "Backing up all namespaces..."
          kubectl get namespaces -o yaml > $BACKUP_DIR/namespaces.yaml
          
          # Backup all resources in each namespace
          for ns in $(kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'); do
            echo "Backing up namespace: $ns"
            mkdir -p $BACKUP_DIR/namespaces/$ns
            
            # Backup common resources
            for resource in configmaps secrets services deployments statefulsets daemonsets replicasets pods persistentvolumeclaims; do
              kubectl get $resource -n $ns -o yaml > $BACKUP_DIR/namespaces/$ns/$resource.yaml 2>/dev/null || true
            done
          done
          
          # Backup cluster-wide resources
          echo "Backing up cluster-wide resources..."
          mkdir -p $BACKUP_DIR/cluster
          kubectl get nodes -o yaml > $BACKUP_DIR/cluster/nodes.yaml
          kubectl get persistentvolumes -o yaml > $BACKUP_DIR/cluster/persistent-volumes.yaml
          kubectl get storageclasses -o yaml > $BACKUP_DIR/cluster/storage-classes.yaml
          kubectl get clusterroles -o yaml > $BACKUP_DIR/cluster/cluster-roles.yaml
          kubectl get clusterrolebindings -o yaml > $BACKUP_DIR/cluster/cluster-role-bindings.yaml
          kubectl get customresourcedefinitions -o yaml > $BACKUP_DIR/cluster/crds.yaml
          
          # Create backup summary
          echo "Creating backup summary..."
          cat > $BACKUP_DIR/backup-info.txt << EOF
          Cluster Backup Information
          ==========================
          Backup Date: $(date)
          Kubernetes Version: $(kubectl version --short --client)
          Cluster Info: $(kubectl cluster-info)
          Node Count: $(kubectl get nodes --no-headers | wc -l)
          Namespace Count: $(kubectl get namespaces --no-headers | wc -l)
          Pod Count: $(kubectl get pods --all-namespaces --no-headers | wc -l)
          EOF
          
          echo "Cluster backup completed: $BACKUP_DIR"
        volumeMounts:
        - name: backup-storage
          mountPath: /backup
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      restartPolicy: Never
      serviceAccountName: cluster-backup-sa
      volumes:
      - name: backup-storage
        hostPath:
          path: /var/backups/cluster
          type: DirectoryOrCreate
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-backup-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-backup-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-backup-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-backup-role
subjects:
- kind: ServiceAccount
  name: cluster-backup-sa
  namespace: kube-system
```

Deploy and run cluster backup:

```bash
# Deploy cluster backup resources
kubectl apply -f cluster-backup-job.yaml

# Monitor backup job
kubectl get jobs -n kube-system cluster-backup -w

# Check backup logs
kubectl logs job/cluster-backup -n kube-system

# Verify backup was created (check on control plane node)
# ls -la /var/backups/cluster/
```

---

## Cluster Upgrade Management

### Pre-Upgrade Validation

Create pre-upgrade checks:

```yaml
# pre-upgrade-check.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-upgrade-check
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: pre-upgrade-check
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "=== Pre-Upgrade Cluster Check ==="
          echo "Date: $(date)"
          echo
          
          echo "=== Current Cluster Version ==="
          kubectl version --short
          echo
          
          echo "=== Node Status ==="
          kubectl get nodes -o wide
          echo
          
          echo "=== System Pod Status ==="
          kubectl get pods -n kube-system
          echo
          
          echo "=== Cluster Component Health ==="
          kubectl get componentstatuses 2>/dev/null || echo "ComponentStatuses not available"
          echo
          
          echo "=== Resource Usage ==="
          kubectl top nodes 2>/dev/null || echo "Metrics server not available"
          echo
          
          echo "=== Persistent Volumes ==="
          kubectl get pv
          echo
          
          echo "=== Failed Pods ==="
          kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded
          echo
          
          echo "=== Recent Events ==="
          kubectl get events --sort-by='.lastTimestamp' | tail -10
          echo
          
          echo "=== Pre-Upgrade Check Complete ==="
          echo "Review the above output before proceeding with upgrade"
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
      restartPolicy: Never
      serviceAccountName: cluster-backup-sa
```

Run pre-upgrade checks:

```bash
# Run pre-upgrade validation
kubectl apply -f pre-upgrade-check.yaml

# Wait for completion and check results
kubectl wait --for=condition=complete --timeout=300s job/pre-upgrade-check -n kube-system
kubectl logs job/pre-upgrade-check -n kube-system

# Clean up
kubectl delete job pre-upgrade-check -n kube-system
```

### Upgrade Verification

Post-upgrade validation job:

```yaml
# post-upgrade-verification.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: post-upgrade-verification
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: verification
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "=== Post-Upgrade Verification ==="
          echo "Date: $(date)"
          echo
          
          echo "=== Cluster Version After Upgrade ==="
          kubectl version --short
          echo
          
          echo "=== Node Status ==="
          kubectl get nodes -o wide
          echo
          
          echo "=== System Pods Health ==="
          kubectl get pods -n kube-system
          echo
          
          echo "=== DNS Test ==="
          kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default 2>/dev/null || echo "DNS test failed"
          echo
          
          echo "=== Metrics Collection ==="
          kubectl top nodes 2>/dev/null || echo "Metrics not available yet"
          echo
          
          echo "=== API Responsiveness Test ==="
          time kubectl get namespaces > /dev/null
          echo
          
          echo "=== Workload Test ==="
          kubectl create deployment upgrade-test --image=nginx:1.21
          kubectl wait --for=condition=available --timeout=120s deployment/upgrade-test
          kubectl get pods -l app=upgrade-test -o wide
          kubectl delete deployment upgrade-test
          echo
          
          echo "=== Post-Upgrade Verification Complete ==="
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      restartPolicy: Never
      serviceAccountName: cluster-backup-sa
```

### Rollback Preparation

Create rollback information:

```yaml
# rollback-info-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: rollback-info-collector
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: rollback-info
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "=== Rollback Information Collection ==="
          echo "Date: $(date)"
          echo
          
          echo "=== Current Configuration Backup ==="
          mkdir -p /backup/rollback-info
          
          # Save critical configurations
          kubectl get nodes -o yaml > /backup/rollback-info/nodes-before-upgrade.yaml
          kubectl get pods -n kube-system -o yaml > /backup/rollback-info/system-pods-before-upgrade.yaml
          kubectl version --output=yaml > /backup/rollback-info/version-info.yaml
          
          echo "Rollback information saved to /backup/rollback-info/"
          echo
          
          echo "=== Manual Rollback Steps ==="
          echo "1. If upgrade fails, drain and reset each node:"
          echo "   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data"
          echo "   # On node: kubeadm reset"
          echo "   # Restore from etcd backup"
          echo "   # Rejoin nodes to cluster"
          echo
          echo "2. Restore etcd from backup if needed"
          echo "3. Verify cluster health after rollback"
          echo
          echo "=== Rollback Info Collection Complete ==="
        volumeMounts:
        - name: backup-storage
          mountPath: /backup
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
      restartPolicy: Never
      serviceAccountName: cluster-backup-sa
      volumes:
      - name: backup-storage
        hostPath:
          path: /var/backups/cluster
          type: DirectoryOrCreate
```

---

## High Availability Monitoring

### HA Health Check DaemonSet

Monitor HA cluster health across all nodes:

```yaml
# ha-health-monitor.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ha-health-monitor
  namespace: kube-system
  labels:
    app: ha-health-monitor
spec:
  selector:
    matchLabels:
      app: ha-health-monitor
  template:
    metadata:
      labels:
        app: ha-health-monitor
    spec:
      hostNetwork: true
      containers:
      - name: health-monitor
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            echo "=== HA Health Check - $(hostname) ==="
            echo "Date: $(date)"
            
            # Check API server connectivity
            if kubectl get nodes > /dev/null 2>&1; then
              echo "✓ API server accessible"
            else
              echo "✗ API server not accessible"
            fi
            
            # Check etcd connectivity (on control plane nodes)
            if [ -f /etc/kubernetes/manifests/etcd.yaml ]; then
              if ETCDCTL_API=3 etcdctl \
                --endpoints=https://127.0.0.1:2379 \
                --cacert=/etc/kubernetes/pki/etcd/ca.crt \
                --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
                --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
                endpoint health > /dev/null 2>&1; then
                echo "✓ etcd healthy"
              else
                echo "✗ etcd unhealthy"
              fi
            fi
            
            echo "---"
            sleep 60
          done
        volumeMounts:
        - name: etcd-certs
          mountPath: /etc/kubernetes/pki/etcd
          readOnly: true
        - name: manifests
          mountPath: /etc/kubernetes/manifests
          readOnly: true
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
          limits:
            cpu: 50m
            memory: 64Mi
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      volumes:
      - name: etcd-certs
        hostPath:
          path: /etc/kubernetes/pki/etcd
      - name: manifests
        hostPath:
          path: /etc/kubernetes/manifests
      serviceAccountName: cluster-backup-sa
```

Deploy HA monitoring:

```bash
# Deploy HA health monitor
kubectl apply -f ha-health-monitor.yaml

# Check DaemonSet status
kubectl get daemonset ha-health-monitor -n kube-system

# View logs from all nodes
kubectl logs -l app=ha-health-monitor -n kube-system --tail=20

# Follow logs in real-time
kubectl logs -l app=ha-health-monitor -n kube-system -f
```

---

## Cluster Monitoring Dashboard

### Create monitoring namespace and resources

```yaml
# monitoring-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
---
# monitoring-dashboard-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-dashboard
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-dashboard
  template:
    metadata:
      labels:
        app: cluster-dashboard
    spec:
      serviceAccountName: dashboard-sa
      containers:
      - name: dashboard
        image: bitnami/kubectl:latest
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            clear
            echo "=== Kubernetes Cluster Dashboard ==="
            echo "Updated: $(date)"
            echo
            
            echo "=== Cluster Overview ==="
            kubectl get nodes --no-headers | wc -l | xargs echo "Nodes:"
            kubectl get pods --all-namespaces --no-headers | wc -l | xargs echo "Total Pods:"
            kubectl get namespaces --no-headers | wc -l | xargs echo "Namespaces:"
            echo
            
            echo "=== Node Status ==="
            kubectl get nodes
            echo
            
            echo "=== Resource Usage ==="
            kubectl top nodes 2>/dev/null || echo "Metrics server not available"
            echo
            
            echo "=== System Pods ==="
            kubectl get pods -n kube-system --no-headers | grep -v Running | wc -l | xargs echo "Non-running system pods:"
            echo
            
            echo "=== Recent Events ==="
            kubectl get events --sort-by='.lastTimestamp' --all-namespaces | tail -5
            echo
            
            sleep 30
          done
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
      restartPolicy: Always
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dashboard-reader
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dashboard-reader
subjects:
- kind: ServiceAccount
  name: dashboard-sa
  namespace: monitoring
```

Deploy monitoring dashboard:

```bash
# Create monitoring resources
kubectl apply -f monitoring-namespace.yaml

# Check deployment
kubectl get all -n monitoring

# View dashboard output
kubectl logs deployment/cluster-dashboard -n monitoring -f
```

---

## Verification and Maintenance

### Complete Cluster Health Check

```bash
# Comprehensive cluster status
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory | head -20

# Check critical components
kubectl get pods -n kube-system
kubectl get componentstatuses 2>/dev/null

# Verify monitoring components
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-state-metrics

# Check backup jobs
kubectl get cronjobs -n kube-system
kubectl get jobs -n kube-system

# View recent events
kubectl get events --sort-by='.lastTimestamp' --all-namespaces | tail -20

# Storage and persistence
kubectl get pv
kubectl get pvc --all-namespaces

# Network and services
kubectl get svc --all-namespaces
kubectl get ingress --all-namespaces
```

### Regular Maintenance Tasks

```bash
# Weekly cluster health check
kubectl get nodes
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded

# Monthly resource review
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check for deprecated APIs (before upgrades)
kubectl api-resources --verbs=list --namespaced -o name

# Certificate expiration check
kubectl get pods -n kube-system -o yaml | grep -i "certificate"

# Clean up completed jobs
kubectl delete jobs --field-selector status.successful=1 -n kube-system

# Verify backups are current
kubectl get cronjobs -n kube-system
```
