
# 🛠️ Feature Proposal: Automated Re-encryption in kubeseal CLI

  

## 🔐 Bitnami Sealed Secrets (kubeseal) - Key Concepts

  

### What Are Sealed Secrets?

Sealed Secrets are Kubernetes custom resources that allow secure management of secrets using asymmetric encryption. They are:

- Safe to store in Git.

- Decrypted only by the Sealed Secrets controller running in the Kubernetes cluster.

  

### ✅ Why Use Sealed Secrets?

Kubernetes Secrets are only base64-encoded — not secure for version control. Sealed Secrets:

- Use asymmetric encryption.

- Encrypt secrets with a public key.

- Can only be decrypted inside the cluster using a private key.

- Are GitOps-friendly, allowing safe Git storage.

  

### ⚙️ How It Works (Core Components)

  

#### 🔐 Encryption Logic

The kubeseal CLI encrypts a Kubernetes Secret using the cluster’s public key.

  

```go

func (s *SealingKey) Encrypt(data []byte, namespace, name  string) (*SealedSecret, error)

```

  

#### 🔑 Key Pair Handling

Handles generation, storage, and rotation of RSA key pairs used for sealing and unsealing.

  

```go

func (r *KeyRegistry) GenerateKeyPair(now  metav1.Time) error

```

  

#### 🎛️ Controller Logic

The Sealed Secrets controller watches for new SealedSecret resources and converts them into Kubernetes Secret objects.

  

```go

func (r *ReconcileSealedSecret) Reconcile(ctx  context.Context, req  ctrl.Request) (ctrl.Result, error)

```

  

---

  

## 🔁 Automated SealedSecrets Re-encryption — Feature Design Document

  

### 📘 Overview

This document outlines a proposal to extend the Bitnami kubeseal CLI tool with a reencrypt feature that automates the re-encryption of all SealedSecret objects in a Kubernetes cluster. This enhancement will address a current manual process that becomes critical after sealing key rotation, ensuring all secrets remain compatible with the latest key and maintain security compliance.

  

### 🎯 Objectives

- Automate the discovery and re-encryption of all SealedSecrets in a Kubernetes cluster.

- Eliminate manual steps currently required after key rotation.

- Seamlessly integrate with the existing kubeseal CLI and controller codebase.

- Provide optional logging and error tracking.

- Scale for production environments with potentially hundreds of secrets.

  

### 🔧 Feature Workflow

  

1.  `kubeseal reencrypt --namespace=all`

2. Authenticate with Kubernetes cluster (kubectl context)

3. List all SealedSecret resources

4. For each SealedSecret:

- Retrieve original encrypted data and metadata

- Ask the controller to decrypt it (secure, private key stays internal)

- Re-encrypt it using the latest active public key

- Replace/update the SealedSecret object in the cluster

5. Report success/failure for each secret

  

---

  

### 📌 Implementation Plan

  

1.  **🔍 Discover Existing SealedSecrets**

  

```bash

kubectl  get  sealedsecrets  --all-namespaces  -o  json

```

# Go Code Implementation to Retrieve SealedSecrets from Kubernetes

This Go code retrieves SealedSecrets from a Kubernetes cluster using the `client-go` library and the SealedSecrets CRD provided by Bitnami.



## Code Implementation

```go
package main

import (
    "context"
    "fmt"
    "log"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "bitnami.com/sealed-secrets/pkg/apis/sealedsecrets/v1alpha1" // import SealedSecrets CRD
)

func getSealedSecrets(clientset *kubernetes.Clientset) ([]v1alpha1.SealedSecret, error) {
    // Replace 'default' with the namespace where you expect SealedSecrets
    sealedSecretsList, err := clientset.SealedsecretsV1alpha1().SealedSecrets("default").List(context.Background(), metav1.ListOptions{})
    if err != nil {
        return nil, err
    }
    return sealedSecretsList.Items, nil
}

func main() {
    kubeconfig := "/path/to/kubeconfig"
    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        log.Fatalf("Failed to build kubeconfig: %v", err)
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        log.Fatalf("Failed to create Kubernetes clientset: %v", err)
    }

    sealedSecrets, err := getSealedSecrets(clientset)
    if err != nil {
        log.Fatalf("Failed to get SealedSecrets: %v", err)
    }

    fmt.Println("Found SealedSecrets:", sealedSecrets)
}


  

Traverse and parse all SealedSecret resources. Extract name, namespace, and encrypted content.


```


2.  **🔐 Fetch Active Public Keys**

  

