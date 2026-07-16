---
name: higress-helm
description: Deploy, configure, or upgrade Higress AI Gateway on Kubernetes via Helm chart. Covers Gateways, plugins, Ingress/CRD routing, observability, and production patterns.
---

# Higress — Helm Chart

**Repo:** `https://higress.io/helm-charts`  
**Charts:** `higress`, `higress-core`, `higress-console`  
**Latest:** 2.2.2  
**Images:** `higress-registry.cn-hangzhou.cr.aliyuncs.com/higress/` (hub)

## Quick Install

```bash
helm repo add higress.io https://higress.io/helm-charts
helm repo update

helm install higress higress.io/higress \
  -n higress-system --create-namespace
```

### Minimal Gateway-Only

```bash
helm install higress higress.io/higress -n higress-system \
  --create-namespace \
  --set global.o11y.enabled=false \
  --set controller.replicas=1 \
  --set gateway.replicas=2
```

## Charts Overview

### `higress` (umbrella)

Deploys everything: `higress-core` + `higress-console` + optional o11y stack.

| Sub-component | Chart | Enabled by default |
|--------------|-------|-------------------|
| Gateway + Controller + Pilot | `higress-core` | ✅ |
| Web UI Console | `higress-console` | ✅ |
| Redis (AI caching, rate limiting) | included | ❌ (`global.enableRedis`) |
| Grafana + Prometheus + Loki | included | ❌ (`global.o11y.enabled`) |
| Plugin Server | included | ❌ (`global.enablePluginServer`) |

### `higress-core`

Core engine only: controller, gateway, pilot, optional Redis.

### `higress-console`

Web UI dashboard only (requires `higress-core` for backend).

## Global Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `global.hub` | `higress-registry.cn-hangzhou.cr.aliyuncs.com` | Image registry |
| `global.imagePullPolicy` | `""` | Image pull policy |
| `global.imagePullSecrets` | `[]` | Image pull secrets |
| `global.ingressClass` | `higress` | IngressClass to watch |
| `global.watchNamespace` | `""` | Restrict to single namespace |
| `global.enableH3` | `false` | Enable HTTP/3 (QUIC) |
| `global.enableIPv6` | `false` | Enable IPv6 |
| `global.enableProxyProtocol` | `false` | Proxy protocol |
| `global.enableRedis` | `false` | Deploy Redis for AI caching |
| `global.enablePluginServer` | `false` | Deploy Wasm plugin server |
| `global.enableIstioAPI` | `true` | Watch Istio CRDs |
| `global.enableGatewayAPI` | `true` | Watch Gateway API CRDs |
| `global.enableInferenceExtension` | `false` | Gateway API Inference Extension |
| `global.enableStatus` | `true` | Update Ingress status field |
| `global.local` | `false` | Local/kind cluster mode |
| `global.o11y.enabled` | `false` | Deploy observability stack |
| `global.logging.level` | `default:info` | Log level |
| `global.defaultResources` | `{cpu: 10m}` | Default resource requests |
| `global.priorityClassName` | `""` | Priority class |
| `global.multiCluster.enabled` | `true` | Multi-cluster support |
| `global.multiCluster.clusterName` | `""` | Cluster name |

## Gateway

