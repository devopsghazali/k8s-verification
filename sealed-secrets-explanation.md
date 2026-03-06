# Kubernetes Sealed Secrets – Complete Workflow

## 1. Problem: Secrets in GitOps

GitOps ka rule hota hai:

```
Everything lives in Git
```

Matlab Kubernetes manifests jaise:

* deployment.yaml
* service.yaml
* configmap.yaml
* secret.yaml

sab Git repository me store kiye jaate hain.

Lekin problem yeh hai ki Kubernetes Secret actually encrypted nahi hota.

Example:

```yaml
apiVersion: v1
kind: Secret
data:
  password: YWRtaW4=
```

Yeh **Base64 encoded** hota hai, encrypted nahi.

Koi bhi decode kar sakta hai:

```
echo YWRtaW4= | base64 -d
```

Output:

```
admin
```

Isliye **Git me plain Secret store karna insecure hai**.

---

# 2. Solution: Sealed Secrets

Sealed Secrets ek system hai jo:

* Secret ko encrypt karta hai
* Git me safe store karta hai
* Kubernetes cluster ke andar decrypt karta hai

Flow:

```
Plain Secret
     │
     ▼
kubeseal encrypt karta hai
     │
     ▼
SealedSecret banta hai
     │
     ▼
Git me store hota hai
     │
     ▼
Cluster decrypt karta hai
```

---

# 3. GitOps Workflow

Real workflow:

```
Developer
   │
   ▼
Secret.yaml
   │
   ▼
kubeseal
   │
   ▼
SealedSecret.yaml
   │
   ▼
Git
   │
   ▼
Kubernetes Controller
   │
   ▼
Secret create
```

Important point:

```
Git me kabhi plain Secret nahi hota
sirf encrypted SealedSecret hota hai
```

---

# 4. Install Sealed Secrets Controller

Helm repo add karo:

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
```

Helm repo update:

```
helm repo update
```

Controller install karo:

```
helm install sealed-secrets sealed-secrets/sealed-secrets \
-n security-system
```

Verify installation:

```
kubectl get pods -n security-system
```

Expected output:

```
sealed-secrets-controller
```

---

# 5. Plain Secret Create

File create karo:

```
secret.yaml
```

Content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
  namespace: demo-app
type: Opaque
stringData:
  password: supersecret
```

---

# 6. Fetch Controller Certificate

kubeseal cluster se public certificate fetch karta hai.

Command:

```
kubeseal --fetch-cert \
--controller-name sealed-secrets \
--controller-namespace security-system
```

Yeh verify karta hai ki CLI controller se connect kar raha hai.

---

# 7. Encrypt Secret

Secret ko encrypt karne ke liye command:

```
kubeseal \
--controller-name sealed-secrets \
--controller-namespace security-system \
-o yaml < secret.yaml > sealed-secret.yaml
```

Yeh command:

```
secret.yaml
```

ko convert karega:

```
sealed-secret.yaml
```

---

# 8. Check Encrypted File

File check karo:

```
cat sealed-secret.yaml
```

Expected structure:

```yaml
kind: SealedSecret
spec:
  encryptedData:
```

Iska matlab secret encrypt ho chuka hai.

---

# 9. Apply Sealed Secret

Cluster me apply karo:

```
kubectl apply -f sealed-secret.yaml
```

Yeh Kubernetes me ek **SealedSecret object** create karega.

---

# 10. Controller Decryption

Cluster ke andar running controller automatically karega:

```
SealedSecret
     │
     ▼
decrypt
     │
     ▼
Kubernetes Secret create
```

---

# 11. Verify Secret Created

Check karo:

```
kubectl get secret -n demo-app
```

Expected:

```
demo-secret
```

---

# 12. Check Secret Value

Secret inspect karo:

```
kubectl get secret demo-secret -n demo-app -o yaml
```

Example output:

```yaml
data:
  password: c3VwZXJzZWNyZXQ=
```

Decode karo:

```
echo c3VwZXJzZWNyZXQ= | base64 -d
```

Output:

```
supersecret
```

---

# 13. Final Architecture

```
Developer
   │
   ▼
Secret.yaml
   │
   ▼
kubeseal encrypt
   │
   ▼
SealedSecret.yaml
   │
   ▼
Git Repository
   │
   ▼
ArgoCD / GitOps
   │
   ▼
Kubernetes Cluster
   │
   ▼
Sealed Secrets Controller
   │
   ▼
Secret Created
```

---

# 14. Security Advantage

Agar Git repository leak bhi ho jaye:

Attacker ko sirf milega:

```
encryptedData: AgBcdF3s8n....
```

Secret read nahi kar paayega.

Kyuki:

```
Private key sirf Kubernetes cluster ke paas hoti hai
```

---

# 15. DevSecOps Stack

Typical production stack:

```
ArgoCD
Kyverno
Sealed Secrets
Prometheus
Grafana
```

Yeh stack:

* Secure GitOps
* Policy Enforcement
* Secret Encryption
* Observability

provide karta hai.

  