```bash

kubeseal  --controller-namespace=kube-system  --fetch-cert

```
# Go Code Implementation to Fetch Active Public Key from a URL

This Go code fetches an active public key (in PEM format) from a specified URL and prints it.

## Code Implementation

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "io/ioutil"
)

func fetchActivePublicKey() (string, error) {
    resp, err := http.Get("https://<controller-url>/v1/cert.pem")
    if err != nil {
        return "", fmt.Errorf("failed to fetch public key: %v", err)
    }
    defer resp.Body.Close()

    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return "", fmt.Errorf("failed to read response body: %v", err)
    }

    return string(body), nil
}

func main() {
    publicKey, err := fetchActivePublicKey()
    if err != nil {
        log.Fatalf("Error fetching public key: %v", err)
    }
    fmt.Println("Active Public Key:\n", publicKey)
}


  ```

Retrieve current public key via the controller's /v1/cert.pem.

  

3.  **🔓 Secure Decryption via Controller**

Introduce a private `/decrypt` endpoint (POST) in the controller:

- Accepts a SealedSecret object.

- Decrypts and returns the original Kubernetes Secret.

- RBAC-protected and internal-only.

  ## Code Implementation

```go
package controller

import (
    "encoding/json"
    "net/http"
    "log"
)

type DecryptRequest struct {
    SealedSecret string `json:"sealedSecret"`
}

type DecryptResponse struct {
    Secret string `json:"secret"`
}

