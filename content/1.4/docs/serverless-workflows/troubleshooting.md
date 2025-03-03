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
When deploying a workflow in a namespace other than the one where Sonataflow services are running (e.g., `sonataflow-infra`), there are essential steps to follow to enable persistence and connectivity for the workflow. See the following [steps](https://github.com/rhdhorchestrator/orchestrator-helm-operator/blob/main/docs/main/README.md#additional-workflow-namespaces).

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
