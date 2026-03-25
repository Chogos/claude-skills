# cert-manager: Issuers & ClusterIssuers

## Contents

- Issuer vs ClusterIssuer
- ACME / Let's Encrypt
- CA Issuer
- Vault Issuer
- Issuer status and troubleshooting

## Issuer vs ClusterIssuer

- `Issuer` — namespace-scoped. Certificates must be in the same namespace.
- `ClusterIssuer` — cluster-wide. Any namespace can reference it.

Use `ClusterIssuer` when multiple namespaces need certificates from the same authority. Use `Issuer` when teams manage their own certificate infrastructure per namespace.

## ACME / Let's Encrypt

### Staging (for testing)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

### Production

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
      - dns01:
          route53:
            region: eu-west-1
            hostedZoneID: Z1234567890
        selector:
          dnsZones:
            - "example.com"
```

Always test with staging first — production has strict rate limits (50 certs/domain/week).

### Multiple Solvers

Combine HTTP01 and DNS01 on one issuer. Use `selector` to route challenges:

```yaml
solvers:
  - http01:
      ingress:
        ingressClassName: nginx
    selector:
      dnsNames:
        - "app.example.com"
  - dns01:
      cloudflare:
        apiTokenSecretRef:
          name: cloudflare-api-token
          key: api-token
    selector:
      dnsZones:
        - "example.com"
```

Wildcard certificates (`*.example.com`) require DNS01 — HTTP01 cannot validate wildcards.

## CA Issuer

### Self-Signed Root CA

Two-step: create a self-signed issuer, generate a CA cert, then create a CA issuer.

```yaml
# step 1: self-signed issuer (bootstrap only)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
# step 2: CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-ca
  duration: 87600h  # 10 years
  secretName: internal-ca-key-pair
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
  privateKey:
    algorithm: ECDSA
    size: 256
---
# step 3: CA issuer using the generated cert
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

Use for mTLS between internal services where public trust isn't needed.

## Vault Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-pki
spec:
  vault:
    server: https://vault.example.com
    path: pki/sign/my-role
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
        secretRef:
          name: vault-sa-token
          key: token
```

- Configure Vault PKI secrets engine with appropriate role and TTL
- Kubernetes auth method: operator service account authenticates to Vault
- Create the token secret from the SA: `kubectl create token cert-manager -n cert-manager > token && kubectl create secret generic vault-sa-token --from-file=token -n cert-manager`

### AppRole Auth

```yaml
spec:
  vault:
    server: https://vault.example.com
    path: pki/sign/my-role
    auth:
      appRole:
        path: approle
        roleId: "db02de05-fa39-4855-059b-67221c5c2f63"
        secretRef:
          name: vault-approle-secret
          key: secret-id
```

## Issuer Status and Troubleshooting

```bash
# check issuer readiness
kubectl get clusterissuer letsencrypt-prod -o wide
kubectl describe clusterissuer letsencrypt-prod

# common issues
# - "Failed to register ACME account" → wrong server URL or network issue
# - "secret not found" → privateKeySecretRef doesn't exist yet (auto-created on first use)
# - "403 Forbidden" → ACME account terms of service not accepted

# with cmctl (cert-manager CLI)
cmctl status issuer letsencrypt-prod
cmctl check api  # verify cert-manager API is reachable
```