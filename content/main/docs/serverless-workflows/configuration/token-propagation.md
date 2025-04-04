---
title: "Configure workflow for token propagation"
date: 2025-04-04
---
The RHDH Orchestrator plugin is, by default, adding all tokens in the `authTokens` field of the POST request's body in the workflow execuition's request with the following format: `X-Authorization-{provider}: {token}`.
This allows the user identity to be propagated to the third parties/externals services called by the workflow.
To do so, a set of properties must be set in the workflow `application.properties` file.

# Prerequisites
* Having a Keycloak instance running with a client
* Having RHDH with the latest version of the Orchestrator plugins
* Having a workflow using openapi spec file to send REST requests to a service

# Build

When building the workflow's image, you will need to make sure the following extensions is present in the `QUARKUS_EXTENSION`:
* io.quarkus:quarkus-oidc-client-filter

# Configuration
## Oauth2
1. In the openapi spec file(s) for which you want to propagate the incoming token to, write down the security scheme configured for the endpoints of interests for you. All endpoints may use the same security scheme if configured globally.
e.g
```
components:
  securitySchemes:
    BearerToken:
     type: oauth2
     flows:
       clientCredentials:
         tokenUrl: http://<keycloak>/realms/<yourRealm>/protocol/openid-connect/token
         scopes: {}
     description: Bearer Token authentication
```
2. In the `application.properties` of your workflow, for each security scheme, adds the following:
```
quarkus.oidc-client.BearerToken.auth-server-url=https://<keycloak>/realms/<yourRealm>
quarkus.oidc-client.BearerToken.token-path=https://<keycloak>/realms/<yourRealm>/protocol/openid-connect/token
quarkus.oidc-client.BearerToken.discovery-enabled=false
quarkus.oidc-client.BearerToken.client-id=<client ID>
quarkus.oidc-client.BearerToken.grant.type=client
quarkus.oidc-client.BearerToken.credentials.client-secret.method=basic
quarkus.oidc-client.BearerToken.credentials.client-secret.value=<client secret>

quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.token-propagation=true
quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.header-name=X-Authorization-<provider>
```
With:
* `spec_file_yaml_or_json`: the name of the spec file configured with `_` as separator. E.g: if the file name is `dumb-server.yaml` the normalized property name will be `dumb_server_yaml`. This should be the same for every security scheme defined in the file.
* `security_scheme`: the name of the security scheme for which propagate the token located in the header defined by the `header-name` property. In our example it would be `BearerToken`.
* `provider`: the name of the expected provider from which the token comes from. As explained above, for each provider in RHDH, the Orchestrator plugin is adding a header with the format `X-Authorization-{provider}: {token}`.
* `keycloak`: the URL of the running Keycloak instance.
* `yourRealm`: the name of the realm to use.
* `client ID`: the ID of the Keycloak client to use to authenticate against the Keycloak instance.

## Bearer token
1. In the openapi spec file(s) for which you want to propagate the incoming token to, write down the security scheme configured for the endpoints of interests for you. All endpoints may use the same security scheme if configured globally.
e.g
```
components:
  securitySchemes:
    SimpleBearerToken:
     type: http
     scheme: bearer
```
2. In the `application.properties` of your workflow, for each security scheme, adds the following:
```
quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.token-propagation=true
quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.header-name=X-Authorization-<provider>
```
With:
* `spec_file_yaml_or_json`: the name of the spec file configured with `_` as separator. E.g: if the file name is `dumb-server.yaml` the normalized property name will be `dumb_server_yaml`. This should be the same for every security scheme defined in the file.
* `security_scheme`: the name of the security scheme for which propagate the token located in the header defined by the `header-name` property. In our example it would be `SimpleBearerToken`.
* `provider`: the name of the expected provider from which the token comes from. As explained above, for each provider in RHDH, the Orchestrator plugin is adding a header with the format `X-Authorization-{provider}: {token}`.
## Basic auth
Basic auth token propagation is not currently supported.
An PR is opened to add its support: https://github.com/quarkiverse/quarkus-openapi-generator/pull/1078


You can see a full example with different security scheme types here: https://github.com/rhdhorchestrator/workflow-token-propagation-example.