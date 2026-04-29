# hostPath

## What is hostPath?

`hostPath` mounts a file or directory from the **host node's filesystem** directly into a Pod. Unlike `emptyDir`, data written to a hostPath volume **persists after the Pod is deleted** — it lives on the node itself.

> Warning: hostPath gives the Pod access to the node's filesystem. Avoid it in production unless you have a specific need (e.g., DaemonSet reading `/sys` or `/var/log`). Prefer PersistentVolumes with a StorageClass instead.

## How it works

```
Node filesystem
└── /mnt/storage          ← actual data lives here
        ▲
        │  hostPath mount
┌───────┴────────┐
│      Pod       │
│  /data ────────┘
└────────────────┘
```

The path on the node is shared by any Pod (on the same node) that mounts it. Data survives Pod restarts and deletions — it is only gone if you delete it from the node directly.

## hostPath Types

| Type | Behaviour |
|---|---|
| `""` (empty) | No pre-flight check |
| `DirectoryOrCreate` | Create directory if absent |
| `Directory` | Directory must already exist |
| `FileOrCreate` | Create file if absent |
| `File` | File must already exist |
| `Socket` | Unix socket must already exist |
| `CharDevice` | Character device must exist |
| `BlockDevice` | Block device must exist |

## emptyDir vs hostPath

| | emptyDir | hostPath |
|---|---|---|
| **Lifecycle** | Pod | Node (survives pod deletion) |
| **Shared across pods** | No (single pod only) | Yes (any pod on same node) |
| **Cross-node** | No | No |
| **Production use** | Scratch/cache/sidecar | Node agents, log collectors |

## Use Cases

| Use Case | Example |
|---|---|
| Node-level log collection | DaemonSet mounts `/var/log` to ship logs |
| Node monitoring agent | Prometheus node-exporter reads `/sys`, `/proc` |
| Development / local testing | Persist data across pod restarts without a PVC |

## Demo

**File:** [`hostpath-demo.yaml`](hostpath-demo.yaml)

The demo runs two separate Pods on the same worker node, both mounting `/mnt/storage` (pre-wired in the kind cluster via `extraMounts` → [`kind/kind-config.yaml`](../../kind/kind-config.yaml)):

- **`hostpath-writer`** — appends a timestamped line to `/data/log.txt` every 5 seconds
- **`hostpath-reader`** — reads the same file, proving data is shared across pods and persists

### Run

```bash
# Apply both pods
kubectl apply -f knowledge-base/storage/hostPath/hostpath-demo.yaml

# Watch writer accumulate lines
kubectl logs -f hostpath-writer -c writer

# Reader sees all data including lines written before it started
kubectl logs -f hostpath-reader -c reader

# Delete the writer and re-create — data is still on the node
kubectl delete pod hostpath-writer
kubectl apply -f knowledge-base/storage/hostPath/hostpath-demo.yaml
kubectl exec hostpath-reader -c reader -- cat /data/log.txt  # full history intact

# Cleanup
kubectl delete -f knowledge-base/storage/hostPath/hostpath-demo.yaml
```

### Verify data on the node directly

```bash
# Enter the worker node container (kind-specific)
docker exec -it k8s-storage-worker sh
cat /mnt/storage/log.txt
```

## Key Properties

| Property | Value |
|---|---|
| Lifetime | Node (independent of Pod lifecycle) |
| Scope | Any Pod scheduled on the same node |
| Persistence | Yes — until manually deleted from the node |
| Cross-node | No |
| Production recommendation | Avoid; use PVC + StorageClass instead |
