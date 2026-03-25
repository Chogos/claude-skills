# cert-manager: Challenge Solvers

## Contents

- HTTP01 solver
- DNS01 solver
- Webhook solvers
- Solver selection
- Common issues

## HTTP01 Solver

Default solver. cert-manager creates a temporary pod + service to respond to ACME challenges.

```yaml
solvers:
  - http01:
      ingress:
        ingressClassName: nginx
```

### Pod and Service Customization

Override solver pod resources or node affinity:

```yaml
solvers:
  - http01:
      ingress:
        ingressClassName: nginx
        podTemplate:
          spec:
            resources:
              requests:
                cpu: 10m
                memory: 16Mi
            nodeSelector:
              kubernetes.io/os: linux
        serviceType: ClusterIP  # default; use NodePort if needed
```

### Gateway API HTTP01

```yaml
solvers:
  - http01:
      gatewayHTTPRoute:
        parentRefs:
          - name: app-gateway
            namespace: default
            kind: Gateway
```

Limitations:
- Cannot validate wildcard certificates
- Requires the domain to be publicly reachable on port 80
- Ingress controller must route `/.well-known/acme-challenge/` to the solver

## DNS01 Solver

Creates a TXT record to prove domain ownership. Required for wildcards and private networks.

### AWS Route53

```yaml
solvers:
  - dns01:
      route53:
        region: eu-west-1
        hostedZoneID: Z1234567890  # optional, auto-detected from domain
        # authentication: IRSA (recommended) or static credentials
```

IRSA setup (recommended):
```yaml
# cert-manager ServiceAccount annotation
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/cert-manager-route53
```

IAM policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:GetChange"],
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/Z1234567890"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
```

### Cloudflare

```yaml
solvers:
  - dns01:
      cloudflare:
        apiTokenSecretRef:
          name: cloudflare-api-token
          key: api-token
```

Token permissions: Zone → DNS → Edit. Scope to specific zones.

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<token>
```

### Google Cloud DNS

```yaml
solvers:
  - dns01:
      cloudDNS:
        project: my-gcp-project
        serviceAccountSecretRef:
          name: gcp-dns-sa
          key: key.json
```

### Azure DNS

```yaml
solvers:
  - dns01:
      azureDNS:
        subscriptionID: <sub-id>
        resourceGroupName: dns-rg
        hostedZoneName: example.com
        environment: AzurePublicCloud
        managedIdentity:
          clientID: <client-id>
```

## Webhook Solvers

For DNS providers not natively supported. Third-party webhook solvers extend cert-manager.

```yaml
solvers:
  - dns01:
      webhook:
        groupName: acme.example.com
        solverName: my-dns-solver
        config:
          apiKey:
            secretRef:
              name: dns-provider-secret
              key: api-key
```

Common third-party webhooks:
- `cert-manager-webhook-hetzner` — Hetzner DNS
- `cert-manager-webhook-ovh` — OVH DNS
- `cert-manager-webhook-infoblox` — Infoblox NIOS
- `cert-manager-webhook-pdns` — PowerDNS

Install via Helm chart provided by the webhook project.

## Solver Selection

Route challenges to different solvers using `selector`:

```yaml
solvers:
  # wildcard and zone-level: use DNS01
  - dns01:
      route53:
        region: eu-west-1
    selector:
      dnsZones:
        - "example.com"
  # specific hosts: use HTTP01
  - http01:
      ingress:
        ingressClassName: nginx
    selector:
      dnsNames:
        - "app.example.com"
        - "api.example.com"
```

Precedence: most specific selector wins. `dnsNames` is more specific than `dnsZones`, which is more specific than no selector.

## Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| HTTP01 challenge pending | Solver pod unreachable | Check ingress routes `/.well-known/acme-challenge/` correctly; verify no firewall blocking port 80 |
| DNS01 challenge pending | TXT record not propagating | Check provider credentials; increase `--dns01-recursive-nameservers-only` timeout; verify correct hosted zone |
| DNS01 "unauthorized" | Wrong IAM/API permissions | Audit provider credentials; ensure zone-level write access |
| Challenge stuck "processing" | Webhook solver unhealthy | Check webhook pod logs; verify webhook service reachable from cert-manager |
| "no matching solver" | No solver selector matches the domain | Add a solver with matching `dnsNames` or `dnsZones` selector |