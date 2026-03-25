# kube-prometheus-stack: Thanos Integration

## Contents

- Architecture overview
- Sidecar setup
- Object storage configuration
- Thanos Query
- Thanos Compactor
- Thanos Store Gateway

## Architecture Overview

Thanos extends Prometheus for long-term storage and multi-cluster querying.

**Sidecar mode** (most common with kube-prometheus-stack):
```
Prometheus Pod
├── prometheus container    → scrapes metrics, writes TSDB blocks
└── thanos-sidecar         → uploads blocks to object storage, serves StoreAPI

Thanos Query  → queries Sidecar (real-time) + Store Gateway (historical)
Thanos Store  → serves historical blocks from object storage
Thanos Compactor → downsamples + compacts blocks in object storage
```

**Receive mode** (alternative — Prometheus remote-writes to Thanos):
- Use when you can't add sidecars (managed Prometheus, multiple clusters with firewall restrictions)
- Configure Prometheus `remoteWrite` to Thanos Receive endpoint

## Sidecar Setup

Enable in kube-prometheus-stack values.yaml:

```yaml
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.36.1
      objectStorageConfig:
        existingSecret:
          name: thanos-objstore
          key: objstore.yml
      # resources for the sidecar container
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 100m
          memory: 256Mi
    # retain local data for 2h (sidecar uploads everything else)
    retention: 2h
    # external labels distinguish this Prometheus from others
    externalLabels:
      cluster: production-eu
      replica: "$(POD_NAME)"
```

The sidecar exposes a gRPC StoreAPI on port 10901 for Thanos Query.

## Object Storage Configuration

### S3

```yaml
# thanos-objstore secret
type: S3
config:
  bucket: thanos-metrics
  endpoint: s3.eu-west-1.amazonaws.com
  region: eu-west-1
  # IRSA: omit access_key/secret_key, use service account
  # static creds:
  # access_key: ...
  # secret_key: ...
```

### GCS

```yaml
type: GCS
config:
  bucket: thanos-metrics
  service_account: ""  # uses workload identity if empty
```

### Azure Blob

```yaml
type: AZURE
config:
  storage_account: thanosmetrics
  storage_account_key: ""  # or use managed identity
  container: thanos
```

Create the secret:

```bash
kubectl create secret generic thanos-objstore \
  --namespace monitoring \
  --from-file=objstore.yml=objstore.yml
```

## Thanos Query

Aggregates data from all StoreAPI endpoints (sidecars, store gateways, other queries).

```yaml
# deploy via thanos helm chart or standalone
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  template:
    spec:
      containers:
        - name: thanos-query
          image: quay.io/thanos/thanos:v0.36.1
          args:
            - query
            - --http-address=0.0.0.0:9090
            - --grpc-address=0.0.0.0:10901
            - --endpoint=dnssrv+_grpc._tcp.thanos-sidecar.monitoring.svc
            - --endpoint=dnssrv+_grpc._tcp.thanos-store.monitoring.svc
            - --query.replica-label=replica  # dedup across replicas
            - --query.auto-downsampling      # use downsampled data for long ranges
```

Deduplication:
- `--query.replica-label=replica` — deduplicates series from HA Prometheus pairs
- Requires matching `externalLabels` with a `replica` label on each Prometheus

Point Grafana datasource at Thanos Query instead of Prometheus for unified view.

## Thanos Compactor

Single-instance process (no HA) that compacts and downsamples blocks in object storage.

```yaml
args:
  - compact
  - --http-address=0.0.0.0:10902
  - --data-dir=/data
  - --objstore.config-file=/etc/thanos/objstore.yml
  - --retention.resolution-raw=30d
  - --retention.resolution-5m=180d
  - --retention.resolution-1h=365d
  - --compact.concurrency=1
  - --downsample.concurrency=1
  - --wait  # run continuously
```

Retention tiers:
- **Raw** (original resolution) — keep 30 days
- **5-minute downsampled** — keep 6 months
- **1-hour downsampled** — keep 1 year

Compactor needs a persistent volume for temporary working space during compaction.

## Thanos Store Gateway

Serves historical blocks from object storage via StoreAPI.

```yaml
args:
  - store
  - --http-address=0.0.0.0:10902
  - --grpc-address=0.0.0.0:10901
  - --data-dir=/data
  - --objstore.config-file=/etc/thanos/objstore.yml
  - --index-cache-size=500MB
  - --chunk-pool-size=2GB
```

- Caches block index in memory — size `index-cache-size` according to block count
- Persistent volume for block metadata caching
- Scales horizontally with `--sharding` for large datasets

## Monitoring Thanos Itself

Key metrics to watch:

```
thanos_objstore_bucket_operations_total{operation="upload"}     # block uploads
thanos_compact_group_compactions_total                          # compaction progress
thanos_store_series_data_fetched_total                          # store gateway queries
thanos_query_store_apis_dns_lookups_total                       # endpoint discovery
```

The kube-prometheus-stack includes Thanos-specific dashboards and alerts when sidecar is enabled.