---
title: "Configure workflow for token exchange"
date: 2025-09-17
---
Token Exchange lets a workflow swap the incoming end‑user token for a new access token tailored to a downstream OpenAPI‑secured service. Use it when you must not forward the original token or when workflows run long enough that the original token may expire.

See the upstream reference for full details: Token Exchange for OpenAPI services in SonataFlow (`https://sonataflow.org/serverlessworkflow/latest/security/token-exchange-for-openapi-services.html`).

# Prerequisites
* Keycloak or another OIDC provider that supports OAuth 2.0 Token Exchange
* A workflow calling an OpenAPI client generated from an OpenAPI specification file using an `oauth2` security scheme

# Build

When building the workflow image, ensure the following extensions are present (e.g., via `QUARKUS_EXTENSION` for the internal builder):
* io.quarkus:quarkus-oidc-client-filter
* org.kie:kie-addons-quarkus-token-exchange

Optional for persistence of exchanged tokens:
* org.kie:kogito-quarkus-serverless-workflow-jdbc-token-persistence


See https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/scripts/build.sh#L180 to see how we do it.

# Configuration

## 1) Define an OAuth2 security scheme in your OpenAPI specification file

The OpenAPI operation(s) you call must be secured by an `oauth2` scheme. The OIDC client name is derived from this scheme name (sanitized by replacing non‑alphanumerics with `_`).

Example:
```
openapi: 3.0.3
paths:
  /secured:
    get:
      operationId: callService
      responses:
        "200": { description: OK }
      security:
        - service-oauth: []
components:
  securitySchemes:
    service-oauth:
      type: oauth2
      flows:
        clientCredentials:
          authorizationUrl: https://<idp>/realms/<realm>/protocol/openid-connect/auth
          tokenUrl: https://<idp>/realms/<realm>/protocol/openid-connect/token
          scopes: {}
```

## 2) Configure `application.properties`

Replace placeholders with your values.

```
# Base URL for the generated client (service id is the sanitized OpenAPI file id)
quarkus.rest-client.<service_id>.url=http://<downstream-service>

# Enable Token Exchange for this auth name (sanitized from OpenAPI scheme name)
sonataflow.security.auth.<auth_name>.token-exchange.enabled=true

# Proactive refresh and monitor (optional, default values are shown below)
sonataflow.security.auth.<auth_name>.token-exchange.proactive-refresh-seconds=300
sonataflow.security.auth.token-exchange.monitor-rate-seconds=60

# Ensure the incoming user token is available to the workflow service
# If the end-user token is sent in a custom header (e.g., from RHDH Orchestrator plugin),
# point Quarkus to that header so the SecurityIdentity is established.
auth-server-url=https://<keycloak>/realms/<yourRealm>
client-id=<client ID>
client-secret=<client secret>

quarkus.oidc.auth-server-url=${auth-server-url}
quarkus.oidc.client-id=${client-id}
quarkus.oidc.credentials.secret=${client-secret}
quarkus.oidc.token.header=X-Authorization-<provider>
quarkus.oidc.token.issuer=any

# OIDC client for Token Exchange corresponding to the auth scheme (name sanitized)
quarkus.oidc-client.<auth_name>.discovery-enabled=false
quarkus.oidc-client.<auth_name>.auth-server-url=${auth-server-url}/protocol/openid-connect/auth
quarkus.oidc-client.<auth_name>.token-path=${auth-server-url}/protocol/openid-connect/token
quarkus.oidc-client.<auth_name>.client-id=${client-id}
quarkus.oidc-client.<auth_name>.grant.type=exchange
quarkus.oidc-client.<auth_name>.credentials.client-secret.method=basic
quarkus.oidc-client.<auth_name>.credentials.client-secret.value=${client-secret}

# Persist inbound headers so tokens survive wait/resume and restarts (recommended)
kogito.persistence.headers.enabled=true
```

