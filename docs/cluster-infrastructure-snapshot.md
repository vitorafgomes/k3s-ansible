# Cluster Infrastructure Snapshot

**Date:** 2026-02-05
**Cluster:** k3s v1.33.4+k3s1 (10 nodes, 2 masters + 8 workers)
**Age:** 126 days

---

## 1. Cluster Topology

### Nodes
| Hostname | Role | IP |
|----------|------|----|
| k3s-m-01 | control-plane, etcd, master | 192.168.50.110 |
| k3s-m-02 | control-plane, etcd, master | 192.168.50.115 |
| k3s-w-01 | worker | 192.168.50.111 |
| k3s-w-02 | worker | 192.168.50.112 |
| k3s-w-03 | worker | 192.168.50.113 |
| k3s-w-04 | worker | 192.168.50.114 |
| k3s-w-05 | worker | 192.168.50.116 |
| k3s-w-06 | worker | 192.168.50.117 |
| k3s-w-07 | worker | 192.168.50.118 |
| k3s-w-08 | worker | 192.168.50.119 |

### Network
- **API VIP:** 192.168.50.3
- **CNI:** Cilium v1.18.2 (native mode, Hubble enabled)
- **LB:** kube-vip v0.8.10 (ARP mode)
- **Pod CIDR:** 10.52.0.0/16
- **Service CIDR:** 10.43.0.0/16

### Namespaces
- cert-manager, cilium-secrets, cloudflare-tunnel, default, development, keda, keycloak, kube-node-lease, kube-public, kube-system, monitoring, smart-management, traefik

---

## 2. cert-manager (v1.14.4)

**Installation:** kubectl apply from official manifest
**Namespace:** cert-manager

### ClusterIssuers
| Name | Type | Server |
|------|------|--------|
| selfsigned-issuer | SelfSigned | - |
| letsencrypt-staging | ACME DNS-01 (Cloudflare) | acme-staging-v02.api.letsencrypt.org |
| letsencrypt-prod | ACME DNS-01 (Cloudflare) | acme-v02.api.letsencrypt.org |

**ACME email:** vitorafgomes@gmail.com
**Cloudflare secret:** `cloudflare-api-token-secret` (namespace: cert-manager, key: `api-token`)

### Certificates
| Name | Namespace | Issuer | DNS Names | Status |
|------|-----------|--------|-----------|--------|
| wildcard-vitorafgomes-net | traefik | letsencrypt-prod | *.vitorafgomes.net, vitorafgomes.net | **EXPIRED** (renewal pending) |
| wildcard-vitorafgomes-local | traefik | selfsigned-issuer | *.vitorafgomes.local, vitorafgomes.local | Ready |
| keycloak-tls | keycloak | letsencrypt-prod | keycloak.vitorafgomes.net | Ready |

**WARNING:** `wildcard-vitorafgomes-net` certificate is expired since 2025-12-30 with a pending renewal order.

---

## 3. Traefik (v3.5.2)

**Installation:** Helm chart traefik/traefik v37.1.1
**Namespace:** traefik
**Release name:** traefik
**Replicas:** 1

