# 🛠️ Feature Proposal: Automated Re-encryption in `kubeseal` CLI
# 🔐 Bitnami Sealed Secrets (`kubeseal`) - Key Concepts

## What Are Sealed Secrets?
Sealed Secrets are Kubernetes custom resources that allow secure management of secrets using **asymmetric encryption**.  
They are:
- Safe to store in Git.
- Decrypted **only** by the Sealed Secrets controller running in the Kubernetes cluster.

---

## ✅ Why Use Sealed Secrets?

- **Kubernetes Secrets** are only base64-encoded — not secure for version control.
- **Sealed Secrets**:
  - Use **asymmetric encryption**.
  - Encrypt secrets with a **public key**.
  - Can only be decrypted inside the cluster using a **private key**.
  - Are **GitOps-friendly**, allowing safe Git storage.

---

## ⚙️ How It Works (Core Components)

### 🔐 Encryption Logic
The `kubeseal` CLI encrypts a Kubernetes Secret using the cluster’s public key.

```go
func (s *SealingKey) Encrypt(data []byte, namespace, name string) (*SealedSecret, error)
```

---

### 🔑 Key Pair Handling
Handles generation, storage, and rotation of RSA key pairs used for sealing and unsealing.

```go
func (r *KeyRegistry) GenerateKeyPair(now metav1.Time) error
```

---

### 🎛️ Controller Logic
The Sealed Secrets controller watches for new `SealedSecret` resources and converts them into Kubernetes `Secret` objects.

```go
func (r *ReconcileSealedSecret) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

---

# 🔁 Automated SealedSecrets Re-encryption — Feature Design Document

## 📘 Overview

This document outlines a proposal to extend the Bitnami `kubeseal` CLI tool with a **`reencrypt`** feature that automates the re-encryption of all `SealedSecret` objects in a Kubernetes cluster. This enhancement will address a current manual process that becomes critical after **sealing key rotation**, ensuring all secrets remain compatible with the latest key and maintain security compliance.

---

## 🎯 Objectives

- Automate the discovery and re-encryption of all SealedSecrets in a Kubernetes cluster.
- Eliminate manual steps currently required after key rotation.
- Seamlessly integrate with the existing `kubeseal` CLI and controller codebase.
- Provide optional logging and error tracking.
- Scale for production environments with potentially hundreds of secrets.

---

## 🔧 Feature Workflow

```text
1. kubeseal reencrypt --namespace=all
2. Authenticate with Kubernetes cluster (kubectl context)
3. List all SealedSecret resources
4. For each SealedSecret:
   a. Retrieve original encrypted data and metadata
   b. Ask the controller to decrypt it (secure, private key stays internal)
   c. Re-encrypt it using the latest active public key
   d. Replace/update the SealedSecret object in the cluster
5. Report success/failure for each secret
```

---

## 📌 Implementation Plan

### 1. 🔍 Discover Existing SealedSecrets

```bash
kubectl get sealedsecrets --all-namespaces -o json
```

- Traverse and parse all SealedSecret resources.
- Extract name, namespace, and encrypted content.

### 2. 🔐 Fetch Active Public Keys

```bash
kubeseal --controller-namespace=kube-system --fetch-cert
```

- This provides the current public key used for new seals.
- Retrieve public keys by calling controller `/v1/cert.pem` endpoint.

### 3. 🔓 Secure Decryption via Controller

- Extend the SealedSecrets controller with a **private `/decrypt` endpoint (POST)** that:
  - Accepts a SealedSecret object.
  - Uses existing private keys to decrypt the secret safely.
  - Returns the raw Kubernetes `Secret` object (base64 encoded).

> ✅ **Security Note**: This endpoint must be:
> - Cluster-internal only.
> - RBAC-protected.
> - Require client auth or mutual TLS.

### 4. 🔁 Re-encrypt Secrets Using New Public Key

- Use existing `kubeseal` logic:

```go
func (s *SealingKey) Encrypt(data []byte, namespace, name string) (*SealedSecret, error)
```

- Re-seal using the newest public key.

### 5. ⬆️ Replace Existing SealedSecrets

- Patch or replace the original SealedSecret in the cluster using:

```bash
kubectl apply -f sealed-secret.yaml
```

> Optionally, backup the old version using a `--backup` flag.

### 6. 📊 Logging and Reporting 

- Add `--log-output=path.log` and `--report=json` flags.
- Capture:
  - Success/failure per object
  - Namespaces/names of updated secrets
  - Errors with line numbers

### 7. 📈 Optimization 

- Process SealedSecrets in batches (e.g., 100 per thread).
- Use Go worker pools and goroutines for concurrency.
- Respect rate limits from the Kubernetes API server.

### 8. 🔐 Security Considerations 

- **Private key never leaves controller pod**.
- If decryption fails (e.g., key too old), log and skip.
- Consider rotating keys that are >90 days old.

---

## 🧪 Usage: New CLI Command

```bash
kubeseal reencrypt   --namespace=all   --controller-namespace=kube-system   --log-output=re-seal.log   --concurrency=10
```

### Optional Flags

| Flag               | Description                                 |
|--------------------|---------------------------------------------|
| `--namespace`      | Target namespace or "all"                   |
| `--backup`         | Save previous sealed version                |
| `--log-output`     | Write logs to file                          |
| `--dry-run`        | Show what would be re-encrypted             |
| `--concurrency`    | Parallel worker count                       |
| `--force`          | Ignore decryption errors and continue       |

---

## ✅ Benefits

- Security: All SealedSecrets get updated to use the latest key.
- Automation: Removes error-prone manual work after key rotation.
- Compliance: Easier to meet security audit requirements.

---

## ⚠️ Challenges

- Ensuring the `/decrypt` API is secure.
- Backward compatibility for older sealed formats.
- Managing secrets at scale without API throttling.

---

## 📦 Integration Plan

- New Go module in `cmd/kubeseal/reencrypt.go`
- Use client-go for Kubernetes API access
- Extend the SealedSecrets controller for decrypt endpoint (optional or hidden behind feature flag)

---

## 📚 References

- [Bitnami SealedSecrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [kubeseal CLI code](https://github.com/bitnami-labs/sealed-secrets/blob/main/cmd/kubeseal/main.go)
- [client-go Kubernetes SDK](https://github.com/kubernetes/client-go)

---

## 🏁 Conclusion

This feature enhances the lifecycle of sealed secrets in GitOps workflows by making re-encryption transparent and automated. It is a security best practice and provides long-term maintainability for Kubernetes-based infrastructures.