| Parameter | Default | Description |
|-----------|---------|-------------|
| `gateway.name` | `higress-gateway` | Gateway deployment name |
| `gateway.replicas` | `2` | Pod count |
| `gateway.kind` | `Deployment` | `Deployment` or `DaemonSet` |
| `gateway.image` | `gateway` | Image name (hub/gateway) |
| `gateway.tag` | `""` (chart appVersion) | Image tag |
| `gateway.httpPort` | `80` | HTTP port |
| `gateway.httpsPort` | `443` | HTTPS port |
| `gateway.hostNetwork` | `false` | Host networking |
| `gateway.service.type` | `LoadBalancer` | Service type (`None` disables) |
| `gateway.service.loadBalancerIP` | `""` | Static LB IP |
| `gateway.service.loadBalancerClass` | `""` | LB class |
| `gateway.service.loadBalancerSourceRanges` | `[]` | LB source ranges |
| `gateway.service.externalTrafficPolicy` | `""` | External traffic policy |
| `gateway.autoscaling.enabled` | `false` | HPA |
| `gateway.autoscaling.minReplicas` | `1` | HPA min |
| `gateway.autoscaling.maxReplicas` | `5` | HPA max |
| `gateway.resources` | `{cpu: 2, mem: 2Gi}` | Container resources |
| `gateway.metrics.enabled` | `false` | PodMonitor/VMPodScrape |
| `gateway.metrics.provider` | `monitoring.coreos.com` | CRD provider |
| `gateway.metrics.podMonitorSelector` | `{release: kube-prome}` | PodMonitor selector |
| `gateway.rollingMaxSurge` | `100%` | Rolling update max surge |
| `gateway.rollingMaxUnavailable` | `25%` | Rolling update max unavailable |
| `gateway.nodeSelector` | `{}` | Node selector |
| `gateway.tolerations` | `[]` | Tolerations |
| `gateway.affinity` | `{}` | Affinity |
| `gateway.topologySpreadConstraints` | `[]` | Topology spread |
| `gateway.automaticHttps.enabled` | `true` | Let's Encrypt auto HTTPS |
| `gateway.automaticHttps.email` | `""` | Let's Encrypt email |

## Controller

| Parameter | Default | Description |
|-----------|---------|-------------|
| `controller.name` | `higress-controller` | Controller deployment name |
| `controller.replicas` | `1` | Pod count |
| `controller.image` | `higress` | Image name (hub/higress) |
| `controller.tag` | `""` | Image tag |
| `controller.service.type` | `ClusterIP` | Service type |
| `controller.resources` | `{cpu: 500m/1, mem: 2Gi}` | Resource requests/limits |
| `controller.autoscaling.enabled` | `false` | HPA |
| `controller.autoscaling.minReplicas` | `1` | HPA min |
| `controller.autoscaling.maxReplicas` | `5` | HPA max |
| `controller.automaticHttps.enabled` | `true` | Auto HTTPS |
| `controller.automaticHttps.email` | `""` | Let's Encrypt email |
| `controller.nodeSelector` | `{}` | Node selector |
| `controller.tolerations` | `[]` | Tolerations |
| `controller.affinity` | `{}` | Affinity |

## Pilot (Istiod)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `pilot.image` | `pilot` | Image name (hub/pilot) |
| `pilot.tag` | `""` | Image tag |
| `pilot.traceSampling` | `1.0` | Trace sampling rate |
| `pilot.resources` | `{cpu: 500m, mem: 2Gi}` | Resources |
| `pilot.env.PILOT_ENABLE_METADATA_EXCHANGE` | `false` | Disable metadata exchange |
| `pilot.keepaliveMaxServerConnectionAge` | `30m` | xDS max connection age |

## Redis (Optional)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `redis.redis.name` | `redis-stack-server` | Redis deployment |
| `redis.redis.image` | `redis-stack-server` | Image name |
| `redis.redis.tag` | `7.4.0-v3` | Image tag |
| `redis.redis.replicas` | `1` | Replicas |
| `redis.redis.password` | `""` | Password (empty = none) |
| `redis.redis.service.port` | `6379` | Service port |
| `redis.redis.persistence.enabled` | `false` | Enable PVC |
| `redis.redis.persistence.size` | `1Gi` | PVC size |

## Plugin Server (Optional)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `pluginServer.name` | `higress-plugin-server` | Plugin server name |
| `pluginServer.replicas` | `2` | Pod count |
| `pluginServer.image` | `plugin-server` | Image name |
| `pluginServer.tag` | `""` | Image tag |
| `pluginServer.service.port` | `80` | Service port |
| `pluginServer.resources` | `{cpu: 200m/500m, mem: 128Mi/256Mi}` | Resources |

## Console (UI)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.repository` | `higress/console` | Console image |
| `image.tag` | `""` | Image tag |
| `replicaCount` | `1` | Replicas |
| `service.port` | `8080` | Service port |
| `ingress.enabled` | `false` | Expose via Ingress |
| `ingress.domain` | `console.higress.io` | Console domain |
| `ingress.tlsSecretName` | `""` | TLS secret |
| `admin.username` | `admin` | Admin user |
| `admin.password` | `""` | Admin password |
| `chat.enabled` | `false` | AI chat in console |
| `chat.endpoint` | `""` | Chat API endpoint |

