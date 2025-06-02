## 1. Local Cluster with Docker and Kind

### 1.1 What Is Kind?

* **Kind** (short for “Kubernetes IN Docker”) is a tool for running one or more Kubernetes clusters inside Docker containers.
* Each Kind “node” is in fact a Docker container running `kubelet`, `kube-proxy`, and a CRI (e.g., containerd).
* Kind is extremely fast to spin up and tear down since you don’t need to provision VMs or physical machines—everything lives 
* inside Docker containers on your local host. This makes it great for CI/CD pipelines, local testing, and learning purposes.

### 1.2 Prerequisites

1. **Docker**

    * A working Docker engine (version ≥ 19.03 recommended).
    * Your user must be able to run Docker commands (e.g., membership in the `docker` group or root privileges).

2. **Kind**

    * Download the Kind binary from its GitHub releases and make it executable. For example:

      ```bash
      curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
      chmod +x ./kind
      sudo mv ./kind /usr/local/bin/kind
      ```
    * (Check [https://github.com/kubernetes-sigs/kind/releases](https://github.com/kubernetes-sigs/kind/releases) for the latest version number.)

3. **kubectl**

    * The Kubernetes CLI—needed to inspect and interact with your Kind cluster. For example:

      ```bash
      curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
      chmod +x kubectl
      sudo mv kubectl /usr/local/bin/
      ```
    * Verify with `kubectl version --client` to ensure it’s installed correctly.

4. **Operating System**

    * Any modern Linux distribution (Ubuntu, Debian, CentOS, Fedora, etc.) with kernel ≥ 4.19 (to support the necessary cgroup features).
    * In practice, Kind also works inside WSL2 (Windows) or Docker Desktop on macOS because Docker is available there.

### 1.3 Creating a Simple Kind Cluster

Once Docker, Kind, and kubectl are installed:

1. **Create a default cluster named `kind-dev`**

   ```bash
   kind create cluster --name kind-dev
   ```

    * By default, Kind will download (or reuse if already present) the node image (e.g., `kindest/node:v1.27.3`).
    * It then creates one “control-plane” container (running the API server, etcd, controller-manager, scheduler, kubelet, kube-proxy) and one “worker” container.
    * Kind automatically sets up a Docker network so these containers can communicate.
    * It writes a new kubeconfig context (`kind-kind-dev`) into your `~/.kube/config`.

2. **Verify the cluster is up**

   ```bash
   kubectl cluster-info --context kind-kind-dev
   kubectl get nodes
   ```

   You should see output like:

   ```
   NAME                         STATUS   ROLES           AGE     VERSION
   kind-dev-control-plane       Ready    control-plane   1m      v1.27.3
   kind-dev-worker              Ready    <none>          1m      v1.27.3
   ```

   That confirms both the control-plane node and the worker node are running (inside Docker).

3. **Run a sample Deployment**
   For example, create a namespace and deploy an NGINX pod:

   ```bash
   kubectl create namespace demo --context kind-kind-dev
   kubectl apply -n demo -f https://k8s.io/examples/application/deployment.yaml --context kind-kind-dev
   kubectl get pods -n demo --context kind-kind-dev
   ```

   You should see the NGINX pods coming up. Because this is all inside Docker, networking is internal to the Docker bridge.

4. **Customize the Kind cluster**
   You can supply a YAML configuration file to control how many nodes you want, port mappings, taints, etc. For example, save this as `kind-cluster.yaml`:

   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   nodes:
     - role: control-plane
       extraPortMappings:
         - containerPort: 30080   # Map host port 80 → container port 30080
           hostPort: 80
           protocol: TCP
     - role: worker
     - role: worker
   networking:
     apiServerAddress: "127.0.0.1"
     apiServerPort: 6443
     disableDefaultCNI: false
   ```

   Then create the cluster with:

   ```bash
   kind create cluster --name dev-cluster --config kind-cluster.yaml
   ```

   This example makes a control-plane and two workers, and also maps your host’s port 80 to port 30080 on the control-plane container so you can directly hit Services exposed on port 80.

5. **Expose a Service externally**
   If you want your demo to be reachable from your host at port 80, you can, for instance, create a Service of type `NodePort` or `LoadBalancer` (though “LoadBalancer” in Kind doesn’t provision a real LB—it ends up being a NodePort). Because of the `extraPortMappings` above, traffic to your host’s port 80 gets directed into the control-plane container’s port 30080, and then `kube-proxy`/iptables will route it to a pod port.

6. **Advantages and Limitations of Kind**

    * **Advantages**:

        * Extremely fast to start and tear down clusters.
        * Very low resource overhead (only Docker containers).
        * Perfect for CI/CD pipelines: spin up a cluster, run tests, destroy it.
    * **Limitations**:

        * Not intended for production (because it’s not running full VMs).
        * Networking is limited to Docker’s virtual bridge, so you can’t easily simulate complicated network topologies.
        * You cannot easily attach multiple physical NICs or advanced firewall rules to these containerized “nodes.”

### 1.4 Additional Tips for Kind

* **Updating the Node Image**
  Kind uses images like `kindest/node:vX.Y.Z`. To use a newer Kubernetes version, specify `--image kindest/node:vX.Y.Z` or update the `kind-cluster.yaml` accordingly.

* **Multiple Clusters & Contexts**
  If you create several clusters (`kind-dev`, `kind-staging`, etc.), they all live in your `~/.kube/config` with different contexts. You can switch contexts by running:

  ```bash
  kubectl config use-context kind-dev
  ```

* **Mounting Host Volumes**
  You can mount a host directory into a Kind node container if you need local storage. In the config:

  ```yaml
  nodes:
    - role: worker
      extraMounts:
        - hostPath: /home/user/localdata
          containerPath: /data
  ```

  Now `/home/user/localdata` on your laptop appears as `/data` inside that worker container, which pods can mount as a volume.

---

## 2. On-Premise Cluster with kubeadm on Linux

If you want to set up a more production-like on-premise cluster—one control-plane node (or multiple for high availability) plus dedicated worker nodes—then **kubeadm** is the recommended tool. Below is a step-by-step guide:

1. **System Requirements & Networking**
2. **Kernel & Linux Configuration (swap, sysctl, firewall)**
3. **Install a CRI (containerd or Docker)**
4. **Install kubeadm, kubelet, kubectl**
5. **Initialize the Control Plane with `kubeadm init`**
6. **Join Worker Nodes**
7. **Install a CNI Plugin**
8. **(Optional) Set Up a LoadBalancer Replacement (e.g., MetalLB) for Services of Type LoadBalancer**
9. **Basic Security & Best Practices**

### 2.1 Hardware & OS Requirements

* **Machines**:

    * At least one machine designated as the control-plane (often called the “master”). In production you usually run 3 or 5 control-plane nodes to form an etcd quorum.
    * At least one (but usually several) worker nodes.
* **Operating System**:

    * Ubuntu 20.04/22.04, Debian 11/12, CentOS 7/8, Rocky Linux 8/9, AlmaLinux 8/9, or similar.
    * Each node should have ≥2 CPU cores, ≥2 GiB RAM (control-plane preferably ≥4 GiB), and ≥20 GiB disk.
* **Network**:

    * All nodes must be able to reach each other via their IP addresses (ideally static IPs; avoid DHCP if possible).
    * DNS or `/etc/hosts` entries so that each node can resolve the control-plane’s hostname (if you plan to reference by name).

### 2.2 Linux Kernel & System Configuration

1. **Disable Swap**
   Kubernetes requires that swap be turned off or kubelet will refuse to start. On each node:

   ```bash
   sudo swapoff -a
   ```

   To make it permanent, open `/etc/fstab` and comment out (or remove) any line that refers to a swap partition or swap file (anything starting with `/swap` or `UUID=… swap`).

2. **Load Necessary Kernel Modules & sysctl Settings**
   Create a file `/etc/modules-load.d/k8s.conf` containing:

   ```
   br_netfilter
   overlay
   ```

   Then create `/etc/sysctl.d/k8s.conf` containing:

   ```
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   net.ipv6.conf.all.forwarding        = 1
   vm.swappiness                        = 0
   ```

   Apply them immediately:

   ```bash
   sudo sysctl --system
   ```

3. **Firewall & SELinux (if using RHEL/CentOS/Fedora)**

    * On Ubuntu/Debian, if you are running `ufw`, either open the required ports (2379/2380 for etcd, 6443 for the API server, 10250 for kubelet, CNI ports, etc.) or disable it temporarily:

      ```bash
      sudo ufw disable
      ```
    * On CentOS/RHEL, you can disable firewalld while setting up:

      ```bash
      sudo systemctl disable firewalld --now
      ```
    * For SELinux on CentOS/Fedora/RHEL, set to permissive or disabled (at least during initial setup):

      ```bash
      sudo setenforce 0
      # Then edit /etc/selinux/config and set SELINUX=permissive
      ```

### 2.3 Install a Container Runtime (CRI)

Since Kubernetes v1.24 removed Docker’s built-in shim, you must install a CRI—either **containerd** or **CRI-O**. Below is an example using **containerd** on Ubuntu or Debian:

1. **Install prerequisites**

   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg lsb-release
   ```

2. **Add Docker’s official GPG key and repository** (because containerd is distributed alongside Docker packages):

   ```bash
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
     https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) stable" | \
     sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

   sudo apt-get update
   ```

3. **Install containerd**

   ```bash
   sudo apt-get install -y containerd.io
   ```

4. **Configure containerd**
   Create the directory (if it doesn’t exist) and generate a default config:

   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   ```

   In `/etc/containerd/config.toml`, find the section:

   ```toml
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
     SystemdCgroup = false
   ```

   Change it to:

   ```toml
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
     SystemdCgroup = true
   ```

   This ensures containerd uses systemd cgroups, which is recommended. Then restart containerd:

   ```bash
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```

5. **(Optional) If you prefer Docker instead**
   You could install Docker Engine:

   ```bash
   sudo apt-get install -y docker.io
   sudo systemctl enable docker --now
   sudo usermod -aG docker $USER   # Then log out/in or restart your shell
   ```

   And configure `/etc/docker/daemon.json` to use the systemd cgroup driver:

   ```json
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": { "max-size": "100m" },
     "storage-driver": "overlay2"
   }
   ```

   Then restart Docker:

   ```bash
   sudo systemctl restart docker
   ```

### 2.4 Install kubeadm, kubelet, and kubectl

1. **Add the Kubernetes apt repository** (Ubuntu/Debian example):

   ```bash
   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
       https://packages.cloud.google.com/apt/doc/apt-key.gpg

   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] \
     https://apt.kubernetes.io/ kubernetes-xenial main" | \
     sudo tee /etc/apt/sources.list.d/kubernetes.list

   sudo apt-get update
   ```
2. **Install specific versions** (you generally want kubelet, kubeadm, and kubectl to match, and kubelet should be no more than one minor version older/newer than your control plane). For example:

   ```bash
   sudo apt-get install -y kubelet=1.27.3-00 kubeadm=1.27.3-00 kubectl=1.27.3-00
   ```

   Replace `1.27.3-00` with the version you intend to run.
3. **Pin the versions to prevent accidental upgrades** (optional):

   ```bash
   sudo apt-mark hold kubelet kubeadm kubectl
   ```
4. **Verify installation**:

   ```bash
   kubeadm version
   kubelet --version
   kubectl version --client
   ```

### 2.5 Initialize the Control Plane (`kubeadm init`)

Ensure on your control-plane node that:

* Swap is turned off (`swapoff -a`).
* sysctl settings are applied (`sysctl net.bridge.bridge-nf-call-iptables` should be `1`, `net.ipv4.ip_forward` should be `1`).
* containerd (or Docker) is running with the correct cgroup driver.

#### 2.5.1 Prepare a `kubeadm` Configuration File (Optional but Recommended)

You can supply a YAML file to configure certain settings, such as the Pod network CIDR, service network, and control-plane endpoint (helpful for HA). For instance, save this as `kubeadm-config.yaml`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.27.3
controlPlaneEndpoint: "master.example.local:6443"
networking:
  podSubnet: "10.244.0.0/16"       # matches Flannel’s default
  serviceSubnet: "10.96.0.0/12"
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "iptables"
```

* **`kubernetesVersion`**: Matches the version you installed for kubeadm/kubelet/kubectl.
* **`controlPlaneEndpoint`**: If you plan multiple control-plane nodes behind a load balancer or VIP, set that address here. Otherwise, use the hostname or IP of this single control-plane node.
* **`podSubnet`**: This must match whatever your CNI expects—e.g., Flannel uses `10.244.0.0/16` by default.
* **`serviceSubnet`**: Defaults to `10.96.0.0/12` unless you have a reason to change it.
* **`KubeProxyConfiguration`**: If you want iptables mode (default), leave it as-is. IPVS is possible but requires extra kernel modules.

Initialize with:

```bash
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

* **`--upload-certs`**: If you intend to add additional control-plane nodes (HA), this uploads the certificates to a secret in `kube-system` so other control-plane nodes can retrieve them.

#### 2.5.2 Follow the Post-Init Instructions

Once initialization succeeds, you’ll see something like:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, run:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join master.example.local:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

1. **Copy the kubeconfig** so you can run `kubectl` as a regular user:

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

   Now `kubectl get nodes` (run as your user) will talk to the newly created cluster.

2. **Note the `kubeadm join` command** (it includes a token and the hash of the CA certificate). You’ll use that exact command on each worker node to join them to this control plane.

3. **Verify control-plane pods** are running:

   ```bash
   kubectl get pods -n kube-system
   ```

   You should see pods like:

    * `coredns-...` (two replicas)
    * `etcd-master` (if you have a standalone etcd on this node)
    * `kube-apiserver-master`, `kube-controller-manager-master`, `kube-scheduler-master`
    * `kube-proxy-...` (one per node, including the control-plane)
    * `storage-provisioner` (if using hostPath dynamic provisioning, for example)

### 2.6 Join Worker Nodes

On each worker node, after you’ve installed and configured containerd and kubeadm/kubelet/kubectl in the same way (swap off, sysctl applied, correct cgroup driver, etc.), run the join command you copied above. Example:

```bash
sudo kubeadm join master.example.local:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

* If your control plane has multiple endpoints behind a load balancer, use that VIP or hostname instead of a single IP.
* Once that command succeeds, on your control-plane node you should see the new worker appear in `kubectl get nodes`.

### 2.7 Install a CNI Plugin

Kubernetes does not provide pod networking out of the box—you must install a Container Network Interface plugin so pods across nodes can talk to each other. Some popular choices:

1. **Flannel** (simple UDP or VXLAN overlay)

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.23.0/Documentation/kube-flannel.yml
   ```

   Make sure the `podSubnet` in your `kubeadm-config.yaml` matches Flannel’s default `10.244.0.0/16`.

2. **Calico** (L3 network with optional BGP, plus NetworkPolicy support)

   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```

   Calico can also operate in a VXLAN mode or use BGP. It is popular for its rich network policy features.

3. **Weave Net** (mesh overlay)

   ```bash
   kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
   ```

   Weave works well out of the box with minimal configuration.

4. **Cilium** (eBPF-based, high performance)

   ```bash
   kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.14.0/install/kubernetes/quick-install.yaml
   ```

   Cilium requires a Linux kernel with eBPF support (≥4.19). It provides very efficient data plane and advanced security features.

After installing your chosen CNI, wait a few minutes and then verify that the CNI pods are all `Running` in `kube-system`:

```bash
kubectl get pods -n kube-system
```

Only once the CNI pods are ready is inter-pod networking functional. Then you can launch your application workloads.

### 2.8 (Optional) Use MetalLB for Services of Type LoadBalancer

On-premise, there is no cloud provider LB. If you want to create a Service of type `LoadBalancer` and have it receive a real IP on your local network, you can install MetalLB:

1. **Deploy MetalLB’s components**

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.11/config/manifests/metallb-native.yaml
   ```
2. **Configure an address pool** by creating a ConfigMap in the `metallb-system` namespace. For example, if your LAN has IPs 192.168.1.240–192.168.1.250 reserved for LoadBalancer Services:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     namespace: metallb-system
     name: config
   data:
     config: |
       address-pools:
       - name: default
         protocol: layer2
         addresses:
         - 192.168.1.240-192.168.1.250
   ```

   ```bash
   kubectl apply -f metallb-config.yaml
   ```
3. **Now whenever you create** a Service of type `LoadBalancer`, MetalLB will pick an unused IP from that pool, announce it via ARP (in layer2 mode), and route traffic to the appropriate node(s).

### 2.9 Basic Security and Best Practices

1. **RBAC (Role-Based Access Control)**
   By default, `kubeadm` sets up RBAC but gives the `admin.conf` user full `cluster-admin` privileges. For production or shared environments, create Roles/ClusterRoles and bind them to individual users or service accounts rather than handing out the admin kubeconfig to everyone.

2. **Pod Security Admission**

    * In newer Kubernetes versions, avoid using Pod Security Policies (PSPs—deprecated) and instead enable **Pod Security Admission**. Label each namespace with the desired enforcement level. For example:

      ```bash
      kubectl label namespace demo pod-security.kubernetes.io/enforce=baseline
      kubectl label namespace demo pod-security.kubernetes.io/enforce-version=v1.27.0
      ```
    * You can also set an “audit” or “warn” policy on other namespaces. This immediately blocks (or warns about) pods that request privileged or unsafe settings.

3. **PersistentVolumes & StorageClasses**

    * For on-premise storage, consider NFS, iSCSI, Ceph, or LVM-backed storage.
    * You can create `PersistentVolume` resources that reference NFS servers, or run an NFS-Client Provisioner in Kubernetes so that `PersistentVolumeClaims` dynamically bind to NFS shares.

4. **Upgrades**

    * Check available versions with

      ```bash
      sudo kubeadm upgrade plan
      ```
    * To upgrade the control-plane:

      ```bash
      sudo kubeadm upgrade apply v1.28.0
      ```
    * Then upgrade `kubelet` and `kubectl` on each node to match (e.g., via `apt-get install kubelet=1.28.0-00 kubectl=1.28.0-00`) and restart the `kubelet`:

      ```bash
      sudo systemctl restart kubelet
      ```
    * Always respect the version skew policy: kubelets may be one minor version older or newer than the control-plane, but you cannot skip more than one minor version.

5. **Monitoring & Metrics**

    * Install the **Metrics Server** so that Horizontal Pod Autoscaler works:

      ```bash
      kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
      ```
    * For production-grade monitoring, deploy Prometheus + Grafana. Include exporters like `node-exporter` and `kubestate-metrics`.

6. **etcd Backup & Restore**

    * On a single control-plane node (non-HA), take periodic etcd snapshots:

      ```bash
      sudo ETCDCTL_API=3 etcdctl snapshot save ~/etcd-snapshot-$(date +%F).db \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
        --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
      ```
    * Restoring from a snapshot involves stopping the control-plane components, running `etcdctl snapshot restore ...` into a new data directory, and adjusting configuration to point etcd to that new data directory before bringing the API server back up.

## Kubernetes Object Management

### 1.1 Introduction to Kubernetes Objects

Kubernetes is a declarative, API-driven system for orchestrating containerized applications. At its core are **objects**, which represent the desired state of various cluster components (for example, pods, services, deployments, config maps, and more). When you create or update an object, you’re expressing to Kubernetes “This is how I want my cluster to look.” Kubernetes then works to make the actual state of the cluster match your declared (desired) state.

Every Kubernetes object has:

* **`apiVersion`**: Specifies which version of the Kubernetes API you’re using (e.g., `v1`, `apps/v1`).
* **`kind`**: The type of object (e.g., `Pod`, `Deployment`, `Service`).
* **`metadata`**: A collection of fields that uniquely identify the object, such as `name`, `namespace`, `labels`, and `annotations`.
* **`spec`** (and sometimes **`status`**): The “specification” section describes the desired state, whereas the `status` section (populated by the system) reflects the current observed state.

> **Example Skeleton of a Kubernetes Object**
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   namespace: default
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>       - name: nginx
>         image: nginx:1.21
>         ports:
>         - containerPort: 80
> ```

### 1.2 Managing Objects Declaratively

Kubernetes encourages a **declarative** approach (rather than imperative). You write a YAML (or JSON) file that declares the object’s desired state, then ask Kubernetes to apply it. Kubernetes compares that desired state to the current state and makes any necessary changes.

#### 1.2.1 Creating Objects

To create a new object from a file:

```bash
kubectl apply -f <filename>.yaml
```

* If the object doesn’t exist, Kubernetes creates it.
* If it exists, Kubernetes attempts to **merge** changes into the live object (a “patch”).

Alternatively, you can use:

```bash
kubectl create -f <filename>.yaml
```

But note that `create` will error out if an object with the same name already exists (whereas `apply` will update).

#### 1.2.2 Viewing Objects

* **List all objects of a given kind**:

  ```bash
  kubectl get pods               # show all Pods in the current namespace
  kubectl get deployments        # show all Deployments in the current namespace
  ```
* **Show detailed information**:

  ```bash
  kubectl describe deployment nginx-deployment
  ```

  This prints events, status, current replica count vs desired, and other helpful debugging info.
* **Output in YAML or JSON**:

  ```bash
  kubectl get svc myservice -o yaml
  kubectl get pods mypod -o json
  ```

#### 1.2.3 Updating Objects

Since Kubernetes is declarative, updating means modifying the YAML and re-applying:

1. Edit your local YAML file (e.g., change `replicas: 3` to `replicas: 5`).
2. Run:

   ```bash
   kubectl apply -f deployment.yaml
   ```

Kubernetes will perform a rolling update (for workloads that support it). You can also use **`kubectl edit`** to directly edit an object’s live configuration:

```bash
kubectl edit deployment nginx-deployment
```

This opens your default editor with the live spec; saving your edits triggers an update.

#### 1.2.4 Deleting Objects

To delete an object:

```bash
kubectl delete -f deployment.yaml
```

or

```bash
kubectl delete deployment nginx-deployment
```

By default, Kubernetes uses a graceful deletion process (sending a `TERM` signal to containers, waiting for them to shut down). You can force immediate deletion with `--grace-period=0 --force`, but this is usually discouraged unless necessary.

### 1.3 Object Fields and Structure

Every Kubernetes object YAML is structured similarly, which makes learning one kind helpful for understanding others. Let’s break down the sections:

1. **`apiVersion`**

    * Determines which API group and version to use. E.g., `v1` for core objects, `apps/v1` for Deployments/DaemonSets/ReplicaSets, `batch/v1` for Jobs/CronJobs, etc.
2. **`kind`**

    * The object’s type, such as `Pod`, `Service`, `ConfigMap`, `Deployment`, and so on.
3. **`metadata`**

    * **`name`**: A unique identifier for the object within its namespace.
    * **`namespace`**: Logical partition of the cluster; if omitted, defaults to the `default` namespace.
    * **`labels`**: Key/value pairs for grouping and selection.
    * **`annotations`**: Key/value pairs for storing arbitrary non-identifying metadata.
    * **`labels`** and **`annotations`** live under `metadata:`.
4. **`spec`**

    * Defines the “desired state” of the object. For a `Pod`, it lists the containers; for a `Deployment`, it sets replica count and pod template; for a `Service`, it describes ports and selectors; and so on.
5. **`status`**

    * Automatically populated by the Kubernetes control plane reflecting the real-time state (e.g., how many replicas are actually running, pod IP addresses, conditions). Typically, you should not set it yourself in YAML; Kubernetes manages it.

> **Tip**: Use `kubectl explain <kind>.<field>` to see documentation for each field. For example:
>
> ```bash
> kubectl explain deployment.spec.template.spec.containers
> ```

### 1.4 Applying Labels, Annotations, and Namespaces (Preview)

Although these topics are treated in depth later, it’s worth noting here how they fit into object management:

* **Labels** (`metadata.labels`) are used for grouping, selectors, and querying (e.g., telling a Service which pods to route to).
* **Annotations** (`metadata.annotations`) hold non-identifying metadata (e.g., build/release IDs, URLs, arbitrary JSON).
* **Namespaces** partition objects into virtual clusters. Placing `metadata.namespace: my-namespace` in an object’s YAML causes it to belong to that namespace. If you omit the namespace, Kubernetes assumes `default` (or the value of your current `kubectl` context).

### 1.5 Practical Examples of Object Lifecycle

#### 1.5.1 Creating a Simple Pod

```yaml
# pod-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Apply the Pod**:

