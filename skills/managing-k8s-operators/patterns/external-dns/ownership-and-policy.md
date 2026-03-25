# external-dns: Ownership & Policy

## Contents

- TXT registry
- Policy modes
- Multi-cluster DNS
- DNS record types
- Safety practices

## TXT Registry

external-dns uses TXT records to track ownership. Each managed record has a corresponding TXT record that stores the owner ID.

```
app.example.com          A       1.2.3.4
extdns-app.example.com   TXT     "heritage=external-dns,external-dns/owner=my-cluster,..."
```

### Configuration

```yaml
extraArgs:
  - --registry=txt                  # always use TXT registry (default)
  - --txt-owner-id=prod-cluster     # unique per external-dns instance
  - --txt-prefix=extdns-            # prefix for TXT records
```

`--txt-prefix` prevents TXT record name collisions when the managed record is itself a TXT record or when using CNAME records (some providers don't allow TXT + CNAME on the same name).

### Ownership Rules

- external-dns only modifies records it owns (matching `txt-owner-id`)
- Records created manually or by another external-dns instance are untouched
- Changing `txt-owner-id` orphans all existing TXT records — the new instance won't recognize old records

### Migrating Owner ID

If you need to change the owner ID:

1. Scale down old external-dns instance
2. Update TXT records in DNS to reflect the new owner ID (script or manual)
3. Deploy with the new `txt-owner-id`

Or: delete all TXT ownership records and let the new instance recreate them (the managed A/CNAME records remain, external-dns re-adopts them).

## Policy Modes

| Policy | Creates | Updates | Deletes | Use Case |
|--------|---------|---------|---------|----------|
| `upsert-only` | Yes | Yes | No | Safe default — prevents accidental deletion |
| `sync` | Yes | Yes | Yes | Full lifecycle management — removes records when source is deleted |
| `create-only` | Yes | No | No | One-time creation, manual management afterward |

### upsert-only (Recommended Default)

```yaml
extraArgs:
  - --policy=upsert-only
```

Records persist even after the Ingress/Service is deleted. Clean up manually or switch to `sync` once confident.

### sync

```yaml
extraArgs:
  - --policy=sync
```

When an Ingress is deleted, external-dns deletes the corresponding DNS records. Use with:
- Tight `--domain-filter` to limit blast radius
- TXT registry to prevent deleting records owned by others
- Confidence that no external process manages records in the same zone

### create-only

```yaml
extraArgs:
  - --policy=create-only
```

Creates records once, never touches them again. Useful when DNS records need manual tuning after initial creation.

## Multi-Cluster DNS

### Unique Owner IDs

Each cluster must have a distinct `--txt-owner-id`:

```yaml
# cluster-a
extraArgs:
  - --txt-owner-id=cluster-a

# cluster-b
extraArgs:
  - --txt-owner-id=cluster-b
```

Both clusters can manage records in the same hosted zone without conflict — each only modifies records it owns.

### Shared Hosted Zone Patterns

**Split by subdomain:**
```yaml
# cluster-a manages eu.example.com
- --domain-filter=eu.example.com

# cluster-b manages us.example.com
- --domain-filter=us.example.com
```

**Shared domain with weighted routing (Route53):**
Both clusters create records for `app.example.com` with different `set-identifier` and weight annotations.

**Conflict prevention:**
- Always use TXT registry with unique owner IDs
- Use `upsert-only` policy when learning
- Monitor DNS changes via provider audit logs

## DNS Record Types

| Type | When Created | Notes |
|------|-------------|-------|
| A | LoadBalancer with IP, Ingress with IP | Most common |
| AAAA | IPv6 addresses | Automatic for dual-stack |
| CNAME | LoadBalancer with hostname (e.g., AWS ELB) | Cannot coexist with other record types on the same name |
| ALIAS | Route53 alias records | Set via annotation; avoids CNAME limitations at zone apex |
| TXT | Ownership records | Auto-managed by external-dns; don't edit manually |

### Zone Apex Records

DNS spec doesn't allow CNAME at zone apex (`example.com`). Use:
- Route53 ALIAS records (`external-dns.alpha.kubernetes.io/alias: "true"`)
- Cloudflare CNAME flattening (automatic)
- For other providers: use A records with a static IP

## Safety Practices

### Initial Setup

1. **Start with `--dry-run`** to preview changes:
   ```yaml
   extraArgs:
     - --dry-run
     - --log-level=debug
   ```
   Check logs: `kubectl logs -n external-dns deploy/external-dns`

2. **Use `upsert-only` policy** to prevent deletions

3. **Restrict scope** with domain and namespace filters:
   ```yaml
   extraArgs:
     - --domain-filter=example.com
     - --namespace=production
   ```

4. **Remove `--dry-run`** once satisfied with the planned changes

### Graduating to sync Policy

1. Run with `upsert-only` for a stabilization period
2. Verify TXT ownership records exist for all managed records
3. Confirm no external tooling manages records in the same zone
4. Switch to `sync` and monitor for unexpected deletions

### Monitoring

```bash
# check managed records
kubectl logs -n external-dns deploy/external-dns | grep "Desired"

# verify DNS resolution
dig +short app.example.com
dig +short TXT extdns-app.example.com
```

Key metrics (if metrics enabled):
- `external_dns_source_endpoints_total` — endpoints discovered from sources
- `external_dns_registry_endpoints_total` — endpoints after registry filtering
- `external_dns_controller_last_sync_timestamp_seconds` — last successful sync