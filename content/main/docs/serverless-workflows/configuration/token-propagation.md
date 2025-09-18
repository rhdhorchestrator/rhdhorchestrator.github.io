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
## Openshift Serverless Logic (OSL) / SonataFlow related
By default, the workflow is not persisting the request headers in the database. Therefore, any token in the header will be lost if the workflow flushes its context (e.g: sleeps, goes idle, is resumed, ...) as the headers will not be restored to the context from the database.

By setting the property `kogito.persistence.headers.enabled` to `true` in the `application.properties` file or in the config map representing it on the cluster, the workflow will persist the headers. This will enable the workflow to keep using the token from the headers even after it was interupted and restored.


You can exclude headers from being persisted using `kogito.persistence.headers.excluded`. See https://sonataflow.org/serverlessworkflow/main/core/configuration-properties.html and/or https://sonataflow.org/serverlessworkflow/main/use-cases/advanced-developer-use-cases/persistence/persistence-with-postgresql.html#ref-postgresql-persistence-configuration for more information. 

## Security related
### Oauth2
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

### Bearer token
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

### Basic auth
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

# Configuring OIDC properties at SonataFlowPlatform level (Cluster-wide OIDC configuration) {#platform-oidc}
This short guide shows how to inject the Quarkus OIDC settings once at platform‑scope so that all present and future workflows automatically authenticate incoming requests and expose $WORKFLOW.identity.

# Prerequisites
* Namespace where the workflows run
* Keycloak Realm URL
* Client‑ID
* Client‑secret

### There is an assumption that the workflows and the platform are installed in the sonataflow-infra here.
export TARGET_NS='sonataflow-infra' # target namespace of workflows and sonataflowplatform CR

Keep the client secret in a Secrets vault; don’t embed it as clear‑text in the CR.

## Create the supporting objects
1. Secret: holds the confidential client secret

e.g
```
oc create secret generic oidc-client-secret \
  -n $TARGET_NS \
  --from-literal=cred=swf-client-secret  # This is a sample value. You need to replace it with actual value.
``` 

## Patch the SonataFlowPlatform CR
1. Create patch.yaml (or paste inline):

e.g
```
#### All the values below need to be replaced by actual values.
spec:
  properties:
    flow:
    - name: quarkus.oidc.auth-server-url
      value: https://keycloak-host/realms/dev
    - name: quarkus.oidc.client-id
      value: swf-client
    - name: quarkus.oidc.token.header
      value: X-Authorization
    - name: quarkus.oidc.token.issuer
      value: any
    - name: quarkus.oidc.credentials.secret
      valueFrom:
        secretKeyRef:
          key: cred
          name: oidc-client-secret
```

2. Apply the patch:

e.g
```
oc patch sonataflowplatform <Platform CR name> \
  -n $TARGET_NS \
  --type merge \
  -p "$(cat patch.yaml)"
```

Wait a few seconds for the operator reconcile loop.

## Verify the managed properties

e.g
```
oc get sonataflowplatform <Platform CR name> -n $TARGET_NS -o yaml
``` 
You should see all five keys.

Restart running workflow deployments once so Quarkus reloads the file:

e.g
```
oc rollout restart deployment -l sonataflow.org/workflow -n $TARGET_NS
``` 