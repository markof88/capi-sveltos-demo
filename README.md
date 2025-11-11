# CAPI + Sveltos on Azure: From Zero to GitOps

This repository demonstrates how to use **Cluster API (CAPI)** to provision a Kubernetes cluster on **Microsoft Azure**, install **Cloud Provider Azure**, deploy **Cilium** as the CNI (with Hubble), and then prepare for a **GitOps-style Day‑2 management** workflow with **Sveltos**.

> **Public repo notice:** All IDs, secrets, and personally identifying values must be replaced with placeholders. Do **not** commit real subscription IDs, client secrets, or SSH keys.

---

## Table of Contents

* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Quick Start](#quick-start)
* [1) Create a local management cluster (kind)](#1-create-a-local-management-cluster-kind)
* [2) One-time Azure setup](#2-one-time-azure-setup)
* [3) Export environment variables](#3-export-environment-variables)
* [4) Initialize Cluster API on the management cluster](#4-initialize-cluster-api-on-the-management-cluster)
* [5) Generate a workload-cluster manifest](#5-generate-a-workload-cluster-manifest)
* [6) Apply the manifest](#6-apply-the-manifest)
* [7) Fetch kubeconfig for the new cluster](#7-fetch-kubeconfig-for-the-new-cluster)
* [8) Install Cloud Provider Azure (CCM/CNM)](#8-install-cloud-provider-azure-ccmcnm)
* [9) Install Cilium (CNI) + Hubble](#9-install-cilium-cni--hubble)
* [10) Quick smoke test](#10-quick-smoke-test)
* [Troubleshooting](#troubleshooting)
* [Cleanup](#cleanup)
* [Next: Manage with Sveltos (GitOps)](#next-manage-with-sveltos-gitops)
* [Repository hygiene](#repository-hygiene)
* [License](#license)

---

## Architecture

**Local (your laptop/WSL/macOS/Linux):**

* A single-node **kind** cluster acts as the **management** cluster that runs the CAPI controllers (Core, Kubeadm CP/Bootstrap, CAPZ for Azure, and cert-manager).

**Azure:**

* A CAPI-provisioned **workload cluster** (control plane + worker nodes).
* **Cloud Provider Azure** for node lifecycle and cloud integration.
* **Cilium** for networking (kube-proxy replacement enabled) with **Hubble** UI.

---

## Prerequisites

* **Azure** subscription and the **Azure CLI** (`az`) logged in: `az login`
* **kubectl**, **helm ≥ 3.17**, **kind v0.30.0**, **clusterctl v1.11.1**, **yq**
* Tested from a standard terminal on Linux/WSL/macOS

> Versions listed are those used in this guide; newer patch versions generally work.

---

## Quick Start

```bash
# Clone your repo and enter it
git clone git@github.com:<YOUR_GH_USER>/capi-sveltos-demo.git
cd capi-sveltos-demo

# Optional: create Python venv/direnv, etc.
```

Then follow sections 1 → 10 below.

---

## 1) Create a local management cluster (kind)

Save as `kind_cluster_extramounts.yaml`:

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: mgmt
networking:
  ipFamily: dual
nodes:
- role: control-plane
  extraMounts:
    - hostPath: /var/run/docker.sock
      containerPath: /var/run/docker.sock
```

Create & verify:

```bash
kind create cluster --config kind_cluster_extramounts.yaml
kubectl cluster-info --context kind-mgmt
kubectl get nodes
```

> The node may show **NotReady** until CAPI and cert-manager are installed in the next steps.

---

## 2) One-time Azure setup

> Replace all placeholder values like `<AZURE_SUBSCRIPTION_ID>` with your own. **Do not commit real values.**

### 2.1 Resource Group

```bash
az group create --name <AZURE_RESOURCE_GROUP> --location <AZURE_LOCATION>
# Example: az group create --name capi-test --location germanywestcentral
```

### 2.2 Managed Identity (for cloud provider)

```bash
az identity create \
  --name cloud-provider-user-identity \
  --resource-group <AZURE_RESOURCE_GROUP> \
  --location <AZURE_LOCATION>
```

Record the following for later: **subscriptionId**, **tenantId**, **clientId**, and **resourceId** of this identity.

### 2.3 (Optional) Check CAPI image availability in your region

```bash
az sig image-version list-community \
  --public-gallery-name ClusterAPI-f72ceb4f-5159-4c26-a0fe-2ea738f0d019 \
  --gallery-image-definition capi-ubun2-2404 \
  --location <AZURE_LOCATION>
```

If your exact Kubernetes patch version isn’t available in your region yet, pick another `v1.34.x`.

### 2.4 App Registration / Service Principal

If you don’t already have one, create an SP and grant it access to the resource group (Contributor is sufficient for a demo):

```bash
az ad sp create-for-rbac \
  --name <SP_NAME> \
  --role Contributor \
  --scopes /subscriptions/<AZURE_SUBSCRIPTION_ID>/resourceGroups/<AZURE_RESOURCE_GROUP>
```

Record the outputs: `appId` (**CLIENT_ID**), `password` (**CLIENT_SECRET**), and `tenant` (**TENANT_ID**).

> 🔒 **Security:** Never commit your `CLIENT_SECRET` or any real IDs.

---

## 3) Export environment variables

```bash
export CLUSTER_TOPOLOGY=true

export AZURE_SUBSCRIPTION_ID="<AZURE_SUBSCRIPTION_ID>"
export AZURE_TENANT_ID="<AZURE_TENANT_ID>"

# Service Principal from step 2.4
export AZURE_CLIENT_ID="<APP_CLIENT_ID>"
export AZURE_CLIENT_SECRET="<APP_CLIENT_SECRET>"
export AZURE_CLIENT_ID_USER_ASSIGNED_IDENTITY=$AZURE_CLIENT_ID

# Secret/identity in mgmt cluster
export AZURE_CLUSTER_IDENTITY_SECRET_NAME="cluster-identity-secret"
export CLUSTER_IDENTITY_NAME="cluster-identity"
export AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE="default"

# Location / VM sizes / Resource Group
export AZURE_LOCATION="<AZURE_LOCATION>"             # e.g., germanywestcentral or francecentral
export AZURE_RESOURCE_GROUP="<AZURE_RESOURCE_GROUP>"  # e.g., capi-test
export AZURE_CONTROL_PLANE_MACHINE_TYPE="Standard_D2s_v3"
export AZURE_NODE_MACHINE_TYPE="Standard_D2s_v3"
```

Create the secret in the management cluster:

```bash
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" \
  --from-literal=clientSecret="${AZURE_CLIENT_SECRET}" \
  --namespace "${AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE}"
```

---

## 4) Initialize Cluster API on the management cluster

```bash
clusterctl init --infrastructure azure
kubectl get ns
kubectl get pods -A
```

Wait until the CAPI/CAPZ and cert-manager pods are **Running**.

> You may see warnings about v1beta1 deprecations. Newer providers use `v1beta2`. The demo works with the generated templates; you can mass‑update apiVersions later if desired.

---

## 5) Generate a workload-cluster manifest

```bash
clusterctl generate cluster az04 \
  --infrastructure azure \
  --kubernetes-version v1.34.1 \
  --control-plane-machine-count=3 \
  --worker-machine-count=2 \
  > az04_capi_cluster.yaml
```

**Important manual fixes in `az04_capi_cluster.yaml`:**

A) In `KubeadmControlPlane.spec.kubeadmConfigSpec.files[0].contentFrom.secret.name` set the secret name to **`az04-md-0-azure-json`** (to match the worker template name that gets generated).

B) In `AzureMachineTemplate.spec.template.spec.userAssignedIdentities[0].providerID` set your identity resource path, e.g.:

```
/subscriptions/<AZURE_SUBSCRIPTION_ID>/resourceGroups/<AZURE_RESOURCE_GROUP>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/cloud-provider-user-identity
```

Also ensure:

* `AzureCluster.spec.location` and `resourceGroup` match your values.
* `AzureCluster.spec.subscriptionID` uses `<AZURE_SUBSCRIPTION_ID>`.
* `AzureClusterIdentity.spec.clientID` and `.tenantID` reference your SP **IDs** (not secrets). The secret name/namespace should match step 3.
* `sshPublicKey` fields contain your **base64‑encoded** SSH public key, or switch templates to reference an external secret if preferred.

> Tip: Use `yq` to patch the file reproducibly and avoid manual edits.

---

## 6) Apply the manifest

```bash
kubectl apply -f az04_capi_cluster.yaml

# Watch progress
watch -n2 'kubectl get clusters; echo; kubectl get machines,azuremachines'
# Optional: view recent events
kubectl get events --sort-by=.lastTimestamp -A
```

In the Azure Portal, under the chosen resource group, you’ll see the VMs/resources being created.

---

## 7) Fetch kubeconfig for the new cluster

```bash
clusterctl get kubeconfig az04 > az04.kubeconfig
export KUBECONFIG=$PWD/az04.kubeconfig
kubectl get nodes -o wide
```

Nodes may be **NotReady** until the cloud provider and CNI are installed in steps 8–9.

---

## 8) Install Cloud Provider Azure (CCM/CNM)

```bash
helm install \
  --repo https://raw.githubusercontent.com/kubernetes-sigs/cloud-provider-azure/master/helm/repo \
  cloud-provider-azure \
  --generate-name \
  --set infra.clusterName=az04 \
  --set cloudControllerManager.clusterCIDR="192.168.0.0/16"

kubectl -n kube-system get pods | grep -i cloud
```

Expect `cloud-node-manager-*` to be **Running**; the `cloud-controller-manager` may be **Pending** until CNI is present.

---

## 9) Install Cilium (CNI) + Hubble

```bash
helm repo add cilium https://helm.cilium.io
helm repo update

helm install cilium cilium/cilium \
  --namespace kube-system --create-namespace \
  --version 1.17.7 \
  --set ipam.mode="cluster-pool" \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="{192.168.0.0/16}" \
  --set ipam.operator.clusterPoolIPv4MaskSize=24 \
  --set kubeProxyReplacement=true \
  --set bpf.masquerade=true \
  --set hubble.ui.enabled=true \
  --set hubble.relay.enabled=true

kubectl get pods -n kube-system
kubectl exec -it ds/cilium -n kube-system -- cilium status
kubectl get nodes
```

At this point all nodes should be **Ready**.

---

## 10) Quick smoke test

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=3
kubectl expose deployment nginx --port=80 --target-port=80
kubectl get pods,svc

kubectl run curl-test --image=busybox:latest --restart=Never -it --rm -- /bin/sh
# inside the shell
wget -qO- http://nginx
# you should see: "Welcome to nginx!"
```

---

## Troubleshooting

* **Deprecated apiVersion warnings:**

  * `cluster.x-k8s.io/v1beta1` (Cluster/MachineDeployment), `controlplane.cluster.x-k8s.io/v1beta1` (KCP), and `bootstrap.cluster.x-k8s.io/v1beta1` may warn about deprecation. Consider upgrading to `v1beta2` when convenient.

* **Nodes stay NotReady:**

  * Ensure CNI (Cilium) and Cloud Provider Azure are installed and healthy.
  * Check `kubelet` logs on nodes; verify `/etc/kubernetes/azure.json` exists from the secrets you referenced.

* **Cloud controller manager Pending:**

  * Expected until CNI is installed. After Cilium is healthy, the CCM should schedule and run.

* **Images not available in region:**

  * Use an alternative `v1.34.x` image that exists in your region (see step 2.3) or switch region.

* **Identity/permissions issues:**

  * Confirm the SP has **Contributor** on the resource group scope.
  * Ensure `AzureClusterIdentity` references the correct `clientID`, `tenantID`, secret **name/namespace**, and the `userAssignedIdentities[*].providerID` path is valid.

---

## Cleanup

From the **management** cluster context:

```bash
kubectl delete cluster az04
```

Wait for Azure resources to deprovision. Then you can delete the kind cluster:

```bash
kind delete cluster --name mgmt
```

---

## Next: Manage with Sveltos (GitOps)

With a healthy cluster, you can use **Sveltos** to apply and drift-manage add‑ons and policies from Git.

High‑level steps (detailed docs to be added in a separate folder):

1. **Install Sveltos** in the management or workload cluster (depending on your topology).
2. Define **ClusterProfiles** and **Profiles** that reference Git repositories of Helm charts/manifests.
3. Point Sveltos to your Git repository branches/tags and enable **drift remediation**.
4. Start with a small set of add‑ons (e.g., Metrics Server, Ingress Controller, Hubble UI exposure) and grow incrementally.

A `sveltos/` directory will be added with ready‑to‑use examples.

---

## Repository hygiene

* Add a `.gitignore` at the repo root:

<details>
<summary>.gitignore (click to expand)</summary>

```gitignore
# Kubernetes
*.kubeconfig
*.kube
kubeconfig

# Secrets and env
.env
*.secret
*.pem
*.key
secrets/**
!secrets/README.md

# OS / editor noise
.DS_Store
Thumbs.db
.vscode/
.idea/
*.swp

# Logs
*.log
```

</details>

* Add a `LICENSE` (recommendation: Apache-2.0 or MIT for demos). See below.
* Keep **all** real IDs, secrets, and SSH keys **out of Git**. Use environment variables or sealed secrets in a private repo.
* Consider adding a `Makefile` for common tasks (e.g., `make mgmt`, `make init`, `make create`, `make cleanup`).

---

## License

Choose a license and add it as `LICENSE` at the repo root. For example, **Apache License 2.0**:

```
Apache License
Version 2.0, January 2004
http://www.apache.org/licenses/
```

(You can add the full text from apache.org or via GitHub’s *Add file → Create new file → Choose a license template*.)

---

## Contributing

Issues and PRs are welcome. Please avoid submitting any credentials or sensitive information in issues, PRs, or commit history.

---

### Attribution

This guide is based on hands‑on steps for provisioning Azure Kubernetes clusters with CAPI/CAPZ and configuring Cilium and Cloud Provider Azure, adapted and sanitized for a public repository.