### Helm Values (user-supplied)
```yaml
additionalArguments:
  - --serverstransport.insecureskipverify=true
  - --certificatesresolvers.cf.acme.email=vitorafgomes@gmail.com
  - --certificatesresolvers.cf.acme.storage=/data/acme.json
  - --certificatesresolvers.cf.acme.dnschallenge=true
  - --certificatesresolvers.cf.acme.dnschallenge.provider=cloudflare
  - --certificatesresolvers.cf.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
  - --metrics.prometheus=true
  - --metrics.prometheus.addEntryPointsLabels=true
  - --metrics.prometheus.addRoutersLabels=true
  - --metrics.prometheus.addServicesLabels=true
  - --tracing=true
  - --tracing.otlp=true
  - --tracing.otlp.grpc.endpoint=alloy.monitoring.svc.cluster.local:4317
  - --tracing.otlp.grpc.insecure=true

api:
  dashboard: true
  insecure: true

env:
  - name: USER
    value: traefik
  - name: OTEL_PROPAGATORS
    value: tracecontext,baggage
  - name: CF_DNS_API_TOKEN
    valueFrom:
      secretKeyRef:
        key: CF_DNS_API_TOKEN
        name: cloudflare-api-token

service:
  type: ClusterIP
  spec:
    externalIPs:
      - 192.168.50.111
      - 192.168.50.112
      - 192.168.50.113
      - 192.168.50.114
      - 192.168.50.116
      - 192.168.50.117
      - 192.168.50.118
      - 192.168.50.119

persistence:
  enabled: true
  size: 128Mi

logs:
  access:
    enabled: true
    format: json
    fields:
      defaultMode: keep
      headers:
        defaultMode: drop
        names:
          Content-Type: keep
          Referer: keep
          User-Agent: keep
          X-Forwarded-For: keep

ports:
  websecure:
    http3:
      enabled: true
    tls:
      enabled: true

providers:
  kubernetesCRD:
    allowCrossNamespace: true
    allowEmptyServices: true
    enabled: true
  kubernetesGateway:
    enabled: true
  kubernetesIngress:
    enabled: false

tracing:
  otlp:
    grpc:
      enabled: true
      endpoint: alloy.monitoring.svc.cluster.local:4317
      insecure: true
```

### Secrets (namespace: traefik)
- `cloudflare-api-token` (key: `CF_DNS_API_TOKEN`)
- `wildcard-vitorafgomes-net-tls` (TLS cert)
- `wildcard-vitorafgomes-local-tls` (TLS cert)

