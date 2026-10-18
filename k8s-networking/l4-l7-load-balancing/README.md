# Load Balancing: Layer 4 (Service) vs Layer 7 (Gateway API)

Same Deployment, two ways to balance traffic to it:

- **L4 – `Service`**: kube-proxy picks a pod **once per TCP connection**. It only sees IP + port.
- **L7 – `Gateway` + `HTTPRoute`** (Traefik): the proxy terminates HTTP and picks a pod **per request**,
  and can route on host, path, headers and weights.

| File | What it deploys |
|---|---|
| `kind-config.yaml` | kind cluster `lb-lab`: 1 control-plane + 2 workers |
| `01-app.yaml` | Namespace `lb-lab`, `web` (v1, 3 pods), `web-v2` (canary, 1 pod), ClusterIP backends, `client` pod |
| `02-service-l4.yaml` | `web-l4` (`LoadBalancer`), `web-l4-sticky` (`sessionAffinity: ClientIP`) |
| `traefik-values.yaml` | Helm values: Traefik with the Gateway API provider |
| `03-gateway-l7.yaml` | `GatewayClass`, `Gateway` (`lb.local`), `HTTPRoute` with header / weighted / default rules |

```
                       ┌──────────── L4: web-l4 (Service) ─────────────┐
                       │  kube-proxy: 1 pod per TCP connection         │
 client ──── TCP ──────┤  sees only IP:port                            ├──► web v1 (x3)
                       └───────────────────────────────────────────────┘
                       ┌──────────── L7: traefik (Gateway) ────────────┐
                       │  x-version: v2   → web-v2                     │──► web v1 (x3)
 client ──── HTTP ─────┤  /split          → web 80% / web-v2 20%       │
                       │  /               → web (round-robin/request)  │──► web-v2 (x1)
                       └───────────────────────────────────────────────┘
```

`traefik/whoami` echoes the pod `Hostname`, its `--name` (`v1`/`v2`), the `RemoteAddr` and the request
headers, so every response shows who served it and what the backend saw.

---

## 1. Deploy

```bash
kind create cluster --config kind-config.yaml

# Gateway API CRDs (v1.5 includes TLSRoute v1, which Traefik v3.7 watches)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml

kubectl apply -f 01-app.yaml -f 02-service-l4.yaml

helm repo add traefik https://traefik.github.io/charts && helm repo update
helm install traefik traefik/traefik -f traefik-values.yaml -n traefik --create-namespace
kubectl -n traefik rollout status deploy/traefik

kubectl apply -f 03-gateway-l7.yaml
kubectl -n lb-lab get gateway,httproute     # web-gw PROGRAMMED=True
```

All experiments run from the `client` pod, so no `LoadBalancer` IP or `/etc/hosts` is needed:

```bash
c() { kubectl -n lb-lab exec client -- sh -c "$1"; }
```

> `curl "http://host/?n=[1-30]"` uses curl URL globbing: 30 requests over **one** keep-alive connection.
> A `for` loop of `curl` calls opens a **new** connection per request.

## 2. L4: balancing per connection

New connection per request: kube-proxy spreads connections (randomly with iptables):

```bash
c 'for i in $(seq 30); do curl -s http://web-l4/; done' | grep Hostname | sort | uniq -c
#   15 Hostname: web-7685b886bf-9d26g
#    5 Hostname: web-7685b886bf-hbxqv
#   10 Hostname: web-7685b886bf-rss24
```

One keep-alive connection: **every** request lands on the same pod:

```bash
c 'curl -s "http://web-l4/?n=[1-30]"' | grep Hostname | sort | uniq -c
#   30 Hostname: web-7685b886bf-9d26g
```

This is why HTTP/2, gRPC and connection-pooled clients end up "unbalanced" behind a plain Service.

Session affinity: each client IP is pinned to one pod, even across new connections:

```bash
c 'for i in $(seq 20); do curl -s http://web-l4-sticky/; done' | grep Hostname | sort | uniq -c
#   20 Hostname: web-7685b886bf-rss24
```

## 3. L7: balancing per request

The same single keep-alive connection, through the Gateway, gets spread evenly. Traefik reads the
Service's EndpointSlices and round-robins each request across the pod IPs:

