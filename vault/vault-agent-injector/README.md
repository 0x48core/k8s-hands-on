# VAULT AGENT INJECTOR: SIDECAR SECRETS IN KUBERNETES

The [Vault Agent Injector](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector) is a Kubernetes admission webhook that modifies pods at creation time and adds Vault Agent init and 
sidecar containers. This allows applications to read secrets from files on disk instead of talking to  Vault directly.

## Why Applications Should Not Talk to Vault Directly

Vault Agent is a helper process that runs next to your application. It handles a few critical tasks:

- Authenticating to Vault.
- Fetching secrets.
- Renewing Vault tokens.
- Refreshing secrets when they change.
- Writing secrets to files.

With Vault Agent in place, your application never calls Vault APIs. It does not manage tokens. It only reads files from disk.

This separation is important. When an application talks to Vault directly:

- It must understand Vault APIs.
- It must store and renew tokens.
- It often needs a restart when secrets change.

Vault Agent removes all of that logic from the app. Vault-related concerns stay outside the application boundary. The app
stays focused on its own job.

## What the Vault Agent Injector Does on Kubernetes

On Kubernetes, Vault Agent runs using the vault-k8s integration. This integration relies on a Mutating Admission Webhook.
The injector adds three things to the Pod:

- An **init container** that runs once before the app starts.
- A **Vault Agent sidecar container** that runs next to the app.
- A **shared volume** where secrets are written.

## The Role of Init Container and Sidecar

Vault Agent supports two execution phases:

### Init Container

The init container:

- Run before the application container starts.
- Authenticates to Vault.
- Fetches secrets.
- Writes them to disk.
- Exits.

This guarantees that secrets exist before the app starts.

### Sidecar Container

The sidecar:

- Start after the init container
- Runs for the entire lifetime of the Pod.
- Renews the Vault token.
- Watches secrets.
- Re-renders files when values change.

This is what allows secrets to change without restarting the application.

## Hands-on: Inject and Read Secrets with the Vault Agent

### Step 1: Install Vault with the Injector Enabled

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dev.enabled=true \
  --set injector.enabled=true
```

### Step 2: Confirm the Injector Is Running

```bash
kubectl get all -n vault
```

### Step 3: Allow Vault to Trust Kubernetes

Vault does not trust Kubernetes by default. We must explicitly configure this trust.

First, enable the Kubernetes auth method:
```bash
kubectl exec -n vault vault-0 -- vault auth enable kubernetes
```
Then configure it:
```bash
kubectl exec -n vault -i vault-0 -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"
```
This setup allows Vault to:
- Verify Service Account tokens.
- Confirm pod identity using Kubernetes API.

> Without this step, authentication will fail later.

### Step 4: Store a Test Secret in Vault

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/my-app/config \
  username="demo-user" \
  password="demo-password"
```

This creates a KV v2 secret in Vault.

Verify it:

```bash
kubectl exec -n vault vault-0 -- vault kv get secret/my-app/config
```

This secret doesn't exist as a Kubernetes Secret because Vault manages it.

### Step 5: Limit Access with Policies

```bash
kubectl exec -n vault -i vault-0 -- vault policy write my-app - <<EOF
path "secret/data/my-app/*" {
  capabilities = ["read"]
}
EOF
```

This policy allows read access only to this app's secrets.

### Step 6: Bind Kubernetes Identity to Vault Access

```bash
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my_app \
  bound_service_account_namespaces=app \
  policies=my-app \
  ttl=1h
```

This role binds together: A namespace, a service account, a vault policy.

Only pods that match all three can authenticate and read secrets.

### Step 7: Create the Application Namespace and Service Account

```bash
kubectl create ns app
kubectl create sa my-app -n app
```

These values must match the role configuration exactly.

### Step 8: Tell the Inject What to Inject

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: app
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app"
    vault.hashicorp.com/agent-inject-secret-database-config: "secret/data/my-app/config"
    vault.hashicorp.com/agent-inject-template-database-config: |
      {{- with secret "secret/data/my-app/config" -}}
      export DB_USERNAME="{{ .Data.data.username }}"
      export DB_PASSWORD="{{ .Data.data.password }}"
      {{- end -}}
spec:
  serviceAccountName: my-app
  containers:
  - name: my-app
    image: nginx:latest
    command: ["/bin/sh"]
    args: ["-c", "while true; do sleep 30; done"]
```

These annotations act as a contract between your Pod and the injector.

- `agent-inject: true` enables injection
- `role` selects the Vault role
- `agent-inject-secret-*` defines the Vault path and output file name
- `agent-inject-template-*` defines how the secret is rendered

The file name comes from the annotation suffix. Multiple annotations create multiple files.

### How Vault Agent Templates Work

Templates run inside the Vault Agent, not inside your application.

They:
- Read data from Vault.
- Render it into files.
- Re-render when the data changes.

In this example:
- Values are mapped to environment variable exports.
- The file is written to `/vault/secrets/database-config`

Your application can source the file or read it directly.

### Step 9: Verify Injection

```bash
kubectl get pods -n app
```