```bash
kubectl apply -f pod-demo.yaml
```

**Verify**:

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

**Delete**:

```bash
kubectl delete pod nginx-pod
```

#### 1.5.2 Creating a Deployment for Rolling Updates

```yaml
# deployment-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Apply**:

```bash
kubectl apply -f deployment-demo.yaml
```

**View rollout status**:

```bash
kubectl rollout status deployment/nginx-deployment
```

**Update image version**: Edit `image: nginx:1.21` → `image: nginx:1.22` in the YAML, then:

```bash
kubectl apply -f deployment-demo.yaml
```

Kubernetes will create new pods with the updated image and terminate the old pods in a rolling fashion.

**Rollback if needed**:

```bash
kubectl rollout undo deployment/nginx-deployment
```

#### 1.5.3 Using `kubectl edit` for Quick Tweaks

Suppose you want to temporarily scale your deployment to 5 replicas without touching your local YAML:

```bash
kubectl edit deployment nginx-deployment
```

Change

```yaml
spec:
  replicas: 2
```

to

```yaml
spec:
  replicas: 5
```

Save and exit. Kubernetes will spin up 3 additional pods.

### 1.6 Best Practices and Patterns for Object Management

1. **Keep manifests declarative and in source control**: Store all YAML files in Git (or another VCS) so you have version history.
2. **Use `kubectl apply` over `kubectl create`** for most workflows: It’s idempotent and merges changes.
3. **Label your resources consistently**: Even if your objects don’t need labels for selection, having common labels (e.g., `app=teamX-serviceY`) can simplify debugging, filtering, and dashboards.
4. **Namespace per environment or team**: For multi-tenant clusters, dedicate namespaces to each team or environment (e.g., `dev`, `staging`, `prod`).
5. **Review `kubectl diff` before apply**: You can run:

   ```bash
   kubectl diff -f deployment-demo.yaml
   ```

   This shows you exactly what will change (fields added, removed, or updated).
6. **Annotate objects with audit or build information**: Store Git commit SHA, CI pipeline run, or ticket IDs in annotations so you can trace back “who did what, and why.”

---

## Labels in Kubernetes

### 2.1 What Are Labels?

**Labels** are key/value pairs attached to Kubernetes objects (pods, services, deployments, etc.) under `metadata.labels`. They serve two main purposes:

1. **Organization and grouping**: You can logically group objects (for example, by application, tier, environment, version).
2. **Selection**: Controllers (Deployments, ReplicaSets, DaemonSets) and Services use **label selectors** to identify the set of pods they should manage or route traffic to.

> **Example of Labels**
>
> ```yaml
> kind: Pod
> apiVersion: v1
> metadata:
>   name: mysql
>   labels:
>     app: myapp
>     tier: database
>     environment: production
> spec:
>   containers:
>   - name: mysql
>     image: mysql:8.0
> ```

### 2.2 Label Keys and Values

* **Key syntax**: Must be at most 63 characters (alphanumeric, `-`, `_`, `.`), optionally prefixed by a DNS subdomain (e.g., `example.com/role`). If prefixed, the prefix and slash together must be ≤ 253 characters.
* **Value syntax**: Up to 63 characters (alphanumeric, `-`, `_`, `.`). Values can be empty strings.

> **Valid Label Keys / Values**
>
> * `app: frontend`
> * `tier: backend`
> * `env: staging`
> * `example.com/version: "v1.0"`

### 2.3 Label Selectors

A **label selector** is how you query or filter objects by labels. There are two styles:

1. **Equality-based selectors** (most common):

    * `key=value` or `key==value`
    * `key!=value`
    * Example: `app=nginx`, `environment!=production`
2. **Set-based selectors** (more advanced):

    * `key in (value1,value2,…)`
    * `key notin (value1,value2,…)`
    * `key` (exists)
    * `!key` (does not exist)

#### 2.3.1 Using Selectors with `kubectl`

* **List objects with a given label**:

  ```bash
  kubectl get pods -l app=nginx
  kubectl get services -l environment=production,tier=frontend
  kubectl get pods -l "release in (stable,canary)"
  ```
* **Watch changes to objects matching a selector**:

  ```bash
  kubectl get pods -l app=nginx --watch
  ```

#### 2.3.2 How Controllers Use Selectors

* A **Deployment** defines a `spec.selector.matchLabels` or `matchExpressions` block. Kubernetes uses that to know which pods belong to the Deployment’s ReplicaSet.
* A **Service** uses `spec.selector` to choose which pods to route traffic to.

> **Deployment Example with Selectors**
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: web-deployment
> spec:
>   replicas: 3
>   selector:
>     matchLabels:            # This selector must match labels in pod template
>       app: web
>       tier: frontend
>   template:
>     metadata:
>       labels:
>         app: web
>         tier: frontend
>     spec:
>       containers:
>       - name: nginx
>         image: nginx:1.21
> ```

> **Service Example with Selectors**
>
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: web-service
> spec:
>   selector:                # Service will route to pods matching these labels
>     app: web
>     tier: frontend
>   ports:
>   - protocol: TCP
>     port: 80
>     targetPort: 80
>   type: ClusterIP
> ```

### 2.4 Best Practices for Labeling

1. **Use meaningful label keys/values** that reflect application structure, environment, version, and role. For instance:

    * `app`: The name of your application (e.g., `app: payment-service`).
    * `component` or `tier`: E.g., `frontend`, `backend`, `database`.
    * `environment`: E.g., `dev`, `staging`, `prod`.
    * `version`: E.g., `v1.0`, `canary`, `stable`.
2. **Adopt a standard labeling scheme** across teams so tools, dashboards, and policies can assume consistent keys.
3. **Don’t over-label**: Only include labels that will be used for filtering, grouping, or selection. Having dozens of arbitrary labels can become confusing.
4. **Use multi-value labels sparingly**: If you need multiple values, consider using a set-based selector rather than duplicate labels.

### 2.5 Common Labeling Scenarios

#### 2.5.1 Blue-Green or Canary Deployments

You can label different versions differently and use a Service selector to switch traffic. For example:

* **Deployment for v1**:

  ```yaml
  metadata:
    name: web-v1
  spec:
    selector:
      matchLabels:
        app: web
        version: v1
    template:
      metadata:
        labels:
          app: web
          version: v1
  ```
* **Deployment for v2 (canary)**:

  ```yaml
  metadata:
    name: web-v2
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: web
        version: v2
    template:
      metadata:
        labels:
          app: web
          version: v2
  ```
* **Service routes only `v1` initially**:

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: web-service
  spec:
    selector:
      app: web
      version: v1
    ports:
    - port: 80
      targetPort: 80
  ```

To shift traffic to v2, you change the Service’s `version` selector to `v2` (or both, or use traffic-splitting via Ingress, Istio, etc.).

#### 2.5.2 Multi-Tenancy and Resource Quotas

If you label pods by team or owner (e.g., `team=payments`), you can:

* Setup **NetworkPolicies** to isolate traffic (`podSelector: { matchLabels: { team: payments } }`).
* Define **ResourceQuotas** per namespace but also reconcile them per label with cluster-level policies.

### 2.6 Adding and Modifying Labels Post-Creation

#### 2.6.1 Using `kubectl label`

* **Add or update a label** on a running object:

  ```bash
  kubectl label pod nginx-pod environment=staging
  ```

  If `environment` didn’t exist, it’s added. If it existed, it’s updated.
* **Remove a label**:

  ```bash
  kubectl label pod nginx-pod environment-
  ```

#### 2.6.2 Patching an Object

You can also use a JSON patch to add/update labels:

```bash
kubectl patch deployment web-deployment -p '{"metadata":{"labels":{"environment":"production"}}}'
```

---

## Namespaces in Kubernetes

### 3.1 What Is a Namespace?

A **namespace** is a logical partition within a Kubernetes cluster. It provides scope for:

* **Names**: Objects (Pods, Services, ConfigMaps, etc.) in different namespaces can share the same `name` without collision (e.g., `dev/nginx-pod` vs `prod/nginx-pod`).
* **Resource isolation**: Though not a full security boundary by default, namespaces let you segment cluster resources (e.g., CPU/memory quotas, network policies, RBAC rules) per team, project, or environment.

By default, every cluster has at least three namespaces:

1. **default**: Where user-created objects go when no namespace is specified.
2. **kube-system**: Reserved for system components (API server, scheduler, etc.).
3. **kube-public**: Read-only, accessible to all authenticated users (typically contains cluster info).

You can create additional namespaces for isolation (e.g., `dev`, `staging`, `prod`, or separate by team).

### 3.2 Creating and Deleting Namespaces

#### 3.2.1 Create a Namespace

```yaml
# namespace-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Apply it:

```bash
kubectl apply -f namespace-dev.yaml
kubectl get namespaces
# NAME          STATUS   AGE
# default       Active   10d
# dev           Active   5m
# kube-system   Active   10d
# kube-public   Active   10d
```

#### 3.2.2 Delete a Namespace

```bash
kubectl delete namespace dev
```

Deleting a namespace is asynchronous: Kubernetes will garbage-collect (delete) all objects in that namespace, then remove the namespace entry once everything is cleaned up.

### 3.3 Working with Namespaced Objects

When you create or query most objects, you specify (or default to) a namespace:

* **Creating an object in a specific namespace**:

  ```yaml
  # pod-in-dev.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    namespace: dev
    labels:
      app: myapp
  spec:
    containers:
    - name: myapp
      image: myapp:latest
  ```

  Then:

  ```bash
  kubectl apply -f pod-in-dev.yaml
  ```
* **Without specifying `metadata.namespace`**, objects go into your current `kubectl` context’s default namespace (often `default`).

#### 3.3.1 Switching Namespace Context

You can set your default namespace for a particular context to avoid specifying `-n` or `metadata.namespace` each time:

```bash
kubectl config set-context --current --namespace=dev
```

After this, any `kubectl apply` or `kubectl get` without `-n` will operate in the `dev` namespace.

#### 3.3.2 List Objects by Namespace

* **All namespaces**:

  ```bash
  kubectl get pods --all-namespaces
  ```
* **Specific namespace**:

  ```bash
  kubectl get deployments -n prod
  ```

### 3.4 Resource Quotas and Limits by Namespace

Organizations often enforce resource consumption boundaries per namespace. This prevents a single team or workload from exhausting the entire cluster.

#### 3.4.1 ResourceQuota Object

```yaml
# quota-dev.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
```

Apply it in the `dev` namespace. Kubernetes then tracks resource usage in that namespace, preventing new pods if quotas are exceeded.

#### 3.4.2 LimitRange Object

You can also set default resource requests/limits for pods or containers in a namespace.

```yaml
# limitrange-dev.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "200m"
      memory: "128Mi"
    type: Container
```

If a pod is created without specifying requests/limits, Kubernetes applies these defaults automatically.

### 3.5 Network Policies by Namespace

Namespaces become network isolation boundaries when combined with NetworkPolicies. For instance, you can allow only pods in the `frontend` namespace to talk to pods in `backend`, while denying other traffic.

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector: {}  # selects all pods in the 'backend' namespace
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
```

This policy says: “Within the `backend` namespace, allow ingress from any pod whose namespace has label `name=frontend`.”

### 3.6 RBAC and Namespaces

You can assign **RoleBindings** or **ClusterRoleBindings** scoped to specific namespaces:

* **RoleBinding** (namespace-scoped) example:

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: dev
    name: pod-reader
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: read-pods-binding
    namespace: dev
  subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: pod-reader
    apiGroup: rbac.authorization.k8s.io
  ```

  This gives user `alice` the ability to list, watch, and get pods in the `dev` namespace.

### 3.7 Best Practices for Namespaces

1. **One namespace per environment**: For example, `dev`, `staging`, `production`. It simplifies resource quotas, RBAC, and resource cleanup.
2. **One namespace per team (if multi-tenant)**: If multiple teams share a cluster, each team can have a dedicated namespace to isolate resources.
3. **Label namespaces consistently**: Annotate or label namespaces with owner, environment, or team metadata (e.g., `environment=prod`, `team=backend`).
4. **Avoid creating too many namespaces**: Each namespace adds overhead (e.g., in etcd). In very large clusters, having thousands of namespaces can degrade performance.
5. **Use `kubectl config set-context`**: Make day-to-day operations easier by switching contexts (and namespaces) rather than supplying `-n` every time.

---

## Annotations in Kubernetes

### 4.1 What Are Annotations?

**Annotations** are arbitrary key/value pairs attached to Kubernetes objects under `metadata.annotations`. Unlike **labels**, which are used for grouping, selection, and identification, annotations are meant for **non-identifying metadata**—often needed for:

* Tooling or automation to record build/release IDs, Git commit hashes, or timestamps.
* Documentation or descriptive text.
* Links to external resources (e.g., URLs to tickets, dashboards).
* Storing configuration data that is too large or verbose for labels.

> **Example of Annotations**
>
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: webapp
>   labels:
>     app: webapp
>   annotations:
>     build-version: "42"
>     git-commit: "a1b2c3d4"
>     changelog-url: "https://git.example.com/myrepo/commit/a1b2c3d4"
>     maintainer: "ops-team@example.com"
> spec:
>   replicas: 3
>   template:
>     metadata:
>       labels:
>         app: webapp
>     spec:
>       containers:
>       - name: webapp
>         image: example/webapp:latest
> ```

### 4.2 Differences Between Labels and Annotations

| Aspect                   | Labels                                         | Annotations                                                          |
| ------------------------ | ---------------------------------------------- | -------------------------------------------------------------------- |
| **Primary Purpose**      | Selecting and grouping objects                 | Storing arbitrary metadata                                           |
| **Selectors**            | Used in label selectors (Deployment, Service)  | Not used for selection; Kubernetes ignores annotations for selection |
| **Intended Size/Length** | Short (≤ 63 characters for keys/values)        | Can be larger (though still limited by overall metadata size)        |
| **Use Cases**            | `app=frontend`, `env=prod`, `version=v1.2`     | `build-timestamp`, `qa-notes`, `documentation-links`, `owner-info`   |
| **Visibility**           | Often shown in `kubectl get` output by default | Not shown by default but visible via `kubectl describe` or `-o yaml` |

### 4.3 Using Annotations in Practice

#### 4.3.1 Adding Annotations to an Object

You can define annotations in your object’s YAML under `metadata.annotations`:

```yaml
metadata:
  name: redis
  annotations:
    backup/schedule: "daily"
    backup/location: "s3://mybucket/redis-backups"
```

To add an annotation after creation, use `kubectl annotate`:

```bash
kubectl annotate deployment webapp build-version="2025-05-30-001"
```

If the key already exists, the command will fail unless you add `--overwrite`:

```bash
kubectl annotate deployment webapp build-version="2025-05-30-002" --overwrite
```

To remove an annotation:

```bash
kubectl annotate pod mypod backup/schedule-
```

#### 4.3.2 Retrieving Annotations

* **Describe object**:

  ```bash
  kubectl describe pod mypod
  ```

  Under the “Annotations” section, you’ll see all key/value pairs.
* **Get object in YAML/JSON**:

  ```bash
  kubectl get deployment webapp -o yaml
  ```

  Look under `metadata.annotations:`.

### 4.4 Common Use Cases for Annotations

#### 4.4.1 Storing CI/CD Metadata

* **Git Commit SHA**: Record which commit an image was built from.
* **Build Timestamp**: When the container image (or Helm chart) was packaged.
* **CI Pipeline ID or URL**: Link back to the CI job that produced the artifact.

**Example**:

```yaml
metadata:
  name: api-server
  annotations:
    ci-pipeline: "https://ci.example.com/job/api-server/1234"
    git-commit: "f7e71abc"
    image-build-date: "2025-05-30T14:22:00Z"
```

#### 4.4.2 External Configuration for Controllers/Operators

Some operators (for example, the Prometheus Operator or cert-manager) read from annotations to drive behavior:

* **cert-manager**:

  ```yaml
  metadata:
    name: example-cert
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
  spec:
    dnsNames:
    - example.com
  ```

  The annotation `cert-manager.io/cluster-issuer` tells cert-manager which Issuer to use to provision a TLS certificate.

* **Ingress Controllers** (e.g., NGINX Ingress Controller) can read annotations to modify behavior on a per-Ingress basis:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: web-ingress
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: "/"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
  spec:
    rules:
    - host: example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: web-service
              port:
                number: 80
  ```

#### 4.4.3 Developer Notes and Documentation

Annotations can hold free-form text, such as a short description, owner contact info, or links to runbooks:

```yaml
metadata:
  name: analytics-job
  annotations:
    description: "Runs nightly data aggregation for reporting"
    owner: "data-team@example.com"
    runbook-url: "https://wiki.example.com/display/DataTeam/Analytics+Job"
```

### 4.5 Design Considerations for Annotations

* **Avoid overly large annotation values**: Although annotations can handle larger strings than labels, extremely large values can increase the size of etcd objects and slow down API calls. If you need to store big blobs (e.g., SSL certificates), consider using a `Secret` instead.
* **Use a consistent naming convention**: Don’t litter random keys; adopt a prefix naming scheme when multiple tools/operators might add annotations:

    * For internal tooling: `internal.example.com/backup-schedule`
    * For cert-manager: `cert-manager.io/cluster-issuer` (as documented by cert-manager)
* **Remember that annotations are not indexed**: You cannot use them for fast label-selector lookups. If you need to filter by a key/value frequently, labels are a better choice.

---

## Common Labels in Kubernetes

### 5.1 Why Standardize on Common Labels?

Although arbitrary labels are powerful, having a **standard set of recommended labels** across your organization—or across the ecosystem—greatly simplifies:

* **Automation**: Scripts and tools can assume the presence of known label keys and perform actions accordingly.
* **Discovery and Dashboards**: Monitoring tools (Prometheus, Grafana, etc.) can aggregate metrics by standardized keys.
* **Governance**: Cluster admins can enforce policies (via OPA, Kyverno, or other admission controllers) that rely on certain labels being present.

Kubernetes provides a list of **common labels** under the [Common Labels documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/). These are conventions—not enforced by Kubernetes itself—but widely adopted by many projects.

### 5.2 List of Recommended Common Labels

Here are the most commonly referenced keys, with typical value examples and descriptions:

| **Key**                         | **Value Examples**       | **Description**                                                                                                               |
| ------------------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `app.kubernetes.io/name`        | `nginx`, `redis`         | The name of the application (e.g., “nginx”, “mysql”).                                                                         |
| `app.kubernetes.io/instance`    | `nginx-1`, `redismaster` | A unique identifier for an instance of an application (e.g., “nginx-1” for a specific deployment).                            |
| `app.kubernetes.io/version`     | `v1.0.0`, `2.3.4`        | The version of the application (often the image tag or release tag).                                                          |
| `app.kubernetes.io/component`   | `frontend`, `cache`      | The component within the architecture (e.g., “frontend”, “backend”, “database”, “cache”).                                     |
| `app.kubernetes.io/part-of`     | `ecommerce`, `analytics` | The name of a higher-level application this component is part of (e.g., a microservice “users” might be part of “ecommerce”). |
| `app.kubernetes.io/managed-by`  | `Helm`, `kustomize`      | The tool or operator managing this object (e.g., “Helm”, “Kustomize”, “operator-name”).                                       |
| `app.kubernetes.io/created-by`  | `GitOps`, `CircleCI`     | The party or tool that created or initiated the deployment (e.g., “GitOps”, “Jenkins”, “CircleCI”).                           |
| `app.kubernetes.io/operated-by` | `team-ops`, `dev-team`   | The team or entity responsible for day-to-day operations of the application.                                                  |
| `helm.sh/chart`                 | `nginx-1.2.3`            | The name and version of the Helm chart used (if deployed via Helm).                                                           |
| `heritage`                      | `Helm`, `Kustomize`      | The tool that originally created this resource (often automatically added by Helm).                                           |

> **Note**: All of these keys (except `heritage`) are under the prefix `app.kubernetes.io/`. This “namespace” helps avoid clashes with other labels.

### 5.3 Applying Common Labels in Practice

#### 5.3.1 Example Deployment with Common Labels

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app.kubernetes.io/name: redis
    app.kubernetes.io/instance: redis-master-1
    app.kubernetes.io/version: "6.2.5"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: "ecommerce-app"
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/operated-by: "db-team"
    helm.sh/chart: "redis-4.5.0"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: redis
      app.kubernetes.io/instance: redis-master-1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: redis
        app.kubernetes.io/instance: redis-master-1
        app.kubernetes.io/version: "6.2.5"
        app.kubernetes.io/component: database
        app.kubernetes.io/part-of: "ecommerce-app"
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/operated-by: "db-team"
        helm.sh/chart: "redis-4.5.0"
    spec:
      containers:
      - name: redis
        image: redis:6.2.5
        ports:
        - containerPort: 6379
```

All these labels serve distinct purposes:

* `app.kubernetes.io/name` and `app.kubernetes.io/instance` identify the application and instance.
* `app.kubernetes.io/version` tracks the specific version of Redis.
* `app.kubernetes.io/part-of` indicates it’s part of a larger “ecommerce-app.”
* `app.kubernetes.io/managed-by` and `helm.sh/chart` show that Helm manages it and which chart version was used.

#### 5.3.2 Service Example Matching on Common Labels

