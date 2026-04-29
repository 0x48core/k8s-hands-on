# kind — Local Kubernetes for This Project

## Prerequisites

```bash
# Install kind (macOS)
brew install kind

# Install kubectl (if not already)
brew install kubectl

# Verify
kind version
kubectl version --client
```

## Cluster Setup (Storage-Focused)

The cluster config at [`kind-config.yaml`](kind-config.yaml) creates:
- 1 control-plane node
- 2 worker nodes, each with a host directory mounted at `/mnt/storage` inside the node

This extra mount is required for **hostPath** and **local PersistentVolume** demos. It maps your Mac's `/tmp/k8s-hands-on/storage` into the kind nodes.

### Create the cluster

```bash
# Create the host directory first
mkdir -p /tmp/k8s-hands-on/storage

# Spin up the cluster
kind create cluster --config kind/kind-config.yaml

# Verify nodes are Ready
kubectl get nodes
```

Expected output:
```
NAME                        STATUS   ROLES           AGE   VERSION
k8s-storage-control-plane   Ready    control-plane   ...
k8s-storage-worker          Ready    <none>          ...
k8s-storage-worker2         Ready    <none>          ...
```

### Switch kubectl context

```bash
kubectl cluster-info --context kind-k8s-storage
kubectl config use-context kind-k8s-storage
```

---

## Storage in kind

### Default Storage Class

kind ships with **local-path-provisioner** out of the box. It automatically provisions PVs on the node's local filesystem — no setup needed.

```bash
kubectl get storageclass
# NAME                 PROVISIONER             ...
# standard (default)   rancher.io/local-path   ...
```

This is enough to run all PVC-based demos without any additional configuration.

### Storage Types Covered in This Project

| Type | Lifecycle | Where |
|---|---|---|
| `emptyDir` | Pod lifetime | [`knowledge-base/storage/emptyDir/`](../knowledge-base/storage/emptyDir/) |
| `hostPath` | Node lifetime | coming soon |
| `PersistentVolume` + `PVC` | Independent | coming soon |
| `StorageClass` (dynamic) | On-demand | coming soon |

---

## Running the emptyDir Demo

```bash
# Deploy
kubectl apply -f knowledge-base/storage/emptyDir/emptydir-demo.yaml

# Watch the reader sidecar receive log lines from writer in real time
kubectl logs -f emptydir-demo -c reader

# Check the init container's seed file
kubectl exec emptydir-demo -c writer -- cat /shared/message.txt

# Cleanup
kubectl delete -f knowledge-base/storage/emptyDir/emptydir-demo.yaml
```

---

## Common Commands

```bash
# List clusters
kind get clusters

# Load a local Docker image into the cluster (avoids Docker Hub pull)
kind load docker-image <image>:<tag> --name k8s-storage

# Delete the cluster
kind delete cluster --name k8s-storage

# Re-create from scratch
kind delete cluster --name k8s-storage && kind create cluster --config kind/kind-config.yaml
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `nodes not ready` | Wait ~30s; kind nodes take a moment to initialise |
| `ImagePullBackOff` on `busybox` | You may need a Docker Hub account — or run `kind load docker-image busybox --name k8s-storage` after pulling locally |
| PVC stuck in `Pending` | Confirm default storage class exists: `kubectl get sc` |
| Wrong context | `kubectl config use-context kind-k8s-storage` |
