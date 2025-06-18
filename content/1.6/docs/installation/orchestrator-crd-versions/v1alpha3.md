---
title: "CRD Version v1alpha3"
date: 2025-04-20
---

The Go-Based Operator was introduced in Orchestrator 1.5 since the helm-based operator is currently in maintenance mode.
Also, with major changes to the CRD, the v1alpha3 version
of [Orchestrator CRD](https://github.com/rhdhorchestrator/orchestrator-go-operator/blob/release-1.5/config/crd/bases/rhdh.redhat.com_orchestrators.yaml)
was introduced and is not backward compatible.

In this version, the CRD field structure has completely changed with most fields either removed or renamed and
restructured.
To see more information about the CRD fields, check out the
full [Parameter list](https://github.com/rhdhorchestrator/orchestrator-go-operator/blob/release-1.5/docs/crd/README.md).

The following Orchestrator CR is a sample of the api v1alpha3 version.

```yaml
apiVersion: rhdh.redhat.com/v1alpha3
kind: Orchestrator
metadata:
  labels:
    app.kubernetes.io/name: orchestrator-sample
  name: orchestrator-sample
spec:
  serverlessLogic:
    installOperator: true # Determines whether to install the ServerlessLogic operator. Defaults to True. Optional
  serverless:
    installOperator: true # Determines whether to install the Serverless operator. Defaults to True. Optional
  rhdh:
    installOperator: true # Determines whether the RHDH operator should be installed.This determines the deployment of the RHDH instance. Defaults to False. Optional
    devMode: true # Determines whether to enable the guest provider in RHDH. This should be used for development purposes ONLY and should not be enabled in production. Defaults to False. Optional
    name: "my-rhdh" # Name of RHDH CR, whether existing or to be installed. Required
    namespace: "rhdh" # Namespace of RHDH Instance, whether existing or to be installed. Required
    plugins:
      notificationsEmail:
        enabled: false # Determines whether to install the Notifications Email plugin. Requires setting of hostname and credentials in backstage secret. The secret, backstage-backend-auth-secret, is created as a pre-requisite. See value backstage-backend-auth-secret. See plugin configuration at https://github.com/backstage/backstage/blob/master/plugins/notifications-backend-module-email/config.d.ts
        port: 587 # SMTP server port. Defaults to 587. Optional
        sender: "" # Email address of the Sender. Defaults to empty string. Optional
        replyTo: "" # Email address of the Recipient. Defaults to empty string. Optional
  postgres:
    name: "sonataflow-psql-postgresql" # The name of the Postgres DB service to be used by platform services. Cannot be empty.
    namespace: "sonataflow-infra" # The namespace of the Postgres DB service to be used by platform services.
    authSecret:
      name: "sonataflow-psql-postgresql" # Name of existing secret to use for PostgreSQL credentials. Required
      userKey: postgres-username # Name of key in existing secret to use for PostgreSQL credentials. Required
      passwordKey: postgres-password # Name of key in existing secret to use for PostgreSQL credentials. Required
    database: sonataflow # Name of existing database instance used by data index and job service. Required
  platform: # Contains the configuration for the infrastructure services required for the Orchestrator to serve workflows by leveraging the OpenShift Serverless and OpenShift Serverless Logic capabilities.
    namespace: "sonataflow-infra"
    resources:
      requests:
        memory: "64Mi" # Defines the Memory resource limits. Optional
        cpu: "250m" # Defines the CPU resource limits. Optional
      limits:
        memory: "1Gi" # Defines the Memory resource limits. Optional
        cpu: "500m" # Defines the CPU resource limits. Optional
    eventing:
      broker: { }
    # To enable eventing communication with an existing broker, populate the following fields: 
    # broker: 
    #   name: "my-knative" # Name of existing Broker instance.
    #   namespace: "knative" # Namespace of existing Broker instance.
    monitoring:
      enabled: false # Determines whether to enable monitoring for platform. Optional
  tekton:
    enabled: false # Determines whether to create the Tekton pipeline and install the Tekton plugin on RHDH. Defaults to false. Optional
  argocd:
    enabled: false # Determines whether to install the ArgoCD plugin and create the orchestrator AppProject. Defaults to False. Optional
    namespace: "orchestrator-gitops" # Namespace where the ArgoCD operator is installed and watching for argoapp CR instances. Optional
```

Migrating to the v1alpha3 CRD version involves upgrading the operator. Please follow
the [Operator Upgrade documentation](https://github.com/rhdhorchestrator/orchestrator-go-operator/tree/release-1.5?tab=readme-ov-file#upgrading-the-operator)
