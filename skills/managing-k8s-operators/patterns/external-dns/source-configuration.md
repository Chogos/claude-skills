# external-dns: Source Configuration

## Contents

- Ingress source
- Service source
- Gateway API source
- Annotations reference
- FQDN templates
- Filtering

## Ingress Source

external-dns creates DNS records from Ingress resources automatically.

### From Ingress Rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.example.com
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  ingressClassName: nginx
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

Without the `hostname` annotation, external-dns extracts hostnames from `.spec.rules[].host` and `.spec.tls[].hosts`.

### Multiple Hostnames

```yaml
annotations:
  external-dns.alpha.kubernetes.io/hostname: "app.example.com,api.example.com"
```

## Service Source

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.example.com
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
```

external-dns reads the LoadBalancer's external IP/hostname from `.status.loadBalancer.ingress` and creates the DNS record.

### Internal DNS (ClusterIP)

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.internal.example.com
    external-dns.alpha.kubernetes.io/internal-hostname: app.internal.example.com
    external-dns.alpha.kubernetes.io/access: private
spec:
  type: ClusterIP
  clusterIP: 10.100.200.50
```

Requires `--source=service` and the provider to support private DNS zones.

### Headless Service

external-dns creates one A record per pod IP for headless services:

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: kafka.example.com
spec:
  type: ClusterIP
  clusterIP: None
```

## Gateway API Source

```yaml
# enable the source
extraArgs:
  - --source=gateway-httproute
  - --source=gateway-grpcroute
  - --source=gateway-tlsroute
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app
  annotations:
    external-dns.alpha.kubernetes.io/hostname: app.example.com
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: app
          port: 8080
```

external-dns extracts hostnames from the HTTPRoute `.spec.hostnames` field or the annotation.

## Annotations Reference

| Annotation | Purpose | Example |
|-----------|---------|---------|
| `external-dns.alpha.kubernetes.io/hostname` | DNS name(s) for the resource | `app.example.com` |
| `external-dns.alpha.kubernetes.io/ttl` | Record TTL in seconds | `"300"` |
| `external-dns.alpha.kubernetes.io/target` | Override the target IP/hostname | `"1.2.3.4"` |
| `external-dns.alpha.kubernetes.io/access` | `public` or `private` zone | `"private"` |
| `external-dns.alpha.kubernetes.io/alias` | Route53 alias record | `"true"` |
| `external-dns.alpha.kubernetes.io/set-identifier` | Route53 routing policy set ID | `"eu-west-1"` |
| `external-dns.alpha.kubernetes.io/aws-weight` | Weighted routing weight | `"50"` |
| `external-dns.alpha.kubernetes.io/aws-region` | Latency-based routing region | `"eu-west-1"` |
| `external-dns.alpha.kubernetes.io/aws-failover` | Failover routing (`PRIMARY`/`SECONDARY`) | `"PRIMARY"` |

### Route53 Alias Records

Alias records avoid extra DNS lookups and don't count toward Route53 query charges:

```yaml
annotations:
  external-dns.alpha.kubernetes.io/hostname: app.example.com
  external-dns.alpha.kubernetes.io/alias: "true"
```

### Weighted Routing (Route53)

```yaml
# cluster-a service
annotations:
  external-dns.alpha.kubernetes.io/hostname: app.example.com
  external-dns.alpha.kubernetes.io/set-identifier: cluster-a
  external-dns.alpha.kubernetes.io/aws-weight: "70"
---
# cluster-b service
annotations:
  external-dns.alpha.kubernetes.io/hostname: app.example.com
  external-dns.alpha.kubernetes.io/set-identifier: cluster-b
  external-dns.alpha.kubernetes.io/aws-weight: "30"
```

## FQDN Templates

Auto-generate hostnames from resource metadata without annotations:

```yaml
extraArgs:
  - --fqdn-template={{.Name}}.{{.Namespace}}.example.com
```

Available template variables:

- `{{.Name}}` — resource name
- `{{.Namespace}}` — resource namespace
- `{{.Annotations}}` — access annotation values

Multiple templates (first match wins):

```yaml
extraArgs:
  - --fqdn-template={{.Name}}.{{.Namespace}}.example.com
  - --fqdn-template={{.Name}}.example.com
```

FQDN templates apply when the resource has no `hostname` annotation.

## Filtering

### Namespace Filtering

```yaml
extraArgs:
  - --namespace=production        # only watch this namespace
  # or multiple:
  - --namespace=production
  - --namespace=staging
```

### Annotation Filtering

```yaml
extraArgs:
  - --annotation-filter=external-dns.alpha.kubernetes.io/enabled=true
```

Only manage resources with the annotation `external-dns.alpha.kubernetes.io/enabled: "true"`.

### Label Filtering

```yaml
extraArgs:
  - --label-filter=app.kubernetes.io/managed-by=helm
```

### Domain Filtering

```yaml
extraArgs:
  - --domain-filter=example.com        # only create records under this domain
  - --exclude-domains=internal.example.com  # exclude subdomain
```
