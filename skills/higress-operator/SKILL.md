---
name: higress-operator
description: Create and manage Higress CRDs — WasmPlugin, Http2Rpc, McpBridge, AI Gateway, 41 Wasm plugins, 16 AI provider integrations, and Gateway configuration.
---

# Higress — Operator CRDs & Configuration

**Repository:** `github.com/alibaba/higress`  
**Latest:** v2.2.2 (May 21, 2026)  
**CNCF:** Sandbox (March 2026)  
**License:** Apache 2.0  
**Stars:** 8.5k

## Architecture

```
Ingress/Gateway API/CRDs → higress-controller (Go) → Istio Pilot → xDS (LDS/RDS/CDS/EDS) → higress-gateway (Envoy + Wasm)
```

- **Control plane**: higress-controller (Go) watches K8s resources, converts to Istio config, pushes via xDS
- **Data plane**: higress-gateway (Envoy-based) handles all traffic, runs Wasm plugins, zero-downtime config reload
- **Three ingress interfaces**: Standard K8s Ingress, Gateway API, Istio API (all simultaneous)

## CRDs (3 custom + 1 bundled)

| CRD | API Version | Kind | Short Names | Purpose |
|-----|-------------|------|-------------|---------|
| WasmPlugin | `extensions.higress.io/v1alpha1` | `WasmPlugin` | — | Wasm plugin lifecycle & config |
| Http2Rpc | `networking.higress.io/v1` | `Http2Rpc` | — | HTTP-to-RPC (Dubbo/gRPC) mapping |
| McpBridge | `networking.higress.io/v1` | `McpBridge` | — | Multi-registry service discovery |
| EnvoyFilter | `networking.istio.io/v1alpha3` | `EnvoyFilter` | — | Envoy filter chain patching (bundled) |

### WasmPlugin — `extensions.higress.io/v1alpha1`

Extends Envoy with Wasm-based plugins for auth, AI proxy, rate limiting, transformation, etc.

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: ai-proxy
  namespace: higress-system
spec:
  # Plugin source
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/ai-proxy:1.0.0
  sha256: abc123...
  pluginName: ai-proxy
  imagePullPolicy: IfNotPresent
  imagePullSecret: my-registry-cred

  # Plugin configuration (global)
  pluginConfig:
    provider:
      type: openai
      apiTokens:
        - "sk-..."
      modelMapping:
        "gpt-4": "gpt-4o"
      timeout: 120000

  # Default config (applied when no matchRules match)
  defaultConfig: {}
  defaultConfigDisable: false

  # Failure behavior
  failStrategy: FAIL_CLOSE  # FAIL_CLOSE | FAIL_OPEN

  # Position in filter chain
  phase: AUTHN  # UNSPECIFIED_PHASE | AUTHN | AUTHZ | STATS
  priority: 100

  # Wasm VM config
  vmConfig:
    env:
      - name: LOG_LEVEL
        value: debug

  # Per-route overrides
  matchRules:
    - domain:
        - "api.example.com"
      config:
        provider:
          type: openai
          apiTokens:
            - "sk-another-token"
      configDisable: false
    - ingress:
        - "my-ingress"
      routeType: HTTP
      configDisable: true
    - service:
        - "llm-service.default.svc"
      config:
        provider:
          type: anthropic
```

#### Spec Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `url` | string | — | Wasm module URL (`oci://...`, `http://...`) |
| `sha256` | string | — | SHA256 checksum |
| `pluginName` | string | — | Plugin name |
| `pluginConfig` | object | — | Plugin config (pass-through to plugin) |
| `defaultConfig` | object | `{}` | Default config for unmatched routes |
| `defaultConfigDisable` | bool | `false` | Disable plugin by default |
| `failStrategy` | enum | `FAIL_CLOSE` | `FAIL_CLOSE` or `FAIL_OPEN` |
| `phase` | enum | — | `UNSPECIFIED_PHASE`, `AUTHN`, `AUTHZ`, `STATS` |
| `priority` | int | — | Ordering within same phase |
| `imagePullPolicy` | enum | — | `UNSPECIFIED_POLICY`, `IfNotPresent`, `Always` |
| `imagePullSecret` | string | — | OCI pull secret name |
| `verificationKey` | string | — | Plugin signature verification key |
| `vmConfig.env` | []EnvVar | — | Wasm VM environment variables |
| `matchRules[]` | []MatchRule | — | Per-route config overrides |

