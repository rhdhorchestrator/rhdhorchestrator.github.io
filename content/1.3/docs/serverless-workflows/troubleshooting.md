---
title: Troubleshooting
date: "2024-07-25"
weight: 120
---


# Troubleshooting Guide

This document provides solutions to common problems encountered with serverless workflows.

## Table of Contents

1. [HTTP Errors](#http-errors)
2. [Workflow Errors](#workflow-errors)
3. [Configuration Problems](#configuration-problems)
4. [Performance Issues](#performance-issues)
5. [Error Messages](#error-messages)
6. [Network Problems](#network-problems)
7. [Common Scenarios](#common-scenarios)
8. [Contact Support](#contact-support)
9. [Air Gapped Problems](#air-gapped-problems)

---

## HTTP Errors
Many workflow operations are REST requests to REST endpoints. If an HTTP error occurs then the workflow will fail and the HTTP code and message will be displayed. Here is an [example](https://github.com/rhdhorchestrator/rhdhorchestrator.github.io/blob/main/content/main/docs/serverless-workflows/409-error.png?raw=true) of the error in the UI.
Please use [HTTP codes documentation](https://developer.mozilla.org/docs/Web/HTTP/Status) for understanding the meaning of such errors.
Here are some examples:
 - `409`. Usually indicates that we are trying to update or create a resource that already exists. E.g. K8S/OCP resources.
 - `401`. Unauthorized access. A token, password or username might be wrong or expired.

 ## Workflow Errors

### Problem: Workflow execution fails

**Solution:**

1. Examine the container log of the workflow
    ```console
        oc logs my-workflow-xy73lj
    ```

### Problem: Workflow is not listed by the orchestrator plugin

**Solution:**

1. Examine the container status and logs
    ```console
        oc get pods my-workflow-xy73lj
        oc logs my-workflow-xy73lj
    ```

2. Most probably the Data index service was unready when the workflow started.
   Typically this is what the log shows:
    ```console
        2024-07-24 21:10:20,837 ERROR [org.kie.kog.eve.pro.ReactiveMessagingEventPublisher] (main) Error while creating event to topic kogito-processdefinitions-events for event ProcessDefinitionDataEvent {specVersion=1.0, id='586e5273-33b9-4e90-8df6-76b972575b57', source=http://mtaanalysis.default/MTAAnalysis, type='ProcessDefinitionEvent', time=2024-07-24T21:10:20.658694165Z, subject='null', dataContentType='application/json', dataSchema=null, data=org.kie.kogito.event.process.ProcessDefinitionEventBody@7de147e9, kogitoProcessInstanceId='null', kogitoRootProcessInstanceId='null', kogitoProcessId='MTAAnalysis', kogitoRootProcessId='null', kogitoAddons='null', kogitoIdentity='null', extensionAttributes={kogitoprocid=MTAAnalysis}}: java.util.concurrent.CompletionException: io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: sonataflow-platform-data-index-service.default/10.96.15.153:80
    ```

3. Check if you use a cluster-wide platform:
    ```console
       $ oc get sonataflowclusterplatforms.sonataflow.org
       cluster-platform
    ```
    If you have, like in the example output, then use the namespace `sonataflow-infra` when you look for the sonataflow services

    Make sure the Data Index is ready, and restart the workflow - notice the `sonataflow-infra` namespace usage:
    ```console
        $ oc get pods -l sonataflow.org/service=sonataflow-platform-data-index-service -n sonataflow-infra
        NAME                                                      READY   STATUS    RESTARTS   AGE
        sonataflow-platform-data-index-service-546f59f89f-b7548   1/1     Running   0          11kh

        $ oc rollout restart deployment my-workflow
    ```

### Problem: Workflow is failing to reach an HTTPS endpoint because it can't verify it

- REST actions performed by the workflow can fail the SSL certificate check if the target endpoint is signed with
a CA which is not available to the workflow. The error in the workflow pod log usually looks like this:

    ```console
        sun.security.provider.certpath.SunCertPathBuilderException - unable to find valid certification path to requested target
    ```

**Solution:**

1. If this happens then we need to load the additional CA cert into the running
   workflow container. To do so, please follow this guile from the SonataFlow guides site:
   https://sonataflow.org/serverlessworkflow/main/cloud/operator/add-custom-ca-to-a-workflow-pod.html


## Configuration Problems

### Problem: Workflow installed in a different namespace than Sonataflow services fails to start

**Solution:**
When deploying a workflow in a namespace other than the one where Sonataflow services are running (e.g., `sonataflow-infra`), there are essential steps to follow to enable persistence and connectivity for the workflow. See the following [steps](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/release-1.3/README.md#additional-workflow-namespaces).

### Problem: sonataflow-platform-data-index-service pods can't connect to the database on startup

1. **Ensure PostgreSQL Pod has Fully Started**\
If the PostgreSQL pod is still initializing, allow additional time for it to become
fully operational before expecting the `DataIndex` and `JobService` pods to connect.
1. **Verify network policies if PostgreSQL Server is in a different namespace**\
If PostgreSQL Server is deployed in a separate namespace from Sonataflow services
(e.g., not in `sonataflow-infra` namespace), ensure that network policies in the
PostgreSQL namespace allow ingress from the Sonataflow services namespace
(e.g., `sonataflow-infra`). Without appropriate ingress rules,
network policies may prevent the `DataIndex` and `JobService` pods from
connecting to the database.


## Air Gapped Problems

### Problem: Cluster is unable to pull images due to air gapped environment

**Solution:**
Create an `ImageDigestMirrorSet` manifest that references the external images to a local registry inside the air gapped environment following the instructions described [here](https://docs.openshift.com/container-platform/4.17/rest_api/config_apis/imagedigestmirrorset-config-openshift-io-v1.html). This manifest configures the cluster to use mirror locations using digest pull specifications, which will enable it to pull the images from the mirrored location accessible to the cluster.

The following manifest is an example that references a local mirror located at `registry.localhost.net`.

```yaml
kind: ImageDigestMirrorSet
apiVersion: config.openshift.io/v1
metadata:
  name: rhdhorchestrator-mirror
spec:
  imageDigestMirrors:
    # Orchestrator
    - source: registry.redhat.io/rhdh-orchestrator-dev-preview-beta/controller-rhel9-operator
      mirrors:
        - registry.localhost.net/rhdh-orchestrator-dev-preview-beta/controller-rhel9-operator
    - source: registry.redhat.io/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle
      mirrors:
        - registry.localhost.net/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle
    # Chart deployment
    - source: registry.redhat.io/openshift4/ose-cli
      mirrors:
        - registry.localhost.net/ose-cli
    - source: registry.access.redhat.com/ubi9-minimal
      mirrors:
        - registry.localhost.net/ubi9-minimal
    # Serverless workflows
    - source: registry.redhat.io/openshift-serverless-1/logic-rhel8-operator
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-rhel8-operator
    - source: registry.redhat.io/openshift-serverless-1/logic-jobs-service-postgresql-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-jobs-service-postgresql-rhel8
    - source: registry.redhat.io/openshift-serverless-1/logic-jobs-service-ephemeral-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-jobs-service-ephemeral-rhel8
    - source: registry.redhat.io/openshift-serverless-1/logic-data-index-postgresql-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-data-index-postgresql-rhel8
    - source: registry.redhat.io/openshift-serverless-1/logic-data-index-ephemeral-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-data-index-ephemeral-rhel8
    - source: registry.redhat.io/openshift-serverless-1/logic-swf-builder-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-swf-builder-rhel8
    - source: registry.redhat.io/openshift-serverless-1/logic-swf-devmode-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/logic-swf-devmode-rhel8
    # RHDH
    - source: registry.redhat.io/rhdh/rhdh-rhel9-operator
      mirrors:
        - registry.localhost.net/rhdh/rhdh-rhel9-operator
    - source: registry.redhat.io/rhdh/rhdh-operator-bundle
      mirrors:
        - registry.localhost.net/rhdh/rhdh-operator-bundle
    - source: registry.redhat.io/rhdh/rhdh-hub-rhel9
      mirrors:
        - registry.localhost.net/rhdh/rhdh-hub-rhel9
    # Knative Serving
    - source: registry.redhat.io/openshift-serverless-1/kn-serving-activator-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-serving-activator-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-serving-autoscaler-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-hpa-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-serving-autoscaler-hpa-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-serving-controller-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-serving-controller-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-serving-webhook-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-serving-webhook-rhel8
    # Knative Serving Ingress
    - source: registry.redhat.io/openshift-serverless-1/kourier-control-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kourier-control-rhel8
    - source: registry.redhat.io/openshift-service-mesh/proxyv2-rhel8
      mirrors:
        - registry.localhost.net/openshift-service-mesh/proxyv2-rhel8
    # Knative Eventing
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-controller-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-controller-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-apiserver-receive-adapter-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-apiserver-receive-adapter-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-webhook-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-webhook-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-channel-controller-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-channel-controller-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-channel-dispatcher-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-channel-dispatcher-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-jobsink-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-jobsink-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-mtchannel-broker-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-mtchannel-broker-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-filter-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-filter-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-ingress-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-ingress-rhel8
    - source: registry.redhat.io/openshift-serverless-1/kn-eventing-mtping-rhel8
      mirrors:
        - registry.localhost.net/openshift-serverless-1/kn-eventing-mtping-rhel8
```