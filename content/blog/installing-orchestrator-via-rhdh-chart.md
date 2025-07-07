---
date: 2025-04-28
title: Installing Orchestrator via the Red Hat Developer Hub Helm Chart
tags: [Orchestrator-1.7]
---

# Installing Orchestrator via the Red Hat Developer Hub Helm Chart

This blog introduces a streamlined installation method that allows users to deploy Orchestrator alongside Red Hat Developer Hub (RHDH) using a Helm Chart and with minimal configuration and effort.

With this approach, the Orchestrator Plugins are installed directly within RHDH and deployed to a single target namespace. As a result, all RHDH components, Openshift Serverless Logic platform services, and serverless workflows coexist within the same namespace, simplifying the deployment footprint and operational model.

A brief explanation of Orchestrator components is given to clarify any gaps before advancing in comparing the different installation types.

## Background: Orchestrator Components 
Orchestrator utilizes several components to build, serve, run and present serverless workflows, for example:

- *Red Hat Developer Hub (RHDH)*: This component acts as the UI, serving the Orchestrator plugins which allows the users to launch, monitor and manage workflows. On RHDH, users can launch software templates, such as the orchestrator's [software templates](https://github.com/rhdhorchestrator/workflow-software-templates) which build serverless workflow projects. RHDH also integrates the ArgoCD and Tekton plugins, allowing for monitoring associated projects to the RHDH and Orchestrator instance. On RHDH, the notifications plugin can be configured, to be used with serverless workflows that send notifications. 

- *Openshift Serverless Logic (OSL)*: This operator serves as the backend engine for running serverless workflows. Orchestrator uses OSL's sonataflow platform resources to launch and deploy workflows, and acts as the bridge between RHDH and OSL. Orchestrator is responsible for installing and configuring this operator. 

- *Openshift Serverless*: This operator manages Knative-Eventing and Knative-Serving services, which allow handling event-driven communication from workflows, and provide the capability to deploy and scale stateless services on demand, respectively. Orchestrator is responsible for installing and configuring this operator. 


## Background: RHDH Helm Chart

The [RHDH Helm Chart](https://github.com/redhat-developer/rhdh-chart) allows users to install Red Hat Developer Hub on an OpenShift or Kubernetes cluster. RHDH can be enriched with custom plugins, extended configurations, and integrated with developer tools such as GitHub, GitLab, ArgoCD, and Tekton. 

## Comparing to the Orchestrator Operator

Until recently, the only supported way to install Orchestrator was through the Orchestrator Operator, available in the Operator Catalog for OpenShift clusters. Now, with the introduction of a Helm chart alternative, users have a more flexible installation option. While both methods enable the core capabilities of Orchestrator, they differ in architecture, permissions, and deployment scope. This section highlights the key similarities and differences between the two approaches.

### Similarity 

The core functionality of Orchestrator is available in both installation methods:

1. The Orchestrator plugins as add-ons for RHDH are included
1. The ability to run and monitor Serverless Workflows (via OpenShift Serverless Logic)
1. Software templates and integration with Openshift Pipelines and Openshift Gitops.

### Differences

1. **Feature Trimming**
   - Installing Orchestrator via the RHDH Helm Chart no longer installs by default all components that were previously bundled in the Orchestrator Operator. Instead, deploying features like integration with Openshift-Gitops, Openshift-Pipelines, and software templates are handled by separate Helm Charts and are configured post-install.

1. **Permissions**
   - Using Orchestrator no longer requires cluster-wide permissions, and can now operate fully within a single namespace.
   -  Admin-level permissions are only required for installing the Openshift Serverless and Openshift Serverless Logic operators and the cluster-wide Knative services. 
   -  To use the Orchestrator, only namespace-scope permissions are required, both for installing an RHDH instance and for deploying and running workflows.

For more information on required infrastructure that requires Admin-level permissions, please advise the[`orchestrator-infra`](https://github.com/redhat-developer/rhdh-chart/blob/main/charts/orchestrator-infra/README.md) chart.

1. **Deployment Scope**
   - All components (Orchestrator, RHDH, workflows) are now deployed within a single namespace.
   - This simplifies resource management and access control.

1. **Installation Method**
   - Previously installed via a dedicated Operator available in Operator Hub.
   - For now installed via Helm Charts (`backstage` and `orchestrator-infra`).

1. **Meta-Operator Behavior**
   - Orchestrator no longer acts as a meta-operator.
   - It no longer installs the RHDH operator, rather the RHDH chart installs Orchestrator related resources.
   - It still installs OpenShift Serverless and Serverless Logic Operators, but does so via the `orchestrator-infra` chart.

1. **Software Templates**
   - Workflow project software templates are no longer installed by default when using the Helm chart. To include them, some post setup is required and is detailed below. 


## Comparing Operator and Helm Chart
| Feature / Aspect            | Orchestrator Operator                                     | Helm Chart                                                                 |
|----------------------------|-------------------------------------------------------|----------------------------------------------------------------------------|
| Installs RHDH              | Yes (Orchestrator acts as a meta-operator)            | Yes (The `backstage` chart installs RHDH and Orchestrator)                                 |
| Privileges                 | Requires cluster-wide permissions                     | Namespace-scoped; elevated permissions isolated in `orchestrator-infra`   |
| Scope                      | Multi-namespace deployment                            | Single namespace for Orchestrator, RHDH, and workflows                    |
| Operator Management        | Installs RHDH and other operators                     | Delegates operator installation to `orchestrator-infra` chart             |
| Meta-Operator Behavior     | Yes (controls RHDH and related operator installs)     | No (installation responsibilities are split and explicit)                 |
| Software Templates         | Installed by default                                  | Not installed by default                                                  |


## Using backstage chart to install Orchestrator

The full Installation steps can be found in the [RHDH Chart README](https://github.com/redhat-developer/rhdh-chart/tree/main/charts/backstage#:~:text=Installing%20RHDH%20with%20Orchestrator%20on%20OpenShift)

## Post-install configurations

// Work in progress

### Dynamic Plugins

Optionally, some workflows that users may use will send notifications to RHDH during their run. To run these types of workflows with Orchestrator, some additional plugins are required to be activated to allow Orchestrator to function. Please add these plugins to the values.yaml file used in the RHDH chart:

```
    plugins: 
      - disabled: false
        package: "./dynamic-plugins/dist/backstage-plugin-notifications"
      - disabled: false
        package: "./dynamic-plugins/dist/backstage-plugin-signals"
      - disabled: false
        package: "./dynamic-plugins/dist/backstage-plugin-notifications-backend-dynamic"
      - disabled: false
        package: "./dynamic-plugins/dist/backstage-plugin-signals-backend-dynamic"
```