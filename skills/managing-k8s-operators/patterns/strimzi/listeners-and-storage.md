# Strimzi: Listeners and Storage

## Contents

- Listener types
- Authentication per listener
- Listener configuration examples
- Storage types
- JBOD configuration
- Volume expansion

## Listener Types

| Type | Use Case | Access |
|------|----------|--------|
| `internal` | In-cluster clients | ClusterIP Service |
| `route` | OpenShift external access | OpenShift Route |
| `loadbalancer` | External access via cloud LB | One LoadBalancer per broker |
| `nodeport` | External access without LB | NodePort on each node |
| `ingress` | External access via Ingress | Requires TLS passthrough |
| `cluster-ip` | Headless in-cluster access | Direct pod IPs |

## Authentication Per Listener

- `tls` — mutual TLS (client certificate)
- `scram-sha-512` — username/password
- `oauth` — OAuth 2.0 token-based
- none (omit `authentication` block)

Each listener can have a different authentication mechanism. Mix internal plain access for trusted workloads with TLS/SCRAM for external clients.

## Listener Configuration Examples

### External LoadBalancer with SCRAM

```yaml
listeners:
  - name: external
    port: 9094
    type: loadbalancer
    tls: true
    authentication:
      type: scram-sha-512
    configuration:
      bootstrap:
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: nlb
          service.beta.kubernetes.io/aws-load-balancer-internal: "true"
```

### Internal with OAuth

```yaml
listeners:
  - name: oauth
    port: 9095
    type: internal
    tls: true
    authentication:
      type: oauth
      validIssuerUri: https://keycloak.example.com/realms/kafka
      jwksEndpointUri: https://keycloak.example.com/realms/kafka/protocol/openid-connect/certs
      userNameClaim: preferred_username
      tlsTrustedCertificates:
        - secretName: keycloak-ca
          certificate: ca.crt
```

### Ingress (TLS Passthrough)

```yaml
listeners:
  - name: ingress
    port: 9094
    type: ingress
    tls: true
    authentication:
      type: tls
    configuration:
      class: nginx
      bootstrap:
        host: kafka-bootstrap.example.com
      brokers:
        - broker: 0
          host: kafka-0.example.com
        - broker: 1
          host: kafka-1.example.com
        - broker: 2
          host: kafka-2.example.com
```

Requires the ingress controller to support TLS passthrough (`nginx.ingress.kubernetes.io/ssl-passthrough: "true"`).

## Storage Types

| Type | Use Case | Persistence |
|------|----------|-------------|
| `ephemeral` | Dev/test | Lost on pod restart |
| `persistent-claim` | Single volume per broker | PVC per broker |
| `jbod` | Multiple volumes per broker | Multiple PVCs per broker |

Storage is defined on the `KafkaNodePool`, not the Kafka CR.

## JBOD Configuration

Multiple volumes per broker for higher throughput:

```yaml
# in KafkaNodePool spec
storage:
  type: jbod
  volumes:
    - id: 0
      type: persistent-claim
      size: 500Gi
      class: gp3
      deleteClaim: false
    - id: 1
      type: persistent-claim
      size: 500Gi
      class: gp3
      deleteClaim: false
```

`deleteClaim: false` preserves PVCs when the cluster is deleted — use for production. Use `true` for dev/staging to avoid orphaned volumes.

Start with one JBOD volume. Add additional volumes only if disk throughput is saturated during broker recovery or consumer replay.

## Volume Expansion

Strimzi supports online volume expansion if the StorageClass allows it (`allowVolumeExpansion: true`):

```yaml
# increase size in the KafkaNodePool — Strimzi patches the PVCs
storage:
  type: jbod
  volumes:
    - id: 0
      type: persistent-claim
      size: 1000Gi  # was 500Gi
      class: gp3
```

Kubernetes does not support shrinking volumes or changing StorageClass. Plan capacity ahead.