When you create a Service, you can match on some of these standard labels:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app.kubernetes.io/name: redis
    app.kubernetes.io/instance: redis-master-1
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
```

This ensures that the Service always routes to pods labeled exactly as that instance, even if new pods are created by a Helm upgrade.

### 5.4 Benefits of Adopting Common Labels

1. **Interoperability with Helm Charts**: Helm automatically injects certain labels (e.g., `heritage: Helm`, `helm.sh/chart`) so that tools can identify which resources belong to which release.
2. **Unified Dashboards and Monitoring**: Prometheus exporters can scrape metrics and add relabeling rules based on `app.kubernetes.io/part-of` or `component`, allowing you to group metrics by higher-level applications.
3. **Policy Enforcement**: Admission controllers or policy engines (OPA Gatekeeper, Kyverno) can enforce that every deployment must have `app.kubernetes.io/operated-by` and `app.kubernetes.io/part-of` labels.
4. **Ease of Searching and Filtering**: Running `kubectl get pods -l app.kubernetes.io/part-of=ecommerce-app` gives all pods that belong to that application, regardless of microservice names or versions.

### 5.5 Other Useful Common Labels (Beyond the Official List)

While the Kubernetes docs list the “apps” prefix keys, many organizations introduce supplemental labels to convey internal metadata. Examples include:

* **`team` or `owner`**: Identifies which team or individual owns the resource.
* **`environment`**: `development`, `test`, `staging`, `production`.
* **`git-repo`**: The Git repository URL.
* **`release`**: A release name (especially in GitOps-driven clusters): e.g., `v2.3.1`.

> **Example**:
>
> ```yaml
> metadata:
>   name: frontend
>   labels:
>     app.kubernetes.io/name: frontend
>     app.kubernetes.io/part-of: ecommerce-app
>     app.kubernetes.io/version: "1.2.0"
>     app.kubernetes.io/managed-by: "GitOps"
>     team: "website-team"
>     environment: "production"
> ```

### 5.6 Migrating to Common Labels

If you have existing clusters with ad-hoc labels, consider:

1. **Inventory current labels**:

   ```bash
   kubectl get all --all-namespaces -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels}{"\n"}{end}'
   ```
2. **Define a standard labeling policy**: Document your organization’s required keys (e.g., `app.kubernetes.io/name`, `app.kubernetes.io/part-of`, `team`, `environment`, etc.).
3. **Use automation (scripts or GitOps)** to patch existing objects with missing labels. For example:

   ```bash
   kubectl label deployment -n dev payment-service app.kubernetes.io/part-of=ecommerce-app --overwrite
   ```
4. **Enforce going forward with Admission Controllers**: Use a policy engine (like Gatekeeper) that rejects any new workloads missing required labels.

## Understanding Kubernetes Pods and Their Container Types

Kubernetes **Pods** are the most fundamental deployable units in a Kubernetes cluster. A Pod represents a group of one or more containers that share storage, network namespaces, and a specification for how to run them. Pods model an application-specific “logical host” and encapsulate tightly coupled containers that must run together on the same node ([Kubernetes][1], [Kubernetes][1]).

---

### 1. Anatomy of a Pod

* **Containers**
  A Pod may contain one or more application containers, each defined under `spec.containers`. When you run a single-container workload, the Pod wraps that container so Kubernetes manages the Pod instead of the container directly. When you need multiple containers that tightly coordinate (for example, sharing an in-memory cache or forwarding logs), you bundle them in a single Pod ([Kubernetes][1], [Kubernetes][1]).

* **Shared Network Namespace**
  All containers in a Pod share:

   * A single IP address.
   * Port space. If container A listens on port 8080, a container B in the same Pod can connect to `localhost:8080`.
   * Inter-process communication (e.g., `localhost`), making inter-container communication straightforward ([Kubernetes][1]).

* **Shared Storage Volumes**
  Kubernetes allows Pods to mount one or more **Volumes** (e.g., `emptyDir`, `hostPath`, PersistentVolumes). All containers in a Pod can read/write to those volumes at the same mount path. This is crucial for init containers and sidecar containers to exchange data with the main application containers ([Kubernetes][2], [Kubernetes][3]).

Below is a minimal Pod manifest illustrating a single-container Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

To create this Pod:

```bash
kubectl apply -f pod-single-container.yaml
```

---

### 2. Pod Lifecycle

Every Pod in Kubernetes advances through a series of **phases**. Understanding these phases and conditions is essential for troubleshooting and designing resilient workloads.

#### 2.1 Pod Phases

1. **Pending**

   * Pod object exists in the API, but no container images have been pulled yet.
   * Kubernetes is still scheduling the Pod or downloading container images.
2. **Running**

   * At least one container is up and running.
   * All other containers in the Pod may still be starting or in `Waiting` states.
3. **Succeeded**

   * All containers in the Pod have terminated successfully (exit code 0).
   * Pod will not be restarted if `restartPolicy: Never` or `restartPolicy: OnFailure`.
4. **Failed**

   * At least one container terminated with a non-zero exit code, and none are configured to restart (depending on `restartPolicy`).
5. **Unknown**

   * The Kubernetes control plane lost communication with the node where the Pod was running ([Kubernetes][4], [Kubernetes][4]).

> Pods in the `Running` phase may still contain containers in different states (`Waiting`, `Running`, or `Terminated`) as reported under `.status.containerStatuses`.

#### 2.2 Pod Conditions

Kubernetes tracks Pod readiness and scheduling via **conditions**:

* **PodScheduled**: Has the Pod been scheduled to a node?
* **Ready**: Are *all* containers (including init and sidecars that block readiness) in a “Ready” state?
* **ContainersReady**: Are application containers deemed ready (i.e., liveness/readiness probes succeeded)?
* **Initialized**: Have all init containers completed successfully?

You can view a Pod’s current phase and conditions with:

```bash
kubectl describe pod nginx-pod
```

#### 2.3 Scheduling and Binding

* When you `kubectl apply -f` a Pod manifest (or a Deployment/ReplicaSet creates Pods), the API server persists the Pod object.
* A **scheduler** selects an appropriate node based on resource availability, taints/tolerations, and affinities/anti-affinities.
* Scheduling (assigning a Pod to a node) is called **binding**.
* Once bound, the kubelet on the target node pulls container images, sets up network namespaces, and starts containers.
* If the kubelet cannot start the Pod (e.g., the node crashes before Pod startup), Kubernetes may create a fresh Pod elsewhere, depending on the controller’s restart policy or higher-level workload construct (like a Deployment) ([Kubernetes][4]).

---

### 3. Init Containers

**Init containers** are special containers defined under `spec.initContainers` that run **before** any application (or sidecar) containers start ([Kubernetes][2], [Kubernetes][2]).

#### 3.1 Key Characteristics

* **Sequential Execution**:

   * Init containers run sequentially, one after another. The next init container begins only when the previous one succeeds (exit code 0).
   * If an init container fails (non-zero exit), Kubernetes restarts it until it succeeds unless `restartPolicy` is `Never`, in which case the Pod is marked as failed ([Kubernetes][2]).

* **Isolation**:

   * Init containers can run with a different filesystem view (via volume mounts) than application containers. This is useful to fetch secrets or apply configurations that the main containers shouldn’t directly access.
   * They support most container fields: resource requests/limits, volumes, security contexts.
   * However, they **do not** support probes (`livenessProbe`, `readinessProbe`, `startupProbe`) or lifecycle hooks. Once they finish, they don’t run again unless the entire Pod restarts.

* **Use Cases**:

   1. **Populate volumes**: Fetch configuration or secrets from external systems into a shared `emptyDir` or an EFS volume before the main containers start.
   2. **Verify preconditions**: Check that a database migration is complete or that another service is reachable before launching the main application.
   3. **Generate artifacts**: Compile configuration templates, generate certificates, or perform setup tasks that must run as root but you don’t want the main container to run as root.

#### 3.2 Example: Init Container for Config Bootstrapping

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-init-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  initContainers:
    - name: init-fetch-config
      image: bitnami/kubectl:1.27.6
      command:
        - sh
        - -c
        - |
          # Simulate fetching config from a remote server:
          echo "config_value=42" > /mnt/shared/config.env
      volumeMounts:
        - name: shared-data
          mountPath: /mnt/shared
      resources:
        requests:
          memory: "50Mi"
          cpu: "50m"
    - name: init-wait-for-db
      image: busybox:1.35
      command:
        - sh
        - -c
        - |
          # Wait until the database service is reachable:
          until nc -z db-service 5432; do
            echo "Waiting for database..."
            sleep 3
          done
      resources:
        requests:
          cpu: "50m"
          memory: "20Mi"
  containers:
    - name: app
      image: myapp:latest
      env:
        - name: CONFIG_VALUE
          valueFrom:
            configMapKeyRef:
              name: dynamic-config
              key: config_value
      volumeMounts:
        - name: shared-data
          mountPath: /etc/app/config
      ports:
        - containerPort: 8080
  restartPolicy: OnFailure
```

* **`init-fetch-config`** writes a configuration file into `/mnt/shared/config.env`.
* **`init-wait-for-db`** polls a `db-service:5432` TCP socket until PostgreSQL is ready.
* Only after both init containers finish successfully does Kubernetes start the `app` container ([Kubernetes][2], [Kubernetes][2]).

---

### 4. Sidecar Containers

**Sidecar containers** run **concurrently** with the main application containers inside a Pod. Unlike init containers, sidecars remain active throughout the Pod’s lifetime and typically provide supporting functionality, such as logging, monitoring, or proxying ([Kubernetes][3], [Kubernetes][5]).

#### 4.1 When to Use Sidecars

* **Logging and Metrics**:

   * Run a log-collector (e.g., Fluentd) as a sidecar that tails application logs from a shared volume or `stdout` and forwards them to a centralized logging backend.
   * Run a metrics exporter (e.g., Prometheus node-exporter) that scrapes metrics from a socket or file the main app exposes.

* **Proxy / Service Mesh**:

   * Deploy an Envoy or Istio proxy as a sidecar to intercept inbound/outbound traffic, enabling traffic routing, retries, and mTLS without changing application code.

* **Security / Credential Refresh**:

   * Run a certificate manager sidecar to automatically renew TLS certificates on a shared volume while the main app mounts them.

* **Data Synchronization**:

   * Run a sidecar that synchronizes local disks with a remote object store (e.g., an S3 syncer) while the application reads/writes data from the shared volume.

#### 4.2 How Kubernetes Implements Sidecars

* **Feature Gate**:

   * Native sidecar containers were stabilized in **Kubernetes v1.29** under the `SidecarContainers` feature gate (enabled by default).
   * Sidecars are configured under `spec.initContainers` with a `restartPolicy` (`Always`, `OnFailure`, etc.), but they run alongside app containers instead of waiting for init order.
   * Under the hood, Kubernetes treats sidecars as a special kind of init container that does not terminate immediately and supports probes ([Kubernetes][3], [Kubernetes][5]).

* **Lifecycle and Termination Ordering**:

   1. Sidecars marked under `initContainers` with `restartPolicy: Always` (or `OnFailure`) start **before** application containers.
   2. When a readiness probe is defined for a sidecar, its probe contributes to the overall Pod `Ready` state.
   3. Upon Pod termination:

      * Kubernetes waits for application containers to finish gracefully.
      * After all primary containers exit, sidecars are terminated in **reverse** order of listing, ensuring they remain up to service the main containers until no longer needed.
      * If sidecars fail to shut down before the grace period ends, Kubernetes sends `SIGKILL` ([Kubernetes][3]).

#### 4.3 Differences from Init and App Containers

| Aspect                     | Init Containers                                     | App Containers                                                         | Sidecar Containers                                                                          |
| -------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Startup Order**          | Run **sequentially**, before app containers         | Start after all init containers finish (concurrently if multiple apps) | Start **before** app containers (due to init mechanism) but continue running alongside them |
| **Purpose**                | Prepare environment (fetch secrets, run migrations) | Run the primary workload                                               | Provide auxiliary services (logging, proxy, metrics)                                        |
| **Restart Behavior**       | Restart until success (unless Pod failed)           | Governed by `restartPolicy` (`Always`/`OnFailure`/`Never`)             | Governed by `restartPolicy` (often `Always`)                                                |
| **Probes/Lifecycle Hooks** | No liveness/readiness/startup probes                | Support all probes and lifecycle hooks                                 | Support probes (`liveness`, `readiness`)                                                    |
| **Termination Order**      | Only runs once; then exits                          | Terminated first on Pod shutdown                                       | Terminated after primary containers (reverse order)                                         |

#### 4.4 Example: Sidecar for Log Forwarding

Below is a Deployment spec with a main application container (`app`) and a sidecar container (`fluentd-sidecar`) that tails logs from a shared emptyDir and ships them to a remote logging cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      volumes:
        - name: log-volume
          emptyDir: {}
      initContainers:
        - name: log-collector
          image: fluent/fluentd:latest
          command:
            - sh
            - -c
            - |
              fluentd -c /fluentd/etc/fluent.conf
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
          restartPolicy: Always
      containers:
        - name: app
          image: my-web-app:1.2.3
          command:
            - sh
            - -c
            - |
              while true; do echo "Request at $(date)" >> /var/log/app/access.log; sleep 2; done
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
          ports:
            - containerPort: 8080
