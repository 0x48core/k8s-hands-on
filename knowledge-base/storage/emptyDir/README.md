# emptyDir

## What is emptyDir?

`emptyDir` is a temporary volume that is created when a Pod is assigned to a Node and exists as long as that Pod is running on that Node. All containers in the Pod share the same `emptyDir` volume and can read/write to it. When the Pod is removed from the Node for any reason, the data in `emptyDir` is deleted permanently.

> Note: A container crash does **not** remove a Pod from a Node, so data in `emptyDir` survives container crashes.

## How it works

```
Pod
├── initContainer: data-generator  ─┐
├── container: writer               ├──► /shared  (emptyDir volume on Node disk)
└── container: reader              ─┘
```

1. The volume is created empty on the Node when the Pod starts.
2. The `initContainer` runs first and can pre-populate the volume (e.g., write a seed file).
3. Once the init container finishes, `writer` and `reader` start — both mount `/shared` and can exchange data through it in real time.
4. When the Pod is deleted, the volume is wiped from the Node.

## Use Cases

| Use Case | Description |
|---|---|
| Shared scratch space | Multiple containers exchange intermediate files without a network hop |
| Cache | Store downloaded or computed data that is only needed for the life of the Pod |
| Init container handoff | Init container seeds config/data that the main container consumes |
| Sidecar log sharing | App container writes logs to the volume; sidecar ships them |

## Memory-backed emptyDir

Set `medium: Memory` to store the volume in a `tmpfs` (RAM-backed) filesystem. Faster, but counts against the container's memory limit.

```yaml
volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 128Mi
```

## Demo

**File:** [`emptydir-demo.yaml`](emptydir-demo.yaml)

The demo runs a single Pod with:
- **`data-generator`** (init container) — writes an initial message to `/shared/message.txt`
- **`writer`** (main container) — appends a timestamped line to `/shared/log.txt` every 5 seconds
- **`reader`** (sidecar container) — tails `/shared/log.txt` in real time

### Run

```bash
# Apply
kubectl apply -f emptydir-demo.yaml

# Watch reader output (shows writer's lines as they arrive)
kubectl logs -f emptydir-demo -c reader

# Verify init container wrote the seed file
kubectl exec emptydir-demo -c writer -- cat /shared/message.txt

# Cleanup
kubectl delete -f emptydir-demo.yaml
```

## Key Properties

| Property | Value |
|---|---|
| Lifetime | Tied to the Pod (deleted when Pod is removed) |
| Scope | Shared across all containers in the Pod |
| Persistence | None — ephemeral only |
| Default storage | Node disk |
| Optional storage | `medium: Memory` for tmpfs |