func decryptHandler(w http.ResponseWriter, r *http.Request) {
    var req DecryptRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    decryptedSecret, err := decryptSealedSecret(req.SealedSecret)
    if err != nil {
        http.Error(w, "Decryption failed", http.StatusInternalServerError)
        return
    }

    resp := DecryptResponse{Secret: decryptedSecret}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func main() {
    http.HandleFunc("/decrypt", decryptHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}

 ``` 

4.  **🔁 Re-encrypt Secrets Using New Public Key**

  

Use `crypto.SealingKey.Encrypt()` from existing implementation.



## Code Implementation

```go
package main

import (
    "fmt"
    "log"

    "github.com/bitnami-labs/sealed-secrets/pkg/sealedsecrets"
)

func reencryptSecret(secret string, publicKey string) (string, error) {
    key, err := sealedsecrets.ParsePublicKey(publicKey)
    if err != nil {
        return "", fmt.Errorf("failed to parse public key: %v", err)
    }

    encryptedSecret, err := sealedsecrets.Encrypt(secret, key)
    if err != nil {
        return "", fmt.Errorf("failed to encrypt secret: %v", err)
    }

    return encryptedSecret, nil
}

func main() {
    publicKey := "<public-key>"
    secret := "<decrypted-secret>"

    encryptedSecret, err := reencryptSecret(secret, publicKey)
    if err != nil {
        log.Fatalf("Error re-encrypting secret: %v", err)
    }

    fmt.Println("Re-encrypted Secret:", encryptedSecret)
}


```  

5.  **⬆️ Replace Existing SealedSecrets**

  

```bash

kubectl  apply  -f  sealed-secret.yaml

```

  

Optionally back up the previous version via `--backup`.

.

## Code Implementation

```go
package main

import (
    "fmt"
    "log"
    "os/exec"
)

func applySealedSecret(filePath string) error {
    cmd := exec.Command("kubectl", "apply", "-f", filePath)
    err := cmd.Run()
    if err != nil {
        return fmt.Errorf("failed to apply SealedSecret: %v", err)
    }
    return nil
}

func main() {
    filePath := "/path/to/sealed-secret.yaml"
    if err := applySealedSecret(filePath); err != nil {
        log.Fatalf("Error applying SealedSecret: %v", err)
    }
    fmt.Println("Successfully applied SealedSecret")
}

```

  

6.  **📊 Logging and Reporting**

  

Add `--log-output=path.log` and `--report=json` flags to capture:

- Success/failure

- Namespace/name

- Error details


## Code Implementation

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Report struct {
    SecretName string `json:"secretName"`
    Namespace  string `json:"namespace"`
    Status     string `json:"status"`
    Error      string `json:"error,omitempty"`
}

func writeReport(report Report, logFile string) error {
    file, err := os.OpenFile(logFile, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer file.Close()

    encoder := json.NewEncoder(file)
    if err := encoder.Encode(report); err != nil {
        return err
    }
    return nil
}

func main() {
    report := Report{
        SecretName: "example-secret",
        Namespace:  "default",
        Status:     "success",
    }
    logFile := "/path/to/log.json"
    if err := writeReport(report, logFile); err != nil {
        fmt.Println("Error writing report:", err)
    }
    fmt.Println("Report written successfully.")
}

```

  

7.  **📈 Optimization**

- Use goroutines for concurrent processing.

- Respect API server rate limits.

  

8.  **🔐 Security Considerations**

- Private key never leaves the cluster.

-  `/decrypt` is RBAC-restricted.

- Skips old/unreadable SealedSecrets.

  

---

  

### 🧪 Usage: New CLI Command

  

```bash

kubeseal  reencrypt  --namespace=all  --controller-namespace=kube-system  --log-output=re-seal.log  --concurrency=10

```

  

| Flag | Description |

|---------------|-------------------------------------------|

| `--namespace` | Target namespace or "all" |

| `--backup` | Save previous sealed version |

| `--log-output`| Write logs to file |

| `--dry-run` | Show what would be re-encrypted |

| `--concurrency`| Parallel worker count |

| `--force` | Ignore decryption errors and continue |

  

---

  

### ✅ Benefits

-  **Security**: Ensures up-to-date encryption.

-  **Automation**: Removes error-prone manual steps.

-  **Compliance**: Helps in passing audits and maintaining secure GitOps workflows.

  

### ⚠️ Challenges

- Ensuring `/decrypt` endpoint is secure and not exposed externally.

- Handling legacy encrypted formats.

- Avoiding Kubernetes API throttling.

  

---

  

### 🔎 Internal Code Design and Integration

  

#### 🧩 New CLI Command

  

Defined in `cmd/kubeseal/reencrypt.go`:

  

```go

var  reencryptCmd = &cobra.Command{

Use: "reencrypt",

Short: "Re-encrypt all SealedSecrets using the latest public key",

RunE: func(cmd *cobra.Command, args []string) error {

return  runReencryption()

},

}

```

  

Ref: `main.go – AddCommand`

  

#### 🔐 Reuse: Public Key Handling

  

From the existing codebase:

  

```go

// Fetch and parse public key

certPEM := FetchCertificate(controllerURL)

block, _ := pem.Decode(certPEM)

pubKey, err := x509.ParsePKIXPublicKey(block.Bytes)

```

  

Ref: `main.go`

  

#### 🔓 New Feature: Controller `/decrypt` Endpoint

  

Expose a secure internal-only decryption API inside the controller:

  

```go

func (r *DecryptionHandler) Decrypt(w  http.ResponseWriter, req *http.Request) {

var  sealedSecret  v1alpha1.SealedSecret

// Deserialize, validate

secret, err := r.unsealer.Unseal(&sealedSecret)

// Return decrypted Secret

}

```

  

  

#### 🔐 Reuse: Encryption Logic

  

```go

sealedSecret, err := crypto.NewSealedSecret(pubKey).Seal(secretObj, namespace, name)

```

  



  

---

  

### 🧱 Implementation Skeleton

  

Functions to implement:

  

```go

func  runReencryption() error

func  fetchAllSealedSecrets(namespace  string) ([]SealedSecret, error)

func  requestDecryption(ss  SealedSecret) (Secret, error)

func  reencryptSecret(secret  Secret, pubKey  rsa.PublicKey) (SealedSecret, error)

func  updateSealedSecret(ss  SealedSecret) error

```

  

---

  

### 💻 GitHub Diff Example

  

```diff

// main.go

func main() {

rootCmd := &cobra.Command{Use: "kubeseal"}

rootCmd.AddCommand(versionCmd)

rootCmd.AddCommand(reencryptCmd) // ➕ Added

...

}

```

  

```diff

+ var reencryptCmd = &cobra.Command{

+ Use: "reencrypt",

+ Short: "Re-encrypt SealedSecrets using the latest public key",

+ RunE: func(cmd *cobra.Command, args []string) error {

+ return runReencryption()

+ },

+ }

```

  

---

  

### 📚 References

- [Bitnami SealedSecrets GitHub](https://github.com/bitnami-labs/sealed-secrets)

- [kubeseal CLI code](https://github.com/bitnami-labs/kubeseal)

- [client-go Kubernetes SDK](https://github.com/kubernetes/client-go)

- [crypto/sealing_key.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/pkg/crypto/sealing_key.go)

- [controller/unsealer.go](https://github.com/bitnami-labs/sealed-secrets/blob/main/pkg/controller/unsealer.go)

  

---

  

## 🏁 Conclusion

The reencrypt command would significantly improve Sealed Secrets lifecycle management by enabling administrators to perform a bulk re-encryption automatically, securely, and reliably. This reduces operational complexity and enforces secure practices at scale.
