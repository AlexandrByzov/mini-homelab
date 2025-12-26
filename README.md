# Mini-Homelab

## Secret Management

This repository uses [SOPS](https://github.com/getsops/sops) (Secrets OPerationS) with [age](https://github.com/FiloSottile/age) encryption to securely store secrets in git. Flux automatically decrypts these secrets during reconciliation.

### Architecture

- **SOPS age key**: Bootstrap secret created directly in the cluster (never stored in git)
- **Application secrets**: Encrypted with SOPS and committed to git
- **Flux**: Automatically decrypts encrypted manifests using the age key from `flux-system` namespace

## Initial Setup

### 1. Create the SOPS Age Key Secret (One-time)

This is the bootstrap secret that Flux uses to decrypt all other secrets. Create it directly in the cluster:

```bash
kubectl create secret generic sops-age \
  --from-literal=age.agekey="$(security find-generic-password -s "sops-age-key" -w | xxd -r -p)" \
  --namespace=flux-system
```

> **Important**: This secret should **never** be stored in git. It's a prerequisite for Flux to decrypt any SOPS-encrypted resources.

### 2. Verify the Secret

```bash
kubectl get secret sops-age -n flux-system
```

## Encrypting Secrets for Git

### Configure SOPS

Create a `.sops.yaml` configuration file in the repository root:

```yaml
creation_rules:
    age: age1your-public-key-here
```

Get your age public key:
```bash
security find-generic-password -s "sops-age-key" -w | xxd -r -p | age-keygen -y
```

### Encrypt a Secret

```bash
# Create a regular Kubernetes secret manifest
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
stringData:
  password: super-secret-value
EOF

# Encrypt it with SOPS
sops --encrypt secret.yaml > secret.yaml

# Or encrypt in-place
sops --encrypt --in-place secret.yaml
```

### Commit Encrypted Secrets

```bash
git add secret.yaml
git commit -m "Add encrypted secret"
git push
```

## How Flux Decrypts Secrets

Flux automatically detects SOPS-encrypted files and decrypts them during reconciliation:

1. Flux pulls the encrypted manifest from git
2. Detects SOPS metadata in the file
3. Uses the `sops-age` secret from `flux-system` namespace to decrypt
4. Applies the decrypted manifest to the cluster

No additional configuration is needed - Flux handles this automatically.