**MatchRule fields:**

| Field | Type | Description |
|-------|------|-------------|
| `domain[]` | []string | Match by domain name |
| `ingress[]` | []string | Match by Ingress resource name |
| `service[]` | []string | Match by K8s service name |
| `routeType` | enum | `HTTP` or `GRPC` |
| `config` | object | Plugin config for this rule |
| `configDisable` | bool | Disable plugin for this rule |

### Http2Rpc — `networking.higress.io/v1`

Maps HTTP endpoints to Dubbo or gRPC services.

```yaml
apiVersion: networking.higress.io/v1
kind: Http2Rpc
metadata:
  name: dubbo-user-service
spec:
  # Dubbo destination (oneOf: dubbo XOR grpc)
  dubbo:
    service: com.example.UserService
    version: 1.0.0
    group: prod
    methods:
      - serviceMethod: getUserById
        httpPath: /api/users/:id
        httpMethods:
          - GET
        headersAttach: x-request-id
        params:
          - paramSource: URL_PATH
            paramKey: id
            paramType: java.lang.Long
      - serviceMethod: createUser
        httpPath: /api/users
        httpMethods:
          - POST
        paramFromEntireBody:
          paramType: com.example.User

  # OR gRPC destination (alternative to dubbo):
  # grpc:
  #   proto_descriptor_str: "...base64..."
  #   proto_descriptor_file_path: /etc/proto/user.pb
  #   services:
  #     - com.example.UserService
```

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `dubbo.service` | string | oneOf | Dubbo service interface |
| `dubbo.version` | string | ✅ | Dubbo service version |
| `dubbo.group` | string | ❌ | Dubbo service group |
| `dubbo.methods[].serviceMethod` | string | ✅ | Dubbo method name |
| `dubbo.methods[].httpPath` | string | ✅ | HTTP path to map |
| `dubbo.methods[].httpMethods[]` | []string | ✅ | Allowed HTTP methods |
| `dubbo.methods[].headersAttach` | string | ❌ | Headers to propagate |
| `dubbo.methods[].params[].paramSource` | string | ✅ | `URL_PATH`, `URL_QUERY`, `REQUEST_HEADER` |
| `dubbo.methods[].params[].paramKey` | string | ✅ | Parameter key name |
| `dubbo.methods[].params[].paramType` | string | ✅ | Java type (e.g. `java.lang.Long`) |
| `dubbo.methods[].paramFromEntireBody.paramType` | string | ❌ | Use entire body as param |
| `grpc.proto_descriptor_str` | string | oneOf | Inline protobuf descriptor |
| `grpc.proto_descriptor_file_path` | string | ❌ | Path to proto descriptor file |
| `grpc.services[]` | []string | ✅ | gRPC service names |

### McpBridge — `networking.higress.io/v1`

Multi-registry service discovery for integrating external service registries.

```yaml
apiVersion: networking.higress.io/v1
kind: McpBridge
metadata:
  name: default
  namespace: higress-system
spec:
  registries:
    # Nacos 2.x (gRPC)
    - type: nacos2
      name: my-nacos
      domain: nacos.example.com
      port: 8848
      protocol: HTTP
      nacosNamespaceId: public
      nacosGroups:
        - DEFAULT_GROUP
      nacosRefreshInterval: 5000

    # Nacos 3.x (MCP auto-discovery)
    - type: nacos3
      name: nacos-mcp
      domain: nacos3.example.com
      port: 8848
      enableMCPServer: true

    # ZooKeeper
    - type: zookeeper
      name: my-zk
      domain: zk.example.com
      port: 2181
      zkServicesPath:
        - /dubbo

    # Consul
    - type: consul
      name: my-consul
      domain: consul.example.com
      port: 8500
      consulDatacenter: dc1
      consulServiceTag: prod

    # Static/DNS
    - type: static
      name: static-backends
      domain: backend.example.com
      port: 8080

  proxies:
    - type: http_connect
      name: corporate-proxy
      serverAddress: proxy.example.com
      serverPort: 3128
      listenerPort: 80
      connectTimeout: 5000
```

