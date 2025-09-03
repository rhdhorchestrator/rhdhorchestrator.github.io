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

In the following upgrade scenarios, we will try to avoid DB migration processes. We will provide solutions to reuse DB instances and connect them to new versions of the operator.

**Note**: Currently, there is no way to plug in an external DB instance for only the Sonataflow resources. When we say "external DB," we mean that the RHDH deployment will use a database that it did not create or provision itself. While you cannot use an external DB for only Sonataflow, you can allow Sonataflow resources to use a separate database within RHDH's primary DB instance.

### Current Orchestrator Architecture

Orchestrator v1.6 currently interacts with several operators and components:

- RHDH operator installation
- OpenShift Serverless and OpenShift Serverless Logic operators (including PostgreSQL database)
- Orchestrator dynamic plugins
- Software templates and GitOps integration

These components remain available in RHDH v1.7. However, instead of the Orchestrator Operator serving as a meta-operator managing RHDH and OSL installations, all deployment logic is integrated into the RHDH installation process.

## Upgrading with RHDH Helm Chart (Reusing Original PostgreSQL Instance)

**Prerequisites**: Familiarize yourself with [deploying RHDH via Helm Chart](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.7/html/installing_red_hat_developer_hub_on_openshift_container_platform/assembly-install-rhdh-ocp-helm).

In this upgrade scenario, we will reuse the PostgreSQL instance that was used for Orchestrator operator, and will treat it as an external DB for the new RHDH installation. No DB migration will be needed, as only a new database will be created in the same instance.

### Steps

1. **Prepare for installation**:

   - Delete the Orchestrator CR and the Backstage CR that was previously used in v1.6
   - Disable or delete your existing SonataFlow Platform.

2. **Configure external database connection**: following the [RHDH external database documentation](https://github.com/redhat-developer/rhdh-chart/blob/main/docs/external-db.md). Create a secret containing PostgreSQL connection properties. This guide will also have additional values for the Helm Chart, that will be used in the next step.

3. **Install RHDH v1.7 with Orchestrator**: using the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage), following the external database configuration guide. Make sure to include the following in the values.yaml:

   - Make sure to have "orchestrator" and "serverlessLogicOperator" enabled.

   ```yaml
   orchestrator:
     enabled: true
     serverlessLogicOperator:
       enabled: true
     serverlessOperator:
       enabled: true # optional
   ```

   - Include the extra values issued by the external-db installation doc from the former point.

   The Helm installation will create a new SonataFlow Platform in the same namespace as RHDH. The installation will reuse your existing 'sonataflow' database if found. The same PostgreSQL instance will be used for both RHDH and SonataFlow.

4. **Migrate workflows**: After installation, migrate any existing workflow deployments (SonataFlow CRs) to the RHDH namespace.

## Upgrading with RHDH Helm Chart (Creating a new PostgreSQL instance for RHDH)

In this upgrade scenario we will create a new PostgreSQL instance for RHDH to use, while keeping intact all resources in the sonataflow-infra namespace, including the PostgreSQL instance used by Orchestrator operator. After installation, we will reconfigure the Orchestrator plugins to point to the old previously used PostgreSQL instance.

### Steps

1. **Prepare for installation**:

   - Delete the Orchestrator CR and the Backstage CR that was previously used in v1.6

2. **Install RHDH v1.7 and Configure Orchestrator Plugin**: using the [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/backstage).
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

   The `backstage-plugin-scaffolder-backend-module-orchestrator-dynamic` and the `backstage-plugin-orchestrator-backend-dynamic` plugins require extra configurations that include the URL for the SonataFlow Data Index.

   Edit the values.yaml in the appropriate places to use the namespace that holds SonataFlow resources.

   ```yaml
   pluginConfig:
     orchestrator:
       dataIndexService:
         url: http://sonataflow-platform-data-index-service.sonataflow-infra
   ```

   Replace "sonataflow-infra" with the namespace you have previously installed your workflows and sonataflow resources in.

   Finally, Helm Install the Chart.

3. **Post Install Configurations**:  
   Configure a network policy to allow traffic only between RHDH, Knative, SonataFlow services, and workflows.

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

## Upgrading with RHDH Operator

**Prerequisites**: Familiarize yourself with [deploying RHDH via Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.6/html/about_red_hat_developer_hub/index).

Using the RHDH Operator, we will not be reinstalling all SonataFlow resources, rather reusing the resources created by the Orchestrator operator.

We will be Installing RHDH v1.7 operator with Orchestrator enabled, but with the _orchestrator plugin dependency disabled_. This will make the installation skip creating new SonataFlow resources.

### Steps

1. **Prepare for installation**:

   - Delete the Orchestrator CR and the Backstage CR that was previously used in v1.6

2. **Install RHDH v1.7 and Configure Orchestrator Plugin** using the [RHDH Operator](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.6/html/about_red_hat_developer_hub/index).

   Please follow the instructions to install RHDH v1.7 with orchestrator enabled, see the [README](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/docs/orchestrator.md).

   As of RHDH 1.7 all of the Orchestrator plugins are included in the default dynamic-plugins.yaml file of install-dynamic-plugins container but disabled by default. To enable the orchestrator plugin, you should refer the dynamic plugins ConfigMap with following data in your Backstage Custom Resource (CR) and put "false" under the "disabled" level.

   Please note to _not_ include the dependencies for the orchestrator plugin:

   ```yaml
   # Do NOT add this
   dependencies:
     - ref: sonataflow
   ```

   Adding the dependencies will trigger the new installation of Sonataflow resources. On this upgrade scenario, we would like to reuse our old sonataflow resources.

3. **Update Orchestrator Plugins**:
   The `backstage-plugin-scaffolder-backend-module-orchestrator-dynamic` and the `backstage-plugin-orchestrator-backend-dynamic` plugins require extra configurations that include the URL for the SonataFlow Data Index.

   Edit the values.yaml in the appropriate places to use the namespace that holds SonataFlow resources.

   ```yaml
   pluginConfig:
     orchestrator:
       dataIndexService:
         url: http://sonataflow-platform-data-index-service.sonataflow-infra
   ```

   Replace "sonataflow-infra" with the namespace you have previously installed your workflows and sonataflow resources in.

   The plugins that require this configuration are the "backstage-plugin-scaffolder-backend-module-orchestrator-dynamic" and the "backstage-plugin-orchestrator-backend-dynamic".

4. **Post Install Configurations**:  
   Configure a network policy to allow traffic only between RHDH, Knative, SonataFlow services, and workflows.

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

Using the RHDH operator, it is also possible to enable workflow deployment in a namespace other than where RHDH Orchestrator infrastructure are deployed and configured.
This will require some additional resources that need cluster-wide permissions, so this option is only available in the operator installation and not the helm chart.

To do so, please follow th steps documented [here](https://github.com/redhat-developer/rhdh-operator/blob/release-1.7/docs/orchestrator.md#optional-enabling-workflow-in-a-different-namespace).
