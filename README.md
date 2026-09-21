# LibreChat Deployment

GitOps deployment for a restricted LibreChat instance used by the Data Space
Dashboard assistant.

The deployment contains:

- LibreChat, pinned to `v0.8.8-rc3` by default
- MongoDB with persistent storage
- a restricted `librechat.yaml` configuration
- optional vLLM OpenAI-compatible endpoint configuration
- optional remote MCP server configuration
- a ManagementAPI application catalog entry and route definition

LibreChat's official Kubernetes guidance recommends stable credentials in a
Kubernetes Secret and supports deployment through its official Helm chart. This
repository keeps the same tenant application shape as the other Data-Space-Lab
deployments so ManagementAPI can render it per tenant.

## Required configuration

Before production deployment, replace the placeholder values in `values.yaml`
or provide a private values override:

- `secrets.credsKey`
- `secrets.credsIv`
- `secrets.jwtSecret`
- `secrets.jwtRefreshSecret`
- `secrets.openidClientSecret`
- `secrets.openidSessionSecret`

The credential values must remain stable across upgrades. Do not commit real
secrets to this repository.

Configure the Keycloak client with this redirect URI:

```text
https://librechat.{tenant_host}/oauth/openid/callback
```

Set the tenant-specific OIDC values:

```yaml
domain:
  host: librechat.example.org
  url: https://librechat.example.org

oidc:
  enabled: true
  issuer: https://dil.collab-cloud.eu/auth/realms/example
  clientId: librechat
  audience: librechat
  scope: openid profile email
```

`oidc.enabled` documents the intended deployment state; LibreChat activation is
controlled by the corresponding `OPENID_*` values and client secret.

## vLLM

Point the custom endpoint at the internal vLLM OpenAI-compatible API:

```yaml
vllm:
  baseUrl: http://vllm.default.svc.cluster.local:8000/v1
  model: data-space-assistant
```

The UI is restricted to the configured Data Space assistant model. Model
selection, presets, web search, file search, code execution, and MCP picker
visibility are disabled in `librechat.yaml`.

## Data Space MCP server

Enable MCP only after the MCP service and its authentication are available:

```yaml
mcp:
  enabled: true
  url: https://mcp.example.org/mcp
  useOpenIdToken: true
```

For a static MCP API key, put it in a private values override as
`secrets.mcpApiKey`. The MCP server remains server-side; the browser never sees
the key.

## Manual deployment

Render the Helm chart:

```bash
helm template librechat . \
  --namespace librechat \
  --set domain.host=librechat.example.org \
  --set domain.url=https://librechat.example.org
```

Install it with a private values file:

```bash
helm upgrade --install librechat . \
  --namespace librechat \
  --create-namespace \
  -f values-private.yaml
```

## ArgoCD

Apply `argocd-application.yaml` after pushing this repository to
`Data-Space-Lab/LibreChat-deployment`. ManagementAPI can use
`application-catalog-entry.json` to create a tenant-scoped Argo application and
Gateway route.
