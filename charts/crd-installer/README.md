# CRD Installer

A generic Helm chart for installing Kubernetes CRDs (Custom Resource Definitions) from remote URLs.

## Overview

This chart uses Helm pre-install/pre-upgrade hooks to download and apply CRD YAML manifests
before the main release is deployed. It is idempotent — running it multiple times will not
duplicate or remove existing CRDs.

## Features

- Install any set of CRDs from any public URL
- Idempotent `kubectl apply` — safe to run repeatedly
- Helm hooks ensure CRDs exist before the main chart resources
- Automatic cleanup of hook resources via `hook-delete-policy`
- Tolerates all node taints (control-plane, NoSchedule, NoExecute) for single-node clusters
- `hostNetwork: true` for reliable API server connectivity

## Usage

### Install

```bash
helm install my-crds ./charts/crd-installer \
  --set crdInstaller.urls[0]=https://example.com/crds/my-crd.yaml
```

Multiple CRDs:

```bash
helm install my-crds ./charts/crd-installer \
  --set crdInstaller.urls[0]=https://example.com/crds/crd-a.yaml \
  --set crdInstaller.urls[1]=https://example.com/crds/crd-b.yaml \
  --set crdInstaller.urls[2]=https://example.com/crds/crd-c.yaml
```

### Using a values file

```yaml
# my-values.yaml
crdInstaller:
  enabled: true
  image:
    repository: alpine/kubectl
    tag: "1.33.3"
  urls:
    - https://raw.githubusercontent.com/example/project/v1.0/config/crd/crd1.yaml
    - https://raw.githubusercontent.com/example/project/v1.0/config/crd/crd2.yaml
```

```bash
helm install my-crds ./charts/crd-installer -f my-values.yaml
```

### Upgrade

```bash
helm upgrade my-crds ./charts/crd-installer -f my-values.yaml
```

### Uninstall

```bash
helm uninstall my-crds
```

> **Note:** Uninstalling the chart does **not** remove the CRDs. CRDs are cluster-scoped
> resources and are never deleted by Helm hooks. To remove CRDs, you must delete them
> manually with `kubectl delete crd <name>`.

## Configuration

| Parameter | Description | Default |
|---|---|---|
| `crdInstaller.enabled` | Enable CRD installation | `true` |
| `crdInstaller.image.repository` | Container image for the installer job | `alpine/kubectl` |
| `crdInstaller.image.tag` | Image tag | `1.33.3` |
| `crdInstaller.name` | Base name for generated Kubernetes resources | `crd-installer` |
| `crdInstaller.urls` | List of URLs to CRD YAML manifests | `[]` (see values.yaml) |

## How It Works

On `helm install` or `helm upgrade`, the chart creates the following resources in order:

| Weight | Resource | Purpose |
|---|---|---|
| `-4` | **ServiceAccount** | Identity for the installer pod |
| `-3` | **ClusterRole** | Permissions to manage CRDs |
| `-2` | **ClusterRoleBinding** | Bind SA to ClusterRole |
| `-1` | **Job** | Downloads and applies CRDs via `kubectl apply -f` |

All resources are annotated with:

- `helm.sh/hook: pre-install,pre-upgrade` — runs before the release is installed/upgraded
- `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded,hook-failed` — cleans up
  after execution

After the job succeeds, the main chart resources (if any) are installed. If the job fails,
the installation/upgrade is aborted.

## Prerequisites

- Kubernetes 1.19+
- Helm 3+

## Examples

### Install Gateway API CRDs

```bash
helm install gateway-api ./charts/crd-installer \
  --set crdInstaller.urls[0]=https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.3.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml \
  --set crdInstaller.urls[1]=https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.3.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml \
  --set crdInstaller.urls[2]=https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.3.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml \
  --set crdInstaller.urls[3]=https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.3.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
```

### Install Solr Operator CRDs

```bash
helm install solr-crds ./charts/crd-installer \
  --set crdInstaller.urls[0]=https://solr.apache.org/operator/downloads/crds/0.9.1/all-with-dependencies.yaml
```

### Install CRDs from a private release artifact (no auth required)

```bash
helm install custom-crds ./charts/crd-installer \
  --set crdInstaller.urls[0]=https://github.com/my-org/my-repo/releases/download/v1.0/crds.yaml
```
