# Upgrading from Orchestrator 1.6 to RHDH 1.7

Starting with Red Hat Developer Hub (RHDH) v1.7, Orchestrator will be shipped as an integrated component of RHDH, available through both the RHDH operator and RHDH Helm Chart. Users currently running Orchestrator v1.6 cannot directly upgrade to "Orchestrator 1.7". Instead, they must install the new RHDH release with Orchestrator enabled and migrate their existing data and resources.

## Background: RHDH and Orchestrator Integration

From RHDH v1.7 onwards, Orchestrator will be shipped alongside RHDH as a plugin with all current capabilities. The components will be installed together, and Orchestrator will no longer have its own dedicated operator. For detailed information about this transition, see our [blog post about installing Orchestrator via RHDH Chart](https://www.rhdhorchestrator.io/blog/installing-orchestrator-via-rhdh-chart/).

### Background: Database Management

RHDH supports two database management options:

### PostgreSQL Database Management in RHDH

The RHDH platform requires a PostgreSQL database. How this database is provisioned depends on whether you are installing via the **RHDH Operator** or the **RHDH Helm Chart**:

- **Local PostgreSQL instance (Operator-managed)**:  
  When installing with the [RHDH Operator](https://github.com/redhat-developer/rhdh-operator), a local PostgreSQL database is created and managed directly by the operator.  
  This does **not** use the Bitnami Helm Chart. Instead, the operator provisions and controls Kubernetes resources such as `StatefulSet`, `Service`, and `PersistentVolumeClaim` to run PostgreSQL natively.  
  You can see the database definition in the operator’s [db-statefulset.yaml](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/config/profile/rhdh/default-config/db-statefulset.yaml).

- **Local PostgreSQL instance (Chart-managed)**:  
  When installing with the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-operator/tree/main/chart), a local PostgreSQL instance is deployed via the [Bitnami PostgreSQL Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/postgresql).  
  In this case, lifecycle management (upgrades, resource handling, etc.) is handled through the chart.

- **External database**: A PostgreSQL instance that is managed separately, and its connection configuration is known to the admin. This is the [recommended](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/configuring_red_hat_developer_hub/configuring-external-postgresql-databases) setup in production.

RHDH has built-in integration for using external databases, both for the [Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/external-db.md) and the [operator](https://github.com/redhat-developer/rhdh-operator/blob/main/docs/external-db.md).

In the following upgrade scenarios, we will try to avoid database migration processes. We will provide solutions to reuse database instances and connect them to new versions of the operator.

**Note**: Currently, there is no way to plug in an external database instance for only the SonataFlow resources. When we say "external database," we mean that the RHDH deployment will use a database that it did not create or provision itself. While you cannot use an external database for only SonataFlow, you can allow SonataFlow resources to use a separate database within RHDH's primary database instance.

### Current Orchestrator Architecture

Orchestrator v1.6 currently interacts with several operators and components:

- RHDH operator installation
- OpenShift Serverless and OpenShift Serverless Logic operators (including PostgreSQL database)
- Orchestrator dynamic plugins
- Software templates and GitOps integration

These components remain available in RHDH v1.7. However, instead of the Orchestrator Operator serving as a meta-operator managing RHDH and OSL installations, all deployment logic is integrated into the RHDH installation process.

## Upgrading with RHDH Helm Chart (Reusing Original PostgreSQL Instance)

**Prerequisites**: Familiarize yourself with [deploying RHDH via Helm Chart](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/installing_red_hat_developer_hub_on_openshift_container_platform/assembly-install-rhdh-ocp-helm).

In this upgrade scenario, we will reuse the PostgreSQL instance that was used for the Orchestrator operator and treat it as an external database for the new RHDH installation. No database migration will be needed, as only a new database will be created in the same instance.

### Steps

1. **Prepare for installation**:

   - Disable the Orchestrator Operator controller:

      `oc scale deploy orchestrator-operator-controller-manager -n openshift-operators --replicas=0`
   - Remove the Orchestrator label from the SonataFlowPlatform resource: 

      `oc label sonataflowplatform sonataflow-platform -n sonataflow-infra rhdh.redhat.com/created-by-`
   - Remove the Orchestrator label from the OpenShift Serverless and OpenShift Serverless Logic subscriptions:

      ```bash 
      oc label subs serverless-operator -n openshift-serverless rhdh.redhat.com/created-by- || true
      oc label subs logic-operator-rhel8 -n openshift-serverless-logic rhdh.redhat.com/created-by- || true
      ```
   - Delete any running RHDH instance via the UI or by deleting the Backstage CR
   - Uninstall the RHDH v1.6 Operator
   - Delete old ConfigMaps used by RHDH (backup before):

      `oc delete cm -n rhdh -l rhdh.redhat.com/created-by=orchestrator`
   - Delete old RHDH Postgres PVCs

      `oc delete pvc <pvc-name> -n rhdh`

**Do not delete the Orchestrator CR, as its removal can delete important resources that cannot be retrieved**

**Note**: There is no need for the orchestrator-infra chart, as you should already have OpenshiftServerless and OpenshiftServerlessLogic operators installed.

2. **Configure external database connection**: Follow the [RHDH external database documentation](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/external-db.md). Create a secret containing PostgreSQL connection properties. This guide will also have additional values for the Helm Chart that will be used in the next step.

3. **Install RHDH v1.7 with Orchestrator**: Use the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage), following the external database configuration guide. Make sure to include the following in the values.yaml:

   - Make sure to have "orchestrator" and "serverlessLogicOperator" enabled.

   ```yaml
   orchestrator:
     enabled: true
     serverlessLogicOperator:
       enabled: true
     serverlessOperator:
       enabled: true # optional
   ```

   - Include the extra values provided by the external database installation documentation from the previous step.

   The Helm installation will create a new SonataFlow Platform in the same namespace as RHDH. The installation will reuse your existing 'sonataflow' database if found. The same PostgreSQL instance will be used for both RHDH and SonataFlow.


## Upgrading with RHDH Helm Chart

### Steps

1. **Prepare for installation**:

   - Disable the Orchestrator Operator controller:

      `oc scale deploy orchestrator-operator-controller-manager -n openshift-operators --replicas=0`
   - Remove the Orchestrator label from the SonataFlowPlatform resource: 

      `oc label sonataflowplatform sonataflow-platform -n sonataflow-infra rhdh.redhat.com/created-by-`
   - Remove the Orchestrator label from the OpenShift Serverless and OpenShift Serverless Logic subscriptions:

      ```bash 
      oc label subs serverless-operator -n openshift-serverless rhdh.redhat.com/created-by- || true
      oc label subs logic-operator-rhel8 -n openshift-serverless-logic rhdh.redhat.com/created-by- || true
      ```
   - Delete any running RHDH instance via the UI or by deleting the Backstage CR
   - Uninstall the RHDH v1.6 Operator
   - Delete old ConfigMaps used by RHDH (backup before):

      `oc delete cm -n rhdh -l rhdh.redhat.com/created-by=orchestrator`
   - Delete old RHDH Postgres PVCs

      `oc delete pvc <pvc-name> -n rhdh`

**Do not delete the Orchestrator CR, as its removal can delete important resources that cannot be retrieved**

**Note**: There is no need for the orchestrator-infra chart, as you should already have OpenshiftServerless and OpenshiftServerlessLogic operators installed.

2. **Install RHDH v1.7 and Configure Orchestrator Plugin**: Use the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage).

   Make sure to include the following in the values.yaml:

   - Make sure to have "orchestrator" enabled and "serverlessLogicOperator" disabled.

   ```yaml
   orchestrator:
     enabled: true
     serverlessLogicOperator:
       enabled: false
     serverlessOperator:
       enabled: false # optional
   ```

   The `backstage-plugin-scaffolder-backend-module-orchestrator-dynamic` and `backstage-plugin-orchestrator-backend-dynamic` plugins require extra configurations that include the URL for the SonataFlow Data Index.

   Edit the values.yaml in the appropriate places to use the namespace that holds SonataFlow resources.

   ```yaml
   pluginConfig:
     orchestrator:
       dataIndexService:
         url: http://sonataflow-platform-data-index-service.sonataflow-infra
   ```

   Replace `sonataflow-infra` with the namespace where you have previously installed your workflows and SonataFlow resources.

   Finally, Helm Install the Chart.

