# external-dns: Provider Setup

## Contents

- AWS Route53
- Cloudflare
- Azure DNS
- Google Cloud DNS
- Common provider settings

## AWS Route53

### IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets"],
      "Resource": "arn:aws:route53:::hostedzone/Z1234567890"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}
```

### IRSA (IAM Roles for Service Accounts)

```yaml
# Helm values
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/external-dns

provider:
  name: aws

extraArgs:
  - --aws-zone-type=public   # public, private, or omit for both
  - --domain-filter=example.com
```

### Hosted Zone Filtering

```yaml
extraArgs:
  - --domain-filter=example.com           # only manage records under this domain
  - --zone-id-filter=Z1234567890          # restrict to specific hosted zone
  - --aws-zone-type=public               # skip private hosted zones
```

Use `--zone-id-filter` when you have multiple hosted zones for the same domain (public + private).

## Cloudflare

### API Token

Create a token with permissions: Zone → DNS → Edit. Scope to specific zones.

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace external-dns \
  --from-literal=cloudflare_api_token=<token>
```

### Helm Values

```yaml
provider:
  name: cloudflare

env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-token
        key: cloudflare_api_token

extraArgs:
  - --cloudflare-proxied             # enable Cloudflare proxy (orange cloud)
  - --domain-filter=example.com
```

Proxied vs DNS-only:
- `--cloudflare-proxied` — routes traffic through Cloudflare CDN/WAF (A/AAAA records only)
- Without flag — DNS-only records, no Cloudflare proxy

## Azure DNS

### Managed Identity

```yaml
provider:
  name: azure

extraArgs:
  - --azure-resource-group=dns-rg
  - --azure-subscription-id=<sub-id>
  - --domain-filter=example.com

podLabels:
  aadpodidbinding: external-dns  # Azure AD Pod Identity
```

### Service Principal

```bash
# create azure.json
cat <<EOF > azure.json
{
  "tenantId": "<tenant-id>",
  "subscriptionId": "<sub-id>",
  "resourceGroup": "dns-rg",
  "aadClientId": "<client-id>",
  "aadClientSecret": "<client-secret>"
}
EOF

kubectl create secret generic azure-config \
  --namespace external-dns \
  --from-file=azure.json
```

```yaml
extraVolumes:
  - name: azure-config
    secret:
      secretName: azure-config
extraVolumeMounts:
  - name: azure-config
    mountPath: /etc/kubernetes
    readOnly: true
```

## Google Cloud DNS

### Workload Identity (GKE)

```yaml
provider:
  name: google

extraArgs:
  - --google-project=my-project
  - --domain-filter=example.com

serviceAccount:
  annotations:
    iam.gke.io/gcp-service-account: external-dns@my-project.iam.gserviceaccount.com
```

### Service Account Key

```bash
kubectl create secret generic gcp-dns-sa \
  --namespace external-dns \
  --from-file=credentials.json=sa-key.json
```

```yaml
extraArgs:
  - --google-project=my-project

env:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /etc/gcp/credentials.json

extraVolumes:
  - name: gcp-creds
    secret:
      secretName: gcp-dns-sa
extraVolumeMounts:
  - name: gcp-creds
    mountPath: /etc/gcp
    readOnly: true
```

## Common Provider Settings

These settings apply regardless of provider:

```yaml
extraArgs:
  - --registry=txt                    # always use TXT registry for ownership tracking
  - --txt-owner-id=my-cluster         # unique per external-dns instance / cluster
  - --txt-prefix=extdns-              # prefix for TXT ownership records
  - --interval=1m                     # sync interval (default 1m)
  - --log-level=info                  # debug for troubleshooting
  - --policy=upsert-only             # safe default — see ownership-and-policy.md
  - --source=ingress                  # sources to watch (ingress, service, gateway-httproute)
  - --source=service
```

`--txt-owner-id` must be unique per external-dns instance. If two instances share an owner ID, they will conflict over record ownership.

First-time setup:

```bash
# start with dry-run to verify behavior
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --values values.yaml \
  --set extraArgs[0]=--dry-run

# check logs for planned changes
kubectl logs -n external-dns deploy/external-dns

# remove --dry-run once verified
```