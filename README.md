
# 🔐 SealedSecrets Re-encryption Feature Plan for kubeseal CLI

## 📘 Overview

This document outlines a detailed plan to implement an automated mechanism within the `kubeseal` CLI for re-encrypting all existing SealedSecrets in a Kubernetes cluster using the latest public key. This feature enhances secret lifecycle management while maintaining strict security standards and GitOps-friendly automation.

## 🧭 Objectives

The proposed `--reencrypt-all` feature will:
- Discover all existing SealedSecrets in the cluster
- Retrieve all active public keys
- Re-encrypt secrets using the most recent key
- Update the existing SealedSecret resources
- Maintain complete security over private key materials (decryption is performed server-side only)
- _(Bonus)_ Log operations and errors
- _(Bonus)_ Handle large volumes efficiently

## 🔍 1. Discover All SealedSecrets

To collect all existing SealedSecrets:

```bash
kubectl get sealedsecrets --all-namespaces -o yaml
```

Optional filters:
- `--namespace`
- `--label-selector="app=my-app"`

## 🔑 2. Fetch Active Public Keys

Existing support:

```bash
kubeseal --fetch-cert
```

Proposed new flag:

```bash
kubeseal --fetch-all-certs
```

Behind the scenes:

```bash
kubectl get secrets -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key=active
```

## 🔓 3. Decryption (Server-Side Inside Controller)

Private keys must remain within the controller. We propose:

1. Add `/reencrypt` endpoint to controller.
2. CLI sends encrypted SealedSecret.
3. Controller:
   - Decrypts with existing private key.
   - Re-encrypts with latest public key.
   - Returns updated SealedSecret.

## 🔐 4. Re-encrypt with Latest Public Key

Controller will:
- Use latest active public key.
- Preserve scope (strict, namespace-wide, cluster-wide).

## 🔁 5. Update SealedSecret Resources

Update manually:

```bash
kubectl apply -f sealedsecret-reencrypted.yaml
```

Optional annotation:

```yaml
annotations:
  sealedsecrets.bitnami.com/key-version: v6
```

## 📝 6. Logging and Reporting (Bonus)

CLI flag:

```bash
--report-file=/path/to/report.json
```

JSON Report structure:

```json
[
  {
    "name": "db-secret",
    "namespace": "production",
    "status": "success",
    "timestamp": "2025-05-10T02:34:00Z"
  },
  {
    "name": "api-key",
    "namespace": "staging",
    "status": "error",
    "message": "decryption failed"
  }
]
```

## 📈 7. Performance and Scalability (Bonus)

Optimizations:
- Use goroutines for concurrency
- Implement batching
- Add rate limiting
- Include retries with exponential backoff

## 🛡️ 8. Security Considerations (Bonus)

- Decryption is done only in-controller  
- Private keys never exposed or logged  
- Data in transit secured via mTLS  
- RBAC scoped to least privilege  
- For external automation tools (e.g., GitOps pipelines) operating outside the cluster network, ensure access is restricted via a secure VPN or private network routing to prevent unauthorized access to the controller's re-encryption endpoint.

## ⚙️ 9. Error Handling and Retries

- Retries for API operations
- Per-secret error capture
- Idempotent updates
- Logging of failures
- Optionally validate secrets pre-update

## 🔄 10. Integration with Key Rotation

Run `--reencrypt-all` after automatic key rotation using CronJobs to keep secrets up to date.

## 🔑 11. User Permissions (RBAC)

Necessary permissions:
- List, get, update SealedSecrets
- (Possibly) get secrets for fetching keys

RBAC example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sealedsecrets-reencrypt-role
  namespace: kube-system
rules:
- apiGroups: ["bitnami.com"]
  resources: ["sealedsecrets"]
  verbs: ["get", "list", "update"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sealedsecrets-reencrypt-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sealedsecrets-reencrypt-binding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: sealedsecrets-reencrypt-sa
  namespace: kube-system
roleRef:
  kind: Role
  name: sealedsecrets-reencrypt-role
  apiGroup: rbac.authorization.k8s.io
```

## 🧪 12. Testing and Rollback Strategy

- Test in non-production first.
- Use Git history as rollback mechanism.
- Revert SealedSecrets via Git and re-apply.

## 🔍 13. Controller Monitoring

- Monitor controller logs during re-encryption.
- Use dashboards/logging systems for insights.

## 📘 Example CLI Usage

```bash
kubeseal reencrypt-all   --namespace production   --label-selector app=backend   --report-file /tmp/sealedsecrets-report.json
```

## 📅 Scheduled Automation Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sealedsecrets-reencrypt
spec:
  schedule: "0 2 */30 * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: sealedsecrets-reencrypt-sa
          containers:
          - name: reencrypt
            image: yourrepo/kubeseal-enhanced
            command: ["kubeseal", "reencrypt-all"]
          restartPolicy: OnFailure
```

## ✅ Benefits

- Improved security with regular re-encryption
- Automation compatible with GitOps
- Secure, scalable, and auditable
- Resilient to key rotation and cluster scale

## 📌 Summary

The proposed `--reencrypt-all` feature extends the `kubeseal` CLI to automate secure secret re-encryption. It integrates naturally with GitOps workflows, respects security boundaries, and provides operational resilience at scale. This approach keeps private keys protected, improves observability, and supports future scalability needs.

---

## 📚 References

- [Bitnami SealedSecrets GitHub Repository](https://github.com/bitnami-labs/sealed-secrets)
- [kubeseal CLI Documentation](https://github.com/bitnami-labs/sealed-secrets#kubeseal)
- [Kubernetes Secrets Management](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Go Client for Kubernetes (client-go)](https://github.com/kubernetes/client-go)