3. **Post-Install Configuration**:  
   Configure these network policies to allow traffic only between RHDH, Knative, SonataFlow services, and workflows. This is crucial for the backstage pod to become live.

   ```yaml
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: rhdh-allow-knative-to-sonataflow-and-workflows
       # Sonataflow and Workflows are using the RHDH target namespace.
       namespace: rhdh
     spec:
       podSelector: {}
       ingress:
         - from:
           - namespaceSelector:
               matchLabels:
                 # Allow knative events to be delivered to workflows.
                 kubernetes.io/metadata.name: knative-eventing
           - namespaceSelector:
               matchLabels:
                 # Allow auxiliary knative function for workflow (such as m2k-save-transformation)
                 kubernetes.io/metadata.name: knative-serving
     ---
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: rhdh-allow-external-communication
       namespace: rhdh
     spec:
       podSelector: {}
       policyTypes:
         - Ingress
       ingress:
         - from:
           - namespaceSelector:
               matchLabels:
                 # Allow knative events to be delivered to workflows.
                 policy-group.network.openshift.io/ingress: ""
     ---
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: rhdh-allow-intra-network
       namespace: rhdh
     spec:
       # Apply this policy to all pods in the namespace
       podSelector: {}
       # Specify policy type as 'Ingress' to control incoming traffic rules
       policyTypes:
         - Ingress
       ingress:
         - from:
           # Allow ingress from any pod within the same namespace
           - podSelector: {}


     ---
     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: rhdh-allow-monitoring-to-sonataflow-and-workflows
       namespace: rhdh
     spec:
       # Apply this policy to all pods in the namespace
       podSelector: {}
       # Specify policy type as 'Ingress' to control incoming traffic rules
       policyTypes:
         - Ingress
       ingress:
         - from:
           - namespaceSelector:
               matchLabels:
                 # Allow openshift-user-workload-monitoring pods to access the workflow.
                 kubernetes.io/metadata.name: openshift-user-workload-monitoring

   ```

