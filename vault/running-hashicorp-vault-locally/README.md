# RUNNING HASHICORP VAULT LOCALLY: WRITE YOUR FIRST SECRETS AND POLICIES

## Installing Vault

In this demo, I'll use MacOS.

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

Verify the installation:

```bash
vault version
```

## Starting Vault in Dev Mode

Dev mode works well for learning because it uses in-memory storage and auto-unseals itself. Unsealing provides secret keys (via Shamir’s Secret Sharing or auto-unseal) to rebuild the master key, enabling Vault to decrypt storage and operate.

```bash
vault server -dev
```

## Set the Vault address

```bash
export VAULT_ADDR=http://127.0.0.1:8200
```

Log in using the root token from the first terminal:

```
vault login <copy the root login from the first terminal>
```

## Checking Available Secrets Engines

List the enabled secrets engines:

```bash
ault secrets list                      
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_88b28959    per-token private secret storage
identity/     identity     identity_b49883a5     identity store
secret/       kv           kv_b71864f0           key/value secret storage
sys/          system       system_3f661946       system endpoints used for control, policy and debugging
```

## Writing Your First Secret

Store the app credentials to a path:

```bash
vault kv put secret/my-app \
  username=appuser \
  password=supersecret
```

This writes data to:

```bash
secret/data/my-app
```

Read the secret:

```bash
vault kv get secret/my-app

=== Secret Path ===
secret/data/my-app

======= Metadata =======
Key                Value
---                -----
created_time       2026-03-01T18:51:55.352502Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    supersecret
username    appuser
```

## Creating a Non-Root Policy

```bash
cat <<EOF > my-app-policy.hcl
path "secret/data/my-app" {
  capabilities = ["read"]
}
EOF
```

```bash
vault policy write my-app my-app-policy.hcl
```

Create a token with this policy:

```bash
vault token create -policy=my-app
```

Copy the new token and log in:

```bash
vault login <new-token>
```

Try reading the secret:

```bash
vault kv get secret/my-app
```

Now try a write operation:

```bash
vault kv put secret/my-app test=fail
```