## O11y (Observability)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `global.o11y.enabled` | `false` | Enable all o11y |
| `global.o11y.grafana.replicas` | `1` | Grafana replicas |
| `global.o11y.grafana.storage` | `1Gi` | Grafana PVC |
| `global.o11y.prometheus.replicas` | `1` | Prometheus replicas |
| `global.o11y.prometheus.storage` | `1Gi` | Prometheus PVC |
| `global.o11y.loki.replicas` | `1` | Loki replicas |
| `global.o11y.loki.storage` | `1Gi` | Loki PVC |

## Production Values Example

```yaml
global:
  ingressClass: higress
  enableGatewayAPI: true
  enableIstioAPI: true
  enableRedis: true
  enablePluginServer: true
  o11y:
    enabled: true
  priorityClassName: system-cluster-critical

gateway:
  replicas: 3
  kind: Deployment
  service:
    type: LoadBalancer
    externalTrafficPolicy: Local
  resources:
    requests:
      cpu: 2
      memory: 2Gi
    limits:
      cpu: 4
      memory: 4Gi
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 80
  metrics:
    enabled: true
    provider: monitoring.coreos.com
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - higress-gateway
          topologyKey: kubernetes.io/hostname

controller:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2
      memory: 4Gi
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5

redis:
  redis:
    persistence:
      enabled: true
      size: 10Gi
      storageClass: ssd
```

## IngressClass Configuration

```yaml
global:
  ingressClass: higress  # default

  # Special: watch nginx Ingress resources (migration mode)
  # ingressClass: nginx
  # - watches both "nginx" class AND no-class Ingress resources
  # - enables smooth migration from ingress-nginx

  # Special: watch all Ingress resources
  # ingressClass: ""
```

## Gateway API Integration

```bash
helm install higress higress.io/higress -n higress-system \
  --set global.enableGatewayAPI=true
```

Requires Gateway API CRDs installed:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

## AI Gateway + Redis

```bash
helm install higress higress.io/higress -n higress-system \
  --set global.enableRedis=true \
  --set global.enablePluginServer=true
```

## Upgrading

```bash
helm repo update
helm upgrade higress higress.io/higress -n higress-system \
  --values values.yaml \
  --version 2.2.2
```

## Uninstalling

```bash
helm uninstall higress -n higress-system
kubectl delete namespace higress-system
```

**Note:** CRDs persist after uninstall. Remove manually:
```bash
kubectl delete crd wasmplugins.extensions.higress.io
kubectl delete crd http2rpcs.networking.higress.io
kubectl delete crd mcpbridges.networking.higress.io
```

## Common Mistakes

- **Hub region** — Default hub is in China (`cn-hangzhou`). Use `us-west-1.cr.aliyuncs.com` for NA, `ap-southeast-7.cr.aliyuncs.com` for SEA. Set `global.hub` before install.
- **Gateway API CRDs missing** — Setting `global.enableGatewayAPI=true` without installing Gateway API CRDs causes controller errors.
- **IngressClass == nginx** — Setting `ingressClass: nginx` makes Higress watch nginx-class Ingresses. Remove or change to `higress` after migration.
- **Redis for AI features** — AI caching, token rate limiting, and quota require `global.enableRedis=true`. Without Redis, these plugins fail.
- **Plugin Server for OCI plugins** — If using `url: oci://...` in WasmPlugin, ensure `global.enablePluginServer=true` or have direct OCI registry access.
- **Standalone console ingress** — `higress-console` needs `ingress.enabled=true` and a domain for external access. Default is ClusterIP only.
- **Gateway metrics** — `gateway.metrics.enabled=true` creates PodMonitor. Requires prometheus-operator or VictoriaMetrics operator CRDs.
- **Controller readiness** — Controller uses `/ready` on port 8888. Ensure network policies allow this.
- **HostNetwork gateway** — `gateway.hostNetwork=true` binds host ports 80/443. Requires host port availability and potential security implications.
- **autoscaling/v2 API** — `global.autoscalingv2API` (default: true) uses `autoscaling/v2`. If your cluster is older, set to false.
- **alpn h2** — `global.disableAlpnH2: false` (default) enables HTTP/2 ALPN. Set true if clients don't support HTTP/2.