## Upgrading with RHDH Operator

**Prerequisites**: Familiarize yourself with [deploying RHDH via Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.6/html/about_red_hat_developer_hub/index).

Using the RHDH Operator, we will not be reinstalling all SonataFlow resources, but rather reusing the resources created by the Orchestrator operator.

We will be installing the RHDH v1.7 operator with Orchestrator enabled, but with the _orchestrator plugin dependency disabled_. This will make the installation skip creating new SonataFlow resources.

### Steps

1. **Prepare for installation**:

Our goal is to disable the Orchestrator operator and avoid it deleting important SonataFlow workloads, pods, and databases.

   - Disable the Orchestrator Operator controller:

      `oc scale deploy orchestrator-operator-controller-manager -n openshift-operators --replicas=0`
   - Remove the Orchestrator label from the SonataFlowPlatform resource: 

      `oc label sonataflowplatform sonataflow-platform -n sonataflow-infra rhdh.redhat.com/created-by-`
   - Remove the Orchestrator label from the OpenShift Serverless and OpenShift Serverless Logic subscriptions:

      ```bash 
      oc label subs serverless-operator -n openshift-serverless rhdh.redhat.com/created-by- || true
      oc label subs logic-operator-rhel8 -n openshift-serverless-logic rhdh.redhat.com/created-by- || true
      ```
   - Delete any running RHDH instance via the UI or by deleting the Backstage CR
   - Uninstall the RHDH v1.6 Operator
   - Delete old ConfigMaps used by RHDH (backup before):

      `oc delete cm -n rhdh -l rhdh.redhat.com/created-by=orchestrator`
   - Delete old RHDH Postgres PVCs

      `oc delete pvc <pvc-name> -n rhdh`

**Do not delete the Orchestrator CR, as its removal can delete important resources that cannot be retrieved**

