# cert-manager: Certificate Resources

## Contents

- Certificate CR basics
- Renewal and rotation
- Ingress integration
- Gateway API integration
- Keystores
- Troubleshooting

## Certificate CR

### Basic Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: app
spec:
  secretName: app-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - app.example.com
    - www.app.example.com
  duration: 2160h    # 90 days
  renewBefore: 720h  # 30 days before expiry
```

### Wildcard Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-tls
  namespace: app
spec:
  secretName: wildcard-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.example.com"
    - example.com
```

Wildcard certs require DNS01 challenge solver on the issuer.

### Private Key Options

```yaml
spec:
  privateKey:
    algorithm: ECDSA       # RSA (default), ECDSA, Ed25519
    size: 256              # ECDSA: 256, 384; RSA: 2048, 4096
    rotationPolicy: Always # regenerate key on each renewal
```

`rotationPolicy: Always` — generates a new private key on every renewal. Use when key rotation is required by compliance.

## Renewal and Rotation

cert-manager renews automatically when `renewBefore` threshold is reached.

```bash
# check certificate status
kubectl get certificate -n app app-tls

# detailed status
cmctl status certificate app-tls -n app

# force renewal
cmctl renew app-tls -n app
```

Renewal lifecycle:
1. cert-manager creates a `CertificateRequest`
2. CertificateRequest triggers an `Order` (ACME) or direct signing (CA/Vault)
3. Order creates `Challenge` resources (ACME only)
4. Challenges are solved → certificate issued → Secret updated
5. Pods using the Secret pick up new cert (may need restart depending on app)

Status conditions to check:
- `Ready=True` — certificate is valid and stored in Secret
- `Issuing=True` — renewal in progress

## Ingress Integration

cert-manager auto-creates Certificate resources from Ingress annotations:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 8080
```

cert-manager watches Ingress resources with the annotation and creates a matching Certificate CR. The `secretName` in `tls` becomes the Certificate's `.spec.secretName`.

## Gateway API Integration

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: app.example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: app-tls-cert
```

## Keystores

### JKS (Java applications)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: java-app-tls
spec:
  secretName: java-app-tls-cert
  issuerRef:
    name: internal-ca
    kind: ClusterIssuer
  dnsNames:
    - java-app.example.com
  keystores:
    jks:
      create: true
      passwordSecretRef:
        name: jks-password
        key: password
```

The Secret will contain `keystore.jks` and `truststore.jks` alongside `tls.crt` and `tls.key`.

### PKCS12

```yaml
  keystores:
    pkcs12:
      create: true
      passwordSecretRef:
        name: pkcs12-password
        key: password
```

## Troubleshooting

```bash
# certificate not becoming Ready
kubectl describe certificate app-tls -n app
cmctl status certificate app-tls -n app

# check CertificateRequest
kubectl get certificaterequest -n app

# check Order and Challenge (ACME only)
kubectl get order -n app
kubectl get challenge -n app

# common issues:
# - Challenge stuck "pending" → HTTP01 solver pod unreachable (firewall, ingress misconfigured)
# - Challenge "failed" → DNS01 propagation timeout, wrong credentials
# - "issuer not ready" → issuer has errors (check issuer status first)
# - Secret not updating → check RBAC, cert-manager controller logs

# operator logs
kubectl logs -n cert-manager deploy/cert-manager --tail=100
```

Challenge debugging for HTTP01:
```bash
# check solver pod and service
kubectl get pods,svc -n app -l acme.cert-manager.io/http01-solver=true

# test solver endpoint
curl http://app.example.com/.well-known/acme-challenge/<token>
```