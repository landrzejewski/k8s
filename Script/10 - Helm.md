## Introduction

Helm is the de facto package manager for Kubernetes, streamlining the deployment, configuration, and management of complex applications on Kubernetes clusters. It uses “charts” as blueprints that bundle Kubernetes manifest templates, default configuration values, and metadata into a single package, allowing consistent, versioned, and repeatable deployments across environments ([middleware.io][1], [devopscube.com][2]). A Helm chart abstracts away repetitive YAML boilerplate by leveraging Go templating to dynamically inject values, which greatly improves maintainability and reduces the risk of configuration drift between development, staging, and production environments ([devopscube.com][2], [datacamp.com][3]).

---

## Prerequisites

Before creating and deploying Helm charts, ensure you have the following:

* **Kubernetes cluster access**: You need a running Kubernetes cluster with kubeconfig properly configured, allowing `kubectl` to operate against your cluster ([devopscube.com][2], [medium.com][4]).
* **Helm CLI installed**: Helm must be installed on your local machine. This guide uses Helm v3 (latest at time of writing). For detailed installation steps, see [Installing and Configuring Helm](#installing-helm) below ([datacamp.com][3], [middleware.io][1]).
* **kubectl installed**: A working `kubectl` binary is required to verify deployments, view resources, and debug outputs.
* **Basic Kubernetes and YAML knowledge**: Familiarity with Kubernetes objects (Deployments, Services, ConfigMaps, etc.) and YAML syntax is assumed ([devopscube.com][2], [datacamp.com][3]).

---

## Installing Helm

Helm can be installed via binary release, script, or package manager across Windows, macOS, and Linux environments ([datacamp.com][3]).

### Windows

1. Download the Windows Helm binary from [Helm’s GitHub releases](https://github.com/helm/helm/releases).
2. Extract the ZIP to a folder, e.g., `C:\Program Files\helm`.
3. Add `C:\Program Files\helm` to your system `PATH` via **System Properties → Advanced → Environment Variables → Path → Edit → New → `C:\Program Files\helm`**.
4. Open a new PowerShell or Command Prompt and verify:

   ```shell
   helm version
   ```

   If successful, you’ll see output like `version.BuildInfo{Version:"v3.x.x", ...}` ([datacamp.com][3]).

### macOS

* **Binary Installation**:

  ```shell
  wget https://get.helm.sh/helm-v3.17.2-darwin-amd64.tar.gz
  tar -zxvf helm-v3.17.2-darwin-amd64.tar.gz
  sudo mv darwin-arm64/helm /usr/local/bin/
  helm version
  ```

* **Homebrew** (preferred):

  ```shell
  brew update
  brew install helm
  helm version
  ```

  Homebrew ensures that `helm` is added to `/usr/local/bin` and auto-updated when you run `brew upgrade helm` ([datacamp.com][3]).

### Linux

1. Download the appropriate Linux binary:

   ```shell
   wget https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
   tar -zxvf helm-v3.17.2-linux-amd64.tar.gz
   sudo mv linux-amd64/helm /usr/local/bin/helm
   helm version
   ```

2. Clean up artifacts:

   ```shell
   rm -rf linux-amd64 helm-v3.17.2-linux-amd64.tar.gz
   ```

This places `helm` in `/usr/local/bin` so it’s accessible in any terminal session ([datacamp.com][3]).

---

## Helm Architecture

Helm v3 uses a **client-only** architecture, removing the server component Tiller (deprecated in Helm v2). The two main components are ([middleware.io][1], [datacamp.com][3]):

1. **Helm Client**: The `helm` CLI interacts directly with the Kubernetes API server using your kubeconfig to install, upgrade, and rollback releases.
2. **Helm Library**: Core Go-based engine that Helm Client invokes to render templates, manage chart dependencies, and communicate with Kubernetes.

By eliminating Tiller, Helm v3 enhances security and simplifies workflows, as no additional service account or in-cluster privilege is required beyond what `kubectl` needs.

---

## Helm Chart Structure

A Helm chart is a directory tree containing everything needed to define and run a Kubernetes application. Run `helm create my-chart` to see the default scaffold ([devopscube.com][2], [middleware.io][1]):

```
my-chart/
├── .helmignore
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── NOTES.txt
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── tests/
        └── test-connection.yaml
```

Below is a breakdown of each component and its purpose:

1. **`.helmignore`**

    * Patterns listed here are excluded when packaging the chart (similar to `.gitignore`). It prevents unnecessary files (e.g., local IDE configs) from being bundled ([devopscube.com][2], [datacamp.com][3]).
2. **`Chart.yaml`**

    * Contains chart metadata:

        * `apiVersion`: Chart API version (`v2` for Helm 3).
        * `name`: Chart name (must be lowercase, hyphenated).
        * `description`: One-line summary.
        * `type`: `application` (deployable) or `library` (reusable).
        * `version`: Chart version (semver).
        * `appVersion`: Upstream application version (e.g., Docker image version).
        * (Optional) `maintainers`, `icon`, `dependencies`, etc. ([devopscube.com][2], [datacamp.com][3]).
3. **`values.yaml`**

    * Default configuration values for the chart’s templates. Users override these via `--values` (`-f`) or `--set` flags when installing/upgrading. Typical keys include image repository/tag, replica count, service ports, resource limits, and environment-specific flags ([devopscube.com][2], [datacamp.com][3]).
4. **`charts/`**

    * Contains subcharts (chart dependencies). If your chart depends on other charts (e.g., Redis, PostgreSQL), place them here or reference them in `Chart.yaml` under `dependencies` ([datacamp.com][3], [middleware.io][1]).
5. **`templates/`**

    * Holds all Kubernetes manifest templates (Go templates). By default, Helm scaffolds common resources:

        * `deployment.yaml`
        * `service.yaml`
        * `ingress.yaml`
        * `hpa.yaml`
        * `serviceaccount.yaml`
        * `NOTES.txt`: Printed at the end of a successful install, often containing post-install instructions.
        * `_helpers.tpl`: Defines reusable template helpers and functions (e.g., `fullname`, `labels`, selector templates) to avoid repetition.
        * `tests/`: Contains pod-based tests (e.g., `test-connection.yaml`) to validate the release ([devopscube.com][2], [middleware.io][1]).

---

## Creating a Helm Chart

The easiest way to start is by generating the chart scaffold using `helm create`. This gives you a working boilerplate that you can customize for your application ([devopscube.com][2], [datacamp.com][3]).

```bash
helm create my-application
```

This command creates:

```
my-application/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── NOTES.txt
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── tests/
        └── test-connection.yaml
```

1. **`Chart.yaml`**: Edit this to set your chart’s `name`, `description`, `version`, `appVersion`, maintainers, and any dependencies. Example:

   ```yaml
   apiVersion: v2
   name: my-application
   description: A Helm chart for deploying MyApp
   type: application
   version: 0.1.0
   appVersion: "2.3.1"
   maintainers:
     - name: DevOps Team
       email: devops@example.com
   ```

2. **`values.yaml`**: Replace default values with your application-specific defaults:

   ```yaml
   replicaCount: 2

   image:
     repository: myorg/myapp
     tag: "2.3.1"
     pullPolicy: IfNotPresent

   service:
     type: ClusterIP
     port: 80

   ingress:
     enabled: false
     annotations: {}
     hosts:
       - host: chart-example.local
         paths: []

   resources: {}
   nodeSelector: {}
   tolerations: []
   affinity: {}
   ```

3. Delete or adjust unused scaffold files in `templates/`. For instance, if you don’t need an HPA or `serviceaccount.yaml`, remove them.

---

## Templating and Customization

Helm leverages Go’s built-in templating engine. Within templates, you access several built-in objects:

* **`.Release`**: Contains release-related information (e.g., `.Release.Name`, `.Release.Namespace`).
* **`.Chart`**: Accesses `Chart.yaml` fields like `.Chart.Name` and `.Chart.Version`.
* **`.Values`**: References keys in `values.yaml` (or user-provided overrides).
* **`include` and `template` built-ins**: Let you reuse snippets defined in `_helpers.tpl`, such as `{{ include "my-application.fullname" . }}` or `{{ include "my-application.labels" . | nindent 4 }}` ([devopscube.com][2], [datacamp.com][3]).

### Example: `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-application.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-application.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
              protocol: TCP
```

* **`include "my-application.fullname" .`**: Calls a helper in `_helpers.tpl` that returns a unique name combining release name and chart name.
* **`{{ .Values.replicaCount }}`**: Inserts the `replicaCount` value from `values.yaml`.
* **`{{ .Values.image.tag | default .Chart.AppVersion }}`**: Uses `image.tag` if provided; otherwise, falls back to `appVersion` defined in `Chart.yaml` ([devopscube.com][2], [datacamp.com][3]).

### Template Helpers (`templates/_helpers.tpl`)

Customize or add functions in `_helpers.tpl`. Example helpers:

```gotemplate
{{- define "my-application.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "my-application.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: Helm
{{- end -}}

{{- define "my-application.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

* **`printf "%s-%s" .Release.Name .Chart.Name`**: Ensures every resource name is unique by prefixing with release name.
* **`trunc 63 | trimSuffix "-"`**: Constrains names to Kubernetes’ 63-character limit and trims trailing hyphens.

These helpers centralize logic, making updates easier if naming conventions change.

---

## Configuration with `values.yaml`

The `values.yaml` file centralizes default configuration values. Its keys are referenced in templates via the `Values` object:

```yaml
replicaCount: 2

image:
  repository: myorg/myapp
  tag: "2.3.1"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

ingress:
  enabled: false
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: []
  tls: []
```

* **Nesting**: Nested keys map to nested Go template notation, e.g., `{{ .Values.image.repository }}` ([devopscube.com][2], [middleware.io][1]).
* **Defaults and Overrides**:

    * User can override `values.yaml` via `-f custom-values.yaml`.
    * CLI overrides:

      ```bash
      helm install my-release \
        ./my-application \
        --set replicaCount=3,image.tag=latest
      ```
    * Unspecified `values.yaml` keys fall back to chart defaults.

To support multiple environments (e.g., dev, staging, prod), create separate files:

```
values-dev.yaml
values-staging.yaml
values-prod.yaml
```

And install accordingly:

```bash
helm install my-release ./my-application -f values-prod.yaml
```

This strategy avoids complex conditional logic in templates and keeps environment-specific values isolated ([middleware.io][1], [datacamp.com][3]).

---

## Validating a Helm Chart

Before installing a chart, it’s crucial to validate structure, syntax, and rendered Kubernetes manifests to catch errors early.

### 1. `helm lint`

Checks chart folder for missing fields, inconsistent indentation, and best-practice violations. Run:

```bash
helm lint ./my-application
```

Output example:

```
==> Linting my-application
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Lint errors typically point to invalid `Chart.yaml` fields, missing required fields, or malformed templates ([devopscube.com][2], [middleware.io][1]).

### 2. `helm template`

Renders templates locally without communicating with Kubernetes API. It shows the fully substituted YAML manifests based on current `values.yaml`:

```bash
helm template my-release ./my-application
```

Sample rendered snippet:

```yaml
# Source: my-application/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-release-my-application
  labels:
    helm.sh/chart: my-application-0.1.0
    app.kubernetes.io/name: my-application
    app.kubernetes.io/instance: my-release
    app.kubernetes.io/version: "2.3.1"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: my-application
      app.kubernetes.io/instance: my-release
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-application
        app.kubernetes.io/instance: my-release
    spec:
      containers:
        - name: my-application
          image: "myorg/myapp:2.3.1"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              protocol: TCP
```

Review the output for correct values, resource definitions, and API versions ([devopscube.com][2], [middleware.io][1]).

### 3. Dry-Run Installation

Use `--dry-run` to simulate an install or upgrade:

```bash
helm install my-release ./my-application --dry-run
helm upgrade my-release ./my-application --dry-run
```

This performs server-side validation against the Kubernetes API server (e.g., missing CRD references, invalid fields), but does not create resources. It further confirms rendered manifests are valid in a cluster context ([devopscube.com][2], [middleware.io][1]).

---

## Deploying a Helm Chart

Once validated, deploy your chart with `helm install`. Make sure you are outside the chart directory:

```bash
helm install my-release ./my-application
```

* **`my-release`**: The release name (arbitrary, but usually related to chart name).
* **`./my-application`**: Path to chart folder or packaged `.tgz` file.

Sample output:

```
NAME: my-release
LAST DEPLOYED: Fri Jun 05 12:00:00 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Verify resources via `kubectl`:

```bash
kubectl get deployments,svc,pods
```

You should see:

```
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-release-my-application   2/2     2            2           1m

NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/my-release-service   ClusterIP   10.96.0.150    <none>        80/TCP    1m

NAME                                             READY   STATUS    RESTARTS   AGE
pod/my-release-my-application-xxxxx-yyyyy         1/1     Running   0          1m
pod/my-release-my-application-xxxxx-zzzzz         1/1     Running   0          1m
```

List all Helm releases:

```bash
helm list
```

Displays:

```
NAME        NAMESPACE   REVISION   UPDATED                  STATUS    CHART                   APP VERSION
my-release  default     1          Fri Jun 05 12:00:00 2025 deployed  my-application-0.1.0    2.3.1
```

To install with custom values:

```bash
helm install my-release ./my-application -f values-prod.yaml
```

Or override inline:

```bash
helm install my-release ./my-application --set replicaCount=3,image.tag="2.4.0"
```

These flags dynamically substitute values without modifying the base `values.yaml`, simplifying environment-specific overrides ([devopscube.com][2], [middleware.io][1]).

---

## Managing Releases

Helm treats each installed chart as a **release**, allowing seamless upgrades, rollbacks, and uninstallation.

### Upgrading a Release

When you update chart templates or `values.yaml`, apply changes to an existing release:

```bash
# After modifying values.yaml or templates
helm upgrade my-release ./my-application
```

To upgrade with a different values file:

```bash
helm upgrade my-release ./my-application -f values-staging.yaml
```

Sample output:

```
Release "my-release" has been upgraded. Happy Helming!
NAME: my-release
LAST DEPLOYED: Fri Jun 05 12:05:00 2025
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

Verify new revision:

```bash
helm list
```

```
NAME        NAMESPACE   REVISION   UPDATED                  STATUS    CHART                   APP VERSION
my-release  default     2          Fri Jun 05 12:05:00 2025 deployed  my-application-0.1.0    2.3.1
```

Inspect rollout:

```bash
kubectl get pods
kubectl describe pod <pod-name> | grep image
```

You should see new pods using the updated image or configuration ([devopscube.com][2], [middleware.io][1], [datacamp.com][3]).

### Rolling Back a Release

If an upgrade misbehaves, revert to a previous revision:

```bash
helm rollback my-release 1
```

Without specifying a revision, it rolls back by one revision:

```bash
helm rollback my-release
```

Output:

```
Rollback was a success! Happy Helming!
```

Check history:

```bash
helm history my-release
```

Example:

```
REVISION   UPDATED                    STATUS     CHART                   APP VERSION   DESCRIPTION
1          Fri Jun 05 12:00:00 2025   deployed   my-application-0.1.0    2.3.1         Install complete
2          Fri Jun 05 12:05:00 2025   deployed   my-application-0.1.0    2.3.1         Upgrade complete
3          Fri Jun 05 12:10:00 2025   deployed   my-application-0.1.0    2.3.1         Rollback to 1
```

Each rollback increments the revision number while applying the last known good configuration ([devopscube.com][2], [middleware.io][1], [datacamp.com][3]).

### Uninstalling a Release

To remove all resources associated with a release:

```bash
helm uninstall my-release
```

Output:

```
release "my-release" uninstalled
```

Verify resource cleanup:

```bash
kubectl get deployments,services,pods
```

No resources associated with `my-release` should remain. Helm tracks history post-uninstallation; you can still view revision logs via `helm history my-release` until the retention limit is reached ([devopscube.com][2], [middleware.io][1]).

---

## Packaging and Distributing Charts

To share your chart, package it into a versioned `.tgz` archive following semantic versioning (semver) guidelines ([devopscube.com][2], [middleware.io][1]):

```bash
helm package my-application
```

Output:

```
Successfully packaged chart and saved it to: /path/to/my-application-0.1.0.tgz
```

You can then:

* **Upload to a Helm repository**: Host on GitHub Pages, S3 bucket, or private Helm repo (e.g., Harbor, ChartMuseum).
* **Share locally**: Team members can `helm install my-release ./my-application-0.1.0.tgz`.

To update repository index:

```bash
helm repo index . --url https://example.com/charts
```

Consumers add the repo:

```bash
helm repo add my-repo https://example.com/charts
helm repo update
helm install my-release my-repo/my-application
```

Packaging ensures reproducible deployments by locking chart versions in `Chart.lock`.

---

## Dependencies and Subcharts

When your application comprises multiple loosely coupled components (e.g., web frontend, database, cache), break them into **subcharts** managed via `dependencies` in `Chart.yaml` ([datacamp.com][3], [middleware.io][1]).

Example `Chart.yaml` with dependencies:

```yaml
apiVersion: v2
name: my-application
version: 0.2.0
appVersion: "2.3.2"
dependencies:
  - name: redis
    version: 17.1.2
    repository: https://charts.bitnami.com/bitnami
  - name: postgresql
    version: 10.3.11
    repository: https://charts.bitnami.com/bitnami
```

To pull in subcharts and update local `charts/` directory:

```bash
helm dependency update my-application
```

This generates a `Chart.lock` file containing exact versions and digests of each dependency, guaranteeing reproducibility across environments. Think of `Chart.lock` as analogous to lockfiles in other package managers like npm’s `package-lock.json` or Terraform’s `.terraform.lock.hcl` ([datacamp.com][3], [datacamp.com][3]).

---

## Best Practices

Adhering to best practices ensures maintainable, secure, and reusable Helm charts ([devopscube.com][2], [middleware.io][1]):

1. **Chart Naming & Versioning**

    * Use lowercase and hyphens only (e.g., `my-application`).
    * Increment `version` in `Chart.yaml` for every chart change.
    * Align `appVersion` with upstream application image tags.

2. **Template Organization**

    * Name manifest files by Kubernetes kind (e.g., `deployment.yaml`, `service.yaml`, `ingress.yaml`).
    * Use `_helpers.tpl` for common labels, selectors, and naming functions to avoid duplication.
    * Utilize logical grouping: if your app has multiple microservices, split into subcharts.

3. **Values Management**

    * Keep `values.yaml` concise; document every key with comments.
    * Use multiple values files (`values-dev.yaml`, `values-prod.yaml`) for environment-specific settings rather than complex `if` statements.
    * Quote string values to prevent parsing issues (e.g., `tag: "1.0.0"`).

4. **Security**

    * Avoid embedding sensitive data (passwords, tokens) directly in `values.yaml`; use Kubernetes Secrets or external secrets management.
    * If pulling from private registries, configure `imagePullSecrets` keys in `values.yaml`.

5. **Linting and Testing**

    * Always run `helm lint` and `helm template` (or dry-run) before installing to detect issues early.
    * Use `helm test` to define simple smoke tests under `templates/tests/`.

6. **Documentation**

    * Include a `README.md` alongside `Chart.yaml` explaining chart usage, configurable values, and examples.
    * In `NOTES.txt`, provide post-install instructions, such as how to access services or any next steps.

7. **Dependencies**

    * Break large charts into subcharts under `charts/` and manage them via `dependencies` in `Chart.yaml`.
    * Pin dependency versions and commit `Chart.lock` to version control for reproducibility.

8. **CI/CD Integration**

    * Automate linting and testing in your CI pipeline (e.g., GitHub Actions, GitLab CI).
    * Publish packaged charts to a repository upon successful pipeline runs.

9. **Resource Management**

    * Define resource `requests` and `limits` for CPU/memory in `values.yaml` to prevent runaway resource consumption.
    * Use `affinity`, `tolerations`, and `nodeSelector` in `values.yaml` to control pod placement in production clusters.

Following these best practices will help you create robust, maintainable, and secure Helm charts that scale with your organization’s needs ([devopscube.com][2], [middleware.io][1]).

---

## Pros and Cons of Using Helm

While Helm offers significant benefits, it also introduces some trade-offs. Understanding these helps you decide when and how to adopt Helm in your Kubernetes workflow ([middleware.io][1]):

### Pros

* **Templated Kubernetes Manifests**: Parameterization via Go templates allows dynamic injection of values across multiple objects (Deployments, Services, ConfigMaps), reducing duplication and human error ([middleware.io][1], [datacamp.com][3]).
* **Version Control & Release Management**: Helm tracks each release’s history, enabling atomic upgrades and rollbacks to previous states, crucial for production stability ([middleware.io][1], [datacamp.com][3]).
* **Dependency Management**: Charts can declare dependencies on other charts (e.g., databases, message queues), and Helm automatically fetches and packages subcharts, simplifying multi-component deployments ([middleware.io][1], [datacamp.com][3]).
* **Ecosystem & Reusability**: A rich ecosystem of community-maintained charts (Artifact Hub, Bitnami) enables rapid adoption of standard architectures, accelerating development ([middleware.io][1], [middleware.io][1]).
* **CI/CD Alignment**: Helm integrates seamlessly into CI/CD pipelines (GitOps, GitLab Runner, GitHub Actions), promoting declarative, automated deployments and rollbacks ([datacamp.com][3], [middleware.io][1]).

### Cons

* **Templating Complexity**: Go templating syntax can become complex for advanced use cases (nested loops, conditionals) and may introduce steep learning curves for new users ([middleware.io][1], [datacamp.com][3]).
* **Overhead for Simple Deployments**: For single-container apps with minimal configuration, Helm’s overhead may outweigh benefits; plain YAML or Kustomize might be simpler.
* **Vendor/Community Lock-In**: Heavy reliance on community charts could introduce lock-in risks if charts aren’t maintained or follow incompatible versions.
* **Debugging Challenges**: Template errors may be cryptic, requiring careful inspection of rendered manifests or enabling verbose debugging.
* **Hybrid Environment Limitations**: Helm is Kubernetes-specific; hybrid or multi-cluster scenarios that span non-Kubernetes environments may require additional tooling.

In most production-grade Kubernetes environments with multi-service architectures, the benefits of Helm outweigh these drawbacks when charts are designed thoughtfully ([middleware.io][1], [devopscube.com][2]).

---

## Helm Commands Reference

Below is a concise cheat sheet of common Helm commands. Save it for quick reference ([devopscube.com][2], [medium.com][4]):

<details>
<summary>**Helm CLI Essentials**</summary>

```bash
# Display Helm version and client/server info
helm version

# Show CLI help
helm help
```

</details>

<details>
<summary>**Chart Management**</summary>

```bash
# Create a new Helm chart scaffold
helm create <chart-name>

# Validate chart structure and templates
helm lint <chart-directory>

# Update chart dependencies (pull subcharts)
helm dependency update <chart-directory>

# Rebuild dependencies based on Chart.lock
helm dependency build <chart-directory>

# List dependencies of a chart
helm dependency list <chart-directory>
```

</details>

<details>
<summary>**Repository Management**</summary>

```bash
# Add a Helm repository
helm repo add <name> <url>

# List all added repositories
helm repo list

# Update all repositories (fetch latest index)
helm repo update

# Remove a repository
helm repo remove <name>

# Search charts in a specific repo
helm search repo <keyword>

# Search charts on Artifact Hub (plugin-based)
helm search hub <keyword>
```

</details>

<details>
<summary>**Release Management**</summary>

```bash
# Install a Helm chart (creates a release)
helm install <release-name> <chart-path-or-repo/chart-name>

# Upgrade an existing release
helm upgrade <release-name> <chart-path-or-repo/chart-name> [-f values.yaml] [--set key=val]

# Roll back a release to a specific revision (0 = previous revision)
helm rollback <release-name> <revision>

# Uninstall (delete) a release
helm uninstall <release-name>

# List all releases in current namespace
helm list

# List all releases in all namespaces
helm list -A

# Show detailed release status
helm status <release-name>

# View release history (all revisions)
helm history <release-name>
```

</details>

<details>
<summary>**Inspect and Debug**</summary>

```bash
# Render chart templates locally (without installing)
helm template <release-name> <chart-path-or-repo/chart-name> [flags]

# Render templates with remote values file
helm template <release-name> <chart-path> -f custom-values.yaml

# Simulate an install (server-side dry run)
helm install <release-name> <chart-path> --dry-run

# Simulate an upgrade (dry run)
helm upgrade <release-name> <chart-path> --dry-run [--set key=val]

# Show default values for a chart
helm show values <chart-path-or-repo/chart-name>

# Show chart metadata (Chart.yaml)
helm show chart <chart-path-or-repo/chart-name>

# Show all information about a chart (metadata + default values)
helm show all <chart-path-or-repo/chart-name>

# Get installed release’s current values
helm get values <release-name>

# Get installed release’s rendered manifest
helm get manifest <release-name>

# Get installed release’s notes
helm get notes <release-name>

# Get installed release’s hooks
helm get hooks <release-name>
```

</details>

<details>
<summary>**Chart Packaging and Sharing**</summary>

```bash
# Package a chart into a versioned .tgz archive
helm package <chart-directory>

# Push a packaged chart to a repository (requires plugin)
helm push <chart.tgz> <repository>

# Index a directory of charts for HTTP hosting
helm repo index . --url https://example.com/charts
```

</details>

<details>
<summary>**Plugin Management**</summary>

```bash
# Install a Helm plugin (e.g., helm-diff)
helm plugin install <git-repo-url>

# List installed plugins
helm plugin list

# Update a specific plugin
helm plugin update <plugin-name>

# Remove a plugin
helm plugin uninstall <plugin-name>
```

</details>

<details>
<summary>**Testing**</summary>

```bash
# Run tests defined under templates/tests/ for a given release
helm test <release-name>
```

</details>

Use this reference to accelerate day-to-day Helm workflows and embed best practices into your operations pipeline ([devopscube.com][2], [medium.com][4]).

---

## Conclusion

Helm charts revolutionize Kubernetes application deployment by packaging manifest templates, default configurations, and metadata into reusable, versioned bundles. By leveraging Go templating, built-in objects (`.Release`, `.Chart`, `.Values`), and helper functions, you can:

1. **Create** a chart quickly using `helm create`, then customize metadata in `Chart.yaml` and defaults in `values.yaml`.
2. **Template** Kubernetes resources to avoid duplication and enforce consistency across environments.
3. **Validate** charts through `helm lint`, `helm template`, and server-side `--dry-run` to catch issues early.
4. **Deploy** releases with `helm install`, managing lifecycles via `helm upgrade`, `helm rollback`, and `helm uninstall`.
5. **Package** charts into `.tgz` archives and distribute them via Helm repositories, ensuring reproducibility.
6. **Manage Dependencies** through subcharts and `Chart.lock`, guaranteeing stable builds with pinned versions.
7. **Adopt Best Practices** such as well-structured templates, clear documentation, environment-specific values files, and automated linting/testing in CI.

While Helm introduces some complexity—particularly in templating and dependency management—its benefits in large-scale, multi-service Kubernetes environments are substantial. By following this professional guide and adhering to best practices, you’ll create maintainable, secure, and reusable Helm charts that accelerate delivery and minimize configuration drift ([devopscube.com][2], [middleware.io][1], [datacamp.com][3]).

[1]: https://middleware.io/blog/helm-chart-tutorial/ "Helm Chart Tutorial: A Complete Guide | Middleware"
[2]: https://devopscube.com/create-helm-chart/ "How to Create Helm Chart [Comprehensive Beginners Guide]"
[3]: https://www.datacamp.com/tutorial/helm-chart "Helm Chart Tutorial: A Step-by-Step Guide with Examples | DataCamp"
[4]: https://medium.com/%40reach2shristi.81/from-zero-to-hero-a-beginners-guide-to-creating-helm-charts-c17d7048ce85 "From Zero to Hero: A Beginner’s Guide to Creating Helm Charts | by Min_Minu | Medium"

Below is a step-by-step tutorial on installing a high‐availability (HA) PostgreSQL cluster on Kubernetes using Helm, and customizing its configuration. We’ll use Bitnami’s `postgresql-ha` chart, which deploys a primary (master) node, one or more replicas (slaves), and Pgpool-II as a load balancer/connection pooler. Citations are provided for each Helm command or configuration reference.

---

## Prerequisites

* **Kubernetes Cluster**: A running Kubernetes cluster (v1.16+). Your `kubectl` context should point to the cluster where you’ll install PostgreSQL.
* **Helm v3 Installed**: Helm CLI (v3) must be installed locally. Verify with:

  ```bash
  helm version
  ```
* **Namespace (Optional)**: In this tutorial, we’ll install into the `database` namespace. You can choose any namespace, or install into `default`.

---

## 1. Add Bitnami Helm Repository

First, add the Bitnami charts repository (which includes `postgresql-ha`) and update your local index:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

([artifacthub.io][1])

* `bitnami/postgresql-ha` is the chart name we’ll reference in subsequent commands.

---

## 2. Inspect Default Values

Before customizing, review the chart’s default configuration:

```bash
helm show values bitnami/postgresql-ha
```

This prints out a lengthy `values.yaml` specification. Pay special attention to sections like:

* `global.postgresql`: Global “postgresqlPassword”, replication settings, etc.
* `postgresql.primary`: Settings for the primary database (resources, persistence, extendedConfiguration).
* `postgresql.secondary`: Replica (standby) configuration (or `replicaCount`).
* `pgpool`: Pgpool-II settings (enabled, resources, service type).
* `persistence`: Default PVC sizes.

Reading the default values helps you understand which keys you’ll override. ([artifacthub.io][1])

---

## 3. Create a Custom `values.yaml`

Below is an example `values.yaml` that customizes a small PostgreSQL HA cluster. Adjust all passwords and resource sizes to your environment’s needs. Save this file as `pg-ha-values.yaml`.

```yaml
# pg-ha-values.yaml

## 1. Global Passwords and Replication
global:
  postgresql:
    # Sets the superuser password for the primary instance
    postgresqlPassword: "SuperSecretPrimaryPass"
    # Configure replication user and password
    replication:
      user: "repl_user"
      password: "ReplUserPass"
    # Force replication to use streaming replication
    overlay: {}  # no additional overlays

## 2. Primary Node Customizations
postgresql:
  primary:
    replicaCount: 1            # Always 1 primary
    image:
      tag: "15.5.0"             # PostgreSQL version (must match chart’s supported versions)
      registry: docker.io/bitnami
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1"
    persistence:
      enabled: true
      storageClass: "standard"  # Change to your cluster’s StorageClass
      size: 10Gi
    ## Example: overriding a PostgreSQL setting (wal_level) via extendedConfiguration
    extendedConfiguration: |-
      wal_level = logical
      max_wal_senders = 5
      max_replication_slots = 5

## 3. Standby (Replica) Nodes
postgresql:
  secondary:
    replicaCount: 2            # Two standby replicas
    image:
      tag: "15.5.0"
      registry: docker.io/bitnami
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1"
    persistence:
      enabled: true
      storageClass: "standard"
      size: 10Gi

## 4. Pgpool-II (Load Balancer) Settings
pgpool:
  enabled: true
  image:
    tag: "5.4.4"               # Pgpool-II version compatible with the chart
    registry: docker.io/bitnami
  replicaCount: 1
  service:
    type: ClusterIP             # Exposes Pgpool-II on ClusterIP service
    port: 5432
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"

## 5. Metrics (Optional)
metrics:
  enabled: false               # Set to true if you have Prometheus/Grafana stack

## 6. Persistence for Pgpool (Optional; usually ephemeral is fine)
pgpool:
  persistence:
    enabled: false             # Pgpool state is ephemeral; set true if you need sticky sessions
```

### Explanations and Sources

1. **Global Replication Settings**

    * `global.postgresql.postgresqlPassword`: Sets the superuser (“postgres”) password for the primary.
    * `global.postgresql.replication.user` and `global.postgresql.replication.password`: Credentials for streaming replication.

   ([artifacthub.io][1], [datree.io][2])

2. **Primary Node (`postgresql.primary`)**

    * `image.tag` and `registry`: Pin PostgreSQL version for consistency.
    * `resources`: Guarantee minimum CPU/memory for primary.
    * `persistence.size`: Size of the primary’s PVC (e.g., 10 Gi).
    * `extendedConfiguration`: Injects extra directives into `postgresql.conf` (e.g., `wal_level = logical`).

        * This uses the same technique as overriding `postgresql.conf` in the single‐node chart (see StackOverflow) ([stackoverflow.com][3]).

3. **Secondary Nodes (`postgresql.secondary`)**

    * `replicaCount`: Number of standby replicas (2 in this example).
    * Each replica will have its own PVC of size 10 Gi.

   ([artifacthub.io][1], [datree.io][2])

4. **Pgpool-II Settings**

    * `enabled: true` deploys Pgpool-II as a separate Deployment and Service.
    * Pgpool-II listens on port 5432 (like a typical PostgreSQL).
    * `replicaCount: 1` is fine for small clusters; you can increase for high availability of Pgpool-II itself.

   ([artifacthub.io][1], [datree.io][2])

> **Note**: Always quote passwords (`"SuperSecretPrimaryPass"`) to avoid YAML parsing issues.

---

## 4. Install the PostgreSQL HA Cluster

With `pg-ha-values.yaml` ready, install the chart:

```bash
helm install pg-ha-cluster \
  --namespace database \
  --create-namespace \
  -f pg-ha-values.yaml \
  bitnami/postgresql-ha
```

([artifacthub.io][1])

* **`pg-ha-cluster`**: Release name.
* **`--namespace database`**: Installs into `database` namespace (created if it doesn’t exist).
* **`-f pg-ha-values.yaml`**: Uses our custom overrides.

---

## 5. Verify Deployment

After installation, check all pods, services, and PVCs:

```bash
kubectl get pods -n database
```

You should see something like:

```
NAME                                              READY   STATUS    RESTARTS   AGE
pg-ha-cluster-postgresql-ha-primary-0             1/1     Running   0          1m
pg-ha-cluster-postgresql-ha-secondary-0           1/1     Running   0          1m
pg-ha-cluster-postgresql-ha-secondary-1           1/1     Running   0          1m
pg-ha-cluster-postgresql-ha-pgpool-566f664b85-xyz 1/1     Running   0          1m
```

Check services:

```bash
kubectl get svc -n database
```

Expect at least:

```
NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
pg-ha-cluster-postgresql-ha-primary          ClusterIP   10.96.0.123     <none>        5432/TCP   1m
pg-ha-cluster-postgresql-ha-secondary        ClusterIP   10.96.0.124     <none>        5432/TCP   1m
pg-ha-cluster-postgresql-ha-pgpool           ClusterIP   10.96.0.125     <none>        5432/TCP   1m
```

Check PVCs:

```bash
kubectl get pvc -n database
```

You should see three PVCs (one for primary, two for replicas) of 10 Gi each. ([artifacthub.io][1])

---

## 6. Connect to PostgreSQL via Pgpool-II

By default, the chart exposes Pgpool-II on a ClusterIP service named `pg-ha-cluster-postgresql-ha-pgpool` on port 5432. To connect from within the cluster or via a local `kubectl port-forward`, do the following.

### 6.1. Port-Forward from Your Local Machine

```bash
kubectl port-forward svc/pg-ha-cluster-postgresql-ha-pgpool 5432:5432 -n database
```

Now, on your local machine you can use `psql` (or any PostgreSQL client) to connect:

```bash
psql "host=127.0.0.1 port=5432 user=postgres password=SuperSecretPrimaryPass"
```

* **User**: `postgres` (default superuser).
* **Password**: As set in `global.postgresql.postgresqlPassword` (“SuperSecretPrimaryPass”).

### 6.2. Basic SQL Check

Once connected, verify the primary/replica status:

```sql
-- On pgpool-II, check which node is primary
SHOW pool_nodes;
```

Example output:

```
 node_hostname | node_port | node_status | node_weight | replication_state |         load_balance_node         | down_after_failover |  last_status_change  | connection_count
---------------+-----------+-------------+-------------+-------------------+------------------------------------+---------------------+----------------------+------------------
 pg-ha-cluster-postgresql-ha-primary-0    |      5432 |        up |           1 |                 primary |            on            |                  30 | 2025-06-05 12:10:00 |               0
 pg-ha-cluster-postgresql-ha-secondary-0  |      5432 |        up |           1 |               standby | off |                  30 | 2025-06-05 12:11:00 |               0
 pg-ha-cluster-postgresql-ha-secondary-1  |      5432 |        up |           1 |               standby | off |                  30 | 2025-06-05 12:11:00 |               0
```

* **`node_status = up`** indicates healthy nodes.
* **`replication_state = primary/standby`** shows which is primary vs. replica.

---

## 7. Testing Failover

1. **Simulate Failure**: Delete the primary pod to force a failover:

   ```bash
   kubectl delete pod pg-ha-cluster-postgresql-ha-primary-0 -n database
   ```

2. **Watch Replica Promotion**: Within \~30 seconds, one of the standby pods will be promoted to primary by Repmgr and Pgpool-II. Check logs:

   ```bash
   kubectl logs -f pg-ha-cluster-postgresql-ha-pgpool-<pod-id> -n database
   ```

   Look for lines indicating a node switch to `primary`.

3. **Verify New Primary**:

   ```bash
   kubectl get pods -n database
   ```

   A new primary will bear the “primary” label. Then, check `SHOW pool_nodes;` again from `psql` to see the updated replication roles.

4. **Re-Create Old Primary**: Kubernetes will spin up a new pod as a standby replica to replace the old primary. Wait for all pods to return to `Ready` state.

Failover should be automatic and transparent to clients connected via Pgpool-II. ([artifacthub.io][1], [datree.io][2])

---

## 8. Modifying Configuration Post-Installation

### 8.1. Patching via `helm upgrade`

If you need to change any of the custom values—say you want to increase the primary’s PVC to 20 Gi, or adjust `wal_level` further—simply update `pg-ha-values.yaml` and run:

```bash
# Update persistence size for primary to 20Gi
yq eval '.postgresql.primary.persistence.size = "20Gi"' -i pg-ha-values.yaml

# Then upgrade
helm upgrade pg-ha-cluster \
  --namespace database \
  -f pg-ha-values.yaml \
  bitnami/postgresql-ha
```

* This resizes the primary’s PVC (if your storage class supports volume expansion).
* Kubernetes will handle the in-place expansion.

> **Overriding `postgresql.conf`**:
> To further customize `postgresql.conf`, use the `extendedConfiguration` block under `postgresql.primary` and/or `postgresql.secondary`. For example, to enable logical replication slots and adjust checkpoint settings:
>
> ```yaml
> postgresql:
>   primary:
>     extendedConfiguration: |-
>       wal_level = logical
>       max_wal_senders = 10
>       max_replication_slots = 10
>       checkpoint_timeout = '5min'
> ```
>
> After editing, run the same `helm upgrade ... -f pg-ha-values.yaml`. Reboot of the Pod may be required to pick up certain settings. ([stackoverflow.com][3])

### 8.2. Adding Additional Standby Replicas

To scale out the number of standbys, increase `secondary.replicaCount`. For example, to go from 2 to 3 replicas:

```bash
yq eval '.postgresql.secondary.replicaCount = 3' -i pg-ha-values.yaml
helm upgrade pg-ha-cluster \
  --namespace database \
  -f pg-ha-values.yaml \
  bitnami/postgresql-ha
```

You’ll see a new standby Pod spin up. Pgpool-II will automatically include it in its load-balancing pool. ([artifacthub.io][1], [datree.io][2])

---

## 9. Upgrading the Chart Version

If Bitnami releases a newer PostgreSQL‐HA chart (e.g., v10.x → v11.x), you can upgrade as follows:

1. **Update Repo**:

   ```bash
   helm repo update
   ```
2. **Check Current Version**:

   ```bash
   helm list -n database
   ```

   Note the current chart version under `CHART`.
3. **Upgrade to Latest**:

   ```bash
   helm upgrade pg-ha-cluster \
     --namespace database \
     -f pg-ha-values.yaml \
     bitnami/postgresql-ha --version 10.0.0
   ```

   Replace `10.0.0` with the desired version. Upgrades are generally safe, but consult the Bitnami chart’s CHANGELOG for breaking changes. ([artifacthub.io][1], [datree.io][2])

---

## 10. Uninstalling and Cleaning Up

To delete the PostgreSQL HA cluster and all associated resources:

```bash
helm uninstall pg-ha-cluster -n database
kubectl delete namespace database
```

* This removes Deployments, Services, PVCs, and Secrets generated by the chart.
* Verify no PVCs remain:

  ```bash
  kubectl get pvc -A | grep pg-ha
  ```

  If any remain, delete them explicitly:

  ```bash
  kubectl delete pvc <pvc-name> -n database
  ```

---

## 11. Summary of Key Values

Below is a succinct summary of the main customization keys used in `pg-ha-values.yaml`:

| Section                | Key                                        | Description                                     |
| ---------------------- | ------------------------------------------ | ----------------------------------------------- |
| `global.postgresql`    | `postgresqlPassword`                       | Superuser (“postgres”) password                 |
|                        | `replication.user`, `replication.password` | Credentials for replication management          |
| `postgresql.primary`   | `image.tag`, `image.registry`              | PostgreSQL image version and registry           |
|                        | `resources.requests/limits`                | CPU/Memory requests and limits for primary      |
|                        | `persistence.size`                         | PVC size for primary                            |
|                        | `extendedConfiguration`                    | Extra settings to inject into `postgresql.conf` |
| `postgresql.secondary` | `replicaCount`                             | Number of standby replicas                      |
|                        | `resources.requests/limits`                | CPU/Memory for each replica                     |
|                        | `persistence.size`                         | PVC size for each replica                       |
| `pgpool`               | `enabled`                                  | Toggles Pgpool-II deployment                    |
|                        | `image.tag`, `image.registry`              | Pgpool-II image version/registry                |
|                        | `replicaCount`                             | Number of Pgpool-II pods                        |
|                        | `service.type`, `service.port`             | Service type and port for Pgpool-II             |
|                        | `resources.requests/limits`                | CPU/Memory for Pgpool-II pods                   |
| `metrics`              | `enabled`                                  | Enable Prometheus exporter                      |

---

## Additional Notes and Best Practices

1. **Secrets Management**

    * The chart automatically creates a Kubernetes `Secret` named `<release>-postgresql-ha` containing `postgresql-password`, `repmgr-password`, and `pgpool-password`. If you need to rotate or externalize secrets, consider using [Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets).

2. **Security Contexts and Network Policies**

    * For production, consider enforcing Pod security policies or OpenShift security context constraints (SCCs) to ensure containers run with non‐root users.
    * Use Kubernetes NetworkPolicies to restrict which Pods or namespaces can talk to your PostgreSQL cluster.

3. **Backups**

    * This chart doesn’t natively include a backup solution. Integrate tools like [Kubernetes CronJobs + `pg_dump`](https://www.datadoghq.com/blog/backup-postgresql-kubernetes/) or use solutions such as [Stash by AppsCode](https://stash.run) for scheduled snapshots and restores.

4. **Monitoring**

    * Enabling the built‐in metrics exporter (set `metrics.enabled: true`) allows Pgpool-II and PostgreSQL to expose Prometheus metrics. Pair this with Grafana dashboards for health and performance monitoring.

5. **High Availability for Pgpool-II**

    * In front of `pgpool`, you might want a Kubernetes `Deployment` with multiple replicas behind a `Service` of `type: LoadBalancer` or `NodePort`. That way, if a Pgpool-II pod fails, another takes over. Adjust `pgpool.replicaCount` accordingly.

---

### Citations

1. **Bitnami PostgreSQL HA Chart (Artifact Hub)**:

    * “The PostgreSQL HA Helm chart deploys a cluster with four nodes by default…” ([artifacthub.io][1])
2. **Customizing `postgresql.conf`**:

    * Use `extendedConfiguration` under `primary` to override `postgresql.conf` keys (e.g., `wal_level = logical`) ([stackoverflow.com][3])
3. **Deployment Overview**:

    * This chart is based on Bitnami’s single-node PostgreSQL chart, but includes “changes to guarantee high availability such as: new deployments for replicas and pgpool-II…” ([datree.io][2])

---

By following this tutorial, you’ll have a fully functional, clustered PostgreSQL deployment on Kubernetes with automatic primary/replica failover, connection pooling via Pgpool-II, and custom configuration settings. Adjust any of the `values.yaml` parameters to suit your environment’s resource quotas, storage classes, and security policies. Happy Helming!

[1]: https://artifacthub.io/packages/helm/bitnami/postgresql-ha?utm_source=chatgpt.com "postgresql-ha 16.0.10 · bitnami/bitnami - Artifact Hub"
[2]: https://www.datree.io/helm-chart/postgresql-ha-bitnami?utm_source=chatgpt.com "Postgresql-ha Helm Chart - Datree"
[3]: https://stackoverflow.com/questions/71214664/how-to-change-bitnami-postgresql-helm-chart-configs-ph-hba-conf-postgresql-co?utm_source=chatgpt.com "How to change bitnami postgresql helm chart configs? (ph_hba.conf ..."

Below is a focused explanation of how “volumes” (that is, Persistent Volumes and Persistent Volume Claims) are used and configured when deploying the Bitnami PostgreSQL HA chart with Helm. We’ll cover:

## 1. Kubernetes Storage Concepts: PV and PVC

* **PersistentVolume (PV)**
  A PV is a piece of network/storage infrastructure (e.g., AWS EBS, GCE PD, NFS, hostPath) that has been provisioned into the cluster. It describes the underlying storage resource—its type, capacity, access modes (e.g., `ReadWriteOnce`, `ReadWriteMany`), reclaim policy, etc.

* **PersistentVolumeClaim (PVC)**
  A PVC is a request for storage by a user (in this case, the Helm chart). It specifies desired storage capacity, access modes, and (optionally) a `StorageClass`. Kubernetes’ control plane will bind that claim to an appropriate PV (either dynamically provisioned or statically bound) that satisfies the request.

When you deploy a Helm chart, the chart’s templates typically include a `PersistentVolumeClaim` definition (often via a `volumeClaimTemplates` block inside a `StatefulSet` or as stand-alone PVC objects). At runtime, Kubernetes ensures that each PVC is bound to a backing PV, so that your database data lives on durable storage rather than ephemeral container disk.

---

## 2. How the PostgreSQL HA Chart Defines PVCs

The Bitnami `postgresql-ha` chart ([https://artifacthub.io/packages/helm/bitnami/postgresql-ha](https://artifacthub.io/packages/helm/bitnami/postgresql-ha)) does the following:

* **Primary Node (StatefulSet)**
  The primary Postgres node is deployed as a StatefulSet named `<release-name>-postgresql-ha-primary`. Under the hood, the chart’s `templates/primary/statefulset.yaml` (or similar) includes a `volumeClaimTemplates:` section. That section references `.Values.postgresql.primary.persistence.*` to create a PVC for the primary’s data directory (`/bitnami/postgresql/data`). By default:

  ```yaml
  postgresql:
    primary:
      persistence:
        enabled: true             # whether to create a PVC
        storageClass: ""          # which StorageClass to use
        accessModes:
          - ReadWriteOnce         # default access mode
        size: 8Gi                 # default size
        annotations: {}           # any PVC annotations
        existingClaim: ""         # if non-empty, use an existing PVC instead of creating a new one
  ```

  If `enabled: true`, Helm will render something like:

  ```yaml
  volumeClaimTemplates:
    - metadata:
        name: data               # name used by StatefulSet to create a PVC called <STSET-NAME>-data-<ordinal>
        annotations: {{ .Values.postgresql.primary.persistence.annotations }}
      spec:
        accessModes: {{ .Values.postgresql.primary.persistence.accessModes }}
        storageClassName: {{ .Values.postgresql.primary.persistence.storageClass | quote }}
        resources:
          requests:
            storage: {{ .Values.postgresql.primary.persistence.size | quote }}
  ```

  Kubernetes will then create a PVC binding to a PV. For example, if your release is `pg-ha-cluster`:

  ```
  NAME                                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  data-pg-ha-cluster-postgresql-ha-primary-0      Bound    pvc-1234abcd-...                          8Gi        RWO            standard       2m
  ```

* **Replica (Secondary) Nodes (StatefulSet or Deployment)**
  Likewise, each standby replica is deployed in a StatefulSet (named `<release>-postgresql-ha-secondary-0`, `<release>-postgresql-ha-secondary-1`, etc.). The chart uses `.Values.postgresql.secondary.persistence.*` to create one PVC per-replica. By default, those values mirror `primary.persistence.*`:

  ```yaml
  postgresql:
    secondary:
      persistence:
        enabled: true
        storageClass: ""
        accessModes:
          - ReadWriteOnce
        size: 8Gi
        annotations: {}
        existingClaim: ""
  ```

  This ensures each replica has its own independent PVC.

* **Pgpool-II (Optional)**
  By default, `pgpool.persistence.enabled` is `false`. Pgpool-II usually doesn’t need durable data, but if you do want it to keep state (for example, to store session affinity info), you can turn on:

  ```yaml
  pgpool:
    persistence:
      enabled: true
      storageClass: "standard"
      accessModes:
        - ReadWriteOnce
      size: 1Gi
      annotations: {}
      existingClaim: ""
  ```

---

## 3. Customizing Volume Settings in `values.yaml`

Below is a minimal example excerpt from `pg-ha-values.yaml` showing how you might override these defaults. This snippet ensures the primary, replicas, and (optionally) Pgpool-II all use 10 Gi of durable storage on a specific StorageClass named `standard`:

```yaml
postgresql:
  primary:
    persistence:
      enabled: true
      storageClass: "standard"
      accessModes:
        - ReadWriteOnce
      size: 10Gi
      annotations:
        volume.beta.kubernetes.io/storage-provisioner: "kubernetes.io/aws-ebs"
      existingClaim: ""

  secondary:
    persistence:
      enabled: true
      storageClass: "standard"
      accessModes:
        - ReadWriteOnce
      size: 10Gi
      annotations: {}
      existingClaim: ""

pgpool:
  persistence:
    enabled: false   # keep pgpool ephemeral; set to true if you need durable caching
    storageClass: ""
    accessModes:
      - ReadWriteOnce
    size: 1Gi
    annotations: {}
    existingClaim: ""
```

* **`storageClass: "standard"`** instructs Kubernetes to dynamically provision a matching PV via your cluster’s “standard” StorageClass (e.g., AWS EBS, GCE PD, or other CSI driver).
* **`size: 10Gi`** requests a volume of 10 GiB.
* **`accessModes: ["ReadWriteOnce"]`** means only one Pod at a time can mount in read-write mode. (Replicas each get their own dedicated PVC.)
* **`annotations`** can be used to target a specific provisioner or pass extra hints (e.g., in AWS EBS you might annotate with snapshot IDs or IOPS requirements).
* **`existingClaim: ""`** is blank here, so Helm creates a new PVC. If you prefer to bind to a PVC you created manually (perhaps because you want to use a hostPath PV or an NFS volume), set `existingClaim` to the exact name of that PVC, and Helm will skip creating a new one.

Once you’ve updated these fields, installing (or upgrading) the chart automatically generates or adjusts the PVC specs under the hood. For example:

```bash
helm upgrade \
  pg-ha-cluster \
  --namespace database \
  -f pg-ha-values.yaml \
  bitnami/postgresql-ha
```

Helm then renders new Kubernetes YAML for the StatefulSets, which includes updated `volumeClaimTemplates` for primary and secondary.

---

## 4. Using an Existing PVC or a Statically-Provisioned PV

Sometimes you want complete control over the underlying PV (e.g., a hostPath, an NFS share, or a specific SSD). In that case:

1. **Create a PV by hand**. For example, if you want to back your primary on a hostPath:

   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: postgresql-primary-pv
     labels:
       type: local
   spec:
     storageClassName: manual
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: /mnt/data/pg-primary
       type: DirectoryOrCreate
   ```

   ([travis.media][1])

2. **Create a matching PVC** that the chart will bind to (the PVC must sit in the same namespace where you install the release—e.g., the `database` namespace):

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: pg-primary-claim
     namespace: database
   spec:
     storageClassName: manual
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
   ```

3. **Point the chart at that existing claim**. In your `values.yaml`, set:

   ```yaml
   postgresql:
     primary:
       persistence:
         enabled: true
         existingClaim: "pg-primary-claim"
         # storageClass, size, accessModes, and annotations are ignored
   ```

   Helm will detect `existingClaim` and skip creating a new PVC for the primary; instead, it uses `pg-primary-claim`. ([stackoverflow.com][2])

You can repeat the above for replicas (e.g., create `pg-replica-claim-0`, `pg-replica-claim-1` and set `postgresql.secondary.persistence.existingClaim` accordingly), although a more common pattern is to let Kubernetes dynamically provision each replica’s PVC.

---

## 5. Access Modes, StorageClasses, and Resizing

### 5.1. Access Modes

* **`ReadWriteOnce (RWO)`** – Mounted as ReadWrite by a single node. This is the most common mode for Postgres data.
* **`ReadWriteMany (RWX)`** – Mounted as ReadWrite by many nodes. Useful if you have an NFS or CSI that provides shared storage, but **not** recommended for standalone Postgres data directories.
* **`ReadOnlyMany (ROX)`** – Mounted as ReadOnly by many nodes (rarely used for Postgres data).

By default, the Bitnami chart sets `accessModes: ["ReadWriteOnce"]`. If your StorageClass doesn’t support `ReadWriteOnce`, you must supply a StorageClass that does, or supply your own PV/PVC manually. ([stackoverflow.com][2])

### 5.2. StorageClass Selection

* If you leave `storageClass: ""`, Kubernetes uses the default StorageClass in your cluster.
* If you set a specific name (e.g. `"standard"` or `"gp2"`), that exact StorageClass is used for dynamic provisioning.
* If you want a completely static PV (manually created), set `existingClaim`.

### 5.3. Volume Resizing

If your cloud provider and StorageClass support volume expansion, you can resize a PVC by:

1. Updating your `values.yaml`. For example, bump the primary PVC from `10Gi` → `20Gi`:

   ```yaml
   postgresql:
     primary:
       persistence:
         size: 20Gi
   ```
2. Running:

   ```bash
   helm upgrade pg-ha-cluster \
     --namespace database \
     -f pg-ha-values.yaml \
     bitnami/postgresql-ha
   ```
3. Kubernetes (if your StorageClass allows expansion) performs an “in-place expansion” of the underlying volume.
4. You may need to restart the Postgres Pod or allow Kubernetes to rollout a new pod so that the file system sees the additional space.

   **Note**: On some local setups (e.g., `hostPath`) you might have to handle resizing manually at the node level. ([stackoverflow.com][2])

---

## 6. Inspecting and Troubleshooting Volumes

Once the chart has been installed or upgraded, you can verify PVC creation and binding status:

1. List PVCs in the `database` namespace:

```bash
kubectl get pvc -n database
```

Expected output (example):

```
NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-pg-ha-cluster-postgresql-ha-primary-0  Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWO            standard       5m
data-pg-ha-cluster-postgresql-ha-secondary-0  Bound  pvc-yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy  10Gi       RWO            standard       5m
data-pg-ha-cluster-postgresql-ha-secondary-1  Bound  pvc-zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz   10Gi       RWO            standard       4m
```

2. Check PVs that back those PVCs:

```bash
kubectl get pv
```

You’ll see which PV objects were dynamically provisioned or which static PVs are bound to your PVCs.
3\.  Describe a PVC if it’s stuck in `Pending`:

```bash
kubectl describe pvc data-pg-ha-cluster-postgresql-ha-primary-0 -n database
```

This often shows why it can’t bind (e.g., no matching StorageClass, insufficient capacity, or a typo in `storageClass`).

---

## 7. Disabling Persistence

If you want a purely ephemeral Postgres (for testing), you can turn off persistence altogether:

```yaml
postgresql:
  primary:
    persistence:
      enabled: false
  secondary:
    persistence:
      enabled: false
pgpool:
  persistence:
    enabled: false
```

This means every time a Pod restarts (or on a chart upgrade without a matching PVC), data is lost. Only use `enabled: false` in non‐production or throwaway environments. ([stackoverflow.com][2])

---

## 8. Summary of Key PVC/Volume-Related Values

Below is a concise checklist of the most important `values.yaml` keys related to volumes:

| Section                                          | Key                                        | Purpose                                                                              |
| ------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------------------------------ |
| `postgresql.primary.persistence.enabled`         | `true` / `false`                           | Whether to create a PVC for the primary’s data directory.                            |
| `postgresql.primary.persistence.storageClass`    | `"<your-storage-class>"` / `""`            | Which StorageClass to request (leave blank for the default StorageClass).            |
| `postgresql.primary.persistence.accessModes`     | `["ReadWriteOnce"]`                        | Access mode for the primary’s PVC.                                                   |
| `postgresql.primary.persistence.size`            | e.g., `"8Gi"`, `"10Gi"`                    | Size of the primary’s PVC.                                                           |
| `postgresql.primary.persistence.annotations`     | `{}` or custom map                         | Any annotations to add to the PVC (e.g., provisioner hints).                         |
| `postgresql.primary.persistence.existingClaim`   | `""` or `"my-precreated-pvc"`              | If non-empty, use this existing PVC and don’t create a new one.                      |
| `postgresql.secondary.persistence.enabled`       | `true` / `false`                           | Whether to create PVCs for each replica.                                             |
| `postgresql.secondary.persistence.storageClass`  | Same as primary                            | StorageClass for replica PVCs.                                                       |
| `postgresql.secondary.persistence.accessModes`   | Same as primary                            | Replica access mode.                                                                 |
| `postgresql.secondary.persistence.size`          | Same as primary                            | Replica PVC size.                                                                    |
| `postgresql.secondary.persistence.existingClaim` | `""` or `"precreated-replica-pvc-0"`       | If non-empty, use this PVC for replica 0 (and so on for additional replicas).        |
| `pgpool.persistence.enabled`                     | `true` / `false`                           | Whether to give Pgpool-II a PVC (most clusters do not need it).                      |
| `pgpool.persistence.storageClass`                | `"<your-storage-class>"` / `""`            | StorageClass for Pgpool-II’s PVC.                                                    |
| `pgpool.persistence.accessModes`                 | `["ReadWriteOnce"]` or `["ReadWriteMany"]` | Access modes for Pgpool-II PVC.                                                      |
| `pgpool.persistence.size`                        | e.g., `"1Gi"`                              | Size for Pgpool-II’s PVC (usually small if you use it at all).                       |
| `pgpool.persistence.existingClaim`               | `""` or `"existing-pgpool-pvc"`            | If non-empty, bind Pgpool-II to an existing PVC (e.g., for a hostPath or NFS share). |