```

* **Volume `log-volume`**: Shared between the `app` container and the `log-collector` sidecar.
* The sidecar is defined under `initContainers` with `restartPolicy: Always`, so it starts first, establishes a connection to the logging backend, and continues running alongside the `app` container.
* When `app` writes to `/var/log/app/access.log`, the sidecar tails that file and forwards log lines.
* If the sidecar or app crashes, Kubernetes restarts them (depending on `restartPolicy`), but the sidecar will not block Pod readiness unless you define a readiness probe for it ([Kubernetes][3], [Kubernetes][5]).

---

### 5. Ephemeral Containers

**Ephemeral containers** are a special type of container that you can **inject into an already-running Pod** to perform debugging, diagnostics, or inspection tasks. Unlike init or sidecar containers, ephemeral containers are not defined in the original Pod specification; they are added on-demand via the Kubernetes API ([Kubernetes][6], [Kubernetes][1]).

#### 5.1 Characteristics of Ephemeral Containers

* **Temporary and Non-Restartable**:

   * Ephemeral containers will **never** be automatically restarted.
   * Once added, they run as long as the Pod persists or until you explicitly delete them.
* **Purpose**:

   * Designed primarily for **interactive debugging** when `kubectl exec` is insufficient (e.g., the original containers lack a shell or the application has crashed).
   * Use cases include:

      * Attaching a debugging shell (e.g., `busybox` or `bash`) to inspect filesystem, network, or processes.
      * Profiling memory or CPU (e.g., running `top` or `strace`).
      * Inspecting logs or core dumps in a container that crashed but still has its mount namespace intact.
* **Limitations** (disallowed or irrelevant fields):

   * **Probes**: `livenessProbe`, `readinessProbe`, `startupProbe` are **not** allowed.
   * **Ports**: Cannot specify `ports`.
   * **Resources**: Cannot request or limit CPU/memory, because Pod resource allocations are immutable once created.
   * **Lifecycle Hooks**: No `lifecycle` or preStop/postStart hooks.
   * **Security Context**: Some security settings may be disallowed if they conflict with Pod-level security.

Because of these limitations, ephemeral containers are not intended for running production workloads; they are solely for debugging or transient support tasks.

#### 5.2 How to Add an Ephemeral Container

Kubernetes v1.25+ introduced the stable API to add ephemeral containers. You typically use:

```bash
kubectl debug --image=busybox -it <pod-name> --target=<existing-container-name>
```

* **`kubectl debug`**:

   * When run against a Pod, it creates a copy of the Pod spec, adds an ephemeral container (with the specified image and target container for namespace attachment), and updates the Pod’s **`ephemeralcontainers`** subresource.
   * Example: Attach a `busybox` shell to an existing Pod named `prod-pod-abc123`:

     ```bash
     kubectl debug prod-pod-abc123 \
       --image=busybox:1.35 \
       --target=app \
       -it -- /bin/sh
     ```

      * `--target=app` means the ephemeral container will share the same network, PID, and IPC namespace as the `app` container.
      * `-it /bin/sh` spawns an interactive shell session.

Once debug tasks are done, you can remove the ephemeral container by editing the Pod’s `ephemeralcontainers` list (this is automatically cleaned up when the Pod restarts or is replaced in a Deployment).

#### 5.3 Example: Injecting a Debug Container

1. **Create a simple Pod to debug**:

   ```bash
   kubectl run pause-pod \
     --image=registry.k8s.io/pause:3.6 \
     --restart=Never \
     -- sleep 3600
   ```

2. **List current containers**:

   ```bash
   kubectl get pod pause-pod -o jsonpath='{.spec.containers[*].name}'
   # Output: pause
   ```

3. **Inject an Ephemeral Container**:

   ```bash
   kubectl debug pause-pod \
     --image=busybox:1.35 \
     --name=debugger \
     --target=pause \
     -it -- /bin/sh
   ```

4. **Inspect the Pod**:

   ```bash
   kubectl describe pod pause-pod
   ```

   Under `Ephemeral Containers:`, you will see:

   ```
   Name:         pause-pod
   ...
   Ephemeral Containers:
     debugger:
       Image: busybox:1.35
       Command:
         sh
         -c
         -- 
         /bin/sh
       TargetContainerName: pause
       State: Running
       ...
   ```

5. **Interact inside the Ephemeral Container**:
   The interactive shell you opened lets you run commands to inspect network state, file structure, or any other debugging needed. When you exit the shell, the ephemeral container terminates but remains in the Pod’s status until the Pod itself is deleted or replaced ([Kubernetes][6], [Kubernetes][7]).

---

### 6. Putting It All Together

Kubernetes Pods support multiple container types—**app containers**, **init containers**, **sidecar containers**, and **ephemeral containers**—to address different concerns. Understanding how they interrelate helps you design robust, maintainable workloads:

1. **Pod Creation**

   * You define a Pod YAML (or a higher-level controller creates Pods for you). The spec may include:

      * `initContainers`: Tasks that must complete before any other containers start.
      * `containers`: One or more primary application containers.
      * `initContainers` reused for sidecars: If you specify a `restartPolicy` for an init container (e.g., `Always`), Kubernetes treats it as a sidecar, running it before and alongside the main containers.
   * The scheduler picks a node; kubelet on that node pulls images and starts init containers in the order listed.

2. **Pod Startup Flow**

   * **Init Containers** run **one at a time**:

      1. **First init container** executes. If it exits successfully (exit code 0), Kubernetes starts the next one.
      2. If any init container fails, Kubernetes restarts it (respecting the Pod’s `restartPolicy`) until it returns 0 or the Pod fails (e.g., `restartPolicy: Never`).
   * Once **all init/sidecar containers** flagged under `spec.initContainers` have started (and, for sidecars, passed readiness probes if defined), Kubernetes concurrently starts the **application containers** defined under `spec.containers`.

3. **Running Phase**

   * **Application containers** run the core workload (e.g., a web server, database, or batch job).
   * **Sidecar containers** (which are simply init containers with a non-`Never` restart policy) run alongside application containers, providing supporting services: log shipping, metrics exporting, or serving as a proxy.
   * **Pod conditions** (e.g., `Ready`, `ContainersReady`, `Initialized`) are updated as containers pass readiness and liveness checks. Only when all required conditions are `True` does Kubernetes mark the Pod as fully “Ready” for service.

4. **Debugging in Flight**

   * If you need to debug a running Pod (e.g., the application container crashed or lacks debugging tools), you **inject** an **ephemeral container**.
   * Ephemeral containers do not affect the Pod’s spec; they are appended to the Pod’s `ephemeralcontainers` subresource.
   * They run once, do not restart, and allow you to inspect or interact with the Pod’s namespaces.
   * When your debugging session ends, you simply exit the ephemeral container’s shell. Kubernetes retains the ephemeral container’s status information, but it is effectively inert from a scheduling or restart perspective.

5. **Termination Sequence**

   * When you delete a Pod (or a higher-level controller tears it down), Kubernetes sends a **graceful termination** signal (`SIGTERM`) to each container in **reverse order** of how they started:

      1. **Application containers** receive `SIGTERM` first.
      2. **Sidecar containers** (because they are init containers with `restartPolicy: Always`) receive `SIGTERM` next, ensuring they remain up to support cleanup or log shipping until after the main containers exit.
   * If containers do not shut down within `terminationGracePeriodSeconds` (default 30s), Kubernetes issues `SIGKILL` to force termination.

[1]: https://kubernetes.io/docs/concepts/workloads/pods/?utm_source=chatgpt.com "Pods - Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/?utm_source=chatgpt.com "Init Containers | Kubernetes"
[3]: https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/?utm_source=chatgpt.com "Sidecar Containers - Kubernetes"
[4]: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/?utm_source=chatgpt.com "Pod Lifecycle - Kubernetes"
[5]: https://kubernetes.io/docs/tutorials/configuration/pod-sidecar-containers/?utm_source=chatgpt.com "Adopting Sidecar Containers - Kubernetes"
[6]: https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/?utm_source=chatgpt.com "Ephemeral Containers - Kubernetes"
[7]: https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/?utm_source=chatgpt.com "Debug Running Pods | Kubernetes"

## ReplicationController

A **ReplicationController** in Kubernetes ensures that a specified number of pod replicas are running at any given time. In essence, it continuously monitors pod counts and takes action to match the desired replica count—creating new pods if too few exist, or deleting excess pods if too many are present. Although largely superseded by higher-level controllers (like ReplicaSets and Deployments), understanding ReplicationControllers remains valuable for grasping Kubernetes’ foundational scaling and self-healing concepts. ([Kubernetes][1])

---

## 1. Purpose and Core Concepts

1. **Ensuring Desired Replica Count**

   * A ReplicationController (RC) declares a `replicas` field (e.g., `replicas: 3`). Once the RC is created, it continuously compares the *actual* number of pods matching its selector to this desired count.
   * If fewer pods exist, it creates new ones based on a Pod template. If more pods exist, it terminates the extras.
   * This behavior guarantees availability: if a pod is evicted, crashes, or is manually deleted, the RC immediately spins up a replacement. Conversely, manual pod creations or mismatches beyond `replicas` are corrected by deletion. ([Kubernetes][1])

2. **Automatic Replacement After Node or Pod Failure**

   * Because Kubernetes treats a node failure (e.g., hardware fault, reboot) as final for any pods on that node, controllers (like the RC) re-create pods elsewhere in the cluster.
   * Even for a single-pod application, an RC is recommended so that if the lone pod dies or its node becomes unreachable, Kubernetes resurrects it on a healthy node. ([Kubernetes][1])

3. **Legacy Status and Supersession**

   * ReplicationControllers were among the first workload controllers in Kubernetes, predating ReplicaSets and Deployments.
   * In modern clusters, **Deployments** (which under the hood create ReplicaSets) have become the preferred mechanism. Deployments offer rolling updates, history tracking, and declarative rollback. While ReplicaSets are conceptually similar to RCs (and can still be created directly), both are largely vestigial compared to Deployments. ([Kubernetes][1], [Kubernetes][2])

---

## 2. Anatomy of a ReplicationController Manifest

A typical ReplicationController YAML manifests defines:

* **`apiVersion`**: Always `v1` for RCs.
* **`kind`**: `"ReplicationController"`.
* **`metadata`**:

   * `name` (unique within the namespace).
   * Optional `labels` and `annotations`.
* **`spec`**:

   1. **`replicas`**: Integer specifying the desired number of pod instances (e.g., `3`).
   2. **`selector`**: A set of key/value pairs (labels) used to identify which existing pods fall under this RC’s control. Only pods whose metadata labels exactly match this selector are counted and managed.
   3. **`template`**: A Pod template (under `template.metadata.labels` and `template.spec`) that describes how newly created pods should be configured (e.g., container image, ports, volumes, resource requests).

Below is a minimal, functional example:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

* The RC is named `nginx-rc` and is labeled `app=nginx`.
* Its `spec.selector` is also `app=nginx`, meaning any pod with that label “belongs” to this RC.
* Under `template`, the pod template also carries the `app=nginx` label so that newly created pods will immediately match the selector.
* Kubernetes will ensure exactly 3 copies of this pod are running. ([Kubernetes][1])

---

## 3. How a ReplicationController Works

1. **Synchronization Loop**

   * Once an RC is created (e.g., via `kubectl apply -f rc.yaml`), the controller manager’s control loop continually checks:

     ```text
     currentCount = (# of Pods matching selector && not marked for deletion)
     desiredCount = spec.replicas
     if currentCount < desiredCount: create (desiredCount − currentCount) pods
     if currentCount > desiredCount: delete (currentCount − desiredCount) pods
     ```
   * If a pod dies (e.g., container crash or node failure), the control loop notices fewer pods than desired and spins up a new one (on any node that meets scheduling criteria). ([Kubernetes][1])

2. **Pod Replacement**

   * Pods created outside of this RC (e.g., manually with `kubectl run` or by another controller) but carrying the same `app=nginx` label are either:

      * Immediately “adopted” by the RC (if the pod lacks any `ownerReferences`).
      * Left untouched if they already have a different controller in `ownerReferences`.
   * Likewise, manually labeled pods can be excluded from RC control by removing or altering their labels; the RC then spins up a replacement because it sees too few pods under its selector. ([Kubernetes][1])

3. **Pod Deletion and Graceful Termination**

   * When deleting pods to reduce surplus replicas, the RC sends a graceful termination signal (`SIGTERM`) to the container(s) in the pod.
   * If a pod does not terminate within its grace period (default 30 seconds), Kubernetes forcefully kills it with `SIGKILL`. ([Kubernetes][1])

4. **Label-Based Isolation and Debugging**

   * You can remove a pod from RC management temporarily by changing its label so it no longer matches the selector. This can be useful for debugging or data recovery on a single pod. The RC, detecting a missing replica, then creates a new pod with the original labels. ([Kubernetes][1])

---

## 4. Creating and Managing a ReplicationController

### 4.1 Creating an RC

1. Save the YAML manifest (e.g., `rc.yaml`).
2. Apply with:

   ```bash
   kubectl apply -f rc.yaml
   ```
3. Verify creation:

   ```bash
   kubectl get rc
   kubectl get pods -l app=nginx
   ```

   This shows the RC and the pods it created.

### 4.2 Viewing Status

* **`kubectl describe rc nginx-rc`**
  Displays:

   * Current vs. desired replicas.
   * Events (e.g., “Scaling up from 2 to 3”, “Created pod nginx-rc-abcde”).
* **`kubectl get pods`**
  Lists all pods; the RC-created pods will have autogenerated names like `nginx-rc-abcde`, `nginx-rc-fghij`, etc.

### 4.3 Scaling an RC

* **Manually** update the `spec.replicas` in the YAML and re-apply:

  ```yaml
  spec:
    replicas: 5
  ```

  Then:

  ```bash
  kubectl apply -f rc.yaml
  ```

  The RC control loop notices the new desired count (5), so it creates two additional pods. ([Kubernetes][1])

* **Imperatively** with `kubectl scale`:

  ```bash
  kubectl scale rc nginx-rc --replicas=5
  ```

  This triggers an in-cluster patch that immediately adjusts `spec.replicas` to 5.

### 4.4 Deleting an RC

* To delete both the RC and its pods:

  ```bash
  kubectl delete -f rc.yaml
  ```

  or

  ```bash
  kubectl delete rc nginx-rc
  ```
* By default, deleting the RC also deletes its pods (because they have the cleanup policy). If you only delete the RC but want to retain pods, you can use `--cascade=false`:

  ```bash
  kubectl delete rc nginx-rc --cascade=false
  ```

  This leaves the existing pods running, but the RC no longer manages or replaces them. ([Kubernetes][1])

---

## 5. Comparison to ReplicaSets and Deployments

1. **ReplicaSet (RS)**

   * A ReplicaSet is functionally similar to a ReplicationController but supports set-based selectors (e.g., `key in (value1,value2)`), whereas RCs only allow equality-based selectors (`key=value`).
   * RS was introduced to support more flexible label matching and has largely replaced RC in practice.
   * Both RC and RS guarantee *N* replicas of pods matching a selector, but RS is the “modern” API. ([Kubernetes][2])

2. **Deployment**

   * A Deployment manages stateless pods by creating and updating ReplicaSets. When you apply a Deployment manifest specifying a new container image or altered pod template, Kubernetes:

      1. Creates a new ReplicaSet.
      2. Gradually scales down the old RS and scales up the new RS (rolling update).
   * Deployments maintain rollout history, support rollbacks, and allow pausing/resuming rollouts—features lacking in RCs or RSs. ([Kubernetes][3])

3. **StatefulSet, DaemonSet, Job**

   * StatefulSet ensures each pod has a unique, persistent identity (stable network ID and associated storage).
   * DaemonSet launches exactly one pod per (matching) node, ideal for node-level services (logging agents, monitoring).
   * Job runs one-off, finite tasks that must complete successfully (e.g., batch jobs).
   * All of these are specialized controllers; if you simply need *N* interchangeable pods, a ReplicaSet (or Deployment) is the recommended choice over RC.

---

## 6. Best Practices and Considerations

1. **Prefer Deployments over RCs**

   * For any stateless service that requires updates (image rollouts, configuration changes), use a Deployment. Deployments automatically create ReplicaSets and provide robust update/rollback semantics.
   * Only use RCs directly if you have a very specific need and understand the limitations (no rolling updates, no revision history). ([Kubernetes][1], [Kubernetes][3])

2. **Design Consistent Labeling**

   * Ensure that the `spec.selector` matches exactly the `template.metadata.labels`. Any mismatch prevents pods from being managed correctly or causes “orphaned” pods.
   * Avoid mixing pod labels: if you place other labels on pods that match the RC’s selector but are not intended to be managed by this RC, the RC may inadvertently adopt them. ([Kubernetes][1])

3. **Use Label-Based Isolation for Debugging**

   * Temporarily remove a pod from an RC by editing its labels (e.g., `kubectl label pod <pod> app-`). This removes it from the RC’s selector scope, and the RC spawns a replacement.
   * Once debugging is complete, revert its labels to reincorporate it under RC control. ([Kubernetes][1])

4. **Monitor Pod Health**

   * Because RCs lack built-in readiness or liveness probe support (unlike Deployments), unhealthy pods are not automatically replaced unless they crash. Combine RCs with manual liveness checks or external monitoring to detect unhealthy but still “running” pods.
   * Consider migrating to Deployments if you need health-based rolling updates.

5. **Resource Quotas and Namespaces**

   * Place RCs in dedicated namespaces per team or environment (e.g., `dev`, `staging`, `prod`) and use ResourceQuotas to enforce limits on CPU, memory, and pod counts.
   * This helps prevent a runaway RC from exhausting cluster resources.

---

## 7. Example Workflow: From YAML to Running Pods

1. **Define the RC YAML** (e.g., `nginx-rc.yaml`):

   ```yaml
   apiVersion: v1
   kind: ReplicationController
   metadata:
     name: nginx-rc
   spec:
     replicas: 3
     selector:
       app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:1.14.2
           ports:
           - containerPort: 80
   ```

2. **Create the RC**:

   ```bash
   kubectl apply -f nginx-rc.yaml
   ```

3. **Verify Creation**:

   ```bash
   kubectl get rc
   # Output: NAME       DESIRED   CURRENT   READY   AGE
   #         nginx-rc   3         3         3       1m

   kubectl get pods -l app=nginx
   # Now you have 3 pods: nginx-rc-abcde, nginx-rc-fghij, nginx-rc-klmno
   ```

4. **Scale Up** to 5 replicas:

   ```bash
   kubectl scale rc nginx-rc --replicas=5
   ```

   or edit `spec.replicas: 5` and re-apply.

   ```bash
   kubectl get rc nginx-rc
   # Output changes to: DESIRED=5, CURRENT=5, READY=5
   ```

5. **Simulate a Pod Failure**:

   ```bash
   kubectl delete pod nginx-rc-abcde
   ```

   * The RC immediately notices only 4 pods remain and creates a new one (e.g., `nginx-rc-pqrst`), returning the count to 5 within seconds.

6. **Scale Down** to 2 replicas:

   ```bash
   kubectl scale rc nginx-rc --replicas=2
   ```

   * The RC deletes 3 of the existing pods (chosen arbitrarily), ensuring only 2 remain.

7. **Delete the RC and Its Pods**:

   ```bash
   kubectl delete rc nginx-rc
   # All associated pods are also removed in a graceful manner.
   ```

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/?utm_source=chatgpt.com "ReplicationController - Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com "ReplicaSet - Kubernetes"
[3]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments | Kubernetes"

## ReplicaSet 

A **ReplicaSet** ensures that a specified number of pod replicas are running at any given time. It continuously compares the *actual* number of pods matching its selector to the *desired* count and creates or deletes pods as needed to achieve that state ([Kubernetes][1]). ReplicaSets are typically managed indirectly by higher-level controllers (such as Deployments), but understanding them clarifies Kubernetes’ core reconciliation logic.

---

## 1. Anatomy of a ReplicaSet Manifest

Every ReplicaSet manifest follows the familiar Kubernetes object structure:

1. **`apiVersion: apps/v1`**
   Indicates the API group and version. ReplicaSets live under `apps/v1` in modern Kubernetes clusters ([Kubernetes][1]).

2. **`kind: ReplicaSet`**
   Declares this object as a ReplicaSet.

3. **`metadata`**

   * **`name`**: A unique identifier within the namespace (e.g., `nginx-rs`).
   * **`labels`** (optional): Key/value pairs for grouping and querying. While not strictly required for ReplicaSet functionality, labeling resources consistently aids observability and policy enforcement.

4. **`spec`**

   * **`replicas`**: An integer specifying how many pod replicas should be running (e.g., `replicas: 3`).
   * **`selector`**: A label selector that identifies which pods this ReplicaSet should “own.” Only pods whose `metadata.labels` match this selector (and that do not already have an `ownerReference` pointing to another controller) are counted and managed. For example:

     ```yaml
     selector:
       matchLabels:
         app: nginx
     ```
   * **`template`**: A Pod template defining the data for new pods created by the ReplicaSet. It must include:

      * **`metadata.labels`** that exactly satisfy the selector. If these labels do not match, the ReplicaSet would never “adopt” the pods it creates.
      * **`spec`** defining the containers, volumes, and other pod-level configuration.

Below is a minimal example manifest:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
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
          image: nginx:1.21.0
          ports:
            - containerPort: 80
```

* The ReplicaSet is named `nginx-rs` and is labeled `app=nginx`.
* The **selector** also matches `app=nginx`.
* Under `template.metadata.labels`, the pod template’s labels are set to `app=nginx`, ensuring newly created pods will satisfy the selector.
* Kubernetes will maintain exactly **3** running pods matching `app=nginx` ([Kubernetes][1]).

---

## 2. How a ReplicaSet Works

### 2.1 Core Reconciliation Loop

Once you create a ReplicaSet (for example, via `kubectl apply -f replicaset.yaml`), the controller manager’s control loop continuously executes the following logic:

1. **Read**: Count the number of pods in the cluster that:

   * Match the ReplicaSet’s **selector**.
   * Do **not** have an `ownerReference` pointing to another controller (or have no `ownerReference` at all).
2. **Compare**: Let `currentCount` be that number, and let `desiredCount = spec.replicas`.
3. **Reconcile**:

   * If `currentCount < desiredCount`, the ReplicaSet creates `(desiredCount − currentCount)` new pods by using its pod template.
   * If `currentCount > desiredCount`, it deletes `(currentCount − desiredCount)` pods, chosen arbitrarily among the matching pods.
   * If `currentCount == desiredCount`, it does nothing.

As pods die (either through crashes, manual deletion, or node failures), the loop notices `currentCount` has dropped below `desiredCount` and spawns replacements. Conversely, if you manually create extra pods matching the selector (and without an existing owner reference), the ReplicaSet instantly “adopts” them by adding itself as owner reference. If there are too many, it deletes surplus pods to reduce the count back to `desiredCount` ([Kubernetes][1]).

### 2.2 OwnerReference and Pod Adoption

* When a ReplicaSet creates pods, it sets each pod’s `metadata.ownerReferences` to reference the ReplicaSet’s UID. This link is how the ReplicaSet knows which pods it owns.
* If you manually create a pod (e.g., `kubectl run mypod --labels="app=nginx"`), and it has no `ownerReferences`, but its labels match the ReplicaSet’s selector, the ReplicaSet will immediately “adopt” this pod by adding itself to its `ownerReferences`.
* If a pod already has an `ownerReference` pointing to a different controller, the ReplicaSet will ignore it (thus preventing conflicts between controllers) ([Kubernetes][1]).

### 2.3 Pod Creation and Deletion

* **Creation**: When the ReplicaSet determines it needs more pods, it uses the **pod template** in `spec.template` to create new pods. Those pods inherit the labels, container specs, volume mounts, and other configuration from the template.
* **Graceful Deletion**: When the ReplicaSet removes excess pods, Kubernetes sends a `SIGTERM` to each container in the pod, allowing applications to shut down cleanly. If the pod does not exit within the `terminationGracePeriodSeconds` (default 30 seconds), Kubernetes force-kills it with `SIGKILL` ([Kubernetes][1]).

---

## 3. When to Use a ReplicaSet Directly

Although **Deployments** are the recommended way to manage stateless workloads—because they provide rolling updates, revision history, and rollbacks—there are scenarios where using a ReplicaSet directly makes sense:

1. **Custom Update Orchestration**
   If you require a bespoke update strategy that differs from Deployment’s rolling-update or recreate policies, you might manage ReplicaSets yourself. For example, you could write a custom controller to manipulate ReplicaSet replicas in a nonstandard pattern.

2. **No Update Requirements**
   If your workload is completely immutable—meaning you never plan to change the pod template once deployed—and you simply want Kubernetes to ensure *N* identical pods, a standalone ReplicaSet suffices.

However, for typical application deployments, using a Deployment is highly recommended. A Deployment automatically creates and manages ReplicaSets under the hood, taking care of scaling, rolling updates, and rollbacks. The ReplicaSet API exists primarily for backward compatibility and for those rare cases where you need direct control over ReplicaSet behavior ([Kubernetes][1], [Kubernetes][2]).

---

## 4. Managing ReplicaSets via `kubectl`

### 4.1 Creating a ReplicaSet

1. Save the YAML manifest as, for example, `nginx-rs.yaml`.
2. Apply it with:

   ```bash
   kubectl apply -f nginx-rs.yaml
   ```
3. Verify that the ReplicaSet and its pods are running:

   ```bash
   kubectl get replicaset
   # NAME       DESIRED   CURRENT   READY   AGE
   # nginx-rs  3         3         3       1m

   kubectl get pods -l app=nginx
   # Lists pods with names like nginx-rs-abcde, each in Ready state.
   ```

### 4.2 Viewing Details

* **Describe the ReplicaSet**:

  ```bash
  kubectl describe replicaset nginx-rs
  ```

  You’ll see:

   * **Desired** vs. **Current** pod counts.
   * **Events** such as “SuccessfulCreate” or “SuccessfulDelete.”
   * The pod template and selector.

* **Get ReplicaSet in YAML/JSON**:

  ```bash
  kubectl get rs nginx-rs -o yaml
  ```

  This shows you the full object, including status, conditions, and events embedded in the status subresource.

### 4.3 Scaling a ReplicaSet

* **Imperatively**:

  ```bash
  kubectl scale rs nginx-rs --replicas=5
  ```

  This updates `spec.replicas` to 5 instantly. The control loop will create two more pods to reach the new desired count.
* **Declaratively**: Edit `nginx-rs.yaml` to change:

  ```yaml
  spec:
    replicas: 5
  ```

  Then run:

  ```bash
  kubectl apply -f nginx-rs.yaml
  ```

  The result is the same: the ReplicaSet control loop notices it must add two additional pods.

### 4.4 Deleting a ReplicaSet

* **Cascade Deletion** (default):

  ```bash
  kubectl delete rs nginx-rs
  ```

  By default, deleting a ReplicaSet also deletes all pods it owns via `ownerReferences`.
* **Orphan Pods**: If you want to delete the ReplicaSet object but keep the pods running, use:

  ```bash
  kubectl delete rs nginx-rs --cascade=false
  ```

  The ReplicaSet is removed from the API, but the pods remain (now with no `ownerReferences`). However, because there’s no controller to manage them, Kubernetes will not replace them if they fail ([Kubernetes][1]).

---

## 5. Comparison: ReplicaSet vs. ReplicationController vs. Deployment

### 5.1 ReplicaSet vs. ReplicationController

* **Selector Support**:

   * **ReplicationController** only supports **equality-based** selectors (e.g., `key=value`).
   * **ReplicaSet** adds support for **set-based** selectors (e.g., `key in (value1,value2)`, `key notin (value1,value2)`, `key` exists, etc.) ([kodekloud.com][3]).
* **Evolution**: ReplicaSet is effectively the “next-generation” ReplicationController. Functionally they both ensure *N* pods are running, but ReplicaSet’s flexible selector syntax is the primary difference.
* **Recommendation**: New clusters and workloads should use ReplicaSets (via Deployments). Direct usage of ReplicationControllers is discouraged and largely maintained for backward compatibility ([Kubernetes][1], [Kubernetes][4]).

### 5.2 ReplicaSet vs. Deployment

* A **Deployment** encapsulates one or more ReplicaSets to provide higher-level features such as:

   * **Declarative rolling updates**: Update container images or pod templates at a controlled rate (e.g., max 25% unavailable at a time).
   * **Rollbacks**: If a new ReplicaSet rollout fails, you can revert to a previous ReplicaSet revision.
   * **Pause/Resume**: Pause an ongoing rollout to make multiple changes before resuming.
   * **Revision history**: Deployments keep an ordered history of ReplicaSet revisions.
* When you create a Deployment, Kubernetes automatically creates a new ReplicaSet under the hood. You should *never* manually modify those child ReplicaSets—always work at the Deployment level. If you need direct access to the ReplicaSets for advanced use cases (e.g., bespoke update logic), you can interact with them directly, but that is rare ([Kubernetes][2], [Kubernetes][4]).

---

## 6. Best Practices for ReplicaSets

1. **Prefer Deployments for Production-grade Workloads**

   * For almost all stateless applications, use a Deployment. It automatically manages ReplicaSets and provides robust update/rollback functionality.
   * Only use a standalone ReplicaSet if you require direct, low-level control over the scaling and adoption behavior, and you do not need rolling updates or history.

2. **Ensure Selector and Template Labels Match Exactly**

   * A common pitfall is misaligned labels. If `spec.selector.matchLabels` does not exactly equal the labels in `spec.template.metadata.labels`, the ReplicaSet will never adopt the pods it creates (or might adopt unintended pods).
   * Always double-check both the selector and the pod template’s labels.

3. **Use Namespaces and ResourceQuotas**

   * Place ReplicaSets in appropriate namespaces (e.g., `dev`, `staging`, `prod`).
   * Define **ResourceQuota** objects per namespace to limit CPU, memory, and pod count. This prevents a runaway ReplicaSet from exhausting cluster resources.

4. **Monitor ReplicaSet and Pod Health**

   * Because ReplicaSets do not inherently perform health-based replacements for non-crashing but unhealthy pods, consider attaching **liveness** and **readiness probes** in the pod template. A failing liveness probe causes the kubelet to restart the container, which in turn may lead the pod to exit if it cannot recover, triggering the ReplicaSet to recreate a fresh pod.
   * Use `kubectl rollout status` on a Deployment to track progress of a ReplicaSet rollout indirectly. For a standalone ReplicaSet, watch pods with `kubectl get pods -l <selector>` and `kubectl describe pods <pod-name>` to examine pod conditions and events.

5. **Label Consistently for Tooling and Policies**

   * Even if you create a ReplicaSet directly, use standard label conventions (e.g., `app.kubernetes.io/name`, `app.kubernetes.io/instance`, `environment`, `team`) so monitoring dashboards, network policies, and admission controllers can operate predictably.

---

## 7. Example Workflow: Using a ReplicaSet

1. **Define the ReplicaSet YAML** (`redis-rs.yaml`):

   ```yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: redis-rs
     labels:
       app: redis
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: redis
     template:
       metadata:
         labels:
           app: redis
       spec:
         containers:
           - name: redis
             image: redis:6.2.6
             ports:
               - containerPort: 6379
   ```

2. **Create**:

   ```bash
   kubectl apply -f redis-rs.yaml
   ```

3. **Verify**:

   ```bash
   kubectl get rs
   # NAME       DESIRED   CURRENT   READY   AGE
   # redis-rs   2         2         2       30s

   kubectl get pods -l app=redis
   # redis-rs-abcde   1/1     Running   0          30s
   # redis-rs-fghij   1/1     Running   0          30s
   ```

4. **Scale Up** to 4 replicas:

   * Imperative:

     ```bash
     kubectl scale rs redis-rs --replicas=4
     ```
   * Declarative: Edit `redis-rs.yaml` → `replicas: 4` → `kubectl apply -f redis-rs.yaml`.
   * Result: two additional pods (`redis-rs-xxxxx`, `redis-rs-yyyyy`) appear within seconds.

5. **Simulate a Pod Crash**:

   ```bash
   kubectl delete pod redis-rs-abcde
   ```

   * The ReplicaSet control loop immediately detects only 3 pods remain, so it creates a new one (e.g., `redis-rs-zzzzz`), restoring the count to 4.

6. **Scale Down** to 1 replica:

   ```bash
   kubectl scale rs redis-rs --replicas=1
   ```

   * Kubernetes deletes three pods arbitrarily to honor `spec.replicas=1`. The remaining pod (e.g., `redis-rs-abcde`) continues running.

7. **Delete the ReplicaSet (Cascade)**:

   ```bash
   kubectl delete rs redis-rs
   ```

   * All pods owned by `redis-rs` are also terminated gracefully.

8. **Delete the ReplicaSet (Orphan Pods)**:

   ```bash
   kubectl delete rs redis-rs --cascade=false
   ```

   * The `redis-rs` object is removed, but the pods continue running. Because there’s no controller owning them, they will not be replaced if they fail.

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com "ReplicaSet - Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments | Kubernetes"
[3]: https://kodekloud.com/community/t/replication-controller-vs-replica-set/50419?utm_source=chatgpt.com "Replication Controller vs Replica set - Kubernetes - KodeKloud"
[4]: https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/?utm_source=chatgpt.com "ReplicationController - Kubernetes"

## DaemonSet 

A **DaemonSet** ensures that a copy of a Pod runs on all (or a subset of) nodes in a cluster. It’s ideal for deploying node-level utilities—such as logging agents, monitoring daemons, or network plugins—that must be present on every host. Below is a comprehensive overview of DaemonSets, their configuration, and practical usage in Kubernetes. ([Kubernetes][1])

---

## 1. Purpose and Common Use Cases

A DaemonSet is fundamentally about **node-local functionality**. When you create a DaemonSet, the controller:

1. **Watches for new nodes**: As each node joins the cluster, Kubernetes automatically schedules a Pod from the DaemonSet onto it.
2. **Removes Pods when nodes leave**: If a node is removed or marked unschedulable, its DaemonSet Pod is garbage-collected.
3. **Ensures a copy per eligible node**: At any time, the desired state (“one copy per node”) is reconciled.

Typical scenarios include:

* **Cluster storage daemons** (e.g., running an iSCSI or Ceph client on every node).
* **Log collection agents** (e.g., Fluentd, Logstash, Prometheus Node Exporter) to capture application or system logs locally.
* **Node monitoring utilities** (e.g., `node-exporter`, `metrics-server`, or custom health checks) that gather node metrics.
* **Network plugins** (e.g., CNI components like Calico or Weave) that must configure networking on each host. ([Kubernetes][1])

You might use multiple DaemonSets for the same purpose if, for example, you need different resource requests or tolerations on heterogeneous hardware (e.g., high-memory vs. low-memory nodes).

---

## 2. Anatomy of a DaemonSet Manifest

Every DaemonSet manifest follows the Kubernetes API conventions:

1. **`apiVersion: apps/v1`**
2. **`kind: DaemonSet`**
3. **`metadata`**:

   * `name`: Unique within the namespace.
   * `namespace` (optional; defaults to the current context’s namespace).
   * `labels`/`annotations` (optional) for grouping or tooling.
4. **`spec`**: The core configuration. This section must include:

   * **`selector`**: A label selector that matches each Pod created by this DaemonSet.
   * **`template`**: A Pod template (just like in Deployments or ReplicaSets) describing the containers, volumes, and node scheduling constraints.

Below is a minimal example that runs a logging agent on every node’s `/var/log` directory: ([Kubernetes][1])

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      # Tolerations to run on control-plane nodes (if needed)
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule

      containers:
        - name: fluentd-elasticsearch
          image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log

      # Ensure graceful shutdown
      terminationGracePeriodSeconds: 30

      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

**Key points about this manifest**:

* **Selector & Pod labels**: Under `.spec.selector.matchLabels` and `.spec.template.metadata.labels`, the label `name: fluentd-elasticsearch` must match exactly. If they differ, the API rejects the configuration.
* **Pod template**: Mirrors a standalone Pod definition (minus `apiVersion`/`kind`). The `restartPolicy` is implicitly `Always` (the only valid policy for DaemonSet Pods).
* **Tolerations**: Allow running on control-plane nodes, which are often tainted `NoSchedule` by default.
* **Volumes**: Use a `hostPath` to capture logs from `/var/log` on each host.

Create this DaemonSet with:

```bash
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

---

## 3. Required Fields and Validations

When defining a DaemonSet, Kubernetes enforces:

1. **`metadata.name`**

   * Must conform to DNS subdomain naming rules (lowercase alphanumeric characters, `-`, and `.`).
2. **`spec.selector`**

   * Must specify either `matchLabels` and/or `matchExpressions`.
   * Once created, `spec.selector` is immutable. Changing it would orphan existing Pods.
   * Must match the labels under `spec.template.metadata.labels` exactly; otherwise, the API server rejects the object. ([Kubernetes][1])
3. **`spec.template`**

   * Requires the same fields as a standard Pod template (`metadata.labels`, `spec.containers`, `volumes`, etc.).
   * `spec.template.spec.restartPolicy` must be `Always` or omitted (default is `Always`).
4. **`spec.template.spec.nodeSelector`** / **`affinity`** (optional)

   * If specified, the DaemonSet controller only creates Pods on nodes matching those selectors or affinities.
   * If omitted, the DaemonSet schedules Pods on **all** eligible (non-tainted or tolerated) nodes. ([Kubernetes][1])

---

## 4. How DaemonSet Scheduling Works

### 4.1 Creating Pods on Existing Nodes

When you apply a DaemonSet:

1. Kubernetes immediately evaluates **all existing nodes**.
2. For each node that is deemed **eligible** (i.e., it matches any `nodeSelector`/`affinity` and does not become `unschedulable` unless tolerated), the DaemonSet controller creates a Pod whose spec includes a **node-affinity rule** ensuring it binds to that exact node.
3. The default scheduler then picks up that Pod with `.spec.nodeName` already set and marks it **Scheduled**.
4. If the Pod cannot fit (due to resource pressure), Kubernetes may preempt (evict) lower-priority Pods to make room—especially important if you assign a high `priorityClassName` to critical DaemonSet Pods. ([Kubernetes][1])

### 4.2 Handling New Nodes

Whenever a **new node** joins the cluster:

1. It is automatically labeled (e.g., with zone, instance type, or custom labels).
2. The DaemonSet controller notices this addition and creates a corresponding Pod on that node—again using a node-affinity rule to tie the Pod to that node.
3. If the node is `NotReady` or unschedulable but the Pod tolerates relevant taints (e.g., `node.kubernetes.io/unschedulable:NoSchedule` is automatically added by the controller), the Pod may start before the node is fully ready. This avoids deadlock scenarios (e.g., a network plugin that can’t run because the node isn’t ready, but the node isn’t ready because the network isn’t set up). ([Kubernetes][1])

### 4.3 Removing Pods from Deleted or Unschedulable Nodes

If a node is removed (manual deletion, hardware failure, or `kubectl drain`), Kubernetes:

1. Detects that the node is gone or that all DaemonSet Pods on it are gone.
2. The DaemonSet controller **garbage-collects** any remaining DaemonSet Pods tied to that node.
3. As long as other nodes still exist that match the DaemonSet’s selector, Pods remain running on them.

---

## 5. Advanced Scheduling Controls

### 5.1 Node Selectors and Affinities

* **`.spec.template.spec.nodeSelector`**: A simple key/value match.

  ```yaml
  spec:
    template:
      spec:
        nodeSelector:
          hardware-type: high-memory
  ```

  Here, only nodes labeled `hardware-type=high-memory` receive a DaemonSet Pod.

* **`.spec.template.spec.affinity.nodeAffinity`**: More expressive.

  ```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: region
                operator: In
                values:
                  - us-west-1
                  - us-west-2
  ```

  This ensures Pods schedule only on nodes in those regions. ([Kubernetes][1])

### 5.2 Tolerations and Taints

* DaemonSet Pods automatically receive a toleration for `node.kubernetes.io/unschedulable:NoSchedule`, letting them run on nodes even if `kubectl cordon` has been used.
* You can add your own tolerations to handle custom taints (e.g., to run on GPU nodes or control-plane nodes).

### 5.3 Priority Classes

If a DaemonSet provides a **critical** node-level service (e.g., network plugin), you may assign a high-priority class:

```yaml
spec:
  template:
    spec:
      priorityClassName: system-node-critical
```

This means that if resources are tight, lower-priority Pods on that node can be evicted to make room for the DaemonSet Pod. ([Kubernetes][1])

---

## 6. Managing DaemonSets

### 6.1 Creating and Applying

* **Imperative**:

  ```bash
  kubectl apply -f daemonset.yaml
  ```
* **Declarative**: Maintain the `daemonset.yaml` (or multiple versions) in Git for version control.

### 6.2 Viewing DaemonSet Status

* **List DaemonSets**:

  ```bash
  kubectl get daemonsets --all-namespaces
  ```

* **Describe a DaemonSet**:

  ```bash
  kubectl describe daemonset fluentd-elasticsearch -n kube-system
  ```

  You’ll see:

   * Desired vs. current number of nodes with Pods.
   * Events (e.g., scheduling failures, Pod creations, evictions).
   * The Pod template, selector, and tolerations.

* **List Pods owned by a DaemonSet**:

  ```bash
  kubectl get pods -l name=fluentd-elasticsearch -n kube-system
  ```

### 6.3 Updating a DaemonSet

#### 6.3.1 Rolling Updates

DaemonSets support rolling updates—meaning that when you change the Pod template (e.g., update the container image), Kubernetes can replace Pods gradually rather than all at once. By default, updates happen in **parallel** on as many nodes as possible. You can control this using `.spec.updateStrategy`:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

* **`maxUnavailable`**: The maximum number of Pods that can be unavailable during an update (either as an integer or as a percentage).

If you set `type: OnDelete`, Pods are only replaced when you delete them manually—no automatic reconciliation.

#### 6.3.2 Example: Change Image Version

1. Edit `daemonset.yaml`, update:

   ```yaml
   containers:
     - name: fluentd-elasticsearch
       image: quay.io/fluentd_elasticsearch/fluentd:v2.6.0
   ```
2. Apply:

   ```bash
   kubectl apply -f daemonset.yaml
   ```
3. The controller creates new Pods with `v2.6.0` on nodes, evicts old Pods (according to `maxUnavailable`), and continues until all nodes run the updated image. ([Kubernetes][1])

### 6.4 Deleting a DaemonSet

* **Cascade Deletion (default)**:

  ```bash
  kubectl delete daemonset fluentd-elasticsearch -n kube-system
  ```

  This deletes the DaemonSet and **all** its Pods across the cluster.

* **Orphan Pods** (rarely recommended):

  ```bash
  kubectl delete daemonset fluentd-elasticsearch -n kube-system --cascade=false
  ```

  The DaemonSet object is removed, but existing Pods remain (without an owner reference). Kubernetes no longer replaces or garbage-collects them.

---

## 7. Best Practices and Considerations

1. **Use DaemonSets for Node-Level Services**

   * Any functionality that must run per node (rather than per-Pod or per-Deployment) should use a DaemonSet. For instance, log collection or metrics exporters.

2. **Segment by Node Attributes**

   * If you only want some nodes to run the DaemonSet (e.g., GPU nodes), use `nodeSelector` or `nodeAffinity` to limit scheduling.

3. **Apply Appropriate Tolerations**

   * If control-plane nodes are tainted `NoSchedule`, include tolerations if the DaemonSet must run on them (e.g., for cluster networking).

4. **Assign High Priority to Critical Daemons**

   * Use a `PriorityClass` (e.g., `system-node-critical`) to ensure your DaemonSet Pods preempt less critical workloads if resources are scarce.

5. **Monitor Resource Usage**

   * A DaemonSet runs one Pod per node, so total resource consumption scales with cluster size. Ensure that each Pod’s `requests`/`limits` are appropriate to avoid node resource exhaustion.

6. **Rolling Update Strategy**

   * If a DaemonSet Pod provides critical functionality (e.g., network setup), consider setting a low `maxUnavailable` to avoid extended periods where some nodes lack that service.

7. **Namespace Organization**

   * Typically, cluster-level DaemonSets (e.g., network plugins) belong in `kube-system`, while application-specific DaemonSets can live in application namespaces (e.g., `logging`, `monitoring`).

8. **Immutable Selector**

   * Because `spec.selector` is immutable once created, plan labels carefully. If you need a different selector, you must delete and recreate the DaemonSet (taking care to avoid service disruption).

---

## 8. Comparison to Other Controllers

| Feature                          | Deployment                                       | ReplicaSet / ReplicationController | DaemonSet                                                                       |
| -------------------------------- | ------------------------------------------------ | ---------------------------------- | ------------------------------------------------------------------------------- |
| **Scale model**                  | Desired replicas across any nodes                | Desired replicas across any nodes  | Exactly one (or selected) per eligible node                                     |
| **Rolling updates**              | Built-in, controlled by strategy fields          | N/A (no automatic rolling update)  | Built-in support via `updateStrategy.rollingUpdate`                             |
| **Use case**                     | Stateless applications (web servers, APIs)       | Basic pod replication (legacy)     | Node-level services (logging, metrics, network plugins)                         |
| **Selector mutability**          | Immutable in ReplicaSet, inherited by Deployment | Immutable                          | Immutable                                                                       |
| **Pod scheduling**               | Default scheduler picks a node per Pod           | Same as Deployments                | Controller schedules one Pod per node first, then binds via `.spec.nodeName`    |
| **Restart policy**               | Pod Spec’s `restartPolicy` (default `Always`)    | Same                               | Always (only valid policy)                                                      |
| **Toleration for unschedulable** | Must be defined manually                         | Same                               | Automatically includes `node.kubernetes.io/unschedulable:NoSchedule` toleration |

Use a **Deployment** (and its ReplicaSets) for stateless, horizontally scalable workloads where you control the replica count. Use a **DaemonSet** when you need a Pod running on each (or a subset of) node. ([Kubernetes][1], [Kubernetes][2])

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/?utm_source=chatgpt.com "DaemonSet - Kubernetes"
[2]: https://v1-32.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/?utm_source=chatgpt.com "DaemonSet - Kubernetes"

## Jobs

Kubernetes **Jobs** are a workload controller designed to run **finite, one-off tasks** until completion rather than long-running services. Jobs create one or more Pods and ensure that a specified number of them terminate successfully. Once the desired completions are reached, the Job is marked as complete and no new Pods are created. Jobs are particularly useful for batch processing, data migrations, scheduled tasks (when combined with CronJobs), or any scenario where you need guaranteed execution of a task to completion.

---

## 1. Core Concepts and Use Cases

1. **Guaranteed Completion**

   * A Job’s primary purpose is to run pods that exit successfully (exit code 0) at least a configured number of times. Kubernetes tracks each Pod’s success or failure and will retry failed Pods up to a `backoffLimit` if necessary.
   * Once the Job’s `.spec.completions` count is met, the Job transitions to the **Completed** state and will not create new Pods.

2. **One-Time or Parallel Processing**

   * **Non-parallel (default) Jobs** run a single Pod to completion.
   * **Parallel Jobs** can split work across multiple Pods in two ways:

      * **Work Queue (Indexed) Mode** (`.spec.completionMode: Indexed` - Kubernetes v1.21+): Each Pod receives a unique index (via the `JOB_COMPLETION_INDEX` environment variable) so you can divide tasks deterministically.
      * **Fork/Join (Non-Indexed) Mode** (default): Multiple Pods run identical work; often they pull tasks from a shared queue or database. You control idempotency and work distribution in your application logic.

3. **Retry and Failure Handling**

   * **`backoffLimit`**: How many times Kubernetes will retry a failing Pod before marking the entire Job as **Failed**. Once Pod crashes more than `backoffLimit` attempts, the Job’s status becomes Failed.
   * **`activeDeadlineSeconds`**: An overall timeout (in seconds) for the Job. If exceeded, Kubernetes terminates any running Pods and marks the Job as Failed.
   * **`ttlSecondsAfterFinished`** (TTL Controller, v1.21+): Once a Job completes (either Succeeded or Failed), the TTL controller can automatically clean up the Job and its Pods after the specified seconds.

4. **Use Cases**

   * **Batch Processing**: Perform compute-intensive work such as video transcoding or image resizing.
   * **Database Migrations**: Run schema migrations upon deployment.
   * **Data Analysis**: Kick off a Spark or MapReduce task with multiple worker Pods.
   * **Backups**: Take snapshots or backups of persistent data.
   * **One-Off Administrative Tasks**: Run a one-time script (e.g., audit logs, cleanup).

---

## 2. Anatomy of a Job Manifest

A Job manifest follows the standard Kubernetes object structure, with a few Job-specific fields under `spec`. Here is a minimal example of a non-parallel Job that prints “Hello, World” and exits successfully:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  # Number of times to complete successfully before considering the Job done
  completions: 1           # default is 1 if omitted
  # Number of pods to run in parallel at any given time
  parallelism: 1           # default is 1 if omitted (non-parallel mode)
  # Template for the pod(s) to run
  template:
    metadata:
      name: hello-pod
    spec:
      restartPolicy: OnFailure
      containers:
      - name: hello
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          echo "Hello, World!"
```

### 2.1 `apiVersion` and `kind`

* **`apiVersion: batch/v1`**
  Jobs live under the `batch` API group, version `v1`. Older clusters (pre-v1.21) might use `batch/v1beta1`, but modern clusters always use `batch/v1`.
* **`kind: Job`**
  Denotes the object type.

### 2.2 `metadata`

* **`name`**: Unique within the namespace (e.g., `hello-job`).
* **`namespace`**: If omitted, defaults to the current namespace in your kubeconfig context.

### 2.3 `spec`

The `spec` field contains all job configuration:

1. **`completions`** *(integer)*

   * How many pods must terminate successfully (exit code 0) before the Job is considered complete.
   * Defaults to `1` if omitted (i.e., a single-run job).
   * In Indexed mode, Kubernetes creates exactly `completions` pods, each with a distinct index.

2. **`parallelism`** *(integer)*

   * Maximum number of pods running in parallel.
   * Defaults to `1`.
   * If `parallelism < completions`, pods run in batches: once a pod completes, Kubernetes spins up another until the total number of successful completions equals `completions`.

3. **`completions` vs. `parallelism`**

   * **Non-parallel mode**: `parallelism: 1`, `completions: 1` (the default). Kubernetes creates one pod, waits for it to succeed, then marks the Job Completed.
   * **Parallel non-indexed mode**: `completions: 10`, `parallelism: 3`. Kubernetes creates 3 pods concurrently. Whenever any single pod terminates successfully, a replacement is created, until a total of 10 successes occur.

4. **`completionMode`** *(string, “NonIndexed” or “Indexed”)*

   * **`NonIndexed`** (default): Pods are fungible workers; Kubernetes does not guarantee a unique index. Each time a pod completes successfully, the Job controller increments a count and, if it is still below `completions`, creates a new pod.
   * **`Indexed`**: Introduced in Kubernetes v1.21. Kubernetes creates exactly `completions` pods up-front, each assigned an index from `0` to `completions−1`. Each pod gets an environment variable `JOB_COMPLETION_INDEX` set to its index. Your application can then use that index to pick a distinct chunk of work.

5. **`activeDeadlineSeconds`** *(integer)*

   * An overall timeout for the Job, in seconds. If the Job does not finish (i.e., achieve `completions` successful pods) before this timeout, the Job is marked **Failed**, and any running pods are terminated.

6. **`backoffLimit`** *(integer)*

   * How many times Kubernetes retries a failing pod before marking the whole Job as **Failed**.
   * Default is `6`. If your application might transiently fail (e.g., due to network hiccups), increase this value; if you want quick failure detection, lower it.

7. **`ttlSecondsAfterFinished`** *(integer, v1.21+)*

   * Once the Job enters a terminal state (Succeeded or Failed), the TTL controller waits this many seconds, then automatically deletes the Job (and its Pods). This prevents accumulation of old Jobs.

8. **`template`**

   * A **PodTemplateSpec** exactly like you’d see under a Deployment or ReplicaSet.
   * Must include:

      * **`metadata.labels`** (optional but recommended).
      * **`spec.containers`**: At least one container must be defined.
      * **`spec.restartPolicy`**: Must be either **`OnFailure`** or **`Never`** (default is `OnFailure`). You cannot use `Always`—Jobs rely on Pod restart policies to determine success/failure.

---

## 3. Job Lifecycle and Status

Kubernetes tracks a Job’s progress via its **Status subresource**, which updates over time:

1. **Pods Pending**

   * Once you create a Job, its status initially shows 0 active and 0 succeeded pods. The scheduler tries to place the first pod(s). They enter `Pending` until images download and scheduling is completed.

2. **Active Pods**

   * The Job’s status `.status.active` increments to reflect the number of pods currently running or in the “Terminating” phase with `restartPolicy: OnFailure`.

3. **Succeeded Pods**

   * Each time a pod exits with exit code 0, it is counted toward `.status.succeeded`.
   * If `.status.succeeded < spec.completions`, and `.status.active < spec.parallelism`, the controller spawns new pods.

4. **Failed Pods**

   * Each time a pod exits with a non-zero code, `.status.failed` increases. If `.status.failed > spec.backoffLimit`, the Job’s `.status.conditions` receives a **Failed** type and the Job is no longer active.

5. **Completion**

   * When `.status.succeeded == spec.completions`, the Job’s `.status.conditions` receives a **Complete** type, and the Job terminates—no new pods are created.

You can inspect status via:

```bash
kubectl describe job <job-name>
# Status:
#  Active: 2
#  Succeeded: 5
#  Failed: 0
#
# Conditions:
#  Type           Status  LastProbeTime        LastTransitionTime   Reason
#  ----           ------  --------------        ------------------   ------
#  Complete       True    2023-06-01T12:01:05Z  2023-06-01T12:01:05Z  All pods succeeded
```

---

## 4. Parallel Job Patterns

### 4.1 Non-Parallel (Sequential) Job

By default, a Job runs a single Pod to completion:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sequential-job
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: task
        image: alpine:3.18
        command: ["sh", "-c", "echo 'Task complete'; exit 0"]
```

* Kubernetes creates one Pod (`sequential-job-xxxxx`).
* When it exits 0, the Job is marked **Complete**.

### 4.2 Parallel Non-Indexed (Work-Queue) Job

Use this pattern when you have many similar tasks that can run in parallel but do not require a unique index. For example, processing messages from a queue or partitioned data:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 10          # Total desired successes
  parallelism: 4           # Run up to 4 pods at once
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: myregistry/worker:latest
        env:
        - name: QUEUE_NAME
          value: "tasks-queue"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

Behavior:

1. Kubernetes starts 4 pods (`parallel-job-aaaaa`, `parallel-job-bbbbb`, etc.).
2. Each pod fetches a task from the shared queue (`tasks-queue`).
3. When any pod succeeds (exit code 0), `.status.succeeded` increments, and if fewer than 10 successes have occurred, Kubernetes launches another pod to maintain up to 4 active.
4. Continues until 10 successful completions.
5. If a pod fails, Kubernetes retries it (up to `backoffLimit` times) before marking the Job as failed if the limit is exceeded.

### 4.3 Parallel Indexed Job (Deterministic Sharding)

In **Indexed** mode (Kubernetes v1.21+), each pod receives a unique index—`JOB_COMPLETION_INDEX`—to coordinate which slice of work to process:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-job
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: shard-worker
        image: myregistry/shard-worker:latest
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

* Kubernetes immediately creates 5 Pods, each annotated with `batch.kubernetes.io/job-completion-index` values `0` through `4`.
* Each worker pod reads its index and processes, for instance, file chunk #0, #1, etc.
* No new pods are created once the 5 pods exist, but if any of these pods fails, Kubernetes retries that exact index until it succeeds or hits `backoffLimit`.
* Use this for deterministic partitioning (e.g., process N data shards in parallel without overlap).

---

## 5. Managing Jobs via `kubectl`

### 5.1 Creating and Viewing Jobs

* **Create from YAML**:

  ```bash
  kubectl apply -f job.yaml
  ```

* **List Jobs**:

  ```bash
  kubectl get jobs
  ```

  Shows:

  ```
  NAME            COMPLETIONS   DURATION   AGE
  hello-job       1/1           5s         1m
  parallel-job    10/10         2m         5m
  ```

* **Describe a Job**:

  ```bash
  kubectl describe job parallel-job
  ```

  Outputs detailed status, including active pods, succeeded/failed counts, events, etc.

* **Get Pod Logs**:

  ```bash
  kubectl logs job/hello-job
  ```

  Equivalent to `kubectl logs <pod-name>` if there is only one pod. For parallel jobs, specify the pod explicitly:

  ```bash
  kubectl get pods -l job-name=parallel-job
  # NAME                     READY   STATUS      RESTARTS   AGE
  # parallel-job-abc123-1    0/1     Completed   0          3m
  # parallel-job-abc123-2    0/1     Completed   0          3m
  kubectl logs parallel-job-abc123-1
  ```

### 5.2 Scaling and Updating Jobs

* **Editing Completions or Parallelism**:
  Jobs are generally immutable once created. If you modify `completions` or `parallelism` and re-apply, Kubernetes rejects the change. Instead, you must delete and recreate the Job with new values.

* **Active Deadline or BackoffLimit**:
  You can update `activeDeadlineSeconds` and `backoffLimit` via `kubectl patch`:

  ```bash
  kubectl patch job parallel-job -p '{"spec":{"backoffLimit":10}}'
  ```

  However, changes only affect newly created pods—not those already running.

### 5.3 Deleting Jobs and Pods

* **Delete a Job** (cascade deletes pods by default):

  ```bash
  kubectl delete job parallel-job
  ```

  This removes the Job object and all its pods.
* **Retain Pods**: If you want to keep pods around for debugging after a failure, you can manually remove the owner reference:

  ```bash
  kubectl delete job parallel-job --cascade=false
  ```

  Now, the Job object disappears but the pods continue running or stay in their final state.

---

## 6. Best Practices and Tips

1. **Choose the Right Completion Mode**

   * If your tasks are truly identical (e.g., consumer workers draining a queue), use **NonIndexed** (default).
   * If you need deterministic sharding (e.g., processing fixed partitions of a dataset), use **Indexed** mode so each pod knows which slice to handle.

2. **Idempotency and Work Distribution**

   * Ensure your application logic is **idempotent**, especially in non-indexed mode, since retries on failure might cause the same work to be attempted multiple times.
   * For non-indexed parallel jobs, use a centralized queue (e.g., Redis, RabbitMQ) or database row-locking mechanism to avoid duplicate work.

3. **Tune `backoffLimit` and `activeDeadlineSeconds`**

   * If a failing pod should be retried more times, increase `backoffLimit`. Otherwise, reduce it to fail fast if you prefer manual inspection.
   * Use `activeDeadlineSeconds` to put an upper bound on how long a Job should run. For example, if nightly backups shouldn’t run beyond two hours, set `activeDeadlineSeconds: 7200`.

4. **Clean Up Completed Jobs**

   * By default, completed Jobs and their pods remain in the cluster indefinitely, cluttering your namespace. Enable **TTL** cleanup:

     ```yaml
     spec:
       ttlSecondsAfterFinished: 3600  # Delete 1 hour after completion
     ```
   * Alternatively, you can run a periodic script or GitOps pipeline that prunes old Jobs older than a threshold.

5. **Monitor Job Status**

   * Use `kubectl get jobs --watch` to track active jobs.
   * Add proper **liveness** and **readiness probes** in the pod template if the task can hang; otherwise, the container may appear Running indefinitely.
   * Implement application-level logging and metrics (e.g., push job progress to Prometheus).

6. **Resource Requests and Limits**

   * Always specify CPU and memory **requests** so the scheduler can place pods appropriately.
   * For resource-intensive batch jobs, set sensible **limits** to prevent runaway resource usage.

7. **Node Selection and Affinity**

   * If certain batch jobs require GPU or high-memory nodes, add a **nodeSelector** or **nodeAffinity** under `template.spec`.
   * Example:

     ```yaml
     spec:
       template:
         spec:
           nodeSelector:
             accelerator: nvidia-gpu
     ```

8. **Security Contexts**

   * Run Jobs with the least privileges necessary. If a pod needs to write to a shared volume, use a specific **`serviceAccountName`** or mount credentials via a **Secret**.
   * If you only need read-only access, set containers to run as non-root with a restricted security context:

     ```yaml
     securityContext:
       runAsUser: 1000
       runAsGroup: 1000
       fsGroup: 2000
     ```

---

## 7. Job vs. Other Controllers

| Controller     | Purpose                                                  | Restart Policy     | Use Case                                                             |
| -------------- | -------------------------------------------------------- | ------------------ | -------------------------------------------------------------------- |
| **Pod**        | Run one or more containers (long-running services)       | Always             | Stateless services, daemons, anything persistent.                    |
| **Deployment** | Run stateless services with rolling updates and rollback | Always             | Web servers, APIs, microservices that need zero-downtime updates.    |
| **ReplicaSet** | Ensure *N* replicas of a Pod run at all times (legacy)   | Always             | Low-level replication; typically managed by Deployments.             |
| **DaemonSet**  | Run a pod on every (or selected) node                    | Always             | Node-level services (logging, monitoring, network plugins).          |
| **Job**        | Run pods *to completion* a specified number of times     | OnFailure or Never | Batch tasks, data processing, one-off scripts, CronJobs.             |
| **CronJob**    | Schedule Jobs on a time-based schedule (“cron” syntax)   | OnFailure or Never | Nightly backups, periodic data pulls, scheduled maintenance scripts. |

* Unlike Deployments, ReplicaSets, or DaemonSets, which target long-lived services, a **Job** is a **finite** workload that completes once its pods succeed.
* If you need recurring execution (e.g., daily backups), wrap your Job in a **CronJob**; CronJobs simply create Jobs on a schedule.

---

## 8. Example Scenarios

### 8.1 Database Migration Job

You deploy a Job that runs database schema migrations before upgrading your application:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: myregistry/db-migrator:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        command:
        - /app/migrate.sh
  backoffLimit: 2
  activeDeadlineSeconds: 600  # Fail if not done within 10 minutes
```

* If the migration script fails, Kubernetes retries up to two times.
* If migrations take longer than 10 minutes, the Job is marked Failed, so you can investigate before rolling out the new version.

### 8.2 Parallel MapReduce-Style Batch Job

Suppose you want to process 1000 data files in parallel, with groups of 50 processed at once:

```bash
# Create a ConfigMap that lists all file paths, one per line
kubectl create configmap datafiles --from-file=files.txt
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mapreduce-job
spec:
  completions: 1000
  parallelism: 50
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mapper
        image: myregistry/mapper:latest
        command:
        - /app/process-file.sh
        - "--file=$(FILE_PATH)"
        env:
        - name: FILE_PATH
          valueFrom:
            configMapKeyRef:
              name: datafiles
              key: files.txt
      volumes:
      - name: datafiles
        configMap:
          name: datafiles
  backoffLimit: 4
```

* Kubernetes launches 50 pods immediately, each reading a different line from `files.txt` (you’d build application logic to pick an unprocessed line atomically).
* When any pod finishes, a replacement starts, until 1000 successful runs occur.

### 8.3 CronJob Example (Schedules a Job)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"  # Run at 2:00 AM every day
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: myregistry/backup-tool:latest
            args:
            - "--backup-database"
            - "--destination=s3://backups/db-$(date +%Y-%m-%d).tar.gz"
```

* The CronJob creates a new Job at 2:00 AM each day.
* That Job runs the backup container to completion.
* You can add `ttlSecondsAfterFinished: 86400` under `jobTemplate.spec` to clean up Jobs 24 hours after they finish.

---

## 9. Troubleshooting and Debugging

1. **Pod Not Starting**

   * `kubectl describe job <job-name>` shows events (e.g., insufficient resources, node affinity mismatches).
   * Check `kubectl get pods -l job-name=<job-name>` to see pod statuses.

2. **Pod Failing Repeatedly**

   * Look at `kubectl logs <pod-name>` to inspect the container’s exit reason.
   * If a pod fails more than `backoffLimit`, the Job is marked Failed.

3. **Job Stuck in Active State**

   * If pods are `Pending` for a long time, the cluster might lack resources or no nodes match a nodeSelector or affinity rule.
   * Use `kubectl get nodes --show-labels` to verify labels, or `kubectl describe pod <pod>` to see unsatisfied scheduling constraints.

4. **Unexpected Multiple Pods**

   * If you see more pods than `parallelism`, ensure you haven’t created new Jobs manually or forgot to clean up old ones.
   * Verify you didn’t mistakenly set `.spec.parallelism` higher than intended.

5. **Jobs Accumulating in Namespace**

   * Set `ttlSecondsAfterFinished` on Job specs or configure a CronJob’s `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`.
   * Alternatively, run a garbage collection script:

     ```bash
     kubectl delete jobs --field-selector status.successful>0,metadata.creationTimestamp<=$(date -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)
     ```

## CronJob

A **CronJob** in Kubernetes is the equivalent of a Unix `cron` entry: it creates one‐time **Jobs** on a repeating schedule. CronJobs are useful for running periodic tasks—backups, report generation, log rotation, cleanup jobs, or any one‐off batch processing that must run on a schedule. ([Kubernetes][1])

---

## 1. What Is a CronJob?

* A CronJob controller watches its own custom resource named `CronJob`, and at specified times it creates a new **Job** object.
* That Job, in turn, creates one or more Pods that run to completion (as described in the Job documentation).
* When the Job finishes successfully (i.e., meets its `.spec.completions`), no further Pods are created until the next CronJob “tick.”
* Conceptually, **one CronJob object is like one line of a crontab** on Unix—which runs a command or script at scheduled intervals. ([Kubernetes][1])

---

## 2. Anatomy of a CronJob Manifest

A CronJob YAML manifest follows Kubernetes standards (similar to a Job or Deployment) but adds scheduling fields. Here is a minimal example:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/5 * * * *"   # Every 5 minutes in standard cron format
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.35
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from Kubernetes CronJob"
```

### Key Fields

1. **`apiVersion: batch/v1`**

   * As of Kubernetes v1.21+, CronJobs are GA under `batch/v1`. Older clusters might use `batch/v1beta1` or `batch/v2alpha1`. Always verify your cluster version. ([Stack Overflow][2])

2. **`kind: CronJob`**

   * Identifies this resource as a CronJob object.

3. **`metadata.name`**

   * The CronJob’s name. Must be a valid DNS subdomain (up to 52 characters, since Kubernetes appends an 11-character suffix when naming created Jobs). ([Kubernetes][1])

4. **`spec.schedule`** (string)

   * A standard crontab‐style schedule (five fields: minute, hour, day of month, month, day of week) ■.
   * Example: `"0 2 * * *"` means “run daily at 02:00.”
   * Wildcards (`*`), lists (`1,15`), ranges (`1-5`), and step values (`*/10`) are all supported (as in a typical Linux cron).
   * For timezone support, prefix the schedule string with `CRON_TZ=<TZ>` (e.g., `"CRON_TZ=UTC 0 23 * * *"`). ([Stack Overflow][2])

5. **`spec.jobTemplate`**

   * This is a **JobTemplateSpec**: it contains a `.spec` block identical to a standard Job’s spec, except `metadata` under `template` cannot include `generateName`, `ownerReferences`, or `clusterName`.
   * In practice, you provide `jobTemplate.spec.template.spec.containers` and `restartPolicy` (must be `OnFailure` or `Never`).

6. **`spec.concurrencyPolicy`** *(optional; string)*

   * Controls what happens if a scheduled time arrives and the previous Job is still running.

      * `Allow` (default): Allows CronJobs to run concurrently; new Job is created even if the old one hasn’t finished.
      * `Forbid`: Skips creating a new Job if the previous Job is still running.
      * `Replace`: Terminates the currently running Job (by deleting its Pods) and immediately creates a new Job. ([Kubernetes][1])

7. **`spec.suspend`** *(optional; boolean)*

   * When set to `true`, the CronJob controller does not create new Jobs (effectively pausing the schedule). Does not delete already-running Jobs or Pods. Default is `false`.

8. **`spec.startingDeadlineSeconds`** *(optional; integer)*

   * An optional deadline in seconds for starting a Job if it misses its scheduled time for any reason (e.g., the controller was offline, or there was a backlog).
   * If a scheduled time is more than `startingDeadlineSeconds` in the past, that execution is skipped.
   * If omitted, missed executions are not re‐queued once the schedule is missed. ([Kubernetes][1])

9. **`spec.failedJobsHistoryLimit`** and **`spec.successfulJobsHistoryLimit`** *(optional; integer)*

   * `failedJobsHistoryLimit`: How many failed Job objects to keep in the cluster. Defaults to `1`.
   * `successfulJobsHistoryLimit`: How many successful Job objects to retain. Defaults to `3`.
   * Older Job objects beyond those counts are automatically deleted. ([Kubernetes][1])

---

## 3. How the CronJob Controller Works

1. **Reconciling Schedules**

   * The CronJob controller maintains an internal schedule of <NextScheduleTime> and continually checks the clock.
   * When the current time ≥ next scheduled time, the controller attempts to create a new Job.
   * It honors `startingDeadlineSeconds`, so if that scheduled moment is now too far in the past (beyond the deadline), the controller skips that run.

2. **Concurrency Handling**

   * If `concurrencyPolicy=Allow`, the controller unconditionally creates a new Job, even if older Jobs remain active (e.g., long-running backups).
   * If `concurrencyPolicy=Forbid`, the controller checks existing Jobs spawned by this CronJob (look at `.status.active`). If any are still active, it skips creating a new Job until that active Job finishes.
   * If `concurrencyPolicy=Replace`, the controller immediately deletes any currently running Jobs (and their Pods) spawned by this CronJob and creates a fresh Job for the new schedule.

3. **Job Naming and Ownership**

   * Every Job created by a CronJob receives an autogenerated name of the form:

     ```
     <cronjob-name>-<time-suffix>
     ```

     where `<time-suffix>` is an 11-character value based on the scheduled timestamp.
   * The Job’s `metadata.ownerReferences` points back to the CronJob, so if you delete the CronJob (and the ownerReference policy is `Foreground`), Kubernetes will garbage‐collect all associated Jobs.

4. **History Limits and Cleanup**

   * After each Job finishes (either Succeeded or Failed), the controller checks how many “old” Jobs of each type (successful vs failed) remain.
   * If the number exceeds the respective history limit, the oldest Jobs are deleted (along with their Pods).
   * This prevents unbounded accumulation of completed or failed Job objects.

5. **Suspend/Resume**

   * Setting `spec.suspend: true` causes the controller to skip future schedules.
   * Toggling `suspend` back to `false` resumes normal scheduling at the next cron‐calculated time.

---

## 4. Cron Schedule Syntax and Timezones

* **Standard Cron Format**
  A CronJob’s schedule is five space‐separated fields:

  ```
  minute hour day-of-month month day-of-week
  ```

  Examples:

   * `"0 2 * * *"` → once daily at 02:00.
   * `"0 0 * * 0"` → once weekly, Sunday at midnight.
   * `"*/15 * * * *"` → every 15 minutes.

* **Step Values and Ranges**

   * Ranges: `"0-30/10 * * * *"` → at minutes 0, 10, 20, 30.
   * Lists: `"0 8,16 * * *"` → at 08:00 and 16:00 daily.

* **Specifying Timezones**

   * Prefix the entire schedule string with `CRON_TZ=<TZ>`.

      * Example:

        ```yaml
        spec:
          schedule: "CRON_TZ=UTC 0 23 * * *"
        ```

        Schedules the job to run daily at **23:00 UTC**, regardless of the cluster’s local timezone. ([Stack Overflow][2])
   * Live clusters must support `CRON_TZ`, which generally works on Kubernetes v1.21+ under `batch/v1` (GA).
   * If you do not specify `CRON_TZ`, the cluster’s default system timezone (usually UTC) is used.

---

## 5. Common Fields and Their Effects

| Field                             | Purpose                                                                                          |
| --------------------------------- | ------------------------------------------------------------------------------------------------ |
| `spec.schedule`                   | Cron expression (or `CRON_TZ=<TZ> …`) for when to run the Job.                                   |
| `spec.startingDeadlineSeconds`    | If a schedule is missed by more than this many seconds, skip that run.                           |
| `spec.concurrencyPolicy`          | `Allow` / `Forbid` / `Replace` behavior when previous Jobs have not finished.                    |
| `spec.suspend`                    | Set to `true` to pause scheduling; `false` to resume.                                            |
| `spec.failedJobsHistoryLimit`     | How many failed Job objects to keep (default: 1).                                                |
| `spec.successfulJobsHistoryLimit` | How many successful Job objects to keep (default: 3).                                            |
| `spec.jobTemplate`                | The Pod‐template spec to use when creating each Job (mirrors a standard Job’s `.spec.template`). |

---

## 6. Example Workflows

### 6.1 Simple CronJob: Run Every Minute

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-every-minute
spec:
  schedule: "*/1 * * * *"   # Every minute
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - date; echo "Hello at $(date)"
          restartPolicy: OnFailure
```

**Apply and Observe:**

```bash
kubectl apply -f cronjob-minute.yaml
kubectl get cronjob hello-every-minute
# NAME                   SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# hello-every-minute     */1 * * * *   False     0        <none>          1m

# After one minute:
kubectl get jobs
# NAME                               COMPLETIONS   DURATION   AGE
# hello-every-minute-1696128000      0/1           20s        20s
# hello-every-minute-1696128060      0/1           15s        15s
# ...
```

Each minute, a new Job named `hello-every-minute‐<timestamp>` is created. Once the Pod in that Job completes successfully, the Job’s `.status.succeeded` flips to 1, and the Job is eventually garbage‐collected according to the history limits. ([Kubernetes][3])

### 6.2 CronJob with ConcurrencyPolicy: Forbid

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cron
spec:
  schedule: "0 2 * * *"            # Daily at 02:00
  concurrencyPolicy: Forbid        # Skip this run if previous Job is still running
  startingDeadlineSeconds: 3600    # If we miss 02:00 by > 1 hour, skip that run
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: daily-backup
            image: myregistry/backup-tool:1.0
            env:
            - name: BACKUP_DEST
              value: "s3://mybucket/db-backups"
            args:
            - "--backup-database"
          restartPolicy: OnFailure
```

* If a Monday backup runs long (e.g., 3 hours), it will still be running at Tuesday 02:00.
* Because `concurrencyPolicy: Forbid`, the Tuesday 02:00 run is skipped.
* If the controller is down at 02:00, then restarts at 03:30, that missed 02:00 run is skipped because `startingDeadlineSeconds: 3600` (any schedule >1 hour ago is considered dead).

Old Jobs beyond the history limits (5 successful, 2 failed) are automatically cleaned up to avoid clutter. ([Kubernetes][1], [Kubernetes][1])

### 6.3 CronJob with Replace Mode

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: stats-cron
spec:
  schedule: "*/10 * * * *"      # Every 10 minutes
  concurrencyPolicy: Replace     # If a previous job is still running, kill it and start a new one
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: compute-stats
            image: myregistry/stats-generator:2.1
            command: ["/bin/bash", "-c", "/app/generate-stats.sh"]
          restartPolicy: OnFailure
```

* Suppose the stats job occasionally lags and overruns into the next 10-minute window.
* With `Replace`, when the next scheduled time arrives, the controller deletes any still-running `stats-cron-<timestamp>` Job’s Pods, then creates a new Job.
* This prevents overlapping executions but means partial results from the previous run might be truncated. Choose this only if truncation is acceptable. ([Kubernetes][1])

---

## 7. Best Practices for CronJobs

1. **Always Set `restartPolicy` to `OnFailure` or `Never`**

   * CronJobs depend on non‐`Always` policies to gauge Pod success/failure. Using `Always` would never mark the Job as complete. ([Kubernetes][1])

2. **Use `startingDeadlineSeconds` to Avoid Stale Runs**

   * Prevent “catch‐up storms” when your cluster is down or overloaded.
   * If a schedule is too far in the past, it is better to skip it than to run a dozen backlogged Jobs all at once.

3. **Tune `concurrencyPolicy` Based on Idempotency**

   * If your Job is idempotent (re‐running doesn’t harm), you may safely use `Allow`.
   * If you absolutely do not want overlapping runs, use `Forbid`.
   * If you need a fresh run each time—even if the last hasn’t finished—use `Replace`.

4. **Clean Up Old Jobs Automatically**

   * Set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` to reasonable values (e.g., 3 or 5). This prevents your namespace from being cluttered with old resources.

5. **Pause a CronJob without Deleting**

   * Setting `spec.suspend: true` lets you temporarily stop new Job creation while preserving the CronJob’s spec and history.

6. **Monitor CronJob Status**

   * Use `kubectl get cronjobs` to see:

     ```
     NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
     hello-cron       */5 * * * *   False     0        2025-05-31T10:00   10m
     ```
   * Check `.status.lastScheduleTime` in `kubectl describe cronjob <name>`.
   * Inspect each spawned Job and its Pods to verify successful or failed runs.

7. **Consider Pod Security and Resource Quotas**

   * Even though CronJobs spawn short-lived workloads, they still consume resources (CPU, memory, ephemeral storage) when running.
   * Declare `resources.requests`/`limits` on your CronJob’s Pod containers so the scheduler can place them appropriately without starving other workloads.

8. **Time Zone Awareness**

   * If your organization runs critical tasks at local business hours, use `CRON_TZ=<Zone>` in `schedule`.
   * Otherwise, by default, schedules are interpreted in UTC.

---

## 8. Troubleshooting Common Issues

1. **CronJob Not Firing at Expected Time**

   * Ensure the CronJob controller is running (usually part of the control‐plane).
   * Check that `schedule` syntax is valid: `kubectl describe cronjob <name>` will report parsing errors.
   * Verify that there is no `suspend: true` flag set inadvertently.

2. **Jobs Piling Up (Too Many Active Jobs)**

   * If `concurrencyPolicy: Allow` and your Jobs take longer than the scheduling interval, multiple Jobs will accumulate.
   * Solution: switch to `Forbid` or `Replace`, or adjust the schedule so each run has ample time to finish.

3. **Unexpected Job Failures**

   * Use `kubectl get pods -l job-name=<job>` and `kubectl logs <pod>` to inspect container exit codes and error messages.
   * If the Job fails early, check your pod’s command, environment variables, Volume mounts, or Secrets.

4. **Jobs Skipped due to Starting Deadline**

   * If you see `Last Schedule` updating but no new Job appears, your cluster might be under heavy load, causing the controller to miss the exact clock tick.
   * If `startingDeadlineSeconds` is small, the controller will skip creating the Job because the current time is already beyond `<scheduled_time> + startingDeadlineSeconds`.
   * Either increase `startingDeadlineSeconds` or ensure the cluster has enough capacity for timely scheduling.

5. **CronJob Name Too Long**

   * Remember that Kubernetes appends an 11-character suffix to CronJob names when naming each spawned Job.
   * Keep the CronJob’s `metadata.name` under 52 characters to avoid tripping the 63-character limit. ([Kubernetes][1])

---

## 9. Advanced Patterns

### 9.1 Using a ConfigMap or Secret with a CronJob

You can mount a ConfigMap or Secret directly into the CronJob’s Pod template:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: report-cron
spec:
  schedule: "0 4 * * *"  # Daily at 04:00
  jobTemplate:
    spec:
      template:
        spec:
          volumes:
          - name: report-config
            configMap:
              name: report-configmap
          containers:
          - name: report-generator
            image: myregistry/reportgen:latest
            volumeMounts:
            - name: report-config
              mountPath: /etc/report
            command: ["/bin/sh", "-c", "/app/generate.sh --config /etc/report/config.yaml"]
          restartPolicy: OnFailure
```

* Here, the container reads its configuration from the mounted ConfigMap. Running the CronJob daily picks up any ConfigMap changes automatically (no need to reapply the CronJob). ([Kubernetes][3])

### 9.2 Using `mytimezone.sh` Wrapper for More Complex Schedules

If you need more dynamic schedules—e.g., only run on the last day of each month—it’s sometimes easier to schedule a placeholder CronJob that runs once per day at a fixed time and then have the container’s entrypoint check the date:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: last-day-report
spec:
  schedule: "0 0 * * *"  # Every midnight
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: daily-check
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - |
              TODAY=$(date +%d)
              DAYS=$(date -d "$(date +%Y-%m-01) +1 month -1 day" +%d)
              if [ "$TODAY" -eq "$DAYS" ]; then
                /app/run-month-end.sh
              else
                echo "Not last day: $TODAY/$DAYS; exiting."
                exit 0
              fi
          restartPolicy: OnFailure
```

* This CronJob runs daily, but `/app/run-month-end.sh` only triggers when the current day equals the last day of the month.
* It’s a useful trick when pure cron syntax cannot express “last day of month.” ([Kubernetes][1])

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/?utm_source=chatgpt.com "CronJob - Kubernetes"
[2]: https://stackoverflow.com/questions/68950893/how-can-i-specify-cron-timezone-in-k8s-cron-job?utm_source=chatgpt.com "How can I specify cron timezone in k8s cron job? - Stack Overflow"
[3]: https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/?utm_source=chatgpt.com "Running Automated Tasks with a CronJob - Kubernetes"

## StatefulSets

A **StatefulSet** is the workload API object in Kubernetes designed to manage stateful applications—those that require stable, unique identities and persistent storage. Unlike Deployments (which treat pods as interchangeable), StatefulSets provide guarantees about pod ordering, naming, and storage persistence. This article explores StatefulSets in depth: their core concepts, manifest structure, lifecycle, use cases, comparisons to other controllers, and best practices.

---

## 1. Why Use a StatefulSet?

### 1.1 Stateful vs. Stateless Workloads

* **Stateless Workloads** (e.g., web servers, front-end services) typically use a Deployment. Any pod can serve any request interchangeably; they have no need for unique identities or persistent volumes.
* **Stateful Workloads** (e.g., databases, caches, message brokers) require each pod to maintain its own data and to have a stable network identity so that clients or peers can reliably refer to "pod A," "pod B," etc. Deleting and re-creating these pods must not disrupt the service contract.

A **StatefulSet** addresses these needs by ensuring:

1. **Stable, unique network identities**: Each pod gets a predictable hostname (e.g., `mysql-0`, `mysql-1`, `mysql-2`) and DNS entry.
2. **Stable, persistent storage**: Each pod can have a dedicated PersistentVolumeClaim (PVC) that is not deleted or reused when the pod is rescheduled.
3. **Ordered, graceful pod creation and deletion**: Pods start in numerical order (`0`, then `1`, etc.) and terminate in reverse order, preserving dependencies (e.g., a primary database must start before replicas). ([Kubernetes][1])

### 1.2 Core Guarantees of a StatefulSet

* **Ordinal Index**: Each pod in a StatefulSet gets an ordinal index in its name (e.g., `<statefulset-name>-0`, `<statefulset-name>-1`).
* **Ordered Creation**: Pods are created sequentially in ordinal order: `pod-0` → `pod-1` → ... → `pod-N`, ensuring that any initialization logic or cluster membership can proceed in a known sequence.
* **Stable Network Identity**: Kubernetes automatically creates a **Headless Service** (no cluster IP) exposing DNS entries like `pod-0.<service-name>.<namespace>.svc.cluster.local`. Even if a pod is rescheduled to a different node, its DNS entry remains the same.
* **Persistent Storage**: With `volumeClaimTemplates`, each pod gets its own PVC named `<volume-claim-template-name>-<statefulset-name>-<ordinal>` (e.g., `data-mysql-0`). These PVCs outlive pod restarts and remain bound to the same underlying PersistentVolume (PV).
* **Ordered Deletion/Scaling-Down**: On scale-down, pods terminate in reverse ordinal order: highest-indexed pod first. This ensures dependent pods (e.g., replicas) are removed before the primary.
* **Controlled Updates**: By default, StatefulSets use a rolling update strategy that updates pods one at a time, maintaining the above order and waiting for readiness before proceeding. ([Kubernetes][1])

---

## 2. Anatomy of a StatefulSet Manifest

A StatefulSet manifest looks similar to other controllers but has a few additional required and specialized fields. Below is a minimal example for a 3-node MySQL cluster:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  serviceName: "mysql-headless"   # Must match a Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql                 # Must match labels in pod template
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data                 # Used to generate PVC names: data-mysql-0, data-mysql-1, etc.
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 2.1 Key Fields Explained

1. **`apiVersion: apps/v1`** and **`kind: StatefulSet`**

   * Modern Kubernetes clusters use `apps/v1` for StatefulSets.

2. **`metadata.name`**

   * The name of the StatefulSet (e.g., `mysql`). Pods will be named `mysql-0`, `mysql-1`, etc.

3. **`spec.serviceName`**

   * **Required**: Must refer to a **Headless Service** object (i.e., a Service with `clusterIP: None`) that governs the network domain for pods. The Headless Service creates DNS records like `mysql-0.mysql-headless.default.svc.cluster.local`.

4. **`spec.replicas`**

   * Desired number of pod replicas (e.g., `3`).

5. **`spec.selector.matchLabels`**

   * A label selector identifying the pods managed by this StatefulSet. Must match exactly the labels in `spec.template.metadata.labels`.

6. **`spec.template`**

   * A standard **PodTemplateSpec**:

      * **`metadata.labels`**: Must include `app: mysql`, matching the selector.
      * **`spec.containers`**: Defines the container image, ports, environment variables, and volume mounts.
      * **`volumeMounts`**: Mounts a volume named `data` into `/var/lib/mysql` (the MySQL data directory).

7. **`volumeClaimTemplates`**

   * **A list of PVC templates**. Each entry defines:

      * **`metadata.name`**: The template’s name (`data`).
      * **`spec`**: The PVC spec, including access modes (e.g., `ReadWriteOnce`) and resource requests (e.g., `storage: 10Gi`).
   * Kubernetes creates one PVC per pod, named `<metadata.name>-<statefulset-name>-<ordinal>` (e.g., `data-mysql-0`). These PVCs are bound to PVs dynamically (if using a StorageClass) or pre-provisioned manually. ([Kubernetes][1])

8. **(Optional) `updateStrategy`**

   * Controls how pod template changes get propagated. Defaults to `RollingUpdate`. See Section 5 for details.

9. **(Optional) `podManagementPolicy`**

   * Can be `OrderedReady` (default) or `Parallel`.

      * **`OrderedReady`**: Ensures pods are created, updated, or deleted strictly in ordinal order.
      * **`Parallel`**: Allows pods to be acted upon (create/delete) in parallel; useful for faster scale-up/scale-down when strict ordering isn’t required.

---

## 3. Headless Service and Network Identity

### 3.1 Why a Headless Service?

A StatefulSet requires a **Headless Service** so that Kubernetes does **not** assign a cluster IP but instead creates DNS A records for each pod. Example manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  labels:
    app: mysql
spec:
  clusterIP: None              # Headless: no virtual IP is allocated
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

* **`clusterIP: None`** tells Kubernetes not to create a load‐balancing virtual IP.
* Instead, the DNS system creates A records:

   * `mysql-0.mysql-headless.default.svc.cluster.local` → IP of pod `mysql-0`
   * `mysql-1.mysql-headless.default.svc.cluster.local` → IP of pod `mysql-1`
   * And so on.

Clients (e.g., a MySQL replication manager) can address each pod via its unique DNS name, even if the pod is rescheduled to a different node (pod IP is stable until pod is deleted). ([Kubernetes][1])

### 3.2 StatefulSet Pod DNS Pattern

For a StatefulSet named `mysql` in the `default` namespace with a Headless Service `mysql-headless`:

```
<statefulset-name>-<ordinal>.<service-name>.<namespace>.svc.cluster.local
```

Examples:

* `mysql-0.mysql-headless.default.svc.cluster.local`
* `mysql-1.mysql-headless.default.svc.cluster.local`

---

## 4. Persistent Storage with `volumeClaimTemplates`

### 4.1 Dedicated PVC per Pod

* **Each pod** in a StatefulSet receives its own PVC, derived from the `volumeClaimTemplates` block.
* For our example above:

   * When `mysql-0` is created, Kubernetes creates a PVC named `data-mysql-0`.
   * When `mysql-1` is created, Kubernetes creates `data-mysql-1`.
   * And so on, up to `data-mysql-2` if `replicas: 3`.

Each PVC is bound to a PV (either dynamically via a StorageClass or manually pre‐provisioned). The pod mounts that PV at `/var/lib/mysql`, ensuring the data directory persists independent of pod restarts or node failures (so long as the PVC is bound).

### 4.2 PVC Lifecycle

* **Creation**: PVCs are created when their corresponding pods are created.
* **Reclaim Policy**: Typically, PVCs use a StorageClass with `persistentVolumeReclaimPolicy: Retain` (the default). Even if you delete a pod, the PVC—and underlying PV—remains, preserving data.
* **Deletion**: By default, deleting a StatefulSet **does not** delete its PVCs (unless you explicitly delete them). This prevents data loss. If you re-create a StatefulSet with the same name and `volumeClaimTemplates`, it can re-use the existing PVCs (and data).

### 4.3 Example: Pre-Provisioned PVs

If you pre-provision PVs manually, ensure that their names match the expected PVC naming scheme (or use a StorageClass that selects existing volumes by label). For dynamic provisioning, specify a `storageClassName` under the PVC template.

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    storageClassName: "fast-ssd"
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```

---

## 5. Pod Lifecycle and Update Strategy

### 5.1 Pod Creation Order (`OrderedReady` vs. `Parallel`)

By default, StatefulSets use **`podManagementPolicy: OrderedReady`**, meaning:

1. Create `mysql-0` → wait until it’s **Ready** (passing readiness probes).
2. Then create `mysql-1` → wait for **Ready**.
3. Continue until reaching the specified `replicas` (e.g., `mysql-2`).

If you set `podManagementPolicy: Parallel`, the StatefulSet controller may create or delete all pods in parallel without waiting for readiness, which can accelerate scale operations at the cost of weaker ordering guarantees.

### 5.2 Rolling Updates (`updateStrategy`)

By default, StatefulSets use `updateStrategy: RollingUpdate`. This ensures a safe, ordered rollout when you update the Pod template (e.g., changing the container image).

* **`spec.updateStrategy.rollingUpdate.partition`** (optional):

   * You can specify a **partition** (an integer) to control which ordinal index to start updating from.
   * For instance, if `partition: 1` in a 3-replica StatefulSet, Kubernetes updates pods `mysql-1` and `mysql-2` but leaves `mysql-0` on the old version. This is useful for canary-style rollouts.

#### RollingUpdate Behavior

1. Identify pods with indices ≥ `partition` that need a template change.
2. For each such pod in ascending ordinal order, execute:

   * Delete the pod (terminating it gracefully).
   * Wait for its replacement to reach **Ready**.
   * Proceed to the next ordinal.

Example snippet controlling partitioned updates:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 1
```

* With `replicas: 3` and `partition: 1`:

   * Kubernetes leaves `mysql-0` untouched.
   * Updates `mysql-1` first, waits for readiness.
   * Then updates `mysql-2`, waits for readiness.
* Only after all pods with ordinal ≥ `partition` are updated is the process considered complete. ([Kubernetes][1])

### 5.3 OnDelete Update Strategy

If you set `updateStrategy: OnDelete`, Kubernetes does **not** automatically replace pods after you change the template. Instead, you must manually delete each pod (`kubectl delete pod mysql-0` → wait for recreation), ensuring control over the exact rollout timing. This is useful for workloads where updates require special coordination (e.g., manual data migration steps).

```yaml
updateStrategy:
  type: OnDelete
```

---

## 6. Scaling and Deletion

### 6.1 Scaling Up

When you increase `spec.replicas` (e.g., from `3` to `5`):

1. Kubernetes creates `mysql-3` and waits for it to become **Ready**.
2. Then creates `mysql-4` and waits for readiness.

Pods and their associated PVCs are provisioned in increasing ordinal order, preserving the creation guarantees.

### 6.2 Scaling Down

When you decrease `spec.replicas` (e.g., from `5` to `2`):

1. Kubernetes deletes pods in decreasing ordinal order: `mysql-4` first, then `mysql-3`.
2. **By default**, PVCs are not deleted. Pods are terminated, but their PVCs (e.g., `data-mysql-3`, `data-mysql-4`) remain in `Released` state, preserving data unless cleaned up manually. This avoids data loss if you later scale up again—Kubernetes will rebind pods to existing PVCs if names match.

If you want PVCs deleted automatically upon scale-down, you must manually delete them or use a custom cleanup process. Kubernetes does not automatically delete PVCs for StatefulSets.

---

## 7. Common Use Cases for StatefulSets

### 7.1 Databases and Data Stores

* **MySQL/MariaDB, PostgreSQL**: Each replica requires its own persistent disk; clients need stable DNS names to connect to the primary or replicas.
* **Cassandra, MongoDB, Elasticsearch**: These distributed databases rely on cluster membership and stable network identities for gossip protocols.
* **Redis Master/Replica**: You can designate `redis-0` as master, and `redis-1`, `redis-2` as replicas; each has a persistent disk for data durability. ([Kubernetes][1])

### 7.2 Message Brokers and Coordinators

* **Kafka, Zookeeper**: Each broker or znode requires a persistent volume (for logs or data) and stable naming so peers can reach each other reliably.
* **Etcd Clusters**: Etcd uses ordinal identities to form stable clusters; StatefulSet ensures etcd-0, etcd-1, etcd-2 come up in order and maintain stable persistence.

### 7.3 Stateful Applications with Leader Election

* **Consul, Vault**: Leader nodes need stable identities; follower nodes need to know how to reach the leader.
* **Custom Stateful Workloads**: Any application that performs peer discovery or bootstraps hosts using a known hostname benefit from StatefulSet’s guarantees.

---

## 8. Comparison with Other Controllers

| Feature                       | Deployment                             | ReplicaSet                         | StatefulSet                                | DaemonSet                                 |
| ----------------------------- | -------------------------------------- | ---------------------------------- | ------------------------------------------ | ----------------------------------------- |
| **Pod Identity**              | Interchangeable; no stable ID          | Same as Deployment                 | Stable, unique identity (e.g., `name-0`)   | Pods per node; identity less important    |
| **Persistent Storage**        | PVCs can be used but no direct mapping | Same as Deployment                 | `volumeClaimTemplates`: one PVC per pod    | Typically no PV per Pod; ephemeral        |
| **Creation Order**            | All pods created in parallel           | Parallel                           | Sequential (`0 → 1 → …`)                   | Parallel (one per node as nodes join)     |
| **Deletion Order**            | All pods deleted in parallel           | Parallel                           | Reverse ordinal (`N → … → 1 → 0`)          | Parallel (one per node)                   |
| **Rolling Updates**           | Declarative rolling-update             | N/A (immutable RS; use Deployment) | Ordered rolling-update (one pod at a time) | RollingUpdate possible across nodes       |
| **Use Case**                  | Stateless web/services                 | Legacy; prefer Deployment          | Stateful systems (DBs, clusters)           | Node-level services (logging, monitoring) |
| **Headless Service Required** | No                                     | No                                 | Yes (for network identity)                 | No                                        |
| **Scaling Guarantees**        | Scale up/down in any order             | Same as Deployment                 | Ordered scale up/down                      | N/A (creates per node)                    |

* **Deployments**/ReplicaSets are for stateless, horizontally scalable workloads where pods are fungible.
* **StatefulSets** are for workloads that need stable identities and persistence.
* **DaemonSets** ensure one pod per node for node-specific services; not intended for replicating sets of pods by count. ([Kubernetes][1], [Kubernetes][2])

---

## 9. Example: Deploying a 3-Node Redis Cluster

Below is a more complete example of a Redis Sentinel cluster using a StatefulSet. The headless service and persistent volumes ensure reliable membership and data durability.

### 9.1 Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
      name: redis
    - port: 26379
      name: sentinel
```

### 9.2 StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  labels:
    app: redis
spec:
  serviceName: "redis"
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: redis
        image: redis:6.2
        ports:
        - containerPort: 6379
          name: redis
        - containerPort: 26379
          name: sentinel
        command:
        - sh
        - -c
        - |
          #!/bin/sh
          if [ "$(hostname)" = "redis-0" ]; then
            redis-server /usr/local/etc/redis/redis.conf --sentinel
          else
            redis-server /usr/local/etc/redis/redis.conf --slaveof redis-0.redis.default.svc.cluster.local 6379
            redis-server /usr/local/etc/redis/sentinel.conf --sentinel
          fi
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

**How It Works**:

1. **`redis-0`** starts as the primary (no `--slaveof`). It creates a data directory under `/data` on its PVC (`data-redis-0`).
2. **`redis-1`** and **`redis-2`** start in ordinal order. They run as replicas, joining `redis-0` via the DNS name `redis-0.redis.default.svc.cluster.local`.
3. **Sentinel** processes run alongside each Redis instance for high availability.
4. Each pod has its own PVC (`data-redis-0`, `data-redis-1`, `data-redis-2`), ensuring persistence across restarts. ([Kubernetes][3])

---

## 10. Best Practices and Considerations

### 10.1 Choose `podManagementPolicy` Wisely

* **`OrderedReady`** (default): Essential for workloads where strict ordering matters (e.g., leader election, sequential initialization).
* **`Parallel`**: Use when you need faster scale‐up/scale‐down and can tolerate pods coming up out of order (e.g., homogeneous replicas without dependencies).

### 10.2 Monitor PVC and PV Usage

* Ensure your StorageClass is configured for **dynamic provisioning** or that enough static PVs exist to satisfy all PVCs (`data-<statefulset>-<ordinal>`).
* If no PV is available, pods will remain in `Pending` state, blocking the StatefulSet progress.

### 10.3 Handle PersistentVolume Reclaim Policy

* By default, PVCs use a reclaim policy (e.g., `Retain`), so PVs remain after a pod or StatefulSet is deleted.
* If you want PVs to be deleted automatically, configure the StorageClass with `reclaimPolicy: Delete`. However, be cautious—this can lead to data loss if the StatefulSet is re-created expecting the same backing storage.

### 10.4 Use Readiness Probes for Correct Ordering

* A pod is considered **Ready** when its readiness probe passes. For MySQL or Redis, configure a readiness probe (e.g., attempt a TCP connection or run a simple command) so the next ordinal does not start prematurely.

Example readiness probe for MySQL:

```yaml
readinessProbe:
  exec:
    command:
    - /usr/bin/mysql
    - -uroot
    - -p$(MYSQL_ROOT_PASSWORD)
    - -e
    - "SELECT 1"
  initialDelaySeconds: 30
  periodSeconds: 10
```

### 10.5 Handle Service and Client Lookup

* Clients that wish to connect to the primary/leader (e.g., a database driver) can look up `mysql-0.mysql-headless.default.svc.cluster.local`.
* For peer-discovery, you can use a headless SRV record (e.g., `_mysql._tcp.mysql-headless.default.svc.cluster.local`) to get all ordinals’ IPs and decide which is the leader.

### 10.6 Clean Up Resources Carefully

* Deleting a StatefulSet does **not** delete its PVCs by default. If you want to remove associated volumes:

   1. Delete the StatefulSet (which deletes pods but keeps PVCs).
   2. Delete the PVCs manually (`kubectl delete pvc -l statefulset.kubernetes.io/pod-name=...`).
   3. If using dynamic provisioning with `reclaimPolicy: Delete`, deleting the PVC automatically deletes the PV.

### 10.7 Plan for Updates and Rollbacks

* Use `partition` in your `rollingUpdate` strategy to control canary or staggered rollouts.
* If you discover a faulty upgrade, you can roll back by editing the StatefulSet’s `.spec.template` back to the previous image and re-applying; Kubernetes will roll back pods in reverse ordinal order.

### 10.8 Scale with Caution

* Scaling a stateful application often requires reconfiguration in the application itself (e.g., telling a database to add a new replica). Automate this via init scripts or cluster managers where possible.
* For large-scale clusters, consider increasing `podManagementPolicy` to `Parallel` for faster scaling but be prepared to handle pods arriving out of order.

---

## 11. Troubleshooting Common Issues

1. **Pod Stuck in `Pending`**

   * Check if a PVC is unbound (`kubectl get pvc`).
   * Inspect the StorageClass and ensure PVs exist or dynamic provisioning is correctly configured.
   * Confirm the node has enough resources or matches any `nodeSelector`/`affinity` constraints.

2. **DNS Resolution Fails**

   * Verify the headless Service exists and that its `selector` matches `app: mysql`.
   * Use `nslookup mysql-0.mysql-headless.default.svc.cluster.local` inside another pod to confirm DNS entries.
   * Check for typos in `serviceName` or the Service’s `metadata.name`. ([Kubernetes][1])

3. **Unexpected Pod Deletions**

   * If you manually delete an ordinal pod (e.g., `mysql-1`), Kubernetes immediately recreates it (with the same PVC and identity).
   * If you want to remove a pod permanently, scale down the StatefulSet (e.g., `kubectl scale sts mysql --replicas=2`), which deletes `mysql-2`—not `mysql-1`—because it deletes in reverse ordinal order.

4. **PVC Name Collisions on Re-Creation**

   * If you delete a StatefulSet and re-create it with the same name, but PVCs still exist, Kubernetes rebinds pods to existing PVCs. If you want fresh volumes, delete PVCs before re-creation.

5. **Rolling Update Fails Because Pod Doesn’t Become Ready**

   * Review logs: `kubectl logs mysql-2`.
   * Check readiness probe: if it never passes, the StatefulSet will not proceed to update the next ordinal.
   * Consider temporarily switching `updateStrategy.type` to `OnDelete` to manually handle the failing pod.

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/?utm_source=chatgpt.com "StatefulSets - Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/controllers/?utm_source=chatgpt.com "Workload Management - Kubernetes"
[3]: https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/?utm_source=chatgpt.com "StatefulSet Basics - Kubernetes"

## Deploymenmts

A **Deployment** in Kubernetes is a higher-level controller that manages a set of Pods (via ReplicaSets) to run a stateless application. It provides declarative updates, rollback capability, and ensures that the desired number of replicas is always running. With a Deployment, you describe your “desired state” (e.g., which container image and how many replicas), and the Deployment controller continuously works to make the actual state match that declaration ([Kubernetes][1]).

---

## 1. Why Use a Deployment?

* **Declarative Updates**
  Instead of manually creating pods one by one, a Deployment lets you specify a Pod template (including container images, resource requests/limits, environment variables, and more) along with the number of replicas. Kubernetes then creates and updates ReplicaSets under the hood to match that state. When you update the Pod template (for example, bumping the container image), the Deployment controller rolls out those changes at a controlled rate, ensuring zero-downtime updates by default ([Kubernetes][1]).

* **Self-Healing**
  A Deployment ensures that if a Pod crashes, is evicted, or a node fails, a replacement Pod is automatically created. It continually compares the “desired replicas” to the “actual running replicas” and reconciles any discrepancies.

* **Rollback and History**
  Deployments maintain a revision history of ReplicaSets, enabling you to roll back to a previous version if a new rollout fails (for instance, due to a misconfiguration or a bug in the updated container) ([Kubernetes][1]).

* **Scaling**
  You can scale a Deployment up or down (changing the number of replicas) either declaratively (by editing the YAML and re-applying) or imperatively (via `kubectl scale`).

* **Use Cases**

   * Stateless workloads where any instance of the application is interchangeable (e.g., front-end web servers, API servers, microservices).
   * Workloads that require rolling updates or rollbacks.
   * Applications that must remain available during updates.

In nearly all modern Kubernetes clusters, **Deployments** are the recommended way to run stateless workloads; direct usage of ReplicaSets (or the older ReplicationController) is discouraged except for very specialized scenarios ([Kubernetes][2], [Kubernetes][3]).

---

## 2. Anatomy of a Deployment Manifest

Below is a minimal example of a Deployment that runs an NGINX web server with 3 replicas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.21.0
        ports:
        - containerPort: 80
```

### 2.1 `apiVersion` and `kind`

* **`apiVersion: apps/v1`**
  Deployments live under the `apps/v1` API group in current Kubernetes versions.
* **`kind: Deployment`**
  Indicates you are creating a Deployment object.

### 2.2 `metadata`

* **`name`**: The unique name of this Deployment within its namespace (e.g., `nginx-deployment`).
* **`labels`** (optional): Key/value pairs for grouping and filtering (e.g., `app=nginx`). Labels on the Deployment itself help tools, dashboards, and organizational policies, but they are not directly used by the Deployment to select Pods.

### 2.3 `spec`

The heart of the Deployment is its `spec`, which consists of:

1. **`replicas`** *(integer)*

   * The desired number of Pod replicas (e.g., `3`). The Deployment controller ensures that exactly 3 Pods with the specified template are running at all times.
   * If you omit `replicas`, Kubernetes defaults to 1.

2. **`selector`** *(LabelSelector)*

   * Required for any Deployment. A label selector (e.g., `matchLabels: { app: nginx }`) that identifies which Pods are managed by this Deployment.
   * **Important**: The `selector.matchLabels` must exactly match the labels under `template.metadata.labels`; otherwise, the Deployment cannot correctly identify and manage its pods.

3. **`template`** *(PodTemplateSpec)*

   * Defines the Pod specification for replicas. Under `template.metadata.labels`, you place the same labels that match the Deployment’s selector (e.g., `app=nginx`).
   * Under `template.spec`, you define one or more containers, their images, resource requests/limits, environment variables, volume mounts, probes, and other standard Pod fields.

   Example container spec:

   ```yaml
   spec:
     containers:
       - name: nginx
         image: nginx:1.21.0
         ports:
           - containerPort: 80
   ```

   You can also specify liveness/readiness probes to help Kubernetes determine Pod health.

4. **`strategy`** *(DeploymentStrategy, optional)*

   * **`type`**: Either `RollingUpdate` (default) or `Recreate`.

      * **`RollingUpdate`**: The Deployment will update Pods in a rolling fashion—terminating some old Pods while creating new ones—maintaining availability.
      * **`Recreate`**: The Deployment deletes all existing Pods before creating new ones (causing downtime).
   * **`rollingUpdate`** parameters (if `type: RollingUpdate`):

      * **`maxUnavailable`**: The maximum number (or percentage) of Pods that can be unavailable during the rollout. Defaults to `25%`.
      * **`maxSurge`**: The maximum number (or percentage) of extra Pods that can be created during the rollout. Defaults to `25%`.

   Example:

   ```yaml
   strategy:
     type: RollingUpdate
     rollingUpdate:
       maxUnavailable: 1
       maxSurge: 1
   ```

5. **`minReadySeconds`** *(integer, optional)*

   * Specifies how many seconds a newly created Pod should be ready (passing readiness probes) before considering it available for traffic. This delay can help avoid launching new Pods too quickly and prematurely terminating old Pods.

6. **`revisionHistoryLimit`** *(integer, optional)*

   * The number of old ReplicaSets to retain in the history (allowing rollbacks). Defaults to `10`. Lower this value if you want to conserve etcd storage.

7. **`progressDeadlineSeconds`** *(integer, optional)*

   * The time in seconds a rollout is allowed to progress before Kubernetes marks it as failed. Defaults to `600` (10 minutes).

---

## 3. Deployment Lifecycle and Behaviors

### 3.1 Creation Flow

1. **Submit `kubectl apply -f deployment.yaml`**

   * Kubernetes API server stores the Deployment object.
2. **Deployment Controller Creates a ReplicaSet**

   * Based on the Pod template in `spec.template`, the controller generates a new ReplicaSet (e.g., `nginx-deployment-5f4c8f7b9f`). That ReplicaSet is responsible for ensuring `spec.replicas` Pods are running.
3. **ReplicaSet Creates Pods**

   * The ReplicaSet controller creates Pods matching the template. As Pods become “Ready,” the ReplicaSet increases its “ReadyReplicas” count in its status, and the Deployment’s status reflects the same.

### 3.2 Rolling Updates

When you update the Deployment (for example, change `image: nginx:1.21.0` → `image: nginx:1.22.0`) and re-apply:

1. **Create a New ReplicaSet**

   * The Deployment controller generates a new ReplicaSet (e.g., `nginx-deployment-6a8df3e7a2`) with the updated Pod template.
2. **Scale Up New ReplicaSet & Scale Down Old ReplicaSet**

   * Kubernetes increases the new ReplicaSet’s replica count by up to `maxSurge` while decreasing the old ReplicaSet’s replica count by up to `maxUnavailable`, ensuring at most the specified number of Pods are unavailable or in excess at any time.
3. **Wait for Readiness**

   * The controller waits for each new Pod to become Ready (and optionally for `minReadySeconds`) before proceeding.
4. **Clean Up Old ReplicaSet**

   * Once all Pods have been updated and are Ready, if the number of old ReplicaSets exceeds `revisionHistoryLimit`, the Deployment controller deletes the oldest ones.

You can monitor rollout progress with:

```bash
kubectl rollout status deployment/nginx-deployment
```

If something goes wrong (for example, new Pods never pass readiness probes), the rollout may time out after `progressDeadlineSeconds`, and you can choose to roll back.

### 3.3 Rollback

If a rollout fails or you decide the new version is not acceptable, you can revert to a previous revision:

```bash
kubectl rollout undo deployment/nginx-deployment
```

This command instructs the Deployment to:

1. Identify the previous ReplicaSet (e.g., `nginx-deployment-5f4c8f7b9f`)
2. Scale that ReplicaSet back up to `spec.replicas`
3. Scale the faulty ReplicaSet down to zero
4. Roll out the stable revision transparently, once again respecting `maxUnavailable` and `maxSurge`.

### 3.4 Scaling

* **Imperative**:

  ```bash
  kubectl scale deployment/nginx-deployment --replicas=5
  ```

  This immediately patches `spec.replicas` to 5, and the ReplicaSet controller creates or deletes Pods to match.
* **Declarative**: Edit `deployment.yaml`:

  ```yaml
  spec:
    replicas: 5
  ```

  Then re-apply:

  ```bash
  kubectl apply -f deployment.yaml
  ```

  Kubernetes reconciles to ensure 5 Pods are running.

### 3.5 Self-Healing

If one of your Pods crashes (e.g., due to a container crash or node failure), the ReplicaSet—and in turn, the Deployment—detects that the “actual ready pods” count is less than `spec.replicas`. It immediately creates a replacement Pod. This ensures that your desired number of replicas is always maintained.

---

## 4. Common Deployment `spec` Fields and Their Effects

| Field                          | Purpose                                                                                                 |
| ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `spec.replicas`                | Desired number of Pod replicas.                                                                         |
| `spec.selector.matchLabels`    | Label selector that identifies Pods managed by this Deployment (must match `template.metadata.labels`). |
| `spec.template`                | Pod template (metadata + spec) containing containers, volumes, probes, etc.                             |
| `spec.strategy.type`           | Update strategy: `RollingUpdate` (default) or `Recreate`.                                               |
| `spec.strategy.rollingUpdate`  | Parameters for rolling updates: `maxUnavailable`, `maxSurge`.                                           |
| `spec.minReadySeconds`         | Seconds to wait after a Pod becomes Ready before considering it available.                              |
| `spec.revisionHistoryLimit`    | How many old ReplicaSets to retain for rollback.                                                        |
| `spec.progressDeadlineSeconds` | Timeout (in seconds) for rollout to make progress before marking failure.                               |

---

## 5. Best Practices for Deployments

1. **Keep Pod Templates in Version Control**

   * Store your Deployment YAML in Git (or another VCS) so you have history and can track changes.

2. **Use Health Probes**

   * Define both **livenessProbes** (to restart unhealthy containers) and **readinessProbes** (so Kubernetes knows when a Pod is ready to serve traffic). This prevents sending traffic to Pods that are not yet ready and helps catch runtime failures.

   Example readiness probe:

   ```yaml
   readinessProbe:
     httpGet:
       path: /healthz
       port: 8080
     initialDelaySeconds: 5
     periodSeconds: 10
   ```

3. **Tune `maxUnavailable` and `maxSurge`**

   * By default, both are `25%`, which works for many cases. However, if you cannot tolerate any downtime, set:

     ```yaml
     rollingUpdate:
       maxUnavailable: 0
       maxSurge: 1
     ```

     This ensures new Pods are brought up before old ones are taken down. If you can tolerate some downtime, you might choose:

     ```yaml
     rollingUpdate:
       maxUnavailable: 1
       maxSurge: 0
     ```

     to minimize the number of extra Pods.

4. **Set `minReadySeconds`**

   * If your application needs a warm-up period (e.g., caches to load, JIT compilation), increase `minReadySeconds` so the Deployment does not proceed with further rollouts until new Pods remain stable for that period.

5. **Limit Revision History**

   * If you have a high volume of rollouts, lower `revisionHistoryLimit` (e.g., to `3`) to prevent accumulating many old ReplicaSets and consuming etcd storage.

6. **Use Resource Requests and Limits**

   * Always specify `resources.requests` so Pods are scheduled appropriately.
   * Use `resources.limits` to prevent runaway resource usage that can starve other workloads.

   Example:

   ```yaml
   resources:
     requests:
       cpu: "100m"
       memory: "200Mi"
     limits:
       cpu: "500m"
       memory: "500Mi"
   ```

7. **Isolate Deployments by Namespace**

   * Use separate namespaces for different environments (e.g., `dev`, `staging`, `production`) to avoid accidental cross-environment deployments and to apply resource quotas at the namespace level.

8. **Label Consistently**

   * Follow standard label conventions (e.g., `app.kubernetes.io/name`, `app.kubernetes.io/version`, `environment`, `team`) so you and your tools can easily filter and group resources.

9. **Monitor Rollout Status**

   * Use:

     ```bash
     kubectl rollout status deployment/<name>
     ```

     to watch a rollout until completion or failure.

10. **Automate with CI/CD**

   * Integrate `kubectl apply` (or use GitOps tools) in your CI/CD pipelines so Deployments happen in a repeatable, auditable fashion.

---

## 6. Deployments vs. Other Controllers

| Controller      | Description                                                                                    | Use Case                                                  |
| --------------- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| **Deployment**  | Manages ReplicaSets to ensure stateless Pods are running; supports rolling updates & rollbacks | Stateless web services, APIs, microservices.              |
| **ReplicaSet**  | Ensures a specified number of Pod replicas are running; lacks rolling updates & history        | Low-level replication; usually managed by a Deployment.   |
| **StatefulSet** | Manages stateful applications requiring stable network IDs and persistent storage              | Databases, clustered caches, message brokers.             |
| **DaemonSet**   | Ensures a Pod runs on each (or a subset of) nodes                                              | Node-level services: logging agents, monitoring daemons.  |
| **Job**         | Runs pods to completion (finite tasks)                                                         | Batch jobs, migrations, data processing, one-off tasks.   |
| **CronJob**     | Schedules Jobs on a cron‐like schedule                                                         | Periodic tasks: backups, report generation, cleanup jobs. |

* **Deployments** are the recommended choice for nearly all stateless workloads because they provide rolling updates and rollback capabilities.
* **ReplicaSets** are rarely used directly. Aside from edge cases needing custom orchestration, you should create Deployments, not ReplicaSets.
* **StatefulSets** are for workloads that need stable identities or persistent volumes; Deployments assume pods are fungible.

---

## 7. Example Workflow: From Zero to a Running Deployment

1. **Write your Deployment YAML** (e.g., `nginx-deployment.yaml`):

   ```yaml
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
           image: nginx:1.21.0
           ports:
           - containerPort: 80
   ```

2. **Create the Deployment**:

   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

3. **Verify Creation**:

   ```bash
   kubectl get deployments
   # NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   # nginx-deployment   3/3     3            3           1m

   kubectl get pods -l app=nginx
   # nginx-deployment-5d8b7f7769-abcde   1/1   Running   0    1m
   # nginx-deployment-5d8b7f7769-fghij   1/1   Running   0    1m
   # nginx-deployment-5d8b7f7769-klmno   1/1   Running   0    1m
   ```

4. **Roll Out an Update** (bump NGINX version to `1.22.0`):

   * Edit `nginx-deployment.yaml` → change `image: nginx:1.21.0` to `image: nginx:1.22.0`.
   * Re-apply:

     ```bash
     kubectl apply -f nginx-deployment.yaml
     ```
   * Observe:

     ```bash
     kubectl rollout status deployment/nginx-deployment
     # deployment "nginx-deployment" successfully rolled out
     ```

   Kubernetes creates a new ReplicaSet with Pods running `nginx:1.22.0`, scales it up, and scales down the old ReplicaSet in a rolling fashion.

5. **Roll Back if Needed**:

   ```bash
   kubectl rollout undo deployment/nginx-deployment
   ```

   This reverts to the previous ReplicaSet (with `nginx:1.21.0`) in the same rolling, controlled manner.

6. **Scale the Deployment** to 5 replicas:

   ```bash
   kubectl scale deployment/nginx-deployment --replicas=5
   ```

   Now you see two additional Pods come up:

   ```bash
   kubectl get pods -l app=nginx
   # ... 5 pods listed, all Running.
   ```

7. **Delete the Deployment**:

   ```bash
   kubectl delete deployment nginx-deployment
   ```

   Kubernetes deletes the Deployment object, its associated ReplicaSets, and all Pods managed by those ReplicaSets (cascade by default).

---

## 8. Advanced Tips

* **Pause & Resume Rollouts**
  To make multiple changes before continuing a rollout, you can pause a Deployment:

  ```bash
  kubectl rollout pause deployment/nginx-deployment
  # make changes (e.g., adjust livenessProbe, add environment vars)
  kubectl apply -f nginx-deployment.yaml
  kubectl rollout resume deployment/nginx-deployment
  ```

* **Inspect Rollout History**

  ```bash
  kubectl rollout history deployment/nginx-deployment
  ```

  Lists revisions (ReplicaSet names and associated Pod templates). You can view details of a specific revision:

  ```bash
  kubectl rollout history deployment/nginx-deployment --revision=2
  ```

* **Set Image via CLI**

  ```bash
  kubectl set image deployment/nginx-deployment nginx=nginx:1.23.0
  ```

  This patches the Pod template, causing a rolling update.

* **Use Canary Deployments with Partitions**
  You can split traffic between old and new versions by manipulating `rollingUpdate` settings (e.g., setting `maxSurge` to a percentage) or using sophisticated blue/green canary tools like **Istio**, **Argo Rollouts**, or **Flagger**.

* **Pod Anti-Affinity**
  To spread Pods across failure domains (nodes or zones), use `podAntiAffinity`. Example:

  ```yaml
  spec:
    template:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: nginx
              topologyKey: "kubernetes.io/hostname"
  ```

  This ensures no two replicas land on the same node, improving fault tolerance.

* **Resource-Based Autoscaling**
  Pair your Deployment with a **Horizontal Pod Autoscaler (HPA)** to automatically adjust `spec.replicas` based on CPU utilization (or custom metrics). Example:

  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: nginx-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-deployment
    minReplicas: 3
    maxReplicas: 10
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  ```

* **Use PodDisruptionBudgets**
  To ensure compliance with minimum availability during voluntary disruptions (e.g., upgrading nodes), define a PodDisruptionBudget (PDB). Example:

  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: nginx-pdb
  spec:
    maxUnavailable: 1
    selector:
      matchLabels:
        app: nginx
  ```

  This ensures that at least `replicas - maxUnavailable` Pods remain available during evictions.

---

## 9. Troubleshooting Common Issues

1. **Pods Not Updating**

   * If you update a Deployment but pods don’t change, verify that:

      * You changed a field under `spec.template` (only changes there trigger a new ReplicaSet).
      * `kubectl rollout status deployment/<name>` shows progress; if it hangs, check Pod events for failed readiness probes.

2. **Rolling Update Stuck**

   * Check `kubectl describe deployment/<name>` and `kubectl describe pods -l <selector>` for events (e.g., image pull errors, insufficient resources, failing probes).
   * Ensure `maxUnavailable` and `maxSurge` settings allow pods to launch.

3. **Rollback Fails**

   * If you attempt `kubectl rollout undo` but it doesn’t revert, verify there is a previous revision:

     ```bash
     kubectl rollout history deployment/<name>
     ```

     If only one revision exists, there’s nothing to roll back to.

4. **Deployment Doesn’t Delete Pods**

   * By default, deleting a Deployment cascades to its ReplicaSets and Pods. If pods remain, check if you used `--cascade=false`.

5. **Unexpected Scaling Behavior**

   * If HPA adjusts replicas independently, remember that any manual `kubectl scale` may be overwritten by HPA. Check `kubectl get hpa` and HPA events.

[1]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/?utm_source=chatgpt.com "Deployments | Kubernetes"
[2]: https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/?utm_source=chatgpt.com "ReplicaSet - Kubernetes"
[3]: https://kubernetes.io/docs/concepts/workloads/controllers/?utm_source=chatgpt.com "Workload Management - Kubernetes"

## Managing Workloads in Kubernetes

## Managing Workloads in Kubernetes

Once you’ve deployed your application and exposed it via a Service, the next step is to manage that workload over time—scaling it up or down, applying updates without downtime, and orchestrating more advanced deployment patterns like canaries. Kubernetes provides several built-in mechanisms (and works well with external tools) to handle these tasks in a declarative, repeatable fashion. This article will walk through:

1. Organizing and applying resource configurations
2. Scaling your application
3. Updating without an outage
4. Managing rollouts
5. Canary deployments
6. Helpful external tools (e.g., Helm)

---

## 1. Organizing Resource Configurations

Large applications often consist of multiple Kubernetes resources—for example, a Deployment, a Service, a ConfigMap, and perhaps a PersistentVolumeClaim. Instead of managing each YAML independently, Kubernetes supports grouping related manifests into a single file (separated by `---`) or into a directory hierarchy that can be applied recursively. This makes it easier to treat an entire microservice or application tier as a single unit.

### 1.1 Multi‐Document YAML Files

You can place multiple resource definitions in a single YAML file by separating them with `---`. For example, an NGINX application might need both a Service and a Deployment:

```yaml
# application/nginx-app.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
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
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

When you run:

```bash
kubectl apply -f application/nginx-app.yaml
```

Kubernetes processes each document in order. In this example, the Service is created first (so that when the Deployment’s Pods come up, they can already be scheduled across nodes that satisfy the Service’s endpoints). Grouping related resources in one file ensures a consistent apply order and makes version-controlling your manifests simpler.

### 1.2 Applying Multiple Files with `–recursive`

If your application’s manifests are spread across multiple files or subdirectories, you can use the `--recursive` (or `-R`) flag so that `kubectl` walks all subdirectories and processes every YAML/JSON it finds:

```
project/k8s/development/
├── configmap/
│   └── my-configmap.yaml
├── deployment/
│   └── my-deployment.yaml
└── pvc/
    └── my-pvc.yaml
```

To create all resources under `project/k8s/development`, run:

```bash
kubectl apply -f project/k8s/development --recursive
```

This will create the ConfigMap, Deployment, and PVC in one go. You can also mix multiple `-f` arguments with `--recursive`:

```bash
kubectl apply \
  -f project/k8s/namespaces \
  -f project/k8s/development \
  --recursive
```

Any command that accepts `-f` (such as `kubectl get`, `kubectl delete`, `kubectl rollout`) can accept `--recursive`. This approach makes it easier to manage a directory of related manifests as a cohesive unit.

---

## 2. Scaling Your Application

When load on your application changes—either increases during peak traffic or decreases during lull—Kubernetes lets you adjust the number of replicas easily. For any resource that supports replication (Deployments, ReplicaSets, StatefulSets, etc.), you can run:

```bash
kubectl scale <resource-type>/<name> --replicas=<count>
```

### 2.1 Example: Scaling a Deployment

Suppose you have an NGINX Deployment named `my-nginx` currently running 3 replicas:

```bash
kubectl get deployment my-nginx
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# my-nginx     3/3     3            3           30m
```

To reduce the replica count from 3 to 1:

```bash
kubectl scale deployment/my-nginx --replicas=1
```

You’ll see output like:

```
deployment.apps/my-nginx scaled
```

Then, if you list pods by label, you’ll have just one Pod left:

```bash
kubectl get pods -l app=nginx
# NAME                       READY   STATUS    RESTARTS   AGE
# my-nginx-2035384211-j5fhi  1/1     Running   0          30m
```

Similarly, to increase back to 3:

```bash
kubectl scale deployment/my-nginx --replicas=3
```

Kubernetes will immediately spawn two more Pods to satisfy the new desired state. If you prefer a declarative approach, edit your Deployment’s `.spec.replicas` field in the YAML and re-apply it with `kubectl apply -f <file>`. The controller then reconciles to match the declared replica count.

---

## 3. Updating Your Application Without an Outage

A core goal for production workloads is zero downtime—even when rolling out new container images or configuration changes. Kubernetes Deployments support rolling updates by default, allowing you to shift traffic gradually from old Pods to new ones, ensuring that some replicas remain available at all times.

### 3.1 Rolling Update Strategies

By default, a Deployment’s update strategy is:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

* **`maxUnavailable`** (default `25%`) indicates how many Pods can be temporarily unavailable during the update. If you have 4 replicas, up to one Pod may be unavailable while the new ones come up.
* **`maxSurge`** (default `25%`) indicates how many extra Pods can be created above the desired replica count. If your Deployment has 4 replicas, one additional Pod (25% of 4) may be created, bringing the total to 5 during the rollout.

These values let you ensure that old Pods are only terminated when new Pods become Ready, maintaining service availability. You can adjust them based on your application’s tolerance for extra capacity or brief unavailability. For instance, to guarantee no Pod is ever down (at the cost of temporarily running one extra replica), set:

```yaml
rollingUpdate:
  maxUnavailable: 0
  maxSurge: 1
```

### 3.2 Patching or Editing to Update the Image

#### Patching via `kubectl patch`

If you want to update from `nginx:1.14.2` to `nginx:1.16.1` without editing the YAML file directly, you can use `kubectl patch`:

```bash
kubectl patch deployment/my-nginx \
  --type='merge' \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.16.1"}]}}}}'
```

This patch changes the Pod template’s container image, which triggers a rolling update. Kubernetes creates a new ReplicaSet for the updated Pods, gradually scales it up (respecting `maxSurge`), and scales down the old ReplicaSet (respecting `maxUnavailable`).

#### Editing with `kubectl edit`

Alternatively, you can open an editor session:

```bash
kubectl edit deployment/my-nginx
```

Find the `image:` line under `spec.template.spec.containers[0]` and change:

```yaml
image: nginx:1.14.2
```

to:

```yaml
image: nginx:1.16.1
```

Save and exit. The Deployment controller notices the change and begins the rolling update automatically.

### 3.3 Allowing Temporary Surge Replicas

By default, RollingUpdate may create up to 25% extra replicas. If you want to allow your Deployment to add up to 100% extra replicas (for very high-availability scenarios), patch it like this:

```bash
kubectl patch deployment/my-nginx \
  --type='merge' \
  -p '{ "spec": { "strategy": { "rollingUpdate": { "maxSurge": "100%" }}}}'
```

This means if you have 3 replicas, Kubernetes can create 3 additional Pods (total 6) during the rollout, so that at least 3 “old” Pods remain serving traffic while 3 new ones come up.

---

## 4. Managing Rollouts

The `kubectl rollout` subcommands let you observe, pause, resume, and roll back Deployments (as well as StatefulSets and DaemonSets).

### 4.1 Viewing Rollout Status

To watch the progress of an ongoing rollout:

```bash
kubectl rollout status deployment/my-nginx --timeout=10m
```

Kubernetes will report lines like:

```
Waiting for deployment "my-nginx" rollout to finish: 2 of 3 updated replicas are available...
```

Once all new Pods pass readiness checks and the old Pods are terminated (in accordance with `maxUnavailable`), you’ll see:

```
deployment "my-nginx" successfully rolled out
```

If the rollout hangs (e.g., because new Pods fail readiness probes), and `progressDeadlineSeconds` (default 600 seconds) is exceeded, the rollout is marked as failed.

### 4.2 Pausing and Resuming a Rollout

You might want to perform multiple changes to a Deployment (for instance, updating environment variables, adding probes, and changing images) but hold off on rolling out until all edits are ready. You can do:

```bash
kubectl rollout pause deployment/my-nginx
# Now edit the Deployment manifest multiple times (kubectl edit or patches)
kubectl rollout resume deployment/my-nginx
```

When you resume, Kubernetes applies all pending changes and begins the rolling update.

### 4.3 Rolling Back to a Previous Revision

If a new rollout is unstable, you can revert to the last working revision:

```bash
kubectl rollout undo deployment/my-nginx
```

Behind the scenes, Kubernetes keeps a history of revisions (ReplicaSets) up to the number specified by `revisionHistoryLimit` (default 10). When you undo, it scales up the previous ReplicaSet and scales down the current one—again adhering to your `maxSurge` and `maxUnavailable` settings for a smooth transition.

You can inspect past revisions with:

```bash
kubectl rollout history deployment/my-nginx
```

To view details of a specific revision (e.g., revision 2):

```bash
kubectl rollout history deployment/my-nginx --revision=2
```

---

## 5. Canary Deployments

A canary deployment is the practice of releasing a new application version to a subset of users before rolling it out to everyone. In Kubernetes, you can achieve this pattern using multiple Deployments (or ReplicaSets) distinguished by labels—such as `track=stable` and `track=canary`—and a single Service that routes traffic to pods matching only a core set of labels (omitting the `track` label).

### 5.1 Example: Canary with Label Splitting

1. **Stable Deployment**
   Create a Deployment for the stable version, labeling its pods with `track=stable`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend-stable
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: guestbook
         tier: frontend
         track: stable
     template:
       metadata:
         labels:
           app: guestbook
           tier: frontend
           track: stable
       spec:
         containers:
           - name: guestbook-frontend
             image: gb-frontend:v3
             ports:
               - containerPort: 80
   ```

2. **Canary Deployment**
   Create a second Deployment for the canary version with only one replica, labeling its pods with `track=canary`:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: frontend-canary
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: guestbook
         tier: frontend
         track: canary
     template:
       metadata:
         labels:
           app: guestbook
           tier: frontend
           track: canary
       spec:
         containers:
           - name: guestbook-frontend
             image: gb-frontend:v4
             ports:
               - containerPort: 80
   ```

3. **Service Definition**
   Define a Service that selects only `app=guestbook, tier=frontend`—i.e., it ignores the `track` label. That way, both stable and canary pods receive traffic, proportional to their replica counts:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: guestbook-service
   spec:
     selector:
       app: guestbook
       tier: frontend
     ports:
       - port: 80
         targetPort: 80
   ```

4. **Adjusting Traffic Split**
   Initially, you might run 3 replicas of stable (`track=stable`) and 1 replica of canary (`track=canary`), so 25% of traffic goes to canary and 75% to stable. Measure metrics (latency, error rates, etc.) from canary pods. If everything looks good, you can:

   * Increase `frontend-canary` replicas to 2 or 3 to shift more traffic.
   * Eventually update `frontend-stable` to the new image (e.g., change to `gb-frontend:v4`) and scale down or remove `frontend-canary`.

Because both Deployments share the same `selector` in the Service (minus the `track` label), traffic is distributed across all matching pods. By varying the number of replicas in stable vs. canary, you control the traffic percentage hitting new code versus old.

### 5.2 Cleaning Up

Once you’re confident in the new version, you have two options:

* **Promote Canary**: Scale the canary Deployment up to replace the stable Deployment, then delete the stable Deployment.
* **Roll Stable Forward**: Modify the stable Deployment’s image to `v4` and scale it to 3 replicas, and delete the canary Deployment.

In either case, the Service’s selector remains unchanged (`app=guestbook, tier=frontend`), so as soon as the new pods become Ready, they automatically receive traffic.

---

## 6. External Tools for Workload Management

While Kubernetes native commands cover most scenarios, many teams use higher-level tools to simplify templating, packaging, and upgrading. The most common is **Helm**.

### 6.1 Helm (Charts)

[Helm](https://helm.sh/) is a package manager for Kubernetes that lets you define a collection of resources—Deployments, Services, ConfigMaps, Secrets, PVCs, etc.—as a single “chart.” A chart can include parameterized templates, so you can deploy the same application with different configuration values (e.g., image tags, resource sizes, environment variables) across environments.

#### Installing a Chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-nginx bitnami/nginx
```

This command fetches the NGINX chart from the Bitnami repository, fills in default values, and installs a Deployment, Service, and any associated resources.

#### Upgrading a Chart

To update the image tag or override values:

```bash
helm upgrade my-nginx bitnami/nginx --set image.tag=1.21.0
```

Helm calculates the diff and applies only the necessary changes, performing a rolling update if the chart’s Deployment uses `RollingUpdate` strategy. You can also roll back with:

```bash
helm rollback my-nginx 1
```

where `1` is the revision number shown by `helm history my-nginx`.

#### Benefits of Helm

* **Chart Reusability**: Standardized templates for common applications (e.g., databases, message queues).
* **Configuration Management**: Values are stored in `values.yaml` or passed on the command line, making environment differences trivial to manage.
* **Revision History**: Helm tracks releases and revisions, enabling easy rollbacks.

---

## 7. Putting It All Together: A Typical Workflow

1. **Define Manifests in Git**

   * Store your multi-document YAMLs (Deployment, Service, ConfigMap) under a directory (e.g., `k8s/guestbook/`).
   * Commit to version control.

2. **Apply Resources**

   ```bash
   kubectl apply -f k8s/guestbook/ --recursive
   ```

   * This creates Services, Deployments, and any other resources in the correct order.

3. **Verify Everything Is Running**

   ```bash
   kubectl get all -l app=guestbook
   # Check that Pods are in Ready state, Services have endpoints, etc.
   ```

4. **Scale as Needed**

   * If traffic grows, scale up:

     ```bash
     kubectl scale deployment/guestbook-front-end --replicas=5
     ```
   * If traffic drops, scale down accordingly.

5. **Roll Out a New Version**

   * Change the image tag in the Deployment’s Pod template (or patch it):

     ```bash
     kubectl patch deployment/guestbook-front-end \
       --type='merge' \
       -p '{"spec":{"template":{"spec":{"containers":[{"name":"guestbook","image":"guestbook:v2"}]}}}}'
     ```
   * Monitor rollout:

     ```bash
     kubectl rollout status deployment/guestbook-front-end
     ```

6. **Pause/Resume or Roll Back**

   * If you need to pause (e.g., to tweak config), run:

     ```bash
     kubectl rollout pause deployment/guestbook-front-end
     ```
   * Once edits are done,:

     ```bash
     kubectl rollout resume deployment/guestbook-front-end
     ```
   * If something goes wrong:

     ```bash
     kubectl rollout undo deployment/guestbook-front-end
     ```

7. **Try a Canary**

   * Duplicate your Deployment with a different label:

     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: guestbook-front-end-canary
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: guestbook
           tier: frontend
           track: canary
       template:
         metadata:
           labels:
             app: guestbook
             tier: frontend
             track: canary
         spec:
           containers:
             - name: guestbook
               image: guestbook:v3
               ports:
                 - containerPort: 80
     ```
   * The existing Service (which selects only `app: guestbook, tier: frontend`) automatically begins sending about 1⁄(1 + 3) = 25% of traffic to the canary pod.
   * Once metrics look good, scale the canary Deployment to 3 replicas and scale down (and eventually delete) the stable Deployment.

8. **Clean Up Old Resources**

   * After you’re confident in v3, delete the old Deployment and any unused ConfigMaps or Secrets.
   * If using Helm, use `helm uninstall <release-name>` or `helm rollback` as appropriate.