#### Spec Fields

| Field | Type | Description |
|-------|------|-------------|
| `registries[]` | []Registry | Service registry configurations |

**Registry fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `nacos2`, `nacos3`, `nacos`, `zookeeper`, `consul`, `eureka`, `static`, `dns` |
| `name` | string | Registry name |
| `domain` | string | Registry host |
| `port` | int | Registry port |
| `protocol` | string | `HTTP` or `HTTPS` |
| `sni` | string | SNI for TLS |
| `nacosNamespaceId` | string | Nacos namespace ID |
| `nacosGroups` | []string | Nacos groups |
| `nacosAccessKey` | string | Nacos auth key |
| `nacosSecretKey` | string | Nacos auth secret |
| `nacosAddressServer` | string | Nacos address server |
| `nacosRefreshInterval` | int | Refresh interval (ms) |
| `zkServicesPath` | []string | ZooKeeper service paths |
| `consulDatacenter` | string | Consul datacenter |
| `consulNamespace` | string | Consul namespace |
| `consulServiceTag` | string | Consul service tag |
| `consulRefreshInterval` | int | Refresh interval (ms) |
| `authSecretName` | string | Secret name for registry auth |
| `enableMCPServer` | bool | Enable MCP server for Nacos 3.x |
| `allowMcpServers` | []string | Allowed MCP server addresses |
| `mcpServerBaseUrl` | string | MCP server base URL |
| `mcpServerExportDomains` | []string | Domains to export via MCP |
| `enableScopeMcpServers` | bool | Scope MCP servers |
| `vport.default` | int | Default virtual port |
| `vport.services` | []object | Per-service vport overrides |
| `metadata` | map | Extra metadata |
| `proxyName` | string | Proxy name |

**Proxy fields:**

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | `http_connect` |
| `name` | string | Proxy name |
| `serverAddress` | string | Proxy server address |
| `serverPort` | int | Proxy server port |
| `listenerPort` | int | Local listener port |
| `connectTimeout` | int | Connection timeout (ms) |

### EnvoyFilter — `networking.istio.io/v1alpha3`

Bundled Istio CRD for low-level Envoy filter chain patching.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: custom-filter
  namespace: higress-system
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
              subFilter:
                name: envoy.filters.http.router
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.lua
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inline_code: |
              function envoy_on_request(request_handle)
                -- custom logic
              end
```

## Wasm Plugin System

### Plugin Categories (41 built-in)

| Category | Plugins | Count |
|----------|---------|-------|
| **AI** | ai-proxy, ai-cache, ai-token-ratelimit, ai-quota, ai-security-guard, ai-statistics, ai-rag, ai-search, ai-agent, ai-transformer, ai-prompt-template, ai-prompt-decorator, ai-intent, ai-history, ai-json-resp, ai-data-masking, ai-load-balancer, ai-image-reader, model-mapper, model-router, mcp-router, mcp-server, chatgpt-proxy | 23 |
| **Auth** | basic-auth, key-auth, hmac-auth, hmac-auth-apisix, jwt-auth, simple-jwt-auth, oauth/oidc, ext-auth, opa | 9 |
| **Security** | waf, bot-detect, cors, ip-restriction, request-block, replay-protection | 6 |
| **Traffic** | cluster-key-rate-limit, key-rate-limit, request-validation, traffic-tag, traffic-editor, response-cache | 6 |
| **Transformation** | transformer, custom-response, cache-control, de-graphql, frontend-gray, nginx-rewrite-compatible | 6 |

### Plugin Languages
- **Go** (69 plugins) — primary, via TinyGo + wasm-go SDK
- **Rust** (5), **C++**, **AssemblyScript**

### Plugin Loading
- **OCI images**: `oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/<name>:<version>`
- **HTTP distribution**: via plugin-server
- **SDK**: [`github.com/higress-group/wasm-go`](https://github.com/higress-group/wasm-go)

### WasmPlugin AI Proxy Example

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: ai-proxy-openai
spec:
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/ai-proxy:1.0.0
  pluginConfig:
    provider:
      type: openai
      apiTokens:
        - "sk-..."
      modelMapping:
        "gpt-4": "gpt-4o"
      timeout: 120000
      protocol: openai  # or "original" for native provider protocol
```

