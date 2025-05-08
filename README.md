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


