---
title: "Requirements"
date: 2024-11-25
---

## Operators
The Orchestrator runtime/deployment is made of two main parts: `OpenShift Serverless Logic operator` and `RHDH operator`

### OpenShift Serverless Logic operator requirements
OpenShift Serverless Logic operator resource requirements are described [OpenShift Serverless Logic Installation Requirements](https://openshift-knative.github.io/docs/docs/latest/serverless-logic/getting-started/preparing-environment.html#proc-minimal-local-environment-setup). This is mainly for local environment settings.  
The operator deploys a Data Index service and a Jobs service.
These are the recommended minimum resource requirements for their pods:  
`Data Index pod`:
```yaml
resources:
      limits:
        cpu: 500m
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 64Mi
```
`Jobs pod:`
```yaml
resources:
      limits:
        cpu: 200m
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 1Gi
```
The resources for these pods are controlled by a CR of type SonataFlowPlatform. There is one such CR in the sonataflow-infra namespace.

### RHDH operator requirements
The requirements for RHDH operator and its components are described [here](https://docs.redhat.com/en/documentation/red_hat_developer_hub/1.3/html/getting_started_with_red_hat_developer_hub/ref-rhdh-sizing_rhdh-getting-started#ref-rhdh-sizing_rhdh-getting-started)

## Workflows
Each workflow has its own logic and therefore different resource requirements that are influenced by its specific logic.  
Here are some metrics for the workflows we provide. For each workflow you have the following fields: cpu idle, cpu peak (during execution), memory.
- greeting workflow
  - cpu idle: 4m
  - cpu peak: 12m
  - memory: 300 Mb
- mtv-plan workflow
  - cpu idle: 4m
  - cpu peak: 130m
  - memory: 300 Mb

### How to evaluate resource requirements for your workflow
Locate the workflow pod in OCP Console. There is a tab for Metrics. Here you'll find the CPU and memory. Execute the workflow a few times. It does not matter whether it succeeds or not as long as all the states are executed. Now you can see the peak usage (execution) and the idle usage (after a few executions).