### IngressRoutes
| Name | Namespace | Match | Service | TLS |
|------|-----------|-------|---------|-----|
| alloy-otel-cors | traefik | Host(\`otel.vitorafgomes.net\`) | alloy.monitoring:4318 | wildcard-vitorafgomes-net-tls |

### Middlewares
| Name | Namespace | Type | Config |
|------|-----------|------|--------|
| circuit-breaker-backend | traefik | CircuitBreaker | NetworkErrorRatio() > 0.30 \|\| ResponseCodeRatio(500,600,0,600) > 0.25, checkPeriod=10s, fallback=30s, recovery=60s |
| ip-whitelist-admin | traefik | IPWhiteList | sourceRange: 192.168.50.0/24, 10.0.0.0/8, ipStrategy.depth=1 |
| rate-limit-api | traefik | RateLimit | average=100, burst=50, period=1s |
| retry-policy | traefik | Retry | attempts=3, initialInterval=100ms |
| cors-otel | traefik | Headers (CORS) | origins: https://smart-management.vitorafgomes.net, https://ui.vitorafgomes.net, credentials=true, headers=*, methods=GET/POST/PUT/DELETE/OPTIONS |
| cors-otel | monitoring | Headers (CORS) | Same as above (duplicate in monitoring namespace) |
| strip-keycloak-prefix | smart-management | StripPrefix | /keycloak-orchestrator |

---

## 4. Cloudflare Tunnel

**Namespace:** cloudflare-tunnel
**Deployment name:** cloudflared-deployment
**Replicas:** 2
**Image:** cloudflare/cloudflared (not pinned)

### Secret
- `tunnel-credentials` (tunnel token from Cloudflare Dashboard)

---

## 5. Keycloak (v26.3.3)

**Namespace:** keycloak
**Image:** quay.io/keycloak/keycloak:26.3.3
**Replicas:** 1

### Database
- **Type:** PostgreSQL (external)
- **Host:** 192.168.50.240:5432
- **Database:** keycloak
- **Schema:** public

### Environment Variables
```
KC_HOSTNAME=keycloak.vitorafgomes.net
KC_HOSTNAME_STRICT=false
KC_HOSTNAME_STRICT_HTTPS=true
KC_HTTP_ENABLED=false
KC_PROXY=edge
KC_PROXY_HEADERS=xforwarded
KC_HEALTH_ENABLED=true
KC_METRICS_ENABLED=true
KC_LOG_LEVEL=INFO
KC_LOG_CONSOLE_OUTPUT=json
KC_TRACING_ENABLED=true
KC_TRACING_ENDPOINT=http://alloy.monitoring.svc.cluster.local:4317
KC_TRACING_PROTOCOL=grpc
KC_TRACING_SAMPLER_TYPE=traceidratio
KC_TRACING_SAMPLER_RATIO=1.0
OTEL_SERVICE_NAME=keycloak
OTEL_EXPORTER_OTLP_ENDPOINT=http://alloy.monitoring.svc.cluster.local:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### PVCs
| Name | Size | Mount |
|------|------|-------|
| kc-data | 5Gi | /opt/keycloak/data |
| kc-themes | 1Gi | /opt/keycloak/themes |
| kc-providers | 500Mi | /opt/keycloak/providers |
| kc-conf | 500Mi | /opt/keycloak/conf |

### TLS
- Certificate `keycloak-tls` (letsencrypt-prod, DNS: keycloak.vitorafgomes.net)
- Mounted at `/opt/keycloak/conf/tls`

### Services
| Name | Type | Ports |
|------|------|-------|
| keycloak | ClusterIP | 8443/TCP (https), 8080/TCP (http) |
| keycloak-metrics | ClusterIP | 9000/TCP (metrics) |

### Secrets
| Name | Keys |
|------|------|
| keycloak-admin | KC_BOOTSTRAP_ADMIN_USERNAME, KC_BOOTSTRAP_ADMIN_PASSWORD |
| keycloak-db | KC_DB_USERNAME, KC_DB_PASSWORD |
| keycloak-tls | tls.crt, tls.key |

### ServiceAccount
- `keycloak` (namespace: keycloak)

### Init Container
- `wait-postgres` (busybox:1.36) - Waits for PostgreSQL at 192.168.50.240:5432

### Resources
- Requests: 500m CPU, 1Gi memory
- Limits: 2000m CPU, 2Gi memory

### Health Checks
- Liveness: HTTPS /health/live:9000 (initial=90s, period=30s)
- Readiness: HTTPS /health/ready:9000 (initial=60s, period=10s)

### Realms (discovered via public endpoints)
- **master** (default Keycloak realm)
- **smart-management** (application realm)

**NOTE:** Keycloak admin credentials were changed after bootstrap. To export full realm/client configuration, access the Keycloak Admin Console at https://keycloak.vitorafgomes.net or use `kcadm.sh` CLI with the current admin password.

---

## 6. Monitoring Stack

### 6.1 kube-prometheus-stack (chart v77.12.0)
**Release:** kps
**Namespace:** monitoring

**User-supplied values:**
```yaml
alertmanager:
  enabled: true
grafana:
  enabled: false   # Grafana deployed separately
prometheus:
  prometheusSpec:
    enableRemoteWriteReceiver: true
    resources:
      limits: { cpu: "1", memory: 4Gi }
      requests: { cpu: 200m, memory: 1Gi }
    retention: 15d
prometheus-node-exporter:
  enabled: true
```

**Components:**
- Prometheus v0.85.0 (operator), retention 15d, remote write receiver enabled
- Alertmanager v0.28.1
- kube-state-metrics v2.17.0
- node-exporter v1.9.1 (DaemonSet, all 10 nodes)

### 6.2 Grafana (chart v10.0.0, app v12.1.1)
**Release:** grafana
**Namespace:** monitoring

**User-supplied values:**
```yaml
adminUser: admin
adminPassword: <REDACTED>

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus (default)
        type: prometheus
        url: http://kps-kube-prometheus-stack-prometheus.monitoring.svc:9090
        uid: PBFA97CFB590B2093
      - name: Loki
        type: loki
        url: http://loki-gateway.monitoring.svc.cluster.local
        uid: P8E80F9AEF21F6940
        # Derived fields: trace_id extraction -> Tempo
      - name: Tempo
        type: tempo
        url: http://tempo.monitoring.svc.cluster.local:3200
        uid: P214B5B846CF3925F
        # tracesToLogs -> Loki
        # tracesToMetrics -> Prometheus
        # tracesToProfiles -> Pyroscope
      - name: Pyroscope
        type: grafana-pyroscope-datasource
        url: http://pyroscope.monitoring.svc.cluster.local:4040
        uid: pyroscope

sidecar:
  dashboards:
    enabled: true
    searchNamespace: ALL
    label: grafana_dashboard
```

### 6.3 Loki (chart v6.41.1, app v3.5.5)
**Release:** loki
**Namespace:** monitoring
**Mode:** SingleBinary (1 replica)

**User-supplied values:**
```yaml
deploymentMode: SingleBinary
singleBinary:
  replicas: 1
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  limits_config:
    allow_structured_metadata: true
    volume_enabled: true
  pattern_ingester:
    enabled: true
  schemaConfig:
    configs:
      - from: "2024-04-01"
        schema: v13
        store: tsdb
        object_store: filesystem
        index: { period: 24h, prefix: loki_index_ }
  storage:
    type: filesystem
chunksCache:
  enabled: true
  allocatedMemory: 2048
  resources: { limits: { cpu: 500m, memory: 2Gi }, requests: { cpu: 100m, memory: 2Gi } }
```

### 6.4 Tempo (chart v1.23.3, app v2.8.2)
**Release:** tempo
**Namespace:** monitoring

**User-supplied values:**
```yaml
tempo:
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }
        http: { endpoint: 0.0.0.0:4318 }
  storage:
    trace:
      backend: local
      local: { path: /var/tempo/traces }
      wal: { path: /var/tempo/wal }
  compactor:
    compaction:
      block_retention: 48h
  metricsGenerator:
    enabled: true
    remoteWriteUrl: http://kps-kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/write
  per_tenant_overrides:
    '*':
      max_bytes_per_trace: 5000000
      metrics_generator_processors: [service-graphs, span-metrics, local-blocks]
  search:
    enabled: true
```

### 6.5 Alloy (chart v1.3.0, app v1.11.0)
**Release:** alloy
**Namespace:** monitoring
**Replicas:** 2

**Key configuration:**
- OTLP receiver (gRPC :4317, HTTP :4318)
- Batch processor (1000 items, 10s timeout)
- Exports: traces -> Tempo, logs -> Loki, metrics -> Prometheus (remote_write)
- Kubernetes discovery: pods, services, endpoints, nodes
- Log collection: all pods (except monitoring self-logs), JSON parsing with structured_metadata
- Metrics scraping: annotated services, Traefik metrics, kubelet/cAdvisor
- Pyroscope profiling: pods with `pyroscope.io/scrape=true` annotation
- RBAC: ClusterRole for node/pod/service read + metrics endpoints

**Extra ports:** 4317 (OTLP gRPC), 4318 (OTLP HTTP)
**Resources:** requests: 300m CPU, 768Mi | limits: 2 CPU, 2Gi

### 6.6 Pyroscope (chart v1.15.0, app v1.14.1)
**Release:** pyroscope
**Namespace:** monitoring

**User-supplied values:**
```yaml
pyroscope:
  replicaCount: 1
  persistence: { enabled: true, size: 10Gi }
  service: { port: 4040, type: ClusterIP }
  resources: { limits: { cpu: 1, memory: 2Gi }, requests: { cpu: 200m, memory: 512Mi } }
  extraArgs: { log.level: info }
```

### 6.7 DB Exporters (kubectl apply manifests)
| Exporter | Image | Target |
|----------|-------|--------|
| mongodb-exporter | percona/mongodb_exporter:0.40.0 | Port 9216 |
| postgres-exporter | prometheuscommunity/postgres-exporter:v0.15.0 | Port 9187 |
| redis-exporter | (not visible in image) | Port 9121 |

### 6.8 OTel Collector (kubectl apply manifest)
- Image: otel/opentelemetry-collector-contrib:latest
- Ports: 12347, 4317 (gRPC)

---

## 7. KEDA (v2.18.1)

**Installation:** Helm chart keda/keda v2.18.1
**Namespace:** keda
**User-supplied values:** none (defaults)

**Components:**
- keda-operator
- keda-admission-webhooks
- keda-operator-metrics-apiserver

---

## 8. Helm Releases Summary

| Release | Chart | Version | Namespace |
|---------|-------|---------|-----------|
| alloy | alloy-1.3.0 | v1.11.0 | monitoring |
| cilium | cilium-1.18.2 | 1.18.2 | kube-system |
| grafana | grafana-10.0.0 | 12.1.1 | monitoring |
| keda | keda-2.18.1 | 2.18.1 | keda |
| kps | kube-prometheus-stack-77.12.0 | v0.85.0 | monitoring |
| loki | loki-6.41.1 | 3.5.5 | monitoring |
| pyroscope | pyroscope-1.15.0 | 1.14.1 | monitoring |
| tempo | tempo-1.23.3 | 2.8.2 | monitoring |
| traefik | traefik-37.1.1 | v3.5.2 | traefik |

---

## 9. Observability Pipeline

```
Applications (OTLP)
       |
       v
  Alloy (2 replicas)
  gRPC:4317 / HTTP:4318
       |
       +---> Tempo (traces)          --> Grafana "Tempo" datasource
       +---> Loki (logs via OTLP)    --> Grafana "Loki" datasource
       +---> Prometheus (metrics)    --> Grafana "Prometheus" datasource
       +---> Pyroscope (profiling)   --> Grafana "Pyroscope" datasource

  Alloy also:
    - Scrapes kubelet/cAdvisor metrics
    - Scrapes annotated services (prometheus.io/scrape=true)
    - Scrapes Traefik metrics
    - Collects pod logs -> Loki
    - Scrapes pprof profiles -> Pyroscope

  Tempo -> Prometheus (metrics_generator: service-graphs, span-metrics)
  Keycloak -> Alloy (OTLP gRPC tracing)
  Traefik -> Alloy (OTLP gRPC tracing)
```

---

## 10. Secrets Inventory (to recreate)

| Secret | Namespace | Keys | Source |
|--------|-----------|------|--------|
| cloudflare-api-token-secret | cert-manager | api-token | Cloudflare API Token (DNS Edit) |
| cloudflare-api-token | traefik | CF_DNS_API_TOKEN | Same Cloudflare API Token |
| tunnel-credentials | cloudflare-tunnel | token | Cloudflare Tunnel Token |
| keycloak-admin | keycloak | KC_BOOTSTRAP_ADMIN_USERNAME, KC_BOOTSTRAP_ADMIN_PASSWORD | Manual |
| keycloak-db | keycloak | KC_DB_USERNAME, KC_DB_PASSWORD | PostgreSQL credentials |
| keycloak-tls | keycloak | tls.crt, tls.key | Auto-generated by cert-manager |
| wildcard-vitorafgomes-net-tls | traefik | tls.crt, tls.key | Auto-generated by cert-manager |
| wildcard-vitorafgomes-local-tls | traefik | tls.crt, tls.key | Auto-generated by cert-manager |

---

## 11. Known Issues

1. **wildcard-vitorafgomes-net certificate EXPIRED** since 2025-12-30. Renewal order is stuck in "pending" state for 66 days.
2. **OTel Collector uses :latest tag** - not pinned, non-reproducible.
3. **Cloudflared uses unpinned image** - should pin to specific version.
4. **Grafana admin password is in plain text** in Helm user-supplied values.

---

## 12. TODO for Keycloak Export

To complete the Keycloak configuration export (realms, clients, roles, users), run:

```bash
# Option 1: via Keycloak Admin REST API
KC_TOKEN=$(curl -sk "https://keycloak.vitorafgomes.net/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" -d "username=admin" -d "password=<ADMIN_PASSWORD>" \
  -d "grant_type=password" | jq -r '.access_token')

# Export master realm
curl -sk -H "Authorization: Bearer $KC_TOKEN" \
  "https://keycloak.vitorafgomes.net/admin/realms/master" | jq . > keycloak-realm-master.json

# Export smart-management realm with clients
curl -sk -H "Authorization: Bearer $KC_TOKEN" \
  "https://keycloak.vitorafgomes.net/admin/realms/smart-management" | jq . > keycloak-realm-smart-management.json

# List clients per realm
curl -sk -H "Authorization: Bearer $KC_TOKEN" \
  "https://keycloak.vitorafgomes.net/admin/realms/smart-management/clients" | jq '.[].clientId' > keycloak-clients-smart-management.txt

# Option 2: via kc.sh export (full realm export including users)
kubectl exec -n keycloak <pod> -- /opt/keycloak/bin/kc.sh export --dir /tmp/export --realm smart-management
kubectl cp keycloak/<pod>:/tmp/export ./keycloak-export/
```