With:
* `service_id`: sanitized OpenAPI file id (e.g., `simple-server.yaml` -> `simple_server_yaml`).
* `auth_name`: sanitized OpenAPI oauth2 security scheme (e.g., `service-oauth` -> `service_oauth`).
* `provider`: the RHDH provider name; the Orchestrator plugin sends user tokens as `X-Authorization-{provider}: {token}`.

### Configuration reference
* SonataFlow Token Exchange guide: [Token Exchange for OpenAPI services](https://sonataflow.org/serverlessworkflow/latest/security/token-exchange-for-openapi-services.html)
* SonataFlow configuration properties (headers persistence): [Core configuration properties](https://sonataflow.org/serverlessworkflow/main/core/configuration-properties.html)
* Quarkus OIDC Client: [OpenID Connect (OIDC) client](https://quarkus.io/guides/security-openid-connect-client)
* Quarkus OIDC Client Filter (REST Client): [REST Client OIDC client filter](https://quarkus.io/guides/security-openid-connect-client#rest-client-oidc-client-filter)
* Quarkiverse OpenAPI Generator client configuration: [Client configuration](https://docs.quarkiverse.io/quarkus-openapi-generator/dev/client.html)
* Token propagation with REST Client (contrast with exchange): [Token propagation](https://quarkus.io/guides/security-openid-connect-client-reference#token-propagation-rest)

## 3) Interplay with token propagation

Do not enable token propagation for the same `<auth_name>` if you need token exchange. If both are enabled, propagation takes precedence and no exchange is performed.

If you do need propagation in another specification file having a similar scheme name, configure per scheme name on the specification level:
```
quarkus.openapi-generator.<another_service_id>.auth.<auth_name>.token-propagation=true
quarkus.openapi-generator.<another_service_id>.auth.<auth_name>.header-name=X-Authorization-<provider>
```

# Caching and persistence

When enabled, exchanged tokens are cached per process instance and auth name, with proactive refresh before expiry. By default, an in‑memory cache is used. To persist cache entries, add the JDBC persistence extension in the `QUARKUS_EXTENSIONS` when building the image: see https://sonataflow-docs-preview-pr-737.surge.sh/serverlessworkflow/latest/cloud/operator/build-and-deploy-workflows.html#passing-build-arguments-to-internal-workflow-builder

For local debug/dev, you can add it in your local `pom.xml` file:

```
<dependency>
  <groupId>org.kie</groupId>
  <artifactId>kogito-quarkus-serverless-workflow-jdbc-token-persistence</artifactId>
  <!-- Configure a JDBC DataSource as usual -->
  <!-- The extension provides a CDI TokenCacheRepository implementation -->
</dependency>
```

# Examples

## Invoke workflow with an Authorization header
```
curl -X POST \
  http://localhost:8080/<workflow_id> \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"value"}'
```

## Invoke workflow with RHDH custom header
If your client sends the token via the Orchestrator plugin header, and you set `quarkus.oidc.token.header=X-Authorization-<provider>`:
```
curl -X POST \
  http://localhost:8080/<workflow_id> \
  -H "X-Authorization-<provider>: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input":"value"}'
```

# Notes
* Security scheme names are global in an OpenAPI specification file; all operations using the same scheme share the same OIDC client and `token‑exchange` configuration.
* Prefer enabling `kogito.persistence.headers.enabled=true` for long‑running workflows so the incoming token is available after wait/resume or restarts.
* For a general comparison and configuration of forwarding the original token, see the token propagation guide in this folder.


# Configuring OIDC properties at SonataFlowPlatform level (Cluster‑wide OIDC configuration)
To avoid duplication, follow the platform‑level OIDC setup here: [Configuring OIDC properties at SonataFlowPlatform level](../token-propagation/#platform-oidc). This centralizes incoming request authentication and `$WORKFLOW.identity`. For Token Exchange, you still need to enable it per auth scheme using `sonataflow.security.auth.<auth_name>.token-exchange.enabled=true`.
