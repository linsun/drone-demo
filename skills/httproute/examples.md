# HTTPRoute Examples

## Same-namespace Gateway, path prefix

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: svc-v1
          kind: Service
          port: 8080
```

## Specific listener port (e.g. MCP)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route-3000
  namespace: mcp
spec:
  parentRefs:
    - name: agentgateway
      namespace: mcp
      port: 3000
  rules:
    - backendRefs:
        - name: mcp-backends
          group: agentgateway.dev
          kind: AgentgatewayBackend
```

## Hostname + path

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: default
spec:
  parentRefs:
    - name: shared-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api-v1
          port: 8080
```

## Multiple backends (single rule)

When a rule has multiple backendRefs, traffic is distributed (implementation-dependent; often round-robin):

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: frontend-a
        port: 80
      - name: frontend-b
        port: 80
```

## Header matching (optional)

Some implementations support header matches in `matches`:

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
      - headers:
          - name: X-Canary
            value: true
    backendRefs:
      - name: canary-service
        port: 80
```

Use only if your Gateway implementation documents support for header matching.
