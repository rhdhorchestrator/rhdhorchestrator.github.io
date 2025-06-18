---
title: "Configure workflow for token propagation"
date: 2025-04-04
---
By default, the RHDH Orchestrator plugin adds *headers* for each token in the 'authTokens' field of the POST request that is used to trigger a workflow execution. Those *headers* will be in the following format: `X-Authorization-{provider}: {token}`.
This allows the user identity to be propagated to the third parties and externals services called by the workflow.
To do so, a set of properties must be set in the workflow `application.properties` file.

# Prerequisites
* Having a Keycloak instance running with a client
* Having RHDH with the latest version of the Orchestrator plugins
* Having a workflow using openapi spec file to send REST requests to a service. Using custom REST function within the workflow will not propagate the token; it is only possible to propagate tokens when using openapi specification file.

# Build

When building the workflow's image, you will need to make sure the following extensions are present in the `QUARKUS_EXTENSION`:
* io.quarkus:quarkus-oidc-client-filter # needed for propagation
* io.quarkus:quarkus-oidc # neded for token validity check thus accessing $WORKFLOW.identity

See https://github.com/rhdhorchestrator/orchestrator-demo/blob/main/scripts/build.sh#L180 to see how we do it.

# Configuration
## Oauth2
1. In the OpenAPI spec file(s) where you want to propagate the incoming token, define the security scheme used by the endpoints you’re interested in. All endpoints may use the same security scheme if configured globally.
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
2. In the `application.properties` of your workflow, for each security scheme, add the following:
```
auth-server-url=https://<keycloak>/realms/<yourRealm>
client-id=<client ID>
client-secret=<client secret>

# Properties to check for identity, needed to use $WORKFLOW.identity within the workflow
quarkus.oidc.auth-server-url=${auth-server-url}
quarkus.oidc.client-id=${client-id}
quarkus.oidc.credentials.secret=${client-secret}
quarkus.oidc.token.header=X-Authorization-<provider>
quarkus.oidc.token.issuer=any # needed in case the auth server url is not the same as the one configured; e.g: localhost VS the k8S service

# Properties for propagation
quarkus.oidc-client.BearerToken.auth-server-url=${auth-server-url}
quarkus.oidc-client.BearerToken.token-path=${auth-server-url}/protocol/openid-connect/token
quarkus.oidc-client.BearerToken.discovery-enabled=false
quarkus.oidc-client.BearerToken.client-id=${client-id}
quarkus.oidc-client.BearerToken.grant.type=client
quarkus.oidc-client.BearerToken.credentials.client-secret.method=basic
quarkus.oidc-client.BearerToken.credentials.client-secret.value=${client-secret}

quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.token-propagation=true
quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.header-name=X-Authorization-<provider>
```
With:
* `spec_file_yaml_or_json`: the name of the spec file configured with `_` as separator. E.g: if the file name is `simple-server.yaml` the normalized property name will be `simple_server_yaml`. This should be the same for every security scheme defined in the file.
* `security_scheme`: the name of the security scheme for which propagates the token located in the header defined by the `header-name` property. In our example it would be `BearerToken`.
* `provider`: the name of the expected provider from which the token comes from. As explained above, for each provider in RHDH, the Orchestrator plugin is adding a header with the format `X-Authorization-{provider}: {token}`.
* `keycloak`: the URL of the running Keycloak instance.
* `yourRealm`: the name of the realm to use.
* `client ID`: the ID of the Keycloak client to use to authenticate against the Keycloak instance.

See https://sonataflow.org/serverlessworkflow/latest/security/authention-support-for-openapi-services.html#ref-authorization-token-propagation and https://quarkus.io/guides/security-openid-connect-client-reference#token-propagation-rest for more information about token propagation.

Setting the `quarkus.oidc.*` properties will enforce the token validity check against the OIDC provider. Once successful, you will be able to use `$WORKFLOW.identity` in the workflow definition in order to get the identity of the user. See https://quarkus.io/guides/security-oidc-bearer-token-authentication and https://quarkus.io/guides/security-oidc-bearer-token-authentication-tutorial for more information.

## Bearer token
1. In the OpenAPI spec file(s) where you want to propagate the incoming token, define the security scheme used by the endpoints you’re interested in. All endpoints may use the same security scheme if configured globally.
e.g
```
components:
  securitySchemes:
    SimpleBearerToken:
     type: http
     scheme: bearer
```
2. In the `application.properties` of your workflow, for each security scheme, add the following:
```
auth-server-url=https://<keycloak>/realms/<yourRealm>
client-id=<client ID>
client-secret=<client secret>

# Properties to check for identity, needed to use $WORKFLOW.identity within the workflow
quarkus.oidc.auth-server-url=${auth-server-url}
quarkus.oidc.client-id=${client-id}
quarkus.oidc.credentials.secret=${client-secret}
quarkus.oidc.token.header=X-Authorization-<provider>
quarkus.oidc.token.issuer=any # needed in case the auth server url is not the same as the one configured; e.g: localhost VS the k8S service

quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.token-propagation=true
quarkus.openapi-generator.<spec_file_yaml_or_json>.auth.<security_scheme>.header-name=X-Authorization-<provider>
```
With:
* `spec_file_yaml_or_json`: the name of the spec file configured with `_` as separator. E.g: if the file name is `simple-server.yaml` the normalized property name will be `simple_server_yaml`. This should be the same for every security scheme defined in the file.
* `security_scheme`: the name of the security scheme for which propagates the token located in the header defined by the `header-name` property. In our example it would be `SimpleBearerToken`.
* `provider`: the name of the expected provider from which the token comes from. As explained above, for each provider in RHDH, the Orchestrator plugin is adding a header with the format `X-Authorization-{provider}: {token}`.

Setting the `quarkus.oidc.*` properties will enforce the token validity check against the OIDC provider. Once successful, you will be able to use `$WORKFLOW.identity` in the workflow definition in order to get the identity of the user. See https://quarkus.io/guides/security-oidc-bearer-token-authentication and https://quarkus.io/guides/security-oidc-bearer-token-authentication-tutorial for more information.

## Basic auth
Basic auth token propagation is not currently supported.
A pull request has been opened to add support for it: https://github.com/quarkiverse/quarkus-openapi-generator/pull/1078

With Basic auth, the `$WORKFLOW.identity` is not available.

Instead you could access the header directly: `$WORKFLOW.headers.X-Authorization-{provider}` and decode it:
```
functions:
- name: getIdentity
  type: expression
  operation: '.identity=($WORKFLOW.headers["x-authorization-basic"] | @base64d | split(":")[0])' # mind the lower case!!
```

You can see a full example here: https://github.com/rhdhorchestrator/workflow-token-propagation-example.