```bash
c 'curl -s -H "Host: lb.local" "http://traefik.traefik/?n=[1-30]"' | grep Hostname | sort | uniq -c
#   10 Hostname: web-7685b886bf-9d26g
#   10 Hostname: web-7685b886bf-hbxqv
#   10 Hostname: web-7685b886bf-rss24
```

Weighted split (canary), `/split` → 80/20:

```bash
c 'curl -s -H "Host: lb.local" "http://traefik.traefik/split?n=[1-100]"' | grep '^Name' | sort | uniq -c
#   80 Name: v1
#   20 Name: v2
```

Header-based routing:

```bash
c 'curl -s -H "Host: lb.local" -H "x-version: v2" "http://traefik.traefik/?n=[1-10]"' | grep '^Name' | sort | uniq -c
#   10 Name: v2
```

Host matching: a hostname that no listener/route matches gets a 404 from the proxy, never reaching a pod:

```bash
c 'curl -s -o /dev/null -w "%{http_code}\n" -H "Host: other.local" http://traefik.traefik/'
# 404
```

## 4. What the backend sees (source IP)

```bash
kubectl -n lb-lab get pod client -o jsonpath='{.status.podIP}{"\n"}'   # 10.244.1.4

c 'curl -s http://web-l4/' | grep RemoteAddr
# RemoteAddr: 10.244.1.4:32938          <- L4: the client's own IP (packets are only DNAT-ed)

c 'curl -s -H "Host: lb.local" http://traefik.traefik/' | grep -E 'RemoteAddr|X-Forwarded-For'
# RemoteAddr: 10.244.2.3:46612          <- L7: Traefik's pod IP (a new connection)
# X-Forwarded-For: 10.244.1.4           <- original client, passed as a header
```

For traffic coming from **outside** the cluster through a `LoadBalancer`/`NodePort`, the default
`externalTrafficPolicy: Cluster` may forward to a pod on another node and SNAT the client IP.
`externalTrafficPolicy: Local` keeps the client IP and avoids the extra hop, but only nodes running a pod
receive traffic.

## 5. Readiness decides who gets traffic (both layers)

```bash
kubectl -n lb-lab get endpointslices -l kubernetes.io/service-name=web

# Break one pod's readiness: whoami's /health accepts POST to change its status code
POD=$(kubectl -n lb-lab get pod -l version=v1 -o jsonpath='{.items[0].metadata.name}')
c "curl -s -X POST -d 500 http://$(kubectl -n lb-lab get pod $POD -o jsonpath='{.status.podIP}'):8080/health"

kubectl -n lb-lab get pods -l version=v1 -w             # $POD goes 0/1 after ~3 failed probes
c 'for i in $(seq 30); do curl -s http://web-l4/; done' | grep Hostname | sort | uniq -c   # $POD gone
c 'curl -s -H "Host: lb.local" "http://traefik.traefik/?n=[1-30]"' | grep Hostname | sort | uniq -c

kubectl -n lb-lab delete pod $POD                        # replaced by a healthy pod
```

## 6. Optional: external LoadBalancer IPs on kind

`web-l4` and `traefik` stay `<pending>` on kind until something implements `type: LoadBalancer`:

```bash
go install sigs.k8s.io/cloud-provider-kind@latest
sudo cloud-provider-kind            # keep it running in another terminal

kubectl -n lb-lab get svc web-l4    # EXTERNAL-IP is assigned
```

On Linux the IP is reachable directly. On macOS the Docker network isn't routable from the host, so
check cloud-provider-kind's docs for its port-mapping mode. On a cloud provider the same manifests create
a real NLB (L4) or a cloud LB in front of Traefik.

## L4 vs L7 summary

| | L4 `Service` | L7 `Gateway` / `HTTPRoute` |
|---|---|---|
| Balancing unit | TCP connection | HTTP request |
| Routing on | IP, port | Host, path, headers, weights |
| Stickiness | `sessionAffinity: ClientIP` | Cookie / hash (controller-specific policy) |
| Client IP at pod | Preserved in-cluster (`Local` for external) | Proxy IP + `X-Forwarded-For` |
| TLS | Passed through to the pod | Can terminate at the gateway |
| Use for | Databases, raw TCP/UDP, TLS passthrough | Web/APIs, gRPC/HTTP2, canary, A/B |

## Cleanup

```bash
kind delete cluster --name lb-lab
```
