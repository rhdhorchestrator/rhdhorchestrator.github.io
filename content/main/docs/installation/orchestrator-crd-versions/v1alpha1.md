---
title: "CRD Version v1alpha1"
date: 2025-02-12
---

The v1alpha1 version of [Orchestrator CRD](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/release-1.3/config/crd/bases/rhdh.redhat.com_orchestrators.yaml) is supported only on Orchestrator 1.3 version. 
It is deprecated and not compatible with future orchestrator versions.


The following Orchestrator CR is an sample of the api v1alpha1 version.


```yaml
apiVersion: rhdh.redhat.com/v1alpha1
kind: Orchestrator
metadata:
  name: orchestrator-sample
spec:
  sonataFlowOperator:
    isReleaseCandidate: false # Indicates RC builds should be used by the chart to install Sonataflow
    enabled: true # whether the operator should be deployed by the chart
    subscription:
      namespace: openshift-serverless-logic # namespace where the operator should be deployed
      channel: alpha # channel of an operator package to subscribe to
      installPlanApproval: Automatic # whether the update should be installed automatically
      name: logic-operator-rhel8 # name of the operator package
      sourceName: redhat-operators # name of the catalog source
      startingCSV: logic-operator-rhel8.v1.34.0 # The initial version of the operator
  serverlessOperator:
    enabled: true # whether the operator should be deployed by the chart
    subscription:
      namespace: openshift-serverless # namespace where the operator should be deployed
      channel: stable # channel of an operator package to subscribe to
      installPlanApproval: Automatic # whether the update should be installed automatically
      name: serverless-operator # name of the operator package
      sourceName: redhat-operators # name of the catalog source
  rhdhOperator:
    isReleaseCandidate: false # Indicates RC builds should be used by the chart to install RHDH
    enabled: true # whether the operator should be deployed by the chart
    enableGuestProvider: false # whether to enable guest provider
    secretRef:
      name: backstage-backend-auth-secret # name of the secret that contains the credentials for the plugin to establish a communication channel with the Kubernetes API, ArgoCD, GitHub servers and SMTP mail server.
      backstage:
        backendSecret: BACKEND_SECRET # Key in the secret with name defined in the 'name' field that contains the value of the Backstage backend secret. Defaults to 'BACKEND_SECRET'. It's required.
      github: #GitHub specific configuration fields that are injected to the backstage instance to allow the plugin to communicate with GitHub.
        token: GITHUB_TOKEN # Key in the secret with name defined in the 'name' field that contains the value of the authentication token as expected by GitHub. Required for importing resource to the catalog, launching software templates and more. Defaults to 'GITHUB_TOKEN', empty for not available.
        clientId: GITHUB_CLIENT_ID # Key in the secret with name defined in the 'name' field that contains the value of the client ID that you generated on GitHub, for GitHub authentication (requires GitHub App). Defaults to 'GITHUB_CLIENT_ID', empty for not available.
        clientSecret: GITHUB_CLIENT_SECRET # Key in the secret with name defined in the 'name' field that contains the value of the client secret tied to the generated client ID. Defaults to 'GITHUB_CLIENT_SECRET', empty for not available.
      k8s: # Kubernetes specific configuration fields that are injected to the backstage instance to allow the plugin to communicate with the Kubernetes API Server.
        clusterToken: K8S_CLUSTER_TOKEN # Key in the secret with name defined in the 'name' field that contains the value of the Kubernetes API bearer token used for authentication. Defaults to 'K8S_CLUSTER_TOKEN', empty for not available.
        clusterUrl: K8S_CLUSTER_URL # Key in the secret with name defined in the 'name' field that contains the value of the API URL of the kubernetes cluster. Defaults to 'K8S_CLUSTER_URL', empty for not available.
      argocd: # ArgoCD specific configuration fields that are injected to the backstage instance to allow the plugin to communicate with ArgoCD. Note that ArgoCD must be deployed beforehand and the argocd.enabled field must be set to true as well.
        url: ARGOCD_URL # Key in the secret with name defined in the 'name' field that contains the value of the URL of the ArgoCD API server. Defaults to 'ARGOCD_URL', empty for not available.
        username: ARGOCD_USERNAME # Key in the secret with name defined in the 'name' field that contains the value of the username to login to ArgoCD. Defaults to 'ARGOCD_USERNAME', empty for not available.
        password: ARGOCD_PASSWORD # Key in the secret with name  defined in the 'name' field that contains the value of the password to authenticate to ArgoCD. Defaults to 'ARGOCD_PASSWORD', empty for not available.
      notificationsEmail:
        hostname: NOTIFICATIONS_EMAIL_HOSTNAME # Key in the secret with name defined in the 'name' field that contains the value of the hostname of the SMTP server for the notifications plugin. Defaults to 'NOTIFICATIONS_EMAIL_HOSTNAME', empty for not available.
        username: NOTIFICATIONS_EMAIL_USERNAME # Key in the secret with name defined in the 'name' field that contains the value of the username of the SMTP server for the notifications plugin. Defaults to 'NOTIFICATIONS_EMAIL_USERNAME', empty for not available.
        password: NOTIFICATIONS_EMAIL_PASSWORD # Key in the secret with name defined in the 'name' field that contains the value of the password of the SMTP server for the notifications plugin. Defaults to 'NOTIFICATIONS_EMAIL_PASSWORD', empty for not available.
    subscription:
      namespace: rhdh-operator # namespace where the operator should be deployed
      channel: fast-1.3 # channel of an operator package to subscribe to
      installPlanApproval: Automatic # whether the update should be installed automatically
      name: rhdh # name of the operator package
      source: redhat-operators # name of the catalog source
      startingCSV: "" # The initial version of the operator
      targetNamespace: rhdh-operator # the target namespace for the backstage CR in which RHDH instance is created
  rhdhPlugins: # RHDH plugins required for the Orchestrator
    npmRegistry: "https://npm.registry.redhat.com" # NPM registry is defined already in the container, but sometimes the registry need to be modified to use different versions of the plugin, for example: staging(https://npm.stage.registry.redhat.com) or development repositories
    scope: "@redhat"
    orchestrator:
      package: "backstage-plugin-orchestrator@1.3.0"
      integrity: sha512-A/twx1SOOGDQjglLzOxQikKO0XOdPP1jh2lj9Y/92bLox8mT+eaZpub8YLwR2mb7LsUIUImg+U6VnKwoAV9ATA==
    orchestratorBackend:
      package: "backstage-plugin-orchestrator-backend-dynamic@1.3.0"
      integrity: sha512-Th5vmwyhHyhURwQo28++PPHTvxGSFScSHPJyofIdE5gTAb87ncyfyBkipSDq7fwj4L8CQTXa4YP6A2EkHW1npg==
    notifications:
      package: "plugin-notifications-dynamic@1.3.0"
      integrity: sha512-iYLgIy0YdP/CdTLol07Fncmo9n0J8PdIZseiwAyUt9RFJzKIXmoi2CpQLPKMx36lEgPYUlT0rFO81Ie2CSis4Q==
    notificationsBackend:
      package: "plugin-notifications-backend-dynamic@1.3.0"
      integrity: sha512-Pw9Op/Q+1MctmLiVvQ3M+89tkbWkw8Lw0VfcwyGSMiHpK/Xql1TrSFtThtLlymRgeCSBgxHYhh3MUusNQX08VA==
    signals:
      package: "plugin-signals-dynamic@1.3.0"
      integrity: sha512-+E8XeTXcG5oy+aNImGj/MY0dvEkP7XAsu4xuZjmAqOHyVfiIi0jnP/QDz8XMbD1IjCimbr/DMUZdjmzQiD0hSQ==
    signalsBackend:
      package: "plugin-signals-backend-dynamic@1.3.0"
      integrity: sha512-5Bl6C+idPXtquQxMZW+bjRMcOfFYcKxcGZZFv2ITkPVeY2zzxQnAz3vYHnbvKRSwlQxjIyRXY6YgITGHXWT0nw==
    notificationsEmail:
      enabled: false # whether to install the notifications email plugin. requires setting of hostname and credentials in backstage secret to enable. See value backstage-backend-auth-secret. See plugin configuration at https://github.com/backstage/backstage/blob/master/plugins/notifications-backend-module-email/config.d.ts
      package: "plugin-notifications-backend-module-email-dynamic@1.3.0"
      integrity: sha512-sm7yRoO6Nkk3B7+AWKb10maIrb2YBNSiqQaWmFDVg2G9cbDoWr9wigqqeQ32+b6o2FenfNWg8xKY6PPyZGh8BA==
      port: 587 # SMTP server port
      sender: "" # the email sender address
      replyTo: "" # reply-to address
  postgres:
    serviceName: "sonataflow-psql-postgresql" # The name of the Postgres DB service to be used by platform services. Cannot be empty.
    serviceNamespace: "sonataflow-infra" # The namespace of the Postgres DB service to be used by platform services.
    authSecret:
      name: "sonataflow-psql-postgresql" # name of existing secret to use for PostgreSQL credentials.
      userKey: postgres-username # name of key in existing secret to use for PostgreSQL credentials.
      passwordKey: postgres-password # name of key in existing secret to use for PostgreSQL credentials.
    database: sonataflow # existing database instance used by data index and job service
  orchestrator:
    namespace: "sonataflow-infra" # Namespace where sonataflow's workflows run. The value is captured when running the setup.sh script and stored as a label in the selected namespace. User can override the value by populating this field. Defaults to `sonataflow-infra`.
    sonataflowPlatform:
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "1Gi"
          cpu: "500m"
  tekton:
    enabled: false # whether to create the Tekton pipeline resources
  argocd:
    enabled: false # whether to install the ArgoCD plugin and create the orchestrator AppProject
    namespace: "" # Defines the namespace where the orchestrator's instance of ArgoCD is deployed. The value is captured when running setup.sh script and stored as a label in the selected namespace. User can override the value by populating this field. Defaults to `orchestrator-gitops` in the setup.sh script.
  networkPolicy:
    rhdhNamespace: "rhdh-operator" # Namespace of existing RHDH instance
```