---
name: httproute
description: Create valid Kubernetes Gateway API HTTPRoute resources. Use when creating or editing HTTPRoute manifests, routing HTTP traffic to services, or when the user asks for HTTPRoute, Gateway API routes, or path-based routing.
---

# HTTPRoute Skill

Create useful and valid HTTPRoute resources using the Kubernetes Gateway API (`gateway.networking.k8s.io/v1`).

## When to Use

- User asks for an HTTPRoute, route manifest, or path-based routing
- User wants to expose a Service through a Gateway
- User needs to add hostname or path matching to existing routing

## Required Structure

Every HTTPRoute must have:

| Field | Purpose |
|-------|---------|
| `apiVersion: gateway.networking.k8s.io/v1` | Gateway API v1 |
| `kind: HTTPRoute` | Resource type |
| `metadata.name`, `metadata.namespace` | Identity |
| `spec.parentRefs` | At least one ref to a Gateway (name, and usually namespace) |
| `spec.rules` | At least one rule with `backendRefs` |

Each rule must have at least one `backendRef` (name, optional port, optional group/kind for custom backends).

## ParentRefs

Reference the Gateway this route attaches to:

```yaml
spec:
  parentRefs:
    - name: <gateway-name>           # required
      namespace: <gateway-namespace>  # required if Gateway is in another namespace
      sectionName: <listener-name>   # optional: target specific listener
      port: 80                       # optional: target specific port
```

- If the Gateway is in the same namespace as the HTTPRoute, `namespace` can be omitted.
- Use `port` when the Gateway has multiple listeners (e.g. 80, 3000, 3001).

**Verify the Gateway exists:** Before applying an HTTPRoute, confirm that the Gateway referenced in `parentRefs` exists in the specified namespace. Resolve namespace as: the `namespace` in the parentRef if set, otherwise the HTTPRoute's `metadata.namespace`. Use `kubectl get gateway <name> -n <namespace>` (or equivalent) to check. If the Gateway is missing or in a different namespace, the route will not attach and may report resolution errors.

## Path Matching

Add `matches` to a rule to restrict by path:

| Type | Value example | Matches |
|------|----------------|---------|
| `PathPrefix` | `/api` | `/api`, `/api/v1`, `/api/foo` |
| `Exact` | `/health` | Only `/health` |

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: /
    backendRefs:
      - name: frontend
        port: 80
```

Omit `matches` to match all requests on that listener (catch-all).

## Hostnames

Optional. Restrict by HTTP Host header:

```yaml
spec:
  hostnames:
    - "app.example.com"
    - "*.example.com"
  rules: ...
```

## BackendRefs

- **Kubernetes Service** (default): `name`, `port` (required if Service has multiple ports), `kind: Service` (optional, default).
- **Custom backend** (e.g. AgentgatewayBackend): set `group` and `kind` and optionally `port`.

```yaml
# Standard Service backend
backendRefs:
  - name: frontend
    kind: Service
    port: 80

# Custom backend (e.g. agentgateway)
backendRefs:
  - name: mcp-backends
    group: agentgateway.dev
    kind: AgentgatewayBackend
```

## Validation Checklist

Before applying, ensure:

- [ ] `apiVersion` is `gateway.networking.k8s.io/v1` and `kind` is `HTTPRoute`
- [ ] `spec.parentRefs` has at least one entry with `name` (and `namespace` if cross-namespace)
- [ ] **Parent Gateway exists:** For each `parentRefs` entry, a Gateway with that `name` exists in the specified namespace (use `parentRef.namespace` if set, else the HTTPRoute's `metadata.namespace`). Verify with `kubectl get gateway <name> -n <namespace>`.
- [ ] Every entry in `spec.rules` has at least one `backendRefs` with `name`
- [ ] Backend `port` matches a port exposed by the Service or custom backend
- [ ] Path `value` starts with `/` when using path matching
- [ ] Gateway allows routes from this HTTPRoute's namespace (per Gateway `allowedRoutes` / listener config)

## Quick Templates

**Catch-all to a Service (same namespace as Gateway):**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: default
spec:
  parentRefs:
    - name: agentgateway-proxy
      namespace: agentgateway-system
  rules:
    - backendRefs:
        - name: my-service
          port: 80
```

**Path prefix to a Service:**
```yaml
spec:
  parentRefs: [...]
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: api-service
          port: 8080
```

**Multiple rules (path order matters; more specific first):**
```yaml
spec:
  parentRefs: [...]
  rules:
    - matches:
        - path:
            type: Exact
            value: /health
      backendRefs:
        - name: health-service
          port: 8080
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

## Additional Resources

- For more examples (hostnames, filters, cross-namespace), see [examples.md](examples.md).