2. **Configuring Orchestrator Plugins**:

   As of RHDH 1.7, all of the Orchestrator plugins are included in the default dynamic-plugins.yaml file of the install-dynamic-plugins container, but disabled by default. To enable the orchestrator plugin, you should reference the dynamic plugins ConfigMap with the following data in your Backstage Custom Resource (CR) and set "false" under the "disabled" level.

   The `backstage-plugin-scaffolder-backend-module-orchestrator-dynamic` and `backstage-plugin-orchestrator-backend-dynamic` plugins require extra configurations that include the URL for the SonataFlow Data Index.

   Edit the values.yaml in the appropriate places to use the namespace that holds SonataFlow resources.

   ```yaml
   pluginConfig:
     orchestrator:
       dataIndexService:
         url: http://sonataflow-platform-data-index-service.{{ namespace }}
   ```

   Replace {{ namespace }} with 'sonataflow-infra', for example, or any other namespace where you have previously installed your workflows and SonataFlow resources.

   Your custom plugin configuration can look like the one [provided here](https://github.com/redhat-developer/rhdh-operator/blob/main/config/profile/rhdh/default-config/dynamic-plugins.yaml). That should contain the latest plugin versions that are released with the operator.

   An example config:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: orchestrator-plugin
   data:
     dynamic-plugins.yaml: |
       includes:
         - dynamic-plugins.default.yaml
       plugins:
         - disabled: false
           package: "https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator-backend-dynamic/-/backstage-plugin-orchestrator-backend-dynamic-1.6.0.tgz"
           integrity: sha512-Kr55YbuVwEADwGef9o9wyimcgHmiwehPeAtVHa9g2RQYoSPEa6BeOlaPzB6W5Ke3M2bN/0j0XXtpLuvrlXQogA==
           pluginConfig:
             orchestrator:
               dataIndexService:
                 url: http://sonataflow-platform-data-index-service.sonataflow-infra
         - disabled: false
           package: "https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator/-/backstage-plugin-orchestrator-1.6.0.tgz"
           integrity: sha512-fOSJv2PgtD2urKwBM7p9W6gV/0UIHSf4pkZ9V/wQO0eg0Zi5Mys/CL1ba3nO9x9l84MX11UBZ2r7PPVJPrmOtw==
           pluginConfig:
             dynamicPlugins:
               frontend:
                 red-hat-developer-hub.backstage-plugin-orchestrator:
                   appIcons:
                     - importName: OrchestratorIcon
                       name: orchestratorIcon
                   dynamicRoutes:
                     - importName: OrchestratorPage
                       menuItem:
                         icon: orchestratorIcon
                         text: Orchestrator
                       path: /orchestrator
         - disabled: false
           package: "https://npm.registry.redhat.com/@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic/-/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic-1.6.0.tgz"
           integrity: sha512-Bueeix4661fXEnfJ9y31Yw91LXJgw6hJUG7lPVdESCi9VwBCjDB9Rm8u2yPqP8sriwr0OMtKtqD+Odn3LOPyVw==
           pluginConfig:
             orchestrator:
               dataIndexService:
                 url: http://sonataflow-platform-data-index-service.sonataflow-infra
         - disabled: false
           package: "https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator-form-widgets/-/backstage-plugin-orchestrator-form-widgets-1.6.0.tgz"
           integrity: sha512-Tqn6HO21Q1TQ7TFUoRhwBVCtSBzbQYz+OaanzzIB0R24O6YtVx3wR7Chtr5TzC05Vz5GkBO1+FZid8BKpqljgA==
           pluginConfig:
             dynamicPlugins:
               frontend:
                 red-hat-developer-hub.backstage-plugin-orchestrator-form-widgets: {}
   ```

   Please note to _not_ include the dependencies for the orchestrator plugin:

   ```yaml
   # Do NOT add this
   dependencies:
     - ref: sonataflow
   ```

   As we don't want the RHDH operator to install an additional SonataFlowPlatform.

3. **Install RHDH v1.7**: Use the [RHDH Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.6/html/about_red_hat_developer_hub/index).

   You can use the Operator Hub UI to install RHDH 1.7. Please follow the instructions to install RHDH v1.7 with Orchestrator enabled. See the [README](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/docs/orchestrator.md).

4. **Prepare necessary configuration before creating the RHDH instance**

   See the [example](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/examples/orchestrator.yaml) for a complete configuration of the orchestrator plugin.

   You should have:

   - A ConfigMap containing the enablement of the orchestrator plugins
   - A secret with the BACKEND_SECRET key/value and update the secret name in the Backstage CR under the extraEnvs field. You should have one already configured from your old orchestrator setup
   - An "app-config-rhdh" containing your migrated app config from your v1.6 installation, containing any auth provider, integration, etc.

   You should also have all operator prerequisites already filled, namely:

    - Preparing to install OpenShift Serverless.
    - Installing the OpenShift Serverless Operator.
    - Installing Knative Serving.
    - Installing Knative Eventing.
    - Installing the OpenShift Serverless Logic Operator.
    

5. **Install the RHDH instance via the RHDH UI**

   While creating the instance, make sure to edit your new Backstage CR as suggested in the Orchestrator example [here](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/examples/orchestrator.yaml).

6. **Post-Install Configuration**:

   Your old network policies should be intact, but in case they were not migrated, configure a network policy to allow traffic only between RHDH, Knative, SonataFlow services, and workflows.

   ```console
   oc create -f - <<EOF
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-infra-ns-to-workflow-ns
     namespace: sonataflow-infra
   spec:
     podSelector: {}
     ingress:
       - from:
         - namespaceSelector:
             matchLabels:
               # Allow traffic from pods in the RHDH namespace.
               kubernetes.io/metadata.name: ${RHDH_INSTALL_NS}
         - namespaceSelector:
             matchLabels:
               # Allow traffic from pods in the Workflow namespace.
               kubernetes.io/metadata.name: ${WORKFLOW_NS}
         - namespaceSelector:
             matchLabels:
               # Allow traffic from pods in the Knative Eventing namespace.
               kubernetes.io/metadata.name: knative-eventing
         - namespaceSelector:
             matchLabels:
               # Allow traffic from pods in the Knative Serving namespace.
               kubernetes.io/metadata.name: knative-serving
         - namespaceSelector:
             matchLabels:
               # Allow traffic from pods in the openshift serverless logic namespace.
               kubernetes.io/metadata.name: openshift-serverless-logic
   EOF
   ```

## Configuring a workflow from different namespace

Using the RHDH operator, it is also possible to enable workflow deployment in a namespace other than where RHDH Orchestrator infrastructure is deployed and configured.
This will require some additional resources that need cluster-wide permissions, so this option is only available in the operator installation and not the Helm chart.

To do so, please follow the steps documented [here](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/docs/orchestrator.md#optional-enabling-workflow-in-a-different-namespace).