### AI Gateway Provider Options

| Provider | `type` value | Auth Method |
|----------|-------------|-------------|
| OpenAI | `openai` | `apiTokens[]` |
| Azure OpenAI | `azure` | `apiTokens[]` |
| Anthropic Claude | `claude` | `apiTokens[]` |
| Google Gemini | `gemini` | `apiTokens[]` |
| AWS Bedrock | `aws-bedrock` | IAM (env) |
| DeepSeek | `deepseek` | `apiTokens[]` |
| Moonshot | `moonshot` | `apiTokens[]` |
| Qwen (Tongyi) | `qwen` | `apiTokens[]` |
| Alibaba Bailian | `bailian` | `apiTokens[]` |
| Doubao | `doubao` | `apiTokens[]` |
| Spark (Xunfei) | `spark` | `apiTokens[]` |
| Cloudflare Workers AI | `cloudflare` | `apiTokens[]` |
| Together AI | `together` | `apiTokens[]` |
| OpenRouter | `openrouter` | `apiTokens[]` |
| Mistral | `mistral` | `apiTokens[]` |
| NVIDIA Triton | `nvidia-triton` | — |

### Common WasmPlugin Patterns

**Rate Limiting:**
```yaml
spec:
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/key-rate-limit:1.0.0
  pluginConfig:
    limit: 100
    window_size: 60
    key_source: X-Forwarded-For
```

**JWT Auth:**
```yaml
spec:
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/jwt-auth:1.0.0
  pluginConfig:
    consumers:
      - name: myapp
        credential: myapp-secret
        iss: https://auth.example.com
```

**CORS:**
```yaml
spec:
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/cors:1.0.0
  pluginConfig:
    allow_origins: "https://app.example.com"
    allow_methods: "GET,POST,PUT,DELETE"
    allow_headers: "Content-Type,Authorization"
    expose_headers: "X-Custom-Header"
    max_age: 3600
```

## Common Mistakes

- **No short names defined** — WasmPlugin/Http2Rpc/McpBridge have no kubectl short names. Use full plural: `kubectl get wasmplugins`, `kubectl get http2rpcs`, `kubectl get mcpbridges`.
- **WasmPlugin url without OCI** — Only `oci://` and `http://` schemes supported. Don't use `docker://`.
- **Image pull secret** — Private OCI registries need `imagePullSecret` referencing a secret in the same namespace.
- **Phase ordering** — Plugins run in order: AUTHN → AUTHZ → STATS. Within same phase, lower priority runs first.
- **matchRules overlap** — If multiple rules match, first match wins. Order `domain` → `ingress` → `service` by priority.
- **Dubbo version required** — `dubbo.version` is required even if not used by the target service.
- **McpBridge registry name** — Each registry in `registries[]` must have a unique `name`.
- **Nacos namespace** — Use `nacosNamespaceId` (not `nacosNamespace` which is legacy) for Nacos multi-tenant.
- **EnvoyFilter complexity** — EnvoyFilter patches are powerful but break on Envoy version upgrades. Prefer WasmPlugin when possible.
- **Gateway API mutual exclusion** — When both Ingress and Gateway API define the same route, Gateway API takes precedence.
- **Fail strategy** — `FAIL_CLOSE` blocks traffic on plugin error. `FAIL_OPEN` passes through. Use `FAIL_CLOSE` for auth, `FAIL_OPEN` for observability